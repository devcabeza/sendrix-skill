# Sendrix AI Agent Skill

[![skills.sh](https://img.shields.io/badge/skills.sh-sendrix--skill-blue?style=flat-square)](https://skills.sh)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg?style=flat-square)](LICENSE)

Official [skills.sh](https://skills.sh) package for integrating **Sendrix** transactional email gateway & proxy into your projects using AI coding agents (**Antigravity**, **Claude Code**, **Cursor**, **Windsurf**, and **GitHub Copilot**).

---

## ⚡ Installation

Install directly into your project using the standard `skills` CLI:

```bash
npx skills add devcabeza/sendrix-skill
```

This will automatically configure the skill inside your agent's directory (e.g. `.agents/skills/sendrix`, `.claude/skills/sendrix`, or `.cursor/rules/sendrix.mdc`).

---

## 🤖 What your AI Agent Learns

Once installed, your AI agent understands how to:

1. **Configure Environment Variables**:
   ```dotenv
   SENDRIX_BASE_URL=https://sendrix.alejandrocabeza.dev
   SENDRIX_KEY=sndx_live_xxxxxxxxxxxxxxxxxxxxxxxx
   ```
2. **Send Transactional Emails**:
   - Single email sending via `POST /api/v1/send`
   - Batch email sending via `POST /api/v1/batch` (up to 100 emails)
   - Asynchronous queueing (`async: true` or `Prefer: respond-async`)
3. **Handle Framework Integrations**:
   - **Laravel**: Zero-change custom mail transport (`SendrixTransport`), Mailables, Horizon background queue (`emails`), and 429 rate limit backoff.
   - **Node.js & TypeScript**: Native fetch client with retries and exponential backoff, Next.js App Router support.
   - **Python**: Requests/httpx client with retry wrappers.
4. **Enforce Rate Limits & Resilience**:
   - Manages Sendrix rate limits (5 req/min) using exponential backoff and `release()`.
   - Prevents duplicate sends on `duplicated: true` responses.
5. **Verify Outbound Webhooks**:
   - Cryptographic HMAC-SHA256 signature verification (`X-Sendrix-Signature`).

---

## 💡 Prompt Your Agent

After installing, simply ask your agent in your project chat:

> *"He instalado el skill de Sendrix. Por favor, integra el envío de correos transaccionales con Sendrix para los correos de bienvenida y recuperación de contraseña en segundo plano."*

---

## 📚 References & Guides

- [SKILL.md](./SKILL.md): Main agent instruction file with dynamic capability discovery.
- [Laravel Integration Guide](./references/laravel.md): Mail transport, queue jobs, Pest tests.
- [Node.js & TypeScript Guide](./references/nodejs.md): TypeScript client and Next.js actions.
- [Python Integration Guide](./references/python.md): Python requests client.
- [API Reference](./references/api-reference.md): Endpoints, schemas, and response codes.
- [Webhooks Verification](./references/webhooks.md): HMAC verification in PHP and Node.

---

## License

MIT License © 2026 Sendrix / devcabeza
