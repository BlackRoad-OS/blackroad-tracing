<div align="center">

# 🛣️🔍 blackroad-tracing

### Distributed Tracing for the BlackRoad OS Platform

Enterprise-grade, end-to-end distributed tracing. Zero-overhead request tracking across every agent, service, and route in your BlackRoad mesh.

[![npm version](https://img.shields.io/npm/v/blackroad-tracing?style=for-the-badge&color=FF0066)](https://www.npmjs.com/package/blackroad-tracing)
[![npm downloads](https://img.shields.io/npm/dm/blackroad-tracing?style=for-the-badge&color=FF0066)](https://www.npmjs.com/package/blackroad-tracing)
[![License](https://img.shields.io/badge/License-Proprietary-9C27B0?style=for-the-badge)](./LICENSE)
[![Platform](https://img.shields.io/badge/Platform-blackroad.io-FF1D6C?style=for-the-badge)](https://blackroad.io)
[![Status](https://img.shields.io/badge/Status-Production-00C853?style=for-the-badge)](https://status.blackroad.io)

</div>

---

## 📋 Table of Contents

1. [Overview](#-overview)
2. [Features](#-features)
3. [Installation](#-installation)
4. [Quick Start](#-quick-start)
5. [Configuration](#-configuration)
6. [API Reference](#-api-reference)
   - [Tracer](#tracer)
   - [Span](#span)
   - [Context Propagation](#context-propagation)
   - [Exporters](#exporters)
7. [Integrations](#-integrations)
   - [BlackRoad Operator](#blackroad-operator)
   - [Jaeger](#jaeger)
   - [OpenTelemetry](#opentelemetry)
   - [Stripe Webhook Tracing](#stripe-webhook-tracing)
8. [Stripe Subscription & Pricing](#-stripe-subscription--pricing)
9. [End-to-End Verification](#-end-to-end-verification)
10. [Security](#-security)
11. [Contributing](#-contributing)
12. [License](#-license)

---

## 🎯 Overview

`blackroad-tracing` is the official distributed tracing package for [BlackRoad OS](https://blackroad.io). It instruments every hop in the BlackRoad routing layer—from the Cloudflare edge through the Operator to downstream AI models, Stripe payments, and Salesforce CRM—giving you a complete, auditable view of every request.

Built on the OpenTelemetry standard and pre-wired for Jaeger, `blackroad-tracing` surfaces latency, errors, and routing decisions across all 30,000+ BlackRoad agents with no additional configuration.

```
[Cloudflare Edge] → [Operator] → [Agent / AI / Stripe / Salesforce]
       ↑                ↑                       ↑
   trace span       trace span             trace span
       └──────────────────────────────────────────┘
                   blackroad-tracing
```

---

## ✨ Features

| Feature | Description |
|---|---|
| **Zero-config instrumentation** | Auto-instruments `fetch`, `http`, `express`, and `fastify` |
| **W3C Trace Context** | Full `traceparent` / `tracestate` header propagation |
| **Jaeger exporter** | Pre-built exporter for BlackRoad Jaeger Enterprise |
| **OTLP exporter** | Export to any OpenTelemetry-compatible backend |
| **Stripe span enrichment** | Automatic span tagging for Stripe payment events |
| **Operator routing spans** | Traces every routing decision in the BlackRoad Operator |
| **Salesforce span support** | Correlates CRM API calls to originating requests |
| **Sampling strategies** | Head-based, tail-based, and adaptive sampling |
| **Async context tracking** | Correct span propagation through `async/await` and Promises |
| **TypeScript-first** | Full type definitions bundled |
| **Edge runtime compatible** | Runs on Cloudflare Workers, Node.js, and Bun |

---

## 📦 Installation

```bash
# npm
npm install blackroad-tracing

# yarn
yarn add blackroad-tracing

# pnpm
pnpm add blackroad-tracing
```

**Peer dependencies (Node.js ≥ 18 / Edge runtime):**

```bash
npm install @opentelemetry/api @opentelemetry/sdk-node
```

---

## 🚀 Quick Start

### Node.js / Express

```typescript
import { BlackroadTracer } from 'blackroad-tracing';

const tracer = new BlackroadTracer({
  serviceName: 'my-service',
  apiKey: process.env.BLACKROAD_API_KEY,
});

tracer.start();

// All outbound HTTP requests are now automatically traced.
```

### Cloudflare Workers / Edge

```typescript
import { edgeTracer } from 'blackroad-tracing/edge';

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const span = edgeTracer.startSpan('handle-request', { request });
    try {
      const response = await handleRequest(request, env);
      span.setStatus({ code: 'OK' });
      return response;
    } catch (err) {
      span.recordException(err as Error);
      throw err;
    } finally {
      span.end();
    }
  },
};
```

### Manual span creation

```typescript
import { tracer } from 'blackroad-tracing';

async function routeRequest(input: string) {
  const span = tracer.startSpan('operator.route', {
    attributes: { 'br.input_length': input.length },
  });

  try {
    const result = await operator.route(input);
    span.setAttribute('br.route_target', result.target);
    return result;
  } finally {
    span.end();
  }
}
```

---

## ⚙️ Configuration

| Option | Type | Default | Description |
|---|---|---|---|
| `serviceName` | `string` | `'blackroad-service'` | Name shown in Jaeger / dashboards |
| `apiKey` | `string` | — | BlackRoad API key (required for managed export) |
| `endpoint` | `string` | `'https://tracing.blackroad.io'` | OTLP/Jaeger collector endpoint |
| `sampler` | `'always' \| 'never' \| 'parent' \| number` | `1.0` | Sampling rate (1.0 = 100%) |
| `propagators` | `string[]` | `['tracecontext', 'baggage']` | W3C propagators to use |
| `exportInterval` | `number` | `5000` | Export batch interval in ms |
| `debug` | `boolean` | `false` | Enable verbose console output |
| `stripeEnrichment` | `boolean` | `true` | Auto-tag Stripe API spans |
| `redactHeaders` | `string[]` | `['authorization', 'cookie']` | Headers to redact from spans |

### Environment variables

```bash
BLACKROAD_API_KEY=br_live_...
BLACKROAD_TRACING_ENDPOINT=https://tracing.blackroad.io
BLACKROAD_SERVICE_NAME=my-service
OTEL_EXPORTER_OTLP_ENDPOINT=https://tracing.blackroad.io
```

---

## 📖 API Reference

### `Tracer`

#### `new BlackroadTracer(config)`

Creates and configures a new tracer instance.

```typescript
const tracer = new BlackroadTracer({
  serviceName: 'operator',
  apiKey: process.env.BLACKROAD_API_KEY,
  sampler: 0.1, // 10% sampling in production
});
```

#### `tracer.start()`

Registers the tracer as the global OpenTelemetry provider and begins instrumenting auto-instrumented libraries.

#### `tracer.stop()`

Gracefully flushes pending spans and shuts down the exporter.

#### `tracer.startSpan(name, options?)`

Returns a new `Span`. Automatically inherits parent context from async storage.

---

### `Span`

| Method | Description |
|---|---|
| `span.setAttribute(key, value)` | Set a single span attribute |
| `span.setAttributes(attrs)` | Set multiple attributes at once |
| `span.addEvent(name, attrs?)` | Record a point-in-time event on the span |
| `span.recordException(error)` | Record an exception with stack trace |
| `span.setStatus({ code, message? })` | Set span status (`'OK'` \| `'ERROR'`) |
| `span.end()` | End the span and schedule export |

---

### Context Propagation

```typescript
import { propagator } from 'blackroad-tracing';

// Inject trace context into outbound headers
const headers: Record<string, string> = {};
propagator.inject(context.active(), headers);

// Extract trace context from inbound headers
const ctx = propagator.extract(context.active(), incomingHeaders);
```

---

### Exporters

#### Managed (BlackRoad Cloud)

```typescript
new BlackroadTracer({ apiKey: process.env.BLACKROAD_API_KEY });
// Exports to https://tracing.blackroad.io automatically
```

#### Self-hosted Jaeger

```typescript
new BlackroadTracer({
  exporter: 'jaeger',
  endpoint: 'http://localhost:14268/api/traces',
});
```

#### OTLP (any backend)

```typescript
new BlackroadTracer({
  exporter: 'otlp',
  endpoint: 'https://your-collector.example.com',
});
```

---

## 🔗 Integrations

### BlackRoad Operator

`blackroad-tracing` is the recommended observability layer for the BlackRoad Operator. Install alongside `@blackroad/operator` for automatic routing span instrumentation:

```typescript
import { BlackroadTracer } from 'blackroad-tracing';
import { Operator } from '@blackroad/operator';

const tracer = new BlackroadTracer({ serviceName: 'operator' });
tracer.start();

const operator = new Operator({ tracer }); // spans created automatically
```

### Jaeger

`blackroad-tracing` ships with first-class support for [BlackRoad Jaeger Enterprise](https://github.com/BlackRoad-OS/blackroad-os-jaeger):

```typescript
new BlackroadTracer({
  exporter: 'jaeger',
  endpoint: process.env.JAEGER_ENDPOINT,
});
```

### OpenTelemetry

The package is built on the OpenTelemetry JS SDK. Any `@opentelemetry/*` compatible exporter, sampler, or propagator can be passed directly:

```typescript
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

new BlackroadTracer({
  exporter: new OTLPTraceExporter({ url: 'https://your-backend/v1/traces' }),
});
```

### Stripe Webhook Tracing

Enable automatic span enrichment for every Stripe event processed through your service:

```typescript
const tracer = new BlackroadTracer({
  serviceName: 'payments',
  stripeEnrichment: true,
  apiKey: process.env.BLACKROAD_API_KEY,
});

// Stripe webhooks will now emit spans with:
//   stripe.event.type, stripe.event.id, stripe.livemode
```

---

## 💳 Stripe Subscription & Pricing

`blackroad-tracing` is available under a usage-based subscription managed through [Stripe](https://stripe.com). Billing is handled automatically when you provide a valid `BLACKROAD_API_KEY`.

| Plan | Price | Spans / month | Retention | SLA |
|---|---|---|---|---|
| **Starter** | Free | 1,000,000 | 3 days | Community |
| **Growth** | $9 / mo | 50,000,000 | 14 days | 99.9% |
| **Scale** | $49 / mo | 500,000,000 | 30 days | 99.95% |
| **Enterprise** | Custom | Unlimited | 90 days | 99.99% |

> **Upgrade or manage your subscription:** [blackroad.io/billing](https://blackroad.io/billing)

All plans include:
- Managed Jaeger UI at `https://tracing.blackroad.io`
- Stripe webhook span enrichment
- W3C Trace Context propagation
- 24/7 status at [status.blackroad.io](https://status.blackroad.io)

---

## ✅ End-to-End Verification

Run the built-in e2e smoke test to verify your installation is fully wired:

```bash
npx blackroad-tracing verify --api-key=$BLACKROAD_API_KEY
```

Expected output:

```
✓ Tracer initialized
✓ Span created and ended
✓ Context propagation: W3C traceparent injected
✓ Exporter: span delivered to https://tracing.blackroad.io
✓ Jaeger UI: trace visible at https://tracing.blackroad.io/trace/<id>
✓ All checks passed
```

---

## 🔒 Security

- All spans are transmitted over TLS 1.3.
- `authorization` and `cookie` headers are redacted from spans by default.
- API keys are never included in span attributes or events.
- To report a security vulnerability, email **security@blackroad.io** or see [`SECURITY.md`](https://github.com/BlackRoad-OS/.github/blob/main/SECURITY.md).

---

## 🤝 Contributing

Contributions are welcome from BlackRoad OS team members. Please review [`CONTRIBUTING.md`](https://github.com/BlackRoad-OS/.github/blob/main/CONTRIBUTING.md) and [`CODE_OF_CONDUCT.md`](https://github.com/BlackRoad-OS/.github/blob/main/CODE_OF_CONDUCT.md) before opening a pull request.

---

## 📄 License

BlackRoad OS, Inc. Proprietary License. See [`LICENSE`](./LICENSE) for full terms.

---

<div align="center">

**Part of [BlackRoad OS](https://blackroad.io)** — 30,000+ AI Agents · 17 Organizations · 1,800+ Repos

*© BlackRoad OS, Inc. All rights reserved.*

</div>
