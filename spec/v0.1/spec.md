# Agent Transfer Protocol — Specification v0.1

**Status:** Draft  
**Date:** February 2026  
**Authors:** Shriram Vasudevan  
**License:** MIT

---

## Abstract

The Agent Transfer Protocol (ATP) defines a standard for websites to expose their capabilities to AI agents through a discoverable, machine-readable JSON manifest. ATP enables any website to be programmatically accessible without browser automation, screen scraping, or ad-hoc integrations.

This specification defines:
- A discovery mechanism via `/.well-known/agent.json`
- A manifest format describing site capabilities, authentication, and policies
- Semantic type annotations for agent reasoning
- Workflow composition for multi-step interactions
- Human-in-the-loop controls for side effects and irreversible actions

## 1. Introduction

### 1.1 Motivation

The World Wide Web was designed for human consumption through visual browsers. As AI agents become primary consumers of web services, this human-centric design creates a fundamental impedance mismatch:

- **Browser automation** (Playwright, Puppeteer, browser-use) achieves ~35% success rates on multi-step workflows, with 5-10% error rates per individual action and 1-5 second latencies per page.
- **Screen scraping** produces 80-95% data accuracy compared to >99% from structured APIs, requires updates every 2-3 months as sites change, and costs $50-100K/year in maintenance for enterprise deployments.
- **No discovery standard** exists for an agent visiting a URL to determine what actions are available, how to authenticate, or what policies govern agent access.

ATP addresses this by defining a simple, file-based standard that any website operator can implement to make their site agent-accessible.

### 1.2 Design Goals

1. **Simplicity**: A single JSON file with no runtime dependencies
2. **Discoverability**: A well-known URL convention following RFC 8615
3. **Semantic richness**: Capability descriptions that enable LLM reasoning
4. **Security-first**: Authentication, authorization, and audit as primitives
5. **Human delegation**: Explicit controls for side effects and confirmation
6. **Backward compatibility**: Complementary to existing standards (OpenAPI, MCP, A2A)

### 1.3 Relationship to Other Standards

| Standard | Relationship to ATP |
|----------|-------------------|
| **robots.txt** (RFC 9309) | ATP governs what agents *can do*; robots.txt governs what crawlers *may access*. Complementary. |
| **OpenAPI** (OAS 3.x) | ATP capabilities MAY reference OpenAPI specs for detailed endpoint documentation. ATP adds discovery, semantics, and policies that OpenAPI lacks. |
| **MCP** (Anthropic) | ATP manifests can be consumed by MCP servers. An ATP-to-MCP bridge is a natural integration. ATP adds web-native discovery that MCP lacks. |
| **A2A** (Google) | A2A's `agent-card.json` describes agent capabilities; ATP's `agent.json` describes *website* capabilities. Complementary — a site may have both. |
| **llms.txt** | llms.txt describes site *content* for LLM comprehension; ATP describes site *capabilities* for agent action. Complementary. |
| **Schema.org** | ATP's semantic types can reference Schema.org vocabularies. ATP adds action semantics that Schema.org's `potentialAction` only partially covers. |

### 1.4 Terminology

- **Agent**: An AI system acting on behalf of a user to accomplish tasks via web services
- **Manifest**: The `agent.json` file describing a site's agent-accessible capabilities
- **Capability**: A discrete action or query an agent can perform on a site
- **Semantic Type**: A namespaced identifier describing the intent of a capability
- **Workflow**: An ordered composition of capabilities representing a multi-step process
- **Side Effect**: Any capability invocation that modifies server state

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

## 2. Discovery

### 2.1 Well-Known URI

An ATP-compliant site MUST serve its manifest at:

```
https://{host}/.well-known/agent.json
```

This follows the Well-Known URI convention defined in RFC 8615.

### 2.2 HTTP Requirements

The server MUST:
- Respond with `Content-Type: application/json`
- Support HTTPS (HTTP MUST redirect to HTTPS)
- Return HTTP 200 with a valid manifest, or HTTP 404 if no manifest is available
- Support CORS headers for cross-origin agent access:
  ```
  Access-Control-Allow-Origin: *
  Access-Control-Allow-Methods: GET, OPTIONS
  Access-Control-Allow-Headers: Accept, Authorization
  ```

The server SHOULD:
- Include `Cache-Control` headers (RECOMMENDED: `max-age=3600`)
- Support `ETag` / `If-None-Match` for conditional requests
- Include a `Link` header pointing to the manifest from the site's homepage:
  ```
  Link: </.well-known/agent.json>; rel="agent-manifest"
  ```

### 2.3 Alternative Discovery (HTML)

Sites MAY additionally include a `<link>` tag in their HTML `<head>`:

```html
<link rel="agent-manifest" href="/.well-known/agent.json" type="application/json">
```

This provides a fallback discovery mechanism for agents that process HTML before checking well-known URIs.

### 2.4 DNS-Based Discovery (Future)

A future version of this spec MAY define a DNS TXT record for ATP discovery:

```
_atp.example.com. IN TXT "v=atp1; manifest=/.well-known/agent.json"
```

This is NOT part of the v0.1 specification.

## 3. Manifest Format

### 3.1 Top-Level Structure

```json
{
  "@context": "https://atp.dev/schema/v1",
  "@type": "AgentManifest",
  "name": "string (REQUIRED)",
  "description": "string (REQUIRED)",
  "version": "string (REQUIRED, semver)",
  "provider": { ... },
  "auth": { ... },
  "rateLimit": { ... },
  "capabilities": [ ... ],
  "workflows": [ ... ],
  "schemas": { ... },
  "policies": { ... }
}
```

### 3.2 Provider Object

```json
{
  "provider": {
    "name": "string (REQUIRED)",
    "url": "string (REQUIRED, URI)",
    "contact": "string (OPTIONAL, email)",
    "logo": "string (OPTIONAL, URI)"
  }
}
```

### 3.3 Authentication Object

The `auth` object describes how agents can authenticate with the site.

```json
{
  "auth": {
    "schemes": [
      {
        "type": "oauth2 | apiKey | bearer | delegated",
        "...scheme-specific properties"
      }
    ],
    "agentIdentity": {
      "required": "boolean",
      "format": "did:web | did:key | custom"
    }
  }
}
```

#### 3.3.1 OAuth 2.1 Scheme

```json
{
  "type": "oauth2",
  "flows": {
    "authorizationCode": {
      "authorizationUrl": "string (URI, REQUIRED)",
      "tokenUrl": "string (URI, REQUIRED)",
      "refreshUrl": "string (URI, OPTIONAL)",
      "scopes": {
        "scope_name": "Human-readable description"
      }
    },
    "clientCredentials": {
      "tokenUrl": "string (URI, REQUIRED)",
      "scopes": { ... }
    }
  }
}
```

ATP RECOMMENDS OAuth 2.1 (draft-ietf-oauth-v2-1) as the primary authentication mechanism. Sites SHOULD support the `authorizationCode` flow for user-delegated access and MAY support `clientCredentials` for machine-to-machine access.

#### 3.3.2 API Key Scheme

```json
{
  "type": "apiKey",
  "in": "header | query | cookie",
  "name": "string (header/query parameter name)",
  "registration": "string (URI where agents can obtain keys)"
}
```

#### 3.3.3 Agent Identity

When `agentIdentity.required` is `true`, agents MUST include a verifiable identity in requests. The RECOMMENDED format is `did:web`, which ties agent identity to a domain the agent operator controls.

```
Agent-Identity: did:web:agent-provider.com:agents:shopping-assistant
```

### 3.4 Rate Limiting Object

```json
{
  "rateLimit": {
    "requests": "integer (max requests per window)",
    "window": "string (duration, e.g., '1h', '1m', '1d')",
    "burstLimit": "integer (OPTIONAL, max concurrent requests)",
    "tierUrl": "string (OPTIONAL, URI to pricing/tier info)"
  }
}
```

Sites SHOULD return standard rate limit headers on API responses:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1735689600
```

### 3.5 Capabilities Array

Each capability describes a discrete action or query. This is the core of the manifest.

```json
{
  "capabilities": [
    {
      "id": "string (REQUIRED, unique within manifest)",
      "name": "string (REQUIRED, human-readable)",
      "description": "string (REQUIRED, natural language for LLM reasoning)",
      "semanticType": "string (OPTIONAL, namespaced type identifier)",
      "endpoint": "string (REQUIRED, relative or absolute URI)",
      "method": "string (REQUIRED, HTTP method)",
      "parameters": [ ...parameter objects ],
      "response": { ...JSON Schema },
      "requiredScopes": [ "string" ],
      "sideEffects": "boolean (default: false)",
      "confirmation": {
        "required": "boolean",
        "message": "string"
      },
      "deprecated": "boolean (default: false)",
      "deprecationMessage": "string (OPTIONAL)"
    }
  ]
}
```

#### 3.5.1 Parameter Object

```json
{
  "name": "string (REQUIRED)",
  "type": "string | number | integer | boolean | array | object (REQUIRED)",
  "required": "boolean (default: false)",
  "description": "string (RECOMMENDED)",
  "default": "any (OPTIONAL)",
  "enum": [ ...allowed values (OPTIONAL) ],
  "format": "string (OPTIONAL, e.g., 'date', 'email', 'uri')",
  "minimum": "number (OPTIONAL)",
  "maximum": "number (OPTIONAL)",
  "pattern": "string (OPTIONAL, regex)"
}
```

#### 3.5.2 Semantic Types

Semantic types use a `namespace:type` convention. ATP defines the following standard namespaces:

| Namespace | Domain | Example Types |
|-----------|--------|---------------|
| `commerce` | E-commerce | `product-search`, `cart-add`, `cart-remove`, `order-create`, `order-status` |
| `content` | Content/Media | `article-search`, `article-read`, `media-stream`, `comment-create` |
| `account` | User accounts | `profile-read`, `profile-update`, `settings-read`, `settings-update` |
| `communication` | Messaging | `message-send`, `message-read`, `notification-subscribe` |
| `scheduling` | Calendar/booking | `availability-check`, `appointment-create`, `appointment-cancel` |
| `data` | Data access | `query`, `export`, `report-generate` |
| `auth` | Authentication | `login`, `logout`, `token-refresh`, `password-reset` |

Sites MAY define custom semantic types using their own namespace (e.g., `acme:custom-action`).

#### 3.5.3 Side Effects and Confirmation

Capabilities that modify server state MUST set `sideEffects: true`. Capabilities that are irreversible or have financial implications SHOULD set `confirmation.required: true` with a descriptive `confirmation.message`.

When `confirmation.required` is `true`, the agent SHOULD present the confirmation message to the user before executing the capability. Agents that execute confirmed capabilities without user approval MAY be rate-limited or banned by the site.

### 3.6 Workflows Array

Workflows describe multi-step processes composed of capabilities.

```json
{
  "workflows": [
    {
      "id": "string (REQUIRED)",
      "name": "string (REQUIRED)",
      "description": "string (REQUIRED)",
      "steps": [ "capability-id-1", "capability-id-2", "..." ],
      "conditional": {
        "step-id": {
          "condition": "response field expression",
          "onTrue": "next-step-id",
          "onFalse": "alternative-step-id"
        }
      }
    }
  ]
}
```

Workflows are advisory — they describe the intended sequence but agents MAY invoke capabilities independently. The `conditional` object allows branching logic (e.g., "if cart total > $100, apply discount before checkout").

### 3.7 Schemas Object

Reusable JSON Schema definitions referenced by capabilities via `$ref`.

```json
{
  "schemas": {
    "Product": {
      "type": "object",
      "properties": {
        "id": { "type": "string" },
        "name": { "type": "string" },
        "price": { "type": "number" },
        "currency": { "type": "string", "default": "USD" }
      },
      "required": ["id", "name", "price"]
    }
  }
}
```

References use JSON Pointer syntax: `{ "$ref": "#/schemas/Product" }`

### 3.8 Policies Object

```json
{
  "policies": {
    "training": "allow | deny | conditional",
    "inference": "allow | deny | conditional",
    "caching": {
      "allowed": "boolean",
      "maxAge": "integer (seconds)"
    },
    "attribution": "required | preferred | none",
    "termsUrl": "string (OPTIONAL, URI to full terms)",
    "privacyUrl": "string (OPTIONAL, URI to privacy policy)"
  }
}
```

- `training: "deny"` — Site data MUST NOT be used for model training
- `training: "allow"` — Site data MAY be used for model training
- `training: "conditional"` — See `termsUrl` for conditions
- `inference: "allow"` — Agents MAY use data in real-time responses
- `attribution: "required"` — Agents MUST cite the source when using data

## 4. Protocol Behavior

### 4.1 Agent Request Flow

1. Agent fetches `https://{host}/.well-known/agent.json`
2. Agent parses manifest and identifies relevant capabilities
3. If authentication is required, agent follows the specified auth flow
4. Agent invokes capabilities via HTTP requests to specified endpoints
5. Agent includes required headers:
   ```
   User-Agent: {agent-name}/{version} (ATP/0.1)
   Authorization: Bearer {token}
   Agent-Identity: {did} (if required)
   X-ATP-Version: 0.1
   ```

### 4.2 Error Handling

ATP endpoints SHOULD return standard HTTP status codes with structured error bodies:

```json
{
  "error": {
    "code": "string",
    "message": "string (human/LLM-readable)",
    "details": { ... },
    "retryAfter": "integer (seconds, OPTIONAL)"
  }
}
```

### 4.3 Versioning

The manifest `version` field follows Semantic Versioning 2.0.0:
- **Major** version changes indicate breaking changes to capabilities
- **Minor** version changes indicate new capabilities added
- **Patch** version changes indicate documentation or metadata updates

Agents SHOULD check the manifest version and handle version mismatches gracefully.

## 5. Security Considerations

### 5.1 Transport Security
All ATP communication MUST use HTTPS. Manifests served over HTTP MUST be rejected.

### 5.2 Agent Identity Verification
Sites MAY verify agent identity by resolving the `did:web` identifier and checking the associated DID document.

### 5.3 Scope Enforcement
Sites MUST enforce capability-level scopes. An agent with `read:products` scope MUST NOT be able to invoke `place-order` (which requires `write:orders`).

### 5.4 Rate Limiting
Sites SHOULD enforce rate limits as declared in the manifest. Agents that exceed rate limits SHOULD receive HTTP 429 responses.

### 5.5 Audit Logging
Sites SHOULD log all agent-invoked capabilities with the agent identity, capability ID, timestamp, and outcome.

### 5.6 Prompt Injection Mitigation
Agent responses from ATP endpoints SHOULD be treated as untrusted data. Sites SHOULD NOT include executable instructions in API responses that could manipulate the calling agent's behavior.

## 6. Implementation Guidelines

### 6.1 For Website Operators

1. Start with read-only capabilities (search, browse, read)
2. Add authentication before exposing write capabilities
3. Set `sideEffects: true` and `confirmation.required: true` for destructive operations
4. Use semantic types from the standard namespaces where possible
5. Keep the manifest under 50KB for context-window efficiency
6. Version your manifest and maintain backward compatibility

### 6.2 For Agent Developers

1. Always check `/.well-known/agent.json` before attempting browser automation
2. Respect `policies.training` and `policies.inference` declarations
3. Present `confirmation.message` to users before executing confirmed capabilities
4. Include proper `User-Agent` and `Agent-Identity` headers
5. Handle rate limiting gracefully with exponential backoff
6. Cache manifests according to `Cache-Control` headers

### 6.3 For Framework Authors

1. Implement ATP discovery as a first-class primitive
2. Auto-generate tool definitions from ATP manifests (e.g., MCP tools, LangChain tools)
3. Respect semantic types for intelligent capability selection
4. Support workflow composition for multi-step tasks

## 7. IANA Considerations

This specification requests registration of the following well-known URI:

- **URI suffix**: `agent.json`
- **Change controller**: ATP Working Group
- **Specification document**: This document
- **Status**: Provisional

## 8. References

- RFC 8615 — Well-Known URIs
- RFC 9309 — Robots Exclusion Protocol
- RFC 6749 — OAuth 2.0 Authorization Framework
- draft-ietf-oauth-v2-1 — OAuth 2.1
- JSON Schema (draft-2020-12)
- Semantic Versioning 2.0.0
- W3C Decentralized Identifiers (DIDs) v1.0

---

*ATP Specification v0.1 — Draft — February 2026*
