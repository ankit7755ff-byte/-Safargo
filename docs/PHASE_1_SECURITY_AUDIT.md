# SafarGo Phase 1 — Foundation & Security Audit

Status: IN PROGRESS  
Target repository: `ankit7755ff-byte/-Safargo`  
Reference implementation reviewed: `sonamankit56-byte/safargo`  
Working branch: `phase-1-foundation-security`

## Scope and safety

This is an audit record, not a claim that the reviewed source has already been merged or deployed. Production `main` remains unchanged. The older unrelated Vercel project must not be modified. Do not copy secrets or production personal data between repositories.

## Confirmed findings from reviewed source files

### Critical — authentication and personal data

- `src/lib/supabase.ts` reads and writes `password` on `safargo_agents`; it also fetches PAN, Aadhaar, bank account and IFSC fields in the general agent query.
- The `AgentUser` type includes an optional `password` field.
- A browser client must never store or compare plaintext passwords. Use Supabase Auth (or another established identity provider); remove password fields from application profile records and never return KYC/bank fields to ordinary agent clients.
- Admin permissions must be enforced by trusted server-side authorization and database policies, not by a client-side email check or a mutable `role` field.

### High — API keys and server endpoints

- `src/lib/emailService.ts` contains a direct browser-side Resend fallback using `VITE_RESEND_API_KEY`. Vite-prefixed variables are client-exposed; Resend secrets must only exist in server runtime environment variables.
- `src/lib/smsService.ts` reads a Fast2SMS key from localStorage and sends it in the request body. The Vite middleware accepts a request-supplied API key. Remove this design; the provider key must remain server-side.
- The Resend, Paybeez and Fast2SMS handlers in `vite.config.ts` are Vite development-server middleware. They are not automatically deployed as production API routes on Vercel.
- Recharge middleware currently allows wildcard CORS and does not show authentication, rate limiting, idempotency or server-side payment verification.
- Email fallback logic forwards messages to a fixed owner inbox on certain provider errors. KYC and personal data must not be silently forwarded to a different recipient.

### High — OTP and session safety

- `src/lib/smsService.ts` generates OTP with `Math.random()`; use a cryptographically secure generator on the server.
- OTP generation, storage, expiry, attempt limits, replay prevention and verification must be server-side. Never log OTP values or report simulated delivery as successful.
- Browser localStorage is not an appropriate place for gateway secrets or privileged configuration.

### High — booking and inventory integrity

- `src/data/busData.ts` generates illustrative bus/seat data; `src/data/hotelData.ts` contains mock hotel inventory. Clearly label demo data and prevent it from being sold as live availability.
- Default fallback values in `src/lib/supabase.ts` can make missing inventory fields look real (e.g. default seat counts and tracking). Remove misleading defaults.
- A booking must remain pending until an authorized operator/admin verifies actual availability and payment. Only then issue the final ticket/PNR and commission.

### Medium — project and deployment

- The reviewed source has a React/Vite/TypeScript structure, but its build has not yet been run in this audit.
- The destination repository currently has a separate lightweight React/Vite MVP. Do not overwrite its entry point or package manifest until the full source tree, dependencies, assets and deployment configuration are transferred and validated together.
- Capacitor dependency versions in the source manifest are inconsistent (`@capacitor/cli` v7 and `@capacitor/core` v8); align versions before Android packaging.
- Review external PostgreSQL/WebSocket dependencies and remove server-only packages from browser bundles where not needed.

## Phase 1 implementation gates

- [ ] Import the complete reviewed source tree into this working branch without secrets or local `.env` files.
- [ ] Reconcile dependencies and run a clean production build.
- [ ] Replace plaintext agent password handling with Supabase Auth.
- [ ] Remove sensitive KYC/bank fields from general client reads; establish least-privilege RLS policies.
- [ ] Enforce admin/operator/agent roles server-side.
- [ ] Move Resend/SMS/payment provider calls to authenticated Vercel server functions.
- [ ] Add OTP rate limits, expiry, attempt limits and secure generation.
- [ ] Mark demo inventory explicitly and block demo data from checkout.
- [ ] Add safe environment variable template and deployment documentation.
- [ ] Test login, access boundaries, booking status transitions and production build.
- [ ] Open a review PR; do not merge to production until checks pass.

## Environment variable policy

Only public, intentionally browser-safe values may use the `VITE_` prefix (for example, a Supabase project URL and publishable/anon key protected by correct RLS). Provider secrets, service-role keys, database passwords, Resend keys, SMS keys and payment credentials must be server-only environment variables and must never be committed.

## Not yet verified

No complete repository tree import, npm build, database/RLS inspection, live provider verification, Android build, or end-to-end booking/payment test has been completed yet. This report must be updated as each gate is actually tested.
