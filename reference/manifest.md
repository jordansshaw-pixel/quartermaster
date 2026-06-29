# Manifest — the Quartermaster's data model

The single source of field names, allowed values, and freshness windows — the one
reference file the charter points to. Every registry record, rule, and example
refers back to this file; change a name here and you change it everywhere. The
**Record format** section below doubles as the blank template for a new record.

## ID convention

Every record has a stable ID: `<entity>-<slug>` — e.g. `lane-foundco`,
`proj-portal`, `asset-foundco-domain`, `acct-cf-personal`, `sub-cloudflare`.

IDs are permanent. Renaming a display name never changes an ID. Cross-links (one
record pointing at another) always use the ID.

## The five entities

Every record, in every entity, carries **`status`** and **`last_confirmed`**.
No undated facts.

### lane — the owner axis (who a thing belongs to)
| field | values / notes |
|---|---|
| `id` | `lane-<slug>` |
| `name` | display name |
| `type` | `personal` \| `client` \| `business` |
| `status` | `active` \| `dormant` \| `wound-down` |
| `notes` | free text |
| `last_confirmed` | date — required |

### project — a unit of work, owned by a lane
| field | values / notes |
|---|---|
| `id` | `proj-<slug>` |
| `name` | display name |
| `lane_id` | → lane (owner) |
| `status` | `active` \| `paused` \| `shipped` \| `abandoned` \| `unknown` |
| `summary` | what it is |
| `last_confirmed` | date — required |

### asset — a thing that runs (the two-axis record)
| field | values / notes |
|---|---|
| `id` | `asset-<slug>` |
| `name` | display name |
| `type` | `domain` \| `dns` \| `registrar` \| `repo` \| `machine` \| `saas` \| `portal` \| `api-key` \| `mailbox` \| `other` |
| `owner_lane_id` | → lane — **OWNS** (which lane it belongs to) |
| `controller_account_id` | → account — **CONTROLLED BY** (which login holds the keys) |
| `project_id` | → project (optional — what it serves) |
| `url` / `location` | where it lives |
| `status` | `live` \| `unconfirmed` \| `lost-access` \| `dead` \| `unknown` |
| `secret_pointer` | where the credential lives — **never the secret** |
| `expiry` | date (optional — e.g. domain expiry; drives a hard trigger) |
| `last_confirmed` | date — required |
| `notes` | free text |

Ownership and control are **two separate links**. A personal login controlling
a business-owned asset is a real, flaggable state — never collapsed into one
field.

### account — the controller axis (a login identity)
| field | values / notes |
|---|---|
| `id` | `acct-<slug>` |
| `login_email` | the email / identity behind it |
| `provider` | e.g. Cloudflare, GitHub, Vercel, Google |
| `home_lane_id` | → lane the login is *meant* to represent (may differ from what it controls) |
| `secret_pointer` | where password / 2FA lives — **never the secret** |
| `recovery_pointer` | where recovery codes live — **never the codes** |
| `status` | `active` \| `lost` (access gone, account still exists → 🔒 LOST) \| `closed` (deliberately deleted/closed → ⏹️ DEAD) \| `shared` \| `unknown` |
| `last_confirmed` | date — required |
| `notes` | free text |

### subscription — a recurring charge
| field | values / notes |
|---|---|
| `id` | `sub-<slug>` |
| `name` | what it pays for |
| `funds_ref` | → asset or account it funds — **the only place the asset↔cost link lives**; find an asset's cost by reverse lookup, never duplicate it onto the asset |
| `lane_id` | → lane that should bear the cost (billed-to) |
| `amount` + `currency` | e.g. 20 USD |
| `cadence` | `monthly` \| `annual` \| `usage` \| `one-time` |
| `next_charge` | date — volatile; a past-due date is a hard trigger |
| `payment_source` | pointer, not a full card number — e.g. "Amex …1234", "App Store: personal Apple ID" |
| `billing_inbox` | which email receives receipts |
| `status` | `active` \| `trial` \| `canceled` \| `unknown` |
| `last_confirmed` | date — required |
| `notes` | free text |

## Confidence states

Derived at read time from `status` + `last_confirmed` versus the freshness
window. **Never stored** — always computed when a record is read.

| State | Meaning | Treatment |
|---|---|---|
| ✅ CONFIRMED | value present, confirmed within its window | shown as live |
| ⚠️ UNCONFIRMED | value present but past its window | **never** shown as live; labeled "unconfirmed as of \<date\>"; offered for re-check |
| ❔ UNKNOWN | never captured — a known gap | surfaced as an open question; **never guessed** |
| 🔒 LOST | access / credential known gone | surfaced with its `recovery_pointer` |
| ⏹️ DEAD | decommissioned / canceled / closed | kept for history; excluded from "live" views |

## Freshness windows

How long a confirmation stays good, by volatility. Tunable — change a number
here and the whole system re-derives.

| Entity / field | Window | Notes |
|---|---|---|
| `subscription` record | re-confirm each billing cycle | — |
| `subscription.next_charge` | — | **hard trigger** when past due |
| `project.status` | 60 days | work moves fast |
| `account` | 120 days | login validity |
| `asset` | 180 days | semi-stable |
| `asset.expiry` (e.g. domain) | — | **hard trigger** within 45 days |
| `lane` | 180 days | — |

A **hard trigger** surfaces every session until resolved, regardless of when the
record was last confirmed.

## Record format

Per-record **stanzas**, not wide tables — diff-friendly, room for status + date
+ notes. The header carries the ID and a derived confidence badge:

```
### asset-foundco-domain   [⚠️ UNCONFIRMED]
type: domain          owner_lane: lane-foundco        controller_account: acct-cf-personal
status: live          secret_pointer: 1Password → Founder vault
expiry: 2026-11-02    last_confirmed: 2026-01-15
notes: renews annually; controller is a PERSONAL login on a business asset → mismatch flag
```

The badge (✅/⚠️/❔/🔒/⏹️) is display only — computed from `status` +
`last_confirmed`, never written into the record.
