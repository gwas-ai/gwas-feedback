# gwas-feedback

Short answer: yes—it’s a very solid pattern. Use **`gwas.feedback`** as the canonical app and point per-site vanity subdomains like **`feedback.gwas.ai`**, **`feedback.gwas.app`**, etc. at it (via CNAME/reverse proxy + Host header–based tenancy). Here’s the trade-off sheet and a blueprint so you can deploy it cleanly.

# Why this is reasonable

* It gives you **one codebase** with **many first-party entry points**. Each project keeps a same-site feedback URL (`feedback.gwas.*`) while all traffic lands on the same service at `gwas.feedback`.
* You avoid the mounting problems of third-party embeds as browsers keep deprecating cross-site cookies/storage. Same-site subdomains per eTLD+1 preserve auth/session where possible.

# Pros

## UX & trust

* **First-party feel**: `gwas.ai` can open `feedback.gwas.ai` in a modal/iframe and remain same-site → fewer consent prompts, smoother auth.
* **Predictable affordance**: users and agents always know “feedback lives at `feedback.<site>`”.
* **Deep-link semantics**: URLs can encode context per property (e.g., `/session/:id?model=gpt-5&topic=search`).

## Technical

* **Single service, multi-tenant**: tenant derived from the Host header; simpler deploys and observability.
* **Cookie/session viability**: same-site on each property (`gwas.ai` ↔ `feedback.gwas.ai`) dodges cross-site cookie partitioning.
* **Cleaner CSP**: allow `frame-ancestors https://*.gwas.ai https://*.gwas.app …` while keeping app logic centralized.
* **Per-site feature flags & throttling**: rate-limit, rollout, and A/B test by tenant (the domain).
* **Analytics clarity**: attribute frustration to the originating property without messy referrer gymnastics.

## Org & brand

* **Canonical hub**: `gwas.feedback` is the public docs/dashboard/API home.
* **Vanity ingress**: `feedback.gwas.*` gives you consistency across 150+ sites without duplicating apps.

# Cons

## Ops overhead

* **DNS sprawl**: you’ll add/maintain a `feedback` record for every domain. Mitigate with templated Cloudflare/Bunny/Route53 APIs.
* **TLS management**: you need **wildcard certs per eTLD+1** (e.g., `*.gwas.ai`, `*.gwas.app`). You can’t wildcard across TLDs, so renewals multiply (ACME automation recommended).
* **CORS/CSRF nuance**: even with same-site subdomains, you’ll juggle multiple origins; lock this down with `SameSite=Lax` cookies, per-tenant CSRF secrets, and strict `Origin` checks.
* **Observability cardinality**: many hostnames explode metrics labels; budget for cost and retention.

## Product/SEO

* **Fragmented SEO**: capture forms themselves shouldn’t be indexed; set `noindex` on `feedback.gwas.*` to avoid duplicate/low-value pages.
* **Inconsistent same-site**: when a property lives on a different eTLD+1 (e.g., `gwas.app` embedding `gwas.feedback`), it’s cross-site. Your `feedback.gwas.<same-tld>` approach fixes this, but only if you actually use the per-site subdomain.

# Recommended architecture

**DNS**

* For each domain: `feedback.gwas.<tld>` → CNAME `edge.gwas.feedback` (your CDN/edge).
* `gwas.feedback` → A/AAAA to the same edge (canonical).

**Routing**

* Edge worker reads `Host` → sets `tenant = host.eTLDplus1` (e.g., `gwas.ai`) and `site = host` (e.g., `feedback.gwas.ai`).
* Forward to a single backend (`/api/*`, `/app/*`) with `X-Tenant`, `X-Site`.

**Auth & sessions**

* OIDC with one IdP; cookie name includes tenant (`gf_sess_<tenant>`).
* `SameSite=Lax`, `Secure`, `HttpOnly`.
* CSRF token per tenant, rotated.

**Storage & schema**

* Multi-tenant tables keyed by `tenant_id`, `site_id`.
  Core event shape: `timestamp, session_id, model, prompt_hash, frustration_type, emotion, score, comment, origin_url`.

**Embedding**

* From each property, prefer iframe to `https://feedback.gwas.<same-tld>/capture?...`.
* Set `frame-ancestors` to allowed origins and use postMessage for handshakes.

**Certificates**

* Automate ACME via your edge provider; create wildcard per TLD (`*.gwas.ai`, `*.gwas.app`, …).

**Analytics**

* Partition metrics/logs by `tenant` and `site`. Create “frustration heatmaps” dashboards with per-site rollups and global views.

# Practical rules of thumb

* **Default to the vanity subdomain** (same eTLD+1) when embedded inside a site → best session behavior.
* **Link out to `gwas.feedback`** for public docs, dashboards, and API references.
* **Centralize code & deploys**; decentralize only DNS and branding.
* **Guardrails**: strict CSP, `Permissions-Policy`, and minimal third-party scripts to preserve first-party status.

# TL;DR

Your plan—**standalone at `gwas.feedback` + per-site subdomains `feedback.gwas.*` that route to it**—is not just reasonable; it’s the *right* architecture for a multi-property, agent-centric feedback layer in a world of tightening browser privacy rules. It keeps UX first-party, keeps code centralized, and scales cleanly to 150+ sites—with some manageable DNS/TLS automation.
