---
name: no-auth-flash
description: Use when a login-gated SPA or SSR app flashes the wrong UI for signed-out or unknown auth states — an app-shell skeleton shown to visitors who will be bounced to login, or a skeleton blocking signed-in users while an auth probe resolves. Covers edge/server session gating, login redirect-back (returnTo), optimistic auth-hint cookies, and how to test the loading window.
---

# Never flash the wrong shell

A login-gated SPA has three auth states: unknown, signed out, signed in. The classic bug is rendering the *signed-in* placeholder (app-shell skeleton: sidebar, nav, content cards) during *unknown* — because the session cookie is HttpOnly and the client can't know who it is until a `/me` probe resolves. Signed-out visitors get shown an app they will never reach, then a jarring swap to login. Signed-in users stare at a skeleton for a full round trip on every load.

The rule: **render the placeholder for the state you have verified, not the state you hope for.** Getting there is three layers, in order of preference. Each layer makes the next one's job smaller.

## Layer 1 — verify the session at the edge, before the document is served

Most session schemes can be verified server-side with **no identity-provider round trip**: sealed/signed session cookies verify with local crypto; JWT access tokens verify against a cached JWKS. So gate document requests in the worker/middleware that serves the SPA's HTML:

- Signed out (no cookie, or cookie fails verification) → `302 /login?returnTo=<path>` before any app HTML exists. The wrong shell can't flash if it is never served.
- Signed in → serve the app. The client now knows something it couldn't before: anyone who receives the SPA at a gated path **is** authenticated.

Scope and edge cases that bite:

- **Gate document navigations only** (GET/HEAD with `sec-fetch-dest: document`, falling back to `Accept: text/html`). API routes, module requests, and health probes answer for themselves.
- **Clear an invalid cookie, don't just redirect.** A cookie that fails verification is worse than none — anything keyed on cookie *presence* (marketing-page routing, the hint below) keeps misbehaving until it's gone. Expire it on the redirect response.
- **Persist rotated sessions.** If verification triggered a token refresh, the new session MUST reach the browser on the response — refresh tokens are typically single-use, and dropping the rotated cookie silently logs the user out at the next expiry. This applies on the serve path too, not just redirects.
- Failures inside verification collapse to "signed out", never to a 500 — the login flow is where the real problem surfaces.

## Layer 2 — an auth-hint cookie for instant signed-in paints

The session cookie is HttpOnly (keep it that way). Give it a non-HttpOnly companion — a display-only snapshot of the identity (name, email, avatar, org) — so the *next* page load can paint the real shell immediately instead of blocking on the `/me` probe:

- The client **writes** it whenever the server confirms identity (every resolved `/me`), and **reads** it on the next load to seed an optimistic "authenticated" state while the probe is in flight.
- It is a **hint, never an authority**. Every API call is still authenticated by the real session server-side. A stale or forged hint can only change which placeholder briefly renders. Keep the payload to display data the user already knows about themselves; schema-validate on read and treat anything malformed as absent.
- The resolved probe **always wins** — reconcile the moment it lands.
- **Clear it everywhere the session dies**: logout response, the edge gate's invalid-cookie path, and client-side when the probe says signed out. A hint that outlives the session paints a signed-in shell for a signed-out browser.
- In an SSR/hydrating app, read the cookie **after mount** (effect/state), not during the first render — the first client render must match the server HTML. The flip costs one frame, not a round trip.

With layers 1+2 in place, the client-side "loading" state is only reachable by verified users with no hint (first visit on a new browser) — so a skeleton there is finally correct.

## Layer 3 — the client gate becomes a safety net

The in-app auth gate no longer handles fresh visits; it handles **mid-session death** (logout in another tab, expiry while the SPA is open):

- `unauthenticated` → redirect to `/login?returnTo=<current>` via a full navigation, rendering a blank screen for the moment that takes — never a skeleton for an app they're out of.
- Unknown routes should render a real 404 page, not the shell or skeleton. If there's no `notFound` route, the gate is probably the thing accidentally rendering a skeleton for garbage URLs — check.

## returnTo: carry it in the OAuth state parameter, validate at every entry

Deep links must survive login. Resist adding a second cookie for it — if the login flow is OAuth-shaped, the `state` parameter already round-trips through the provider verbatim and is already authenticated by the CSRF check (state pinned in a cookie, compared timing-safe at the callback). Ride along: `state = base64url(JSON { nonce, returnTo })`. One value, one cookie, and an attacker can't swap the destination without breaking the comparison.

Wherever a returnTo enters (gate query, login page, callback's decoded state), validate it as a **same-origin relative path**: starts with `/`, not `//` (protocol-relative is an absolute URL in disguise), and not an API path. Anything else falls back to `/`. Decoding the state envelope must be total — providers send callbacks with state you never minted; junk reads as "no returnTo", never a throw.

## Testing the loading window

The bug lives in a timing window, so the test must hold the window open:

- **Intercept the auth probe** and delay it (e.g. Playwright `page.route` on `/me` with a sleep). Assert what is painted *during* the delay: for a signed-out visitor, the login page with zero skeleton elements; for a signed-in user with a hint, the real nav — plus a flag proving the probe had not resolved when it painted.
- **Drive the full login round trip** over the wire: gated deep link → login redirect carrying returnTo → provider → callback → lands on the deep link. Then the forged version: an off-origin returnTo completes login but lands on `/`.
- **Assert cookie hygiene as Set-Cookie headers**: invalid session → cleared (Max-Age=0) alongside the hint; logout → both gone.
- **Drive the refresh path without waiting out expiry**: if the session is a sealed cookie, unseal it with the same library the server uses, corrupt the access token's signature, reseal. The gate sees an invalid-but-refreshable session: assert the page is served, the rotated session is set on the response, and the spent one is refused on replay.
