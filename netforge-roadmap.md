# NetForge — Phased Build Roadmap

*All-in-one network engineering platform · commercial product · v0.1 plan (June 2026)*

This roadmap sequences the work from a sellable MVP to a mature platform. Phases are ordered by dependency and value: each one ships something usable on its own, and earns the right to build the next. Indicative timeline assumes a small team (≈4–6 engineers + 1 designer + 1 PM); adjust to your actual capacity.

---

## Guiding principles

The architecture decision that shapes everything else is the **collector/agent**: a lightweight service deployed inside the customer's network that does the polling, scanning, and config access, then reports to a cloud (or self-hosted) control plane. Almost nothing real works without it, so it lands in Phase 0. We sell outcomes, not features — every phase is anchored to a problem a network engineer will pay to stop having. Security and multi-tenancy are designed in from the start, not retrofitted, because they are nearly impossible to add cleanly later.

---

## Phase 0 — Foundation (Weeks 1–6)

**Goal:** the plumbing that all features depend on. Nothing customer-facing ships, but it de-risks everything.

The collector agent is the centerpiece: a containerized service that runs on-prem, authenticates to the control plane over an outbound mTLS tunnel (no inbound firewall holes), and exposes a job-runner for discovery, polling, and scans. Alongside it, stand up the control plane skeleton — API gateway, time-series store for metrics (e.g. a Postgres + Timescale or VictoriaMetrics), object store for config/firmware artifacts, and the auth layer (SSO/OIDC, RBAC, audit logging) and tenant isolation model. Build CI/CD, secrets management, and the agent auto-update channel now so you are never shipping security patches by hand.

**Exit criteria:** an agent in a lab network can register, receive a job, and return data to the control plane; a user can log in with role-scoped access.

---

## Phase 1 — MVP: Visibility (Weeks 7–16)

**Goal:** "I plug in NetForge and immediately see my whole network." This is the smallest thing worth paying for and the basis of your first design partners.

Ship **discovery and inventory** (ARP/SNMP/LLDP/CDP/mDNS sweeps, device fingerprinting, an auto-built topology), **monitoring and alerting** (ICMP/SNMP polling, interface and CPU/memory metrics, threshold alerts routed to email/Slack/webhook), and the **dashboard** that ties them together. Round it out with the **diagnostics toolkit** (ping, traceroute, DNS, port scan, subnet calc) — cheap to build, used daily, and great for demos.

**Exit criteria:** 3–5 design-partner networks running in production; mean time to "first useful insight" under 30 minutes from install. **This is your first sellable release.**

---

## Phase 2 — Control: Config & IPAM (Weeks 17–28)

**Goal:** move from watching the network to managing it — the jump from a nice-to-have to a system of record.

Add **configuration management**: SSH/NETCONF/RESTCONF access, scheduled backups with version diffing, drift detection against a golden baseline, and rollback. Then **bulk config push** with templates and a dry-run/approval flow (high value, higher risk — gate it behind RBAC and require confirmation). Ship **IPAM**: subnet/VLAN tracking, DHCP scope visibility, and IP conflict detection.

**Exit criteria:** customers trust NetForge as the source of truth for device configs; at least one customer doing scheduled bulk changes through it.

---

## Phase 3 — Secure: Scanning & the Firmware Scanner (Weeks 29–42)

**Goal:** the security story, including the headline firmware vulnerability scanner. This is a strong differentiator and a reason to expand a contract.

Start with **network-level security scanning** (open-port/service detection, default-credential checks, weak SNMP strings, TLS/cipher auditing) — it reuses the discovery engine. Then build the **firmware vulnerability scanner** as its own pipeline: ingest an image (upload or live pull via the agent) → unpack with a binwalk-style extractor → fingerprint components and versions → generate a CycloneDX SBOM → match against the NVD CVE feed plus vendor advisories → produce a CVSS-weighted, risk-ranked report with remediation guidance, plus secret/private-key detection. Add **compliance reporting** (CIS benchmarks) on top.

The scanner is the most research-heavy component, which is why it is deliberately late: it depends on the agent (Phase 0), inventory (Phase 1), and benefits from the config baseline (Phase 2). Lean on mature open-source for extraction and the CVE feed rather than building those from scratch; your value is the integration, SBOM accuracy, and the workflow around remediation.

**Exit criteria:** scanner produces accurate SBOMs and CVE matches against a test corpus of real firmware with low false-positive rate; findings flow into the alerting and reporting built earlier.

---

## Phase 4 — Scale: Automation & Platform (Weeks 43–56)

**Goal:** turn point features into a platform customers build on — and make larger, multi-site deals possible.

Ship **automation and scheduled jobs** (recurring backups, re-scans, reports), a public **REST/GraphQL API** and webhooks, **multi-site / multi-tenant** management at scale, and richer **reporting/export** (PDF/CSV, scheduled delivery). Harden for the enterprise: granular RBAC, SAML, full audit trails, and the SOC 2 groundwork procurement teams will ask for.

**Exit criteria:** a customer manages multiple sites from one tenant; a third party integrates via the API; enterprise security questionnaire is answerable with a yes.

---

## Phase 5 and beyond — Intelligence (ongoing)

Once the data is flowing and trusted, the differentiators get smarter: anomaly detection and predictive alerting on the metrics you already collect, automated remediation playbooks (detect drift or a CVE, propose or apply the fix), AI-assisted root-cause analysis, and integrations with the rest of the stack (ITSM/ticketing, SIEM, IdP). These are high-value but only credible after the foundation is solid — premature "AIOps" on thin data erodes trust.

---

## Sequencing rationale at a glance

| Phase | Theme | Why here | Sellable? |
|------|-------|----------|-----------|
| 0 | Foundation / agent | Everything depends on it | No |
| 1 | Visibility | Fastest path to value; demoable | **Yes — first release** |
| 2 | Config & IPAM | Becomes system of record | Yes — expansion |
| 3 | Security + firmware scanner | Differentiator; depends on 0–2 | Yes — upsell |
| 4 | Automation & platform | Unlocks enterprise/multi-site | Yes — larger deals |
| 5 | Intelligence | Needs trusted data first | Ongoing |

## Key risks to watch

The agent's security posture is the whole product's security posture — a compromised collector is a foothold in every customer network, so it deserves disproportionate review. Bulk config push and any automated remediation are the features most likely to cause an outage; ship them behind dry-run, approval, and easy rollback. Firmware scanner accuracy (false positives/negatives) directly affects trust; validate against a real corpus before GA. And scope discipline matters most in Phases 1–2 — the temptation to build everything at once is what sinks all-in-one products.
