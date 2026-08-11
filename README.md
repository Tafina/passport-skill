# passport-skill

An agent skill for [Passport](https://passport.oway.pw), a hosted OAuth 2.1 /
OpenID Connect auth service. Install it and your coding agent can add
"Log in with Passport" to a web app end to end: register the OAuth app from
the CLI, wire the login, callback and logout routes, and set up social login,
webhooks or branding, without you opening a dashboard.

## Install

```sh
npx skills add Tafina/passport-skill
```

Global install for every project:

```sh
npx skills add Tafina/passport-skill -g -y
```

Or use it once without installing:

```sh
npx skills use Tafina/passport-skill@passport-auth
```

## What it covers

- Creating a Passport app and capturing `client_id` / `client_secret` with the
  `@tafina/passport` CLI, non-interactively
- Login, callback and logout via the `@tafina/passport-node` SDK on Node, Bun
  and edge runtimes
- The raw OAuth flow for every other stack, plus generic OIDC discovery
- Social login connections (Google, GitHub, Discord, any OIDC provider)
- Webhooks, including posting straight to a Discord channel
- User management: search, force sign-out, block and unblock
- Hosted-page branding and custom auth domains
- The common failure modes: redirect URI mismatches, PKCE, expired codes,
  state mismatches

The skill defers to <https://passport.oway.pw/llms.txt> for exact endpoints
and per-language reference implementations, so it stays current as the
service changes.

## Not passport.js

This is the hosted Passport service at passport.oway.pw. It has nothing to do
with `passport` / `passport.js`, the Express strategy middleware.

## Links

- Service and dashboard: <https://passport.oway.pw>
- Integration guide for agents: <https://passport.oway.pw/llms.txt>
- CLI: [`@tafina/passport`](https://www.npmjs.com/package/@tafina/passport)
- Server SDK: [`@tafina/passport-node`](https://www.npmjs.com/package/@tafina/passport-node)

MIT licensed.
