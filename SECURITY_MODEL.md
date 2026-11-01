# Threat model: berry-13/librechat.ai

_Built from the code (commit 5d05fb5, `main`) and Marco's answers. Items marked **[assumed]** are my reading of the code and have not been confirmed by the owner._

## 1. System overview

librechat.ai is LibreChat's public marketing and documentation site: Next.js (App Router, `next build --webpack`), with MDX content under `content/` rendered by fumadocs. **[assumed]** It is hosted on Vercel with Cloudflare in front as CDN/cache (`.github/workflows/cache-purge.yml` purges Cloudflare after Vercel deploys). There are no user accounts and no login. Everything is either static content or a small set of anonymous serverless endpoints that call third-party services with server-held secrets.

## 2. Components

| Component            | Covers                                                                                                                                                                                                                                  | Notes                                                                                                                                          |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Web**              | `app/` pages and layouts, `components/`, `content/`, `proxy.ts` (i18n and markdown-negotiation middleware), `lib/` rendering helpers, `app/llms*.txt`, `app/llms.mdx`, `app/openapi.json`, `app/auth.md`, `app/.well-known/api-catalog` | Static and SSR docs. Client-side widgets (Crisp, Reo, CWV analytics) load after consent.                                                       |
| **Docs Chat**        | `app/api/chat/route.ts`, `components/ai/` (chat panel UI), `lib/ratelimit.ts` (Upstash), `lib/openrouter-attribution.ts`                                                                                                                | Anonymous LLM chat over OpenRouter with `search` and `navigate` tools over local docs.                                                         |
| **Newsletter**       | `app/api/subscribe`, `app/api/unsubscribe`, `lib/supabase.ts`, `lib/rate-limit.ts`                                                                                                                                                      | Supabase `subscribers` table accessed with the **service-role key**.                                                                           |
| **Feedback**         | `app/actions/feedback.ts`                                                                                                                                                                                                               | Server action that creates GitHub Discussions (`GITHUB_FEEDBACK_TOKEN`) and posts Discord/Slack webhooks.                                      |
| **MCP & Agent APIs** | `app/mcp/route.ts`, `lib/mcp-server.ts`, `lib/mcp-documents.ts`, `lib/webmcp.ts`, `lib/web-bot-auth.ts`, `app/.well-known/http-message-signatures-directory`, `app/api/search`, `app/api/og`, `app/api/github-stats`                    | Public read-only MCP server over the docs. Web Bot Auth signs outbound agent requests with an Ed25519 private JWK. OG image rendering. Search. |
| **CI/CD**            | `.github/workflows/*`, `scripts/`                                                                                                                                                                                                       | Translation (OpenRouter), embeddings (Supabase/OpenAI), Cloudflare purge, screenshots (demo credentials), starters.                            |

## 3. Trust boundaries and entry points

- **Internet → Vercel functions:** every route above is anonymous and reachable by anyone. Inputs include JSON bodies, query strings (OG `title`), and headers (`x-chat-mode`, `x-chat-page`, `Accept`, `Accept-Language`, the locale cookie, `Origin` and `mcp-protocol-version` on `/mcp`, `x-forwarded-for` for rate limiting).
- **Server → third parties:** OpenRouter (cost-bearing), Supabase (service role bypasses RLS), GitHub GraphQL (write token), Discord/Slack webhooks, Upstash, GitHub REST.
- **LLM boundary:** user messages and docs content reach the model. Model output (markdown, `navigate` URLs) is rendered in the browser.
- **Content boundary:** MDX in `content/`, including machine-generated translations from `translate_docs.yml` (an LLM writes files that are later compiled as MDX/JSX).
- **GitHub → CI:** `ci.yml` runs on `pull_request` (fork PRs get no secrets). The other workflows run on schedule, push, or dispatch with repository secrets. No `pull_request_target`, `workflow_run` or `issue_comment` triggers were found.
- **Cloudflare cache:** responses that vary by `Accept` or the locale cookie must not be cache-poisoned across users.

## 4. Assets

Marco named three assets as most critical: **server/CI secrets**, **site integrity**, and **spend & availability**. Subscriber emails were not named critical.

1. **Server secrets:** `SUPABASE_SERVICE_ROLE_KEY`, `OPENROUTER_API_KEY`, `WEB_BOT_AUTH_PRIVATE_JWK` (lets anyone sign as LibreChat's agent), `GITHUB_FEEDBACK_TOKEN` (discussions:write), webhook URLs, `LANGFUSE_SECRET_KEY`, `UPSTASH_REDIS_REST_TOKEN`, `MONGODB_URI`. CI-only: `CLOUDFLARE_API_TOKEN`, `OPENAI_API_KEY`, `DEMO_EMAIL`/`DEMO_PASSWORD`.
2. **Subscriber emails** (PII) in Supabase. In scope, but not named as critical.
3. **Site integrity:** visitors trust librechat.ai docs (install commands, config snippets). XSS, defacement, a poisoned cache, or a malicious translated page would push bad instructions to self-hosters.
4. **Spend and availability:** OpenRouter and translation budget, Vercel function time.

## 5. Attackers

Marco: **all of the attackers below matter.**

- **Anonymous internet user** (primary): unlimited requests to the public endpoints, with control over headers and bodies.
- **Malicious contributor:** opens PRs with content or code. Can reach CI only through `pull_request`, which runs without secrets.
- **Indirect prompt injector:** influences text the LLMs read (chat messages, docs pages, translation sources).
- **[assumed] Not modelled:** compromised Vercel, Cloudflare or GitHub accounts, and maintainers with write access.

## 6. Existing controls

- Rate limits: Upstash sliding window, 10/min per IP on chat (it fails open when unconfigured or slow, and uses the first `x-forwarded-for` hop). In-memory per-instance limiters: subscribe and unsubscribe 5/min, feedback 3/min, plus one on `/mcp`.
- `x-chat-page` must start with `/docs/` and contain no `..`, and it is resolved through the page map, not the filesystem.
- Feedback validates `opinion`, uses a relative `url` with no `//`, and caps message length.
- `/mcp`: Origin check, Content-Type and Accept checks, body size cap, protocol-version allow-list.
- JSON-LD escapes `<`. The OG title is truncated to 110 characters.
- Email validation and normalisation before Supabase queries. The service-role key is used only on the server.

## 7. Out of scope / accepted risk

Marco accepts these as risk:

- **Best-effort rate limiting and LLM cost abuse.** This covers in-memory per-instance limiters, bypassing the limit through `x-forwarded-for` spoofing or a missing or fail-open Upstash, and running up OpenRouter spend through the public chat. Findings whose _only_ impact is rate-limit bypass or LLM spend on the docs chat are accepted risk. This does **not** cover leaking or misusing the keys themselves, or DoS that takes the site down.
- **Docs chatbot jailbreaks / off-topic answers.** Prompt injection or jailbreaks that only make the chat model say something off-topic or wrong are accepted risk. Injection that reaches a security impact stays in scope: XSS through rendered model output, an unsafe `navigate` redirect, secret or system-prompt-borne secret disclosure, or tool misuse.
- **Not accepted:** unsubscribing any email without a token, and feedback/subscribe spam. Both stay at the scanner's rating.

## 8. Open questions

- None open on infrastructure: there is no infrastructure repo; Vercel and Cloudflare are configured in their dashboards (Marco). Edge behaviour (whether `x-forwarded-for` is overwritten) can't be confirmed from the repo.
