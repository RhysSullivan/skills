# skills

Agent skills by Rhys Sullivan.

## Install

```bash
npx skills add RhysSullivan/skills
```

Or install a single skill:

```bash
npx skills add RhysSullivan/skills --skill quality-code
```

## Skills

- **[no-auth-flash](skills/no-auth-flash/SKILL.md)** — stop login-gated apps flashing the wrong UI for unknown auth states: verify sessions at the edge, seed signed-in paints from an auth-hint cookie, carry returnTo in the OAuth state parameter, and test the loading window by holding the auth probe open.
- **[quality-code](skills/quality-code/SKILL.md)** — principles for writing quality full-stack TypeScript: branded types, discriminated unions, end-to-end types, real tests over mocks, OpenTelemetry, picking the right abstractions.
- **[write-better-error-messages](skills/write-better-error-messages/SKILL.md)** — review and rewrite product error messages so users understand what happened, what was preserved, and what to do next. Adapted from Jenni Nadler's Wix UX article "When life gives you lemons, write better error messages."
