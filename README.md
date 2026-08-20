# sfoapply — MCP direct-apply server

Application server for the COO/CFO search (single-family office + pre-seed venture fund).
Candidates do not send resumes. They connect their AI agent to this MCP server and apply
through it. The connection itself is the first screen.

## What it does

One serverless function (`api/mcp.js`, zero dependencies) speaking MCP over HTTP with two tools:

- `get_position_description` — returns the blind PD, including how to apply
- `apply` — validates the submission and emails it via Resend

Write-only by design: there is no tool that reads applications back, so candidates can
never see each other's data.

## Deploy (Vercel)

1. Import this repo into Vercel as a new project. No framework, no build step.
2. In Project Settings → Environment Variables, add:
   - `RESEND_API_KEY` (required) — from resend.com
   - `APPLY_TO` (optional) — defaults to dchie@paloaltostaffing.com
   - `APPLY_FROM` (optional) — defaults to "Maple Drive Apply <onboarding@resend.dev>".
     Note: the resend.dev sender only delivers to the email that owns the Resend account.
     For delivery to any address, verify a domain in Resend and set APPLY_FROM to it.
3. Deploy. The live MCP endpoint is:

       https://applyops.mapledrive.com/mcp

   `/api/mcp` resolves to the same handler, so a candidate who guesses that path still gets in.

   Use the custom domain, not a `*.vercel.app` alias. The project has Vercel
   authentication enabled for everything except custom domains, so
   `sfoapply-mdpt.vercel.app` and `sfoapply-git-main-mdpt.vercel.app` both return
   401 to candidates. `sfoapply.vercel.app` happens to answer today, but only the
   custom domain is exempt by configuration rather than by accident.

## Candidate instructions (for the job posting)

> To register interest, connect your agent to this endpoint and apply
> through it:
>
>     https://applyops.mapledrive.com/mcp
>
> Grok, Claude, Codex, Gemini, Cursor, or anything else that speaks MCP.
> There's no resume and no form. The server tells your agent what we need,
> including two short answers you'll write yourself.
>
> Where to add it: Claude → Settings → Connectors → Add custom connector.
> Grok, Codex, Gemini, and Cursor each support custom MCP servers in their
> own settings; check your agent's MCP or connector documentation for the
> exact path. No authentication or API key is needed, so the URL is the
> only thing you have to enter.
>
> On the Claude Code CLI:
>
>     claude mcp add --transport http mapledrive https://applyops.mapledrive.com/mcp
>
> The `--transport http` flag matters. Without it the CLI defaults to stdio
> and tries to run the URL as a local command, which fails in a way that
> looks like the server is down.

## Security posture

- No credentials for anything except Resend; a compromise exposes nothing but the mail key
- Strict validation: LinkedIn/GitHub URLs only from their real domains, size caps, control chars stripped
- Applicant text is labeled UNTRUSTED in the email so downstream AI treats it as data, not instructions
- Best-effort rate limiting (30/hr per IP, 120/hr per instance, tunable via `RATE_PER_IP` / `RATE_GLOBAL`) and 24h dedupe by email. Only real send attempts are charged; validation failures and duplicates are free. The per-IP cap is deliberately generous because hosted agents (claude.ai, Grok) call from shared datacenter egress IPs — one IP can be many candidates
- Every request is logged as a one-line JSON event with the exact response the agent saw: `connect` (which agent, from `clientInfo`), `pd_read`, `page_view`, `unknown_tool`, and `apply` with outcome `sent` / `rejected` (plus which rules failed) / `duplicate` / `rate_limited` / `delivery_failed` / `network_error`. Vercel keeps runtime logs for about an hour, so a log drain (Axiom) provides the retention — the funnel lives there
- Body size capped at 20KB

## Local test

    node test-local.js

Runs the full flow against a mock Resend server: protocol handshake, both tools,
validation failures, dedupe, sanitization, rate limit.
