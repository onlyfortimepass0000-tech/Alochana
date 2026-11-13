# Reverse engineering LISTKIT and INSTANTLY: findings

Date: 13 Nov 2026. Method: public pages only (marketing sites, app shells, public API docs, help centres). No login, no private endpoints. Prices and limits are as published on that date.

## 1. Instantly (instantly.ai)

### What it is
A cold-email sending platform that has grown into a full outbound suite: sending and warmup, lead database (SuperSearch, "450M+ B2B leads"), email verification, inbox-placement tests, CRM with calls/SMS, website-visitor ID, automations, and AI agents (AI SDR, reply agent, Copilot, deliverability agent).

### Pricing (monthly)
| Product | Plan | Price | Limits |
|---|---|---|---|
| Outreach | Growth | $47 | 1,000 uploaded contacts, 5,000 emails/mo |
| Outreach | Hypergrowth | $107 (card also shows $358 in bundle view) | 25,000 contacts, 125,000 emails/mo |
| Outreach | Light Speed | $358 | 100,000 contacts, 500,000 emails/mo, SISR private IPs |
| Credits | Growth / Supersonic / Hyper | $47 / $97 / from $197 | 1,500-2,000 / 5,000-7,500 / 10K-200K credits |
| Bundles | Starter / Scale / Agency | $94 / $194 / $555 | sending plan + credits together |

- Every plan: unlimited email accounts and unlimited warmup. The meter is contacts and emails per month, not inboxes.
- Yearly billing saves about 10%.

### Tech stack (seen in public HTML)
- App `app.instantly.ai`: Next.js (pages router, static export), React, Material UI (JSS), 6 locales, 121 routes in the public build manifest.
- Marketing site: Webflow. Docs: Mintlify. Help centre: Intercom.
- Vendors: Intercom chat, a custom chat widget hosted on Render, VWO (A/B tests), Hotjar, Plausible, Google Tag Manager, Facebook Pixel, ProfitWell, Stripe OAuth.

### App map (from the route manifest)
accounts, campaigns (analytics, leads, options, schedule, sequences, subsequences), unibox, lead-finder, verification (single, bulk, list), inbox-placement, crm (leads, opportunities, calls, sms, tasks, reports, website-visitors), automations, ai-agents, ai-sales-agent, copilot (memory, tasks), settings (advanced-deliverability, blocklist, integrations), accelerator, plus `/client/login` (agency client portal) and `/privacy/*` (opt-out, report abuse).

### API v2
- Base `https://api.instantly.ai/api/v2`, Bearer API key with per-resource scopes (`campaigns:read`, `all:all`, ...).
- Rate limit: 100 requests/second and 6,000/minute, shared across the whole workspace.
- 269 documented endpoints in 37 groups. Biggest: campaign (19), AI sales agent (19), SuperSearch enrichment (18), account (16), AI inbox manager (16), lead (13), subsequence (11), email (9), blocklist (9), webhook (8), DFY email account orders (7).
- Long jobs (for example enable/disable warmup) run as background jobs that you poll.
- Webhook events: `email_sent`, `email_opened`, `reply_received`, `auto_reply_received`, `link_clicked`, `email_bounced`, `lead_unsubscribed`, `account_error`, `campaign_completed`, `lead_interested`, `lead_not_interested`, `lead_neutral`, plus meeting events.

### Data model (how a campaign is built)
- Campaign: `campaign_schedule` (schedules with days, from/to time, timezone), `sequences` -> `steps` (each with `delay`, `delay_unit`) -> `variants` (subject, body, `v_disabled`). That is the A/Z test.
- Sending controls: `daily_limit`, `daily_max_leads`, `email_gap`, `random_wait_max`, `stop_on_reply`, `stop_on_auto_reply`, `stop_for_company`, `text_only`, `first_email_text_only`, `link_tracking`, `open_tracking`, `prioritize_new_leads`, `match_lead_esp`, `allow_risky_contacts`, `disable_bounce_protect`, `insert_unsubscribe_header`, `auto_variant_select`, `provider_routing_rules` (`recipient_esp`, `sender_esp`), `is_evergreen`.
- Senders are attached by `email_list` or `email_tag_list`. Rotation = the campaign spreads sends over those inboxes.
- Account: `daily_limit`, `sending_gap` (minutes), `tracking_domain_name`, `enable_slow_ramp`, and a `warmup` block: `limit`, `increment` (disabled or 0-4 per day), `reply_rate`, `open_rate`, `important_rate`, `spam_save_rate`, `read_emulation`, `weekday_only`, `warmup_custom_ftag`.
- Warmup pools: a standard pool and a premium/private pool (help-centre article titles).

### What is hard to copy vs easy
- Hard: the warmup network (needs thousands of real inboxes), the private IP rotation (SISR), the lead database, and deliverability know-how.
- Easy: the sequencer itself (schedule + steps + variants + per-inbox limits) and the unibox UI. The API and field names above are effectively a spec.

## 2. ListKit (listkit.io)

### What it is now
ListKit started as a lead database. The public site now sells it as an "all in one cold email platform": leads, domains, inboxes, warmup, AI copy and sending in one price. Claims: 977M+ triple-verified contacts, 100M+ emails sent.

### Pricing
- One product: the Bundle, $597/month per 1,000 emails/day. Bundles stack (2 = $1,194, 5 = $2,985, 10 = $5,970, max 50). 6-month billing saves 10%, annual 15%. No free trial.
- One Bundle = 20 domains, 40 Google Workspace inboxes (2 per domain, 25 emails/day each), 10,000 credits/month, 20,000 stored contacts, AI Script Writer, one onboarding call, course and community.
- Credits: 1 per triple-verified business email, 3 per US mobile found, 0.1 per email verified from an outside list, 100 per AI Script Writer run. AI spintax is free. Top-up: 2,000 credits for $47. Credits expire after 60 days.

### How it works
- Verification happens at the moment of export ("waterfall verification"), not from a cached result. Failed emails are dropped and not charged. Results are Good, Risky (mostly catch-all) or Bad.
- Inboxes: pre-warmed (.info domains from a shared pool, ready in minutes) or branded (.com, 72 hours setup + 14 days warmup). Every inbox then ramps 1, 5, 9, 13, 17, 21, 25 emails/day over 7 sending days.
- Search filters: job title, seniority, department, company size, revenue, technology, location (company vs person), buying intent topics, funding, hiring, industry / vertical / micro-industry / SIC, keywords, AI company score, business model, growth stage. Plus AI Search in plain English.

### The key finding: the sending engine is SmartLead
ListKit's own developer docs say its webhooks exist "because SmartLead - which actually generates most of these events - only supports one webhook destination", and webhook endpoints carry a `smartlead_campaign_id`. So ListKit's "Email Engine" is SmartLead underneath, resold with domains, inboxes, data and AI copy on top. The docs also mention order fulfilment moving from an "old GCP pipeline" to a "v4 (Cloudflare) fulfillment engine".

### Tech stack
- App `next.listkit.io`: Vite + React single-page app, Clerk for login, TanStack Query, Google sign-in.
- Marketing: Sanity CMS. Help: Intercom. Booking: Chili Piper. Affiliates: FirstPromoter. Community/checkout: Whop. Video: Wistia.

### API and MCP
- Public REST API at `developer.listkit.io`, `/v1/...`, Bearer key `lk_live_...`, up to 25 keys per account, OpenAPI spec published.
- Resources: account, billing, campaigns (sequence, schedule, senders, AI writer, AI spintax, analytics), enrichment, exclusions, inbox, keys, leads, mailboxes and domains (including the pre-warmed pool), orders, search (people, companies, counts, insights, intent topics, job-title expansion), verifications, webhooks, workflows.
- Rate limits are token buckets per account by class: read, write, search, spend, chat, insights.
- A remote MCP server at `developer.listkit.io/mcp` exposes the same actions to Claude/ChatGPT/Cursor. Payments and mailbox passwords are deliberately kept out of MCP.

## 3. Side by side
| | Instantly | ListKit |
|---|---|---|
| Core | Own sending + warmup engine | Reseller bundle on SmartLead |
| Entry price | $47/mo (bring your own inboxes) | $597/mo (inboxes and domains included) |
| Meter | Contacts + emails per month | Emails per day (bundles) + credits |
| Inboxes | Unlimited, you supply or buy DFY | 40 per bundle, managed for you |
| Leads | SuperSearch credits, separate plan | 10,000 credits included |
| API | 269 endpoints, webhooks | Smaller REST API + MCP server |
| Buyer | Agencies and technical senders | Non-technical business owners |

## 4. Fit for us
1. The ListKit model is a packaging business: SmartLead (or Instantly) for sending + bought domains/inboxes + a data provider + AI copy + onboarding. The margin comes from bundling and hand-holding, not from new technology. An agency could offer the same "done for you outbound" bundle to Indian B2B clients on top of Instantly's $47 plan.
2. Both expose APIs, and ListKit has an MCP server, so Claude can run lead search, campaign creation and reply handling directly. That links with the REA and Remotion work: one agent workflow can find leads, write copy, and attach a personalised video.
3. If we ever build our own sequencer, Instantly's campaign and account fields above are a ready blueprint for the data model.

## 5. Not verified
- Instantly's real warmup pool size and how the pool pairs inboxes (only setting names were visible).
- Where either company's lead data comes from.
- Anything behind login in either app.
