# sabren/cornerops-svn

> The Cornerhost-era hosting operation in code: a web-hosting control
> panel (`cornerhost`) and a subscription-billing system (`duckbill`),
> imported from Subversion into git — Python 2, MySQL, and the sabren
> web stack, with tests. The DNS feature is the part still in live
> use (see below).
> Repo: https://github.com/sabren/cornerops-svn — 208 files, git
> history in this clone 2008-05-25 → 2011-05-15 (CVS `$Id$` tags
> inside files reach back to 2006). **Keep.**

## What it is

`cornerops-svn` is an SVN-layout import (`trunk/`/`branches/`) of
two related Python 2 applications from the Cornerhost web-hosting
business. The name is descriptive: "cornerops" (Cornerhost
operations) as it existed in Subversion.

### cornerhost/ — the hosting control panel

The main app: a control panel for a shared-hosting business, with an
admin interface and a customer panel, built on the sabren stack
(`strongbox` models, `clerks`/`storage` MySQL layer, `weblib`
`.zb`/`.app` templates — the same stack archived in
`sabren/sixthdev` under `@gone/`).

What it manages, from its schema (`cornerhost/schema/`):

- **Domains and DNS** — `Domain.py`, `DNSRec.py`: domains,
  subdomains, and DNS records. The DNS record schema allows exactly
  two record types: `rectype` must be `"TXT"` or `"MX"` (with
  `priority` for MX), and TXT values are auto-quoted. There is no
  A/AAAA/CNAME record type in this schema — the panel's DNS is a
  mail-focused subset, not a general zone editor.
- **Sites** — `Site.py`: docroots and site configuration.
- **Mail** — `Mailbox.py`, `EmailRule.py`: mailboxes and mail
  rules (including catchall forms in the skin templates).
- **Databases, plans, servers, users, signups, usage** —
  `Database.py`, `Plan.py`, `Server.py`, `User.py`, `Signup.py`,
  `Usage.py`, `Trail.py` (audit trail).
- **Features** (`cornerhost/features/`): `dns.py`, `email.py`,
  `site.py`, `user.py`, `panel.py`, `remote.py`, plus a `mock.py`
  for testing. The DNS feature's commands are `SaveDNSRecCommand`,
  `DeleteDNSRecCommand`, `CreateDomainCommand`, `DeleteDomainCommand`
  — with safety checks (`safety.safeDomain`, `safeDNSRec`) and a
  note that a domain cannot be deleted while it is a site or has
  subdomains.
- **Grunt work** (`cornerhost/grunts/`, `*Grunt.py`): `NewSiteGrunt`
  (provisioning), `UserClerk`, `BandwidthGrunt`, `LimitChart`,
  `MailerApp`, `AdminApp`, `UserApp`.
- **Templates** (`cornerhost/skin/`): 28 `.gen` templates covering
  domains, subdomains, DNS (including `frm_dns_mx.gen` and
  `frm_dns_txt.gen`), mailboxes, email rules, databases, servers,
  signups, sites, traffic, and cron.
- **Tests** (`cornerhost/test/`): 22 test files — `DNSFeaturesTest`,
  `DomainTest`, `EmailFeaturesTest`, `SiteFeaturesTest`,
  `UserAppTest`, `BandwidthGruntTest`, `ControlPanelFeatureTest`,
  and more, with a `personas.py` fixture module.

### duckbill/ — subscription billing

The companion billing app: accounts, subscriptions, receivables,
statements, and events, with credit cards encrypted via GnuPG
(`EncryptedCard.py`, `GnuPG.py`; `README-GPG.TXT` documents the
key-generation setup for tests, including a hard-coded test
passphrase). `ducktools/` holds the operational scripts: `autobill`,
`runcollections`, `unearned`, `catchup`, `copycard`, `cardinfo`.
Present twice: `duckbill/trunk/` and `duckbill/branches/genshi-port/`
(a port of the frontend templates to Genshi — the two trees are
near-identical apart from the template layer).

## Tech stack

Python 2 (`except (ValueError, TypeError), e` syntax; `long` type
in the schema), MySQL via the `clerks`/`storage` layer, weblib
`.zb`/`.app` templates, and the sabren libraries (`strongbox`,
`platonic`). GnuPG via `GnuPGInterface` for card encryption. The
code predates the git era: CVS `$Id$` tags (e.g.
`DNSRec.py,v 1.3 2006/07/02`) and SVN `trunk/`/`branches/`
layout throughout.

## What is still in live use — and what is not claimed

The task brief for this doc states he still uses this repo for at
least his DNS. What the repo itself verifies:

- **DNS is a first-class feature, not an afterthought:** a dedicated
  `features/dns.py` command module, a `DNSRec` schema model, DNS
  skin templates (`dns.gen`, `frm_dns_mx.gen`, `frm_dns_txt.gen`),
  and a `DNSFeaturesTest` suite. If DNS is the live use, this is
  the surface that matters — and its limits are visible: MX and TXT
  records only, no A/AAAA/CNAME in the schema.
- **Everything else is unverified as a live use.** The repo also
  covers sites, mailboxes, databases, plans, signups, usage
  tracking, and the whole duckbill billing system — but nothing in
  the repo says which of those, if any, are still operated today.
  No claim is made here about them.

## Modernization / MCP design proposal

This section is a proposal, not a repo fact. The goal: keep the one
live use (DNS) working without keeping the whole Python 2 panel
alive, and make it operable by an agent via MCP.

**Principles:**

1. **Shrink the surface to DNS first.** The live need, as far as
   the repo verifies it, is DNS records (MX/TXT) on domains. A
   modernization that tries to revive the whole panel (sites, mail,
   billing, duckbill) is a rewrite of a dead business, not a
   maintenance task. Phase 1 covers DNS only; everything else stays
   archived.
2. **Do not port the panel — extract the data.** The DNS data
   lives in MySQL behind the `clerks`/`storage` layer. The smallest
   correct move is a read of the live DNS table(s) to confirm the
   actual current schema (the repo's `DNSRec` may predate later
   changes), then a thin service over that data — not a port of
   `cornerhost` to Python 3.
3. **Put a real DNS API in front of it.** Options, in order of
   increasing work: (a) if the domains' DNS already lives at a
   registrar/DNS provider with an API, manage records there and treat
   the panel's DNS table as legacy read-only reference; (b) if the
   panel's table is still authoritative, stand up a minimal service
   that reads/writes that table and regenerates the zone/config the
   way `genbindconf`-era tooling did (the sei-bin scripts in
   `sabren/sei-bin` include `genbindconf` — the config-generation
   pattern already exists in the family). Which is true is
   unverified from the repo and must be checked against the live
   setup before any code is written.

**Proposed MCP server ("cornerops-dns"):**

- **Tools** (each a thin wrapper over the DNS store, never over the
  panel UI):
  - `dns_list_domains` — domains under management.
  - `dns_list_records(domain)` — MX/TXT records for a domain, with
    IDs, priorities, values.
  - `dns_add_record(domain, type, value, priority?)` — type
    restricted to MX/TXT to match the verified schema; adding
    A/AAAA/CNAME requires a schema decision first, since the repo's
    schema does not allow them.
  - `dns_delete_record(domain, id)` — with the repo's own safety
    rule preserved: refuse to delete a domain's records while it is
    a live site (the `is_site` guard in `DeleteDomainCommand`).
  - `dns_diff(domain)` — repo-table state vs. live published DNS,
    so drift is visible before any write.
- **Safety rules (carried over from the repo, not invented):**
  - TXT values auto-quoted (the `DNSRec.get_value` rule).
  - MX values validated as hostnames (the `reDomain` check in
    `SaveDNSRecCommand`).
  - Every write logged (the `Trail` audit model exists for a
    reason; the MCP server should keep an equivalent append-only
    log).
  - Read-only by default: a `CORNEROPS_DNS_READONLY` mode for
    inspection, with writes requiring an explicit flag — DNS
    mistakes are public and cached.
- **What not to build:** no resurrection of the admin/customer panel
  UI, no duckbill revival, no port of `strongbox`/`weblib` to
  Python 3. If a broader need emerges (sites, mail), that is a
  separate proposal with its own verification step.

**Suggested sequencing:**

1. Verify live state: which DNS store is authoritative today, and
   whether its schema still matches `DNSRec` (MX/TXT only).
2. Build the MCP server read-only first (`list` + `diff` tools);
   use it for a few weeks of DNS lookups before enabling writes.
3. Enable writes behind the readonly flag flip, with the Trail-style
   audit log from day one.
4. Leave the rest of `cornerops-svn` archived and untouched.

## Inventory (highlights)

| Path | Contents |
|------|----------|
| `cornerhost/trunk/cornerhost/schema/` | Domain, DNSRec, Site, Mailbox, EmailRule, Database, Plan, Server, User, Signup, Usage, Trail |
| `cornerhost/trunk/cornerhost/features/` | dns, email, site, user, panel, remote, mock |
| `cornerhost/trunk/cornerhost/grunts/` | NewSiteGrunt, UserClerk, BandwidthGrunt, LimitChart |
| `cornerhost/trunk/cornerhost/skin/` | 28 templates (domains, DNS MX/TXT forms, mail, sites, traffic) |
| `cornerhost/trunk/cornerhost/test/` | 22 test files incl. DNSFeaturesTest |
| `cornerhost/trunk/cornerhost/admin/`, `panel/` | Admin and customer panel apps (`.app`, `.zb`, CSS) |
| `duckbill/trunk/` | Billing: accounts, subscriptions, receivables, GPG-encrypted cards, ducktools |
| `duckbill/branches/genshi-port/` | Same billing app with Genshi templates |

## Surprising / notable

- **A complete hosting business in one repo:** panel + billing +
  tests, from an era when a small hoster wrote all of it
  themselves. The `Trail` audit model and the `safety` module
  show it was operated carefully, not hacked together.
- **The DNS subset is the tell:** a panel that only models MX and
  TXT records is a panel built for a mail-centric hoster — the
  schema is a fossil of the business model.
- **duckbill's GPG card handling** (encrypted cards, test keys with
  hard-coded passphrases, `copycard`/`cardinfo` tools) is a
  reminder of pre-PCI-tooling payment handling — archive-only, and
  nothing in it should be revived.
- No linked streams or videos found in the repo.

## Verdict

Keep. The DNS feature is the live thread — preserve the repo as
the reference for what the DNS data model was, and pursue the
narrow MCP proposal above rather than any broad revival. The rest
(panel, billing) is business history, worth keeping as-is.

(Flagged as unverified: which DNS store is authoritative today,
whether the live schema still matches `DNSRec`, and whether any
cornerhost/duckbill function beyond DNS is still operated — none
of these are stated in the repo.)
