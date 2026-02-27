# Agent Transfer Protocol (ATP)

**The missing standard for the agentic web.**

> Every website is an API. Every API is discoverable. Every agent can act.

ATP is an open specification that lets any website expose its capabilities to AI agents through a simple, discoverable JSON manifest. No browser automation. No scraping. Just structured, authenticated, reliable access.

🌐 **[Read the Spec →](https://agentransferprotocol.org)** | 📄 **[Specification (v0.1)](./spec/v0.1/spec.md)** | 💬 **[Discussions](../../discussions)**

---

## The Problem

The web was built for humans. AI agents trying to use websites face:

| Metric | Browser Automation | ATP (Structured Access) |
|--------|-------------------|------------------------|
| Reliability | 35% on 20-step workflows | 99.9%+ via API contracts |
| Latency | 1-5 seconds per page | 20-100ms per call |
| Error rate | 5-10% per action | <0.1% |
| Maintenance | Breaks every 2-3 months | Versioned, backward-compatible |
| Annual cost | $50-100K in maintenance | Near-zero |
| Security | CAPTCHAs, anti-bot, session failures | Authenticated, authorized, auditable |

Over **$200M+ in venture funding** has been raised by companies trying to patch around this problem (Browserbase, Nimble, Firecrawl, MultiOn). Browser-use has 79,000+ GitHub stars because developers are desperate for a solution. But browser automation is a workaround, not a fix.

**ATP is the fix.** Instead of teaching agents to pretend to be humans, we give them a first-class interface.

## How It Works

### 1. Discover → `/.well-known/agent.json`

A JSON manifest at a well-known URL describes what the site can do, how to authenticate, and what capabilities are available.

```bash
curl https://acme.com/.well-known/agent.json
# → 200 OK — AgentManifest
# → 3 capabilities, OAuth 2.1, 4 scopes
```

### 2. Authenticate → OAuth 2.1 / API Keys / Delegated

ATP defines standard auth flows. Agents present credentials; sites grant scoped access. Human delegation is built in.

### 3. Act → Typed API Calls with Semantic Descriptions

Agents call structured endpoints with full type safety. Every action is described semantically so any LLM can reason about what it does.

```json
{
  "id": "search-products",
  "name": "Search Products",
  "description": "Search the product catalog by keyword, category, or price range",
  "semanticType": "commerce:product-search",
  "endpoint": "/api/v1/products/search",
  "method": "GET",
  "parameters": [
    { "name": "q", "type": "string", "required": true, "description": "Search query" },
    { "name": "category", "type": "string", "enum": ["electronics", "clothing", "home"] },
    { "name": "price_max", "type": "number", "description": "Maximum price in USD" }
  ],
  "requiredScopes": ["read:products"]
}
```

## Quick Start

### For Website Owners

1. Create `/.well-known/agent.json` on your domain
2. Define your capabilities, auth, and policies
3. Validate with `npx atp validate ./agent.json`

```json
{
  "@context": "https://atp.dev/schema/v1",
  "@type": "AgentManifest",
  "name": "My Website",
  "description": "What my site does",
  "version": "1.0.0",
  "auth": { ... },
  "capabilities": [ ... ],
  "policies": {
    "training": "deny",
    "inference": "allow"
  }
}
```

### For Agent Developers

```python
import atp

# Discover what a site can do
manifest = atp.discover("https://acme.com")

# List capabilities
for cap in manifest.capabilities:
    print(f"{cap.name}: {cap.description}")

# Call a capability
results = manifest.call("search-products", q="wireless headphones", price_max=100)
```

## Design Principles

| Principle | Description |
|-----------|-------------|
| **Simplicity First** | One JSON file. No new runtime. No build tools. If you can write robots.txt, you can write agent.json. |
| **Discovery by Convention** | `/.well-known/agent.json` is the single entry point. Like robots.txt for capabilities. |
| **Semantic, Not Just Structural** | Capabilities have semantic types (`commerce:product-search`) so agents understand intent, not just parameters. |
| **Auth is First-Class** | OAuth 2.1, API keys, and agent identity (DID:web) are built into the spec. |
| **Human-in-the-Loop by Default** | Side effects require explicit declaration. Irreversible actions require confirmation. |
| **Context-Window Aware** | Designed for LLM consumption. Progressive disclosure: summary first, details on demand. |

## How ATP Compares

| Feature | robots.txt | llms.txt | OpenAPI | MCP | A2A | **ATP** |
|---------|-----------|---------|---------|-----|-----|---------|
| Discovery | ✅ /.well-known | ✅ /llms.txt | ❌ None | ❌ Pre-configured | ✅ /.well-known | ✅ **/.well-known** |
| Purpose | Permissions | Content | API docs | Tool integration | Agent-to-agent | **Full web access** |
| Auth model | None | None | Described | OAuth 2.1 | Flexible | **OAuth 2.1 + DID** |
| Semantic types | ❌ | ❌ | ❌ | ❌ | Partial | ✅ |
| Side effects | N/A | N/A | ❌ | ❌ | ❌ | ✅ |
| Workflows | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Human-in-loop | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Training/inference policy | Partial | ❌ | ❌ | ❌ | ❌ | ✅ |
| Rate limits | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

## Repository Structure

```
agent-transfer-protocol/
├── README.md                  # This file
├── spec/
│   └── v0.1/
│       ├── spec.md            # Full specification document
│       └── schema.json        # JSON Schema for agent.json validation
├── examples/
│   ├── e-commerce.agent.json  # E-commerce site example
│   ├── saas.agent.json        # SaaS platform example
│   └── content.agent.json     # Content/media site example
├── website/
│   └── index.html             # Specification website
└── tools/
    └── atp-convert/           # Website-to-ATP converter (coming soon)
```

## Roadmap

- **Phase 1 — Specification** (Now): Publish ATP v0.1 spec. Reference implementation. Validator tool.
- **Phase 2 — Tooling** (Q2 2026): `atp-convert` CLI. WordPress/Shopify plugins. CMS integrations.
- **Phase 3 — Ecosystem** (Q3 2026): Agent framework integrations (LangChain, CrewAI). MCP bridge. Browser extension.
- **Phase 4 — Standards** (Q4 2026): IETF draft. IANA well-known URI registration. W3C community group.

## Contributing

ATP is an open specification. Contributions are welcome.

1. **Spec feedback** → Open an issue or start a discussion
2. **Examples** → Submit real-world `agent.json` manifests
3. **Tooling** → Help build validators, converters, and framework integrations
4. **Adoption** → Implement ATP on your website and share your experience

## License

MIT License — see [LICENSE](./LICENSE) for details.

---

**ATP is not affiliated with any company. It is an open standard for the open web.**
