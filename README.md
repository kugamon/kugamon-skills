# kugamon-skills
Skills for Kugamon — agentic workflows that manage the full **Quote-to-Cash** and **Subscription Management** lifecycle in Salesforce through the Kugamon managed packages.

> **Status:** Private / in testing.
> This repository is private while we validate the skills against live Kugamon orgs. It will be made public once testing is complete.

## What's inside

| Skill | Location | Version | Status |
|-------|----------|---------|--------|
| `kugamon-full-qtc-submgmt` | [`skills/kugamon-full-qtc-submgmt/`](./skills/kugamon-full-qtc-submgmt/) | 0.2.0 | Beta |

### kugamon-full-qtc-submgmt

Manages the full Kugamon lifecycle — opportunities, quotes, orders, order releases, invoices, payments, shipments, contracts, subscriptions, renewal opportunities, and assets.

**Supports four Kugamon deployment modes:**

| Mode | Flow |
|------|------|
| **CPQ** | Opportunity → Quote → Order → (Order Release) → Asset |
| **Q2C** (Quote to Cash) | Opportunity → Quote → Order → (Order Release) → Asset + Shipment → Invoice → Payment |
| **SubMgmt** (Subscription Management) | Opportunity → Quote → Order → (Order Release) → Asset + Contract + Subscription + Renewal Opportunity |
| **SB** (Subscription Billing) | Opportunity → Quote → Order → (Order Release) → Asset + Shipment + Contract + Subscription + Renewal Opportunity → Invoice → Payment |

The skill **auto-detects** which Kugamon packages are installed in the target org (by checking for the `kuga_sub__Renew__c` field on `OpportunityLineItem`) and adapts accordingly.

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code) or another Claude agent runtime that supports skills.
- A Salesforce MCP connector with access to the target org. This skill assumes the agent can run SOQL, DML, and Apex against the org.
- A Salesforce org with one or more Kugamon managed packages installed:
  - **Kugamon Quote to Cash** (namespace: `kugo2p`)
  - **Kugamon Subscription Management** (namespace: `kuga_sub`) — optional

## Install

### Option A — Install as a Claude Code skill

Clone the repo and copy (or symlink) the skill folder into your local Claude skills directory:

```bash
git clone https://github.com/kugamon/kugamon-skills.git
cp -r kugamon-skills/skills/kugamon-full-qtc-submgmt ~/.claude/skills/
```

Restart Claude Code. The skill will trigger automatically when you ask it to operate on Kugamon objects.

### Option B — Reference it from an agent

Point your agent's skill loader at `skills/kugamon-full-qtc-submgmt/SKILL.md`.

## Usage

Once installed, invoke the skill by asking Claude to perform any Kugamon Quote-to-Cash operation in plain English. Examples:

- *"Create a new quote in my Kugamon sandbox for Acme Corp for 50 user licenses."*
- *"Convert quote Q-0042 into an order."*
- *"Generate an invoice for order O-0099 and record a payment."*
- *"Show me all open renewal opportunities closing this quarter."*

The skill handles package detection, record type routing, line-item population, and amount-field interpretation automatically.

## Testing

During the private testing phase:

1. Install the skill into a **non-production Kugamon sandbox** first.
2. Report bugs, unexpected behavior, or wording issues by opening an issue on this repo.
3. Once the skill passes validation against all four deployment modes, the repository will be made public.

## Contributing

The repository is currently private and not accepting external contributions. Internal contributors: branch from `main`, open a PR, and request review from @kuldiph.

## License

Copyright © 2026 Kugamon LLC. All rights reserved.
See [LICENSE](./LICENSE) for details.

## Support

Questions or issues? Contact [support@kugamon.com](mailto:support@kugamon.com).
