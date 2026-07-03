# kugamon-skills

A Claude Desktop / Cowork **plugin marketplace** that ships a single plugin (`kugamon`) for agentic **Quote-to-Cash** and **Subscription Management** in Salesforce orgs running the Kugamon managed packages.

This repo **does not install an MCP server**. It assumes you already have a Salesforce MCP server connected to a Kugamon org. The plugin's job is to teach Claude the Kugamon object model, lifecycle flows, record type routing, and amount-field semantics so it can run Quote-to-Cash operations end to end.

> **Status: Beta.** Validate against a non-production Kugamon sandbox before using in production orgs.

## Why this plugin

Out of the box, Claude can query and update Salesforce records — but it doesn't know:

- **The Kugamon object model** — how opportunities, quotes, orders, order releases, invoices, payments, shipments, contracts, subscriptions, renewal opportunities, and assets relate
- **Which deployment mode the org runs** (CPQ, Q2C, SubMgmt, or SB) and how the lifecycle differs in each
- **Record type routing** for New, Expansion, and Renewal transactions
- **Amount-field semantics** — when to read `kuga_sub__Amount__c` vs. the standard `Amount` (which may show MRR or prorated values)
- **Line-item population rules** so quotes and orders come out complete

The plugin encodes these rules as a Cowork skill. It **auto-detects** which Kugamon packages are installed (by checking for the `kuga_sub__Renew__c` field on `OpportunityLineItem`) and adapts the workflow accordingly.

## Deployment modes

| Mode | Flow |
|------|------|
| **CPQ** | Opportunity → Quote → Order → (Order Release) → Asset |
| **Q2C** (Quote to Cash) | Opportunity → Quote → Order → (Order Release) → Asset + Shipment → Invoice → Payment |
| **SubMgmt** (Subscription Management) | Opportunity → Quote → Order → (Order Release) → Asset + Contract + Subscription + Renewal Opportunity |
| **SB** (Subscription Billing) | Opportunity → Quote → Order → (Order Release) → Asset + Shipment + Contract + Subscription + Renewal Opportunity → Invoice → Payment |

## Prerequisites

1. **A Salesforce MCP server** connected to the target org — e.g. [salesforce-mcp-auto-auth-chrome](https://github.com/kugamon/salesforce-mcp-auto-auth-chrome). The skill assumes the agent can run SOQL, DML, and Apex against the org.
2. **[salesforce-core-skills](https://github.com/kugamon/salesforce-core-skills)** — install this marketplace first. It provides the general Salesforce skills (sf-data for SOQL/DML, sf-metadata, sf-apex, sf-flow, …) that this plugin builds on for org-level operations, while the `kugamon` plugin adds the Kugamon-specific lifecycle knowledge on top.
3. **A Salesforce org with the Kugamon managed packages installed:**
   - **Kugamon Quote to Cash** (namespace: `kugo2p`) — required
   - **Kugamon Subscription Management** (namespace: `kuga_sub`) — optional

## Repo layout

```
kugamon-skills/                          # repo root = a marketplace
├── .claude-plugin/
│   └── marketplace.json                 # marketplace manifest (lists 1 plugin)
├── README.md                            # you are here
├── CHANGELOG.md
├── LICENSE                              # proprietary
└── plugins/
    └── kugamon/                         # the plugin itself
        ├── .claude-plugin/
        │   └── plugin.json              # plugin manifest
        └── skills/
            └── kugamon-full-qtc-submgmt/
                └── SKILL.md             # the Quote-to-Cash lifecycle skill
```

The marketplace pattern means future plugins (e.g. Kugamon analytics, provisioning) can be added under `plugins/<name>/` and registered in `marketplace.json` — the install URL stays the same.

## Install

### Option 1 — Cowork "Add marketplace" (recommended)

1. Open Claude Desktop → **Customize** → **Marketplace**.
2. Click **+ Add marketplace** (sometimes labeled **Sync from URL**).
3. URL: `kugamon/kugamon-skills` — or the full URL `https://github.com/kugamon/kugamon-skills`.
4. Click **Sync**. You'll see one plugin: `kugamon`.
5. Click **Install**.
6. Restart Claude (Cmd+Q + reopen on macOS) so the skill loads into the system prompt.

### Option 2 — Local folder

1. Clone the repo or download the source.
2. In Claude Desktop → **Customize** → **Personal plugins** → **+** → **Local folder**.
3. Pick `plugins/kugamon/` (the directory that contains `.claude-plugin/plugin.json`).
4. Toggle on. Restart Claude.

### Option 3 — Manually pin to settings.json

```json
{
  "extraKnownMarketplaces": [
    { "url": "https://github.com/kugamon/kugamon-skills" }
  ],
  "enabledPlugins": ["kugamon"]
}
```

## Verify

After installing and restarting, test the skill:

> "Using my Kugamon sandbox, what deployment mode is this org running?"

Claude should check for the `kuga_sub__Renew__c` field on `OpportunityLineItem` and report CPQ/Q2C vs. SubMgmt/SB.

## Sample prompts

- "Create a new quote in my Kugamon sandbox for Acme Corp for 50 user licenses."
- "Convert quote SQ-260519-0020461 into an order."
- "Generate an invoice for order SO-260521-0113570 and record a payment."
- "Show me all open renewal opportunities closing this quarter."

The skill handles package detection, record type routing, line-item population, and amount-field interpretation automatically.

## Troubleshooting

**"This repository isn't a marketplace — no manifest found at .claude-plugin/marketplace.json".** Make sure you're on `main` — the manifest lives at the repo root.

**Plugin loads but the skill doesn't trigger.** Restart Claude Desktop fully (Cmd+Q on macOS). Skills are loaded at session start.

**Wrong amounts on opportunities or quotes.** The skill reads `kuga_sub__Amount__c` for full subscription/contract value; the standard `Amount` field may show MRR or prorated values. If numbers look off, confirm which field you're comparing against.

**Objects missing (contracts, subscriptions).** The org likely runs CPQ or Q2C mode without the `kuga_sub` package — those objects only exist in SubMgmt/SB modes.

## Testing (Beta)

1. Use the skill with a **non-production Kugamon sandbox** first.
2. Report bugs, unexpected behavior, or wording issues by opening an issue on this repo.
3. The repository will be made public once the skill passes validation against all four deployment modes.

## Contributing

Internal contributors: branch from `main`, open a PR, and request review from @kuldiph. External contributions are not being accepted while the repo is private.

## License

Copyright © 2026 Kugamon LLC. **All rights reserved.** See [LICENSE](./LICENSE) for details.

The Kugamon managed packages are commercial products of Kugamon LLC; use of the packages is subject to their own terms.

## Support

Questions or issues? Contact [support@kugamon.com](mailto:support@kugamon.com).
