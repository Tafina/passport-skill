---
name: passport-auth
description: Add "Log in with Passport" to any web app using the Passport hosted auth service (passport.oway.pw). Use when asked to add login, sign-in, sign-up, authentication, OAuth or SSO with Passport, to register or manage a Passport OAuth app, or to add social login (Google/GitHub/Discord/OIDC), passkeys, two-factor auth, auth webhooks, user management or a custom auth domain. This is the hosted Passport service, not the passport.js Express middleware.
version: 0.1.0
---

# Passport hosted auth

Passport (https://passport.oway.pw) is a hosted OAuth 2.1 / OpenID Connect
provider. The app sends the user to Passport, they sign in or sign up there,
and they come back with an authorization code the app's server exchanges for
their identity. Nothing to store: no password hashes, no session tables, no
sign-in UI to build. Users get email + password, magic links, emailed codes,
passkeys, optional TOTP two-factor and per-app social login without the app
implementing any of it.

This is **not** `passport` / `passport.js`, the Express strategy middleware.
If the user wants `passport.authenticate("local")` inside their own app,
this skill does not apply.

## Read the live guide before writing code

`https://passport.oway.pw/llms.txt` is the complete, always-current
integration guide: exact endpoints, reference implementations for Express,
Next.js, Flask, PHP and generic OIDC libraries, id_token verification,
webhook payloads and signature checks, and an error/cause/fix table. Fetch
it whenever this skill does not answer the question outright, and prefer it
over guessing when protocol details matter.

## 1. Get credentials

The integration needs two values: `PASSPORT_CLIENT_ID` and
`PASSPORT_CLIENT_SECRET`. Create them with the CLI instead of sending the
user to the dashboard.

```sh
npx @tafina/passport@latest login          # opens the browser once, user approves
npx @tafina/passport@latest --json apps create \
  --name "My app" \
  --redirect-uri https://myapp.com/auth/callback
# => { "client_id": "...", "client_secret": "...", ... }
```

- `login` uses the OAuth device flow and stores the session in
  `~/.passport/config.json`, so every later command is non-interactive.
  Headless: `login --no-open` prints a URL to approve on another device, or
  set `PASSPORT_TOKEN` to an existing session token.
- The redirect URI must match the app's callback URL exactly, character for
  character. Add the local one too:
  `apps update <client_id> --add-redirect-uri http://localhost:3000/auth/callback`.
- `client_secret` is printed only by `apps create` and `apps rotate-secret`.
  Capture it right there, write it to `.env`, and never commit it or expose
  it to the browser.
- Put `--json` before the subcommand, not after.
- If the user already has credentials from
  https://passport.oway.pw/dashboard, skip this step.

## 2. Wire up login

### Node 20+, Bun or an edge runtime: use the SDK

```sh
npm install @tafina/passport-node
```

```ts
import { Passport } from "@tafina/passport-node";

export const passport = new Passport({
  clientId: process.env.PASSPORT_CLIENT_ID!,
  clientSecret: process.env.PASSPORT_CLIENT_SECRET!,
  baseUrl: process.env.APP_URL!, // the app's own public origin
});
```

Mount the handler on `/auth/*`. It takes a standard `Request` and returns a
standard `Response`:

```ts
// Hono
app.all("/auth/*", (c) => passport.handler(c.req.raw));

// Next.js App Router, app/auth/[...path]/route.ts
export const GET = passport.handler;
export const POST = passport.handler;

// Bun
Bun.serve({ fetch: (req) => passport.handler(req) });
```

That gives four routes for free:

| route | does |
|---|---|
| `/auth/login` | starts login, optional `?redirect=/after` |
| `/auth/callback` | code exchange, id_token verification, session cookie |
| `/auth/logout` | signs out, optional `?redirect=/` |
| `/auth/user` | the session user as JSON, for client-side fetch |

Read the user anywhere on the server:

```ts
const user = await passport.session.getUser(request);
// { sub, email, email_verified, name } | null
```

The registered redirect URI is `APP_URL + "/auth/callback"`.

### Any other stack: the raw flow

1. `/auth/login`: generate 16+ random bytes of hex as `state`, store it in an
   httpOnly SameSite=Lax cookie, redirect the browser to
   `https://passport.oway.pw/api/auth/oauth2/authorize?client_id=...&redirect_uri=<encoded callback>&response_type=code&scope=openid+profile+email&state=...`
2. `/auth/callback`: handle an `error` query param, compare `state` against
   the cookie and clear it, then `POST` to
   `https://passport.oway.pw/api/auth/oauth2/token` with
   `grant_type=authorization_code&code=...&redirect_uri=<same callback>&client_id=...&client_secret=...`
   as form-encoded body.
3. `GET https://passport.oway.pw/api/auth/oauth2/userinfo` with
   `Authorization: Bearer <access_token>` returns
   `{ sub, name, given_name, family_name, email, email_verified }`.
4. Create the app's own session from that. Key users by `sub`, which is
   stable and unique. Never key by email alone.
5. `/auth/logout` clears the app's session cookie. No call to Passport needed.

Dashboard-created clients are confidential and do **not** require PKCE: send
`client_id` + `client_secret` in the token request body and omit
`code_challenge`. Access tokens are opaque (not JWTs) and live 10 minutes;
authorization codes are single use and expire in 10 minutes.

Copy a working implementation for the target language from section 5 of
`https://passport.oway.pw/llms.txt` rather than writing it from scratch.
Generic OIDC libraries work too: point them at
`https://passport.oway.pw/api/auth/.well-known/openid-configuration`.

## 3. Environment variables

```sh
PASSPORT_CLIENT_ID=...
PASSPORT_CLIENT_SECRET=...       # server only, never in a client bundle
APP_URL=https://myapp.com        # the app's own public origin
```

The SDK defaults to the hosted service, so `PASSPORT_ORIGIN` is only needed
when hand-rolling the flow or pointing at a self-hosted instance.

## 4. Finish the job

- Add a visible "Log in" link pointing at `/auth/login`.
- Show the signed-in user's name or email plus a logout link.
- Run the flow end to end in a browser and confirm the callback lands back
  on the app with a session, not on an error page.

## Beyond login

All of this is CLI-driven, so it can be done without the user opening the
dashboard. Run `npx @tafina/passport@latest <group> --help` for full flags.

| want | command |
|---|---|
| match the app's look on hosted pages | `apps branding <client_id> --logo-uri ... --accent-color "#7c3aed" --sign-in-title "Welcome back"` |
| add Google / GitHub / Discord / OIDC buttons | `connections create <client_id> --provider google --client-id ... --client-secret ...` |
| get notified on sign-in, block, revoke | `webhooks create <client_id> --url https://myapp.com/webhooks/passport` |
| post events to Discord with no endpoint | `webhooks create <client_id> --url <discord webhook url> --discord` |
| list, search, force sign-out or block users | `apps users <client_id> --q ada@`, `apps revoke-user`, `apps block-user` |
| serve sign-in from auth.myapp.com | dashboard → app → Domains, then publish the DNS records shown |

Free accounts get 2 apps, 2 social connections per app and 1 webhook per
app. Pro lifts those and unlocks generic-OIDC connections and custom
domains. Sign-in methods are never gated.

## Troubleshooting

| symptom | fix |
|---|---|
| 400 at authorize, no redirect back | the callback URL is not registered, exactly. Add it with `apps update <client_id> --add-redirect-uri <url>` |
| `invalid_request`, "pkce is required" | the client requires PKCE. Drop `code_challenge` for dashboard/CLI-created clients, or send the verifier consistently |
| `401 invalid_client` at the token endpoint | wrong `client_secret`. Re-check the env var, or `apps rotate-secret` |
| `invalid_grant`, "invalid code" | the code was reused or expired. Restart the flow |
| state mismatch in the callback | the state cookie was blocked or login started twice. Set it httpOnly with SameSite=Lax |

The passkey UI is hidden on custom domains by design: WebAuthn credentials
are bound to the canonical host, so passkeys only work on
`passport.oway.pw`. Every other method works on custom domains.
