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
3. Deploy. The MCP endpoint is `https://<your-domain>/mcp` (also `/api/mcp`).

## Candidate instructions (for the job posting)

> We do not accept resumes. Add this connector to your AI agent
> (Claude: Settings → Connectors → Add custom connector) and apply through it:
>
>     https://<your-domain>/mcp

## Security posture

- No credentials for anything except Resend; a compromise exposes nothing but the mail key
- Strict validation: LinkedIn/GitHub URLs only from their real domains, size caps, control chars stripped
- Applicant text is labeled UNTRUSTED in the email so downstream AI treats it as data, not instructions
- Best-effort rate limiting (5/hr per IP, 60/hr global per instance) and 24h dedupe by email
- Body size capped at 20KB

## Local test

    node test-local.js

Runs the full flow against a mock Resend server: protocol handshake, both tools,
validation failures, dedupe, sanitization, rate limit.
