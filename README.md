# gtm-persist-original-source-landing-fbclid

A **Google Tag Manager (GTM) container export** (JSON) — not a single tag template — that you import directly into a GTM workspace to persist three pieces of acquisition context as first-party cookies: the **site referral source** (where the visitor came from), the **site landing page** (the first page they hit), and the **`fbclid`** URL parameter (Facebook's click identifier). It follows the same `Cookie Creator`/`Get Root Domain` pattern used across this author's other GTM container exports, and adds a URL-parsing layer on top for the landing-page data.

This README is based on a direct inspection of the container export's actual contents (9 tags, 9 triggers, 19 variables, 4 folders, 5 built-in variables, and 3 bundled custom templates), cross-checked against the repository's own (unusually detailed) description.

## What problem does this solve?

Marketers often populate `utm_source` inconsistently or with placeholder/garbage values, which makes `utm_source` alone an unreliable signal for where traffic actually came from. This container captures two more objective signals as a cross-check: the **`document.referrer`** the browser itself reports (the actual referring site, independent of anything a marketer typed into a UTM tag), and the **landing page URL** the visitor first arrived on (useful for multi-domain/multi-subdomain sites where knowing *which* subdomain or path started the journey matters, especially since many subdomains just resolve to `/`). It additionally demonstrates the same persistence pattern applied to `fbclid`, Facebook's click-identifier query parameter, as a template for capturing **any** other single URL parameter the same way.

## The three things this container persists

| Concept | GTM folder | Cookie names | Source |
|---|---|---|---|
| Site referral source | `Site Referral Source Cookie Creator` | `_cc_site_referral_src`, `_cc_1st_site_referral_src` | GTM's built-in Referrer variable, Host component only, with `www.` stripped |
| Site landing page | `Site Landing Page Cookie Creator` | `_cc_site_landing_page`, `_cc_1st_site_landing_page` | GTM's built-in URL variable (full page URL) |
| `fbclid` | `fbclid Cookie Creator` | `_cc_fbclid`, `_cc_1st_fbclid` | The `fbclid` URL query parameter |

A fourth folder, **`Analytics`**, holds shared/general-purpose variables (`Root Domain`, `Shared Event Settings`) rather than its own cookie tags.

## How the tag/trigger logic works

Each of the three concepts uses the now-familiar **three-tag pattern** seen across this author's containers — `Create <X>` (session, current/last value), `Create 1st <X>` (persistent, first-touch, write-once), `Rewrite 1st <X>` (renews the persistent cookie's expiration without changing its value) — but with an important behavioral difference depending on the concept:

### Referral source & landing page: session cookie is also write-once
For **Site Referral Source** and **Site Landing Page**, even the plain **session** `Create` tag only fires when its own session cookie **isn't already set** — i.e., it captures the value exactly once per browser session (the first pageview) and never overwrites it as the visitor navigates further. This makes sense semantically: "referral source" and "landing page" are properties of how a *session* began, not something that should change mid-session as the visitor clicks around your own site (where `document.referrer` would otherwise just become your own domain, and the "landing page" would become whatever page they're currently on).

### `fbclid`: session cookie updates on every matching page
By contrast, the **`Create fbclid`** session tag fires **every time** `fbclid` appears in the URL (gated only by "the URL contains `?`" and "the URL contains `fbclid`," with no check on the existing cookie) — so the session cookie always reflects the *most recent* `fbclid` seen, consistent with the "current/last touch" pattern used for UTM parameters in this author's sibling [UTM-persistence container](https://github.com/drewspen/gtm-persist-utm-url-query-strings-1st-party-cookie).

### The persistent (`1st_`) cookies follow the standard pattern for all three
For all three concepts, `Create 1st <X>` fires only when the `_cc_1st_*` cookie doesn't already exist (capturing first-touch, write-once, 12-month expiry), and `Rewrite 1st <X>` fires whenever that cookie *does* already exist, re-writing it with its own existing value purely to renew its 12-month expiration on each return visit — identical in spirit to the renewal pattern documented in this author's other containers.

All nine triggers are `INIT`-type (evaluated on page load via trigger filter conditions), matching the UTM-persistence container's approach (rather than `PAGEVIEW`, used in the UUID-persistence container).

## Custom templates bundled in this export

Unlike several of this author's other exports (which bundle templates that go unused), **all three** custom templates here are actively used:

| Template | Purpose |
|---|---|
| **Cookie Creator** | Sets cookies via sandboxed JavaScript, avoiding extra CSP `script-src`/hash allowances — used by all 9 tags. |
| **Get Root Domain** | Extracts the registrable root domain from `{{Page Hostname}}`, used as every cookie's `Domain` value (via the `Root Domain` variable). |
| **URL Components Parser** | Extracts a specific component (domain, path, query string, or hash fragment) from a given URL string — used by eight variables that parse the *persisted* landing-page cookie value back into its parts. |

## Variables

### Core capture variables
| Variable | Type | Purpose |
|---|---|---|
| `Site Referral Source Built In Variable` | Built-in Referrer, Host component, `www.` stripped | The raw referral source value written to the referral-source cookies. |
| `Site Landing Page Built In Variable` | Built-in URL | The full landing-page URL written to the landing-page cookies. |
| `fbclid URL Query Value` | URL (Query component) | Reads the raw `fbclid` value from the query string. |
| `Root Domain` | Custom (Get Root Domain) | Root domain from `{{Page Hostname}}`, used as every cookie's `Domain`. |

### Cookie read-back variables
`Site Referral Source Cookie Value`, `Site Referral Source 1st Cookie Value`, `Site Landing Page Cookie Value`, `Site Landing Page 1st Cookie Value`, `fbclid Cookie Value` *(as `fbclid from Cookie`)*, `fbclid 1st Cookie Value` — each reads back its corresponding `_cc_*` cookie, feeding the trigger conditions described above (and, for the landing-page cookies, feeding the parser variables below).

### Landing-page URL component parser variables
Eight variables (four components × current/first-touch) use the **URL Components Parser** template to break the *persisted* landing-page cookie value back into its constituent parts:

- `Site Landing Domain Cookie Value` / `Site Landing Domain 1st Cookie Value`
- `Site Landing Path Cookie Value` / `Site Landing Path 1st Cookie Value`
- `Site Landing Query String Cookie Value` / `Site Landing Query String 1st Cookie Value`
- `Site Landing Hash Cookie Value` / `Site Landing Hash 1st Cookie Value`

None of these eight are consumed by any tag inside this container itself — they exist to make the stored landing-page URL immediately usable elsewhere (e.g., in other tags, or for reporting) without every downstream consumer needing to re-parse the raw URL string. Note the underlying cookie-read variables they build on (`Site Landing Page Cookie Value`, `Site Landing Page 1st Cookie Value`) have `decodeCookie: true`, so percent-encoded characters in the stored URL are decoded before parsing — unlike the `fbclid` cookie-read variables, which use `decodeCookie: false` (appropriate, since `fbclid` values are already URL-safe tokens with no encoding to reverse).

### Analytics scaffold
`Shared Event Settings` is a GA4 **Event Settings Variable** (type `gtes`), matching point 4 of the repository's own description: *"An Analytics folder that has the shared event settings. You'll want to turn the shared event settings into event and user custom definitions in GA4."* As imported, this variable's `eventSettingsTable` and `userProperties` are both **empty** — it's a scaffold, not a working configuration. This container does **not** include a GA4 Configuration or Event tag that references it; you're expected to wire this variable into your own GA4 tags yourself (e.g., referencing the cookie-read-back variables above as custom event/user parameters) after import.

## Cookie configuration

- **Domain:** `{{Root Domain}}` — shared across subdomains.
- **Path:** `/`
- **Expiration:** 12 months for all `_cc_1st_*` persistent cookies; session-only for the plain `_cc_*` cookies.
- **SameSite:** `None`, with the SameSite checkbox enabled, on all nine tags.
- **Secure:** disabled (`checkbox1Secure: false`) on all nine tags — see [Known issue](#known-issue-samesitenone-without-secure) below.
- **Consent:** every tag requires **`analytics_storage`** consent (`consentStatus: NEEDED`) — note this differs from this author's UTM- and UUID-persistence containers, which gate on `functionality_storage` instead. If you're running more than one of this author's containers in the same GTM setup, double-check which consent category each expects, since they aren't uniform.

## Prerequisites

1. **A GTM web container** — ideally a scratch/sandbox workspace, since importing can create duplicate tags/triggers/variables if names collide with what's already in your container.
2. **A consent management setup** wired into GTM's consent mode capable of granting `analytics_storage` consent — all nine tags require it to fire.
3. If you also plan to use this author's [UTM-persistence container](https://github.com/drewspen/gtm-persist-utm-url-query-strings-1st-party-cookie) or [UUID-persistence container](https://github.com/drewspen/gtm-custom-user-uuid-persist-1st-party-cookie), note that all three exports bundle their **own copies of `Cookie Creator` and `Get Root Domain`** — importing more than one into the same container may prompt you to resolve duplicate template conflicts.

## Getting started

### Import into Google Tag Manager

1. In GTM, go to **Admin** → **Import Container**.
2. Choose `gtm_persist_original_source_landing_fbclid.json` from this repository.
3. Select the target container and **choose a new workspace** (recommended) rather than overwriting an existing one, so you can review the merge before publishing.
4. Choose **Merge**, and review the import summary — note this export's account/container IDs (`1315341986` / `219401138`) are **different placeholder values** than this author's other exports (which mostly use `999999999`/`GTM-AAAAAAAA` or `GTM-99999999`) — either way, they're scrubbed and won't match your own container, so GTM will import everything by name.
5. Confirm the import. All three bundled custom templates will import alongside the tags, triggers, and variables that depend on them.

### Post-import checklist

- **Confirm your consent setup** grants `analytics_storage` appropriately, or none of these nine tags will fire.
- **Review the `Secure` cookie setting** (see below) before publishing to a production HTTPS site.
- **Populate `Shared Event Settings`** and wire it into your own GA4 tags if you want the persisted values available as GA4 event/user parameters, per the repository author's own guidance.
- **Test in Preview mode**:
  - Land on your test site from an external referrer with `?fbclid=test123` in the URL, and confirm `_cc_site_referral_src`, `_cc_site_landing_page`, and `_cc_fbclid` (plus their `_cc_1st_*` counterparts) are all set correctly.
  - Navigate to a second page on the same site and confirm the session referral-source and landing-page cookies **do not change**, while a fresh `fbclid` on that second page (if present) **would** update `_cc_fbclid`.
  - Start a new session and confirm the `_cc_1st_*` cookies are **not** overwritten (their values persist from the original first-touch), while their expiration silently renews.

### Using the persisted values

Once set, these cookies (and their corresponding read-back and URL-parsing variables already included in this container) can be read back later in the visit — for example, to populate hidden form fields for a CRM/MAP submission, to add richer acquisition context to GA4 events via the `Shared Event Settings` variable, or to reconcile cases where `utm_source` looks unreliable against the actual browser-reported referrer.

## Known consideration: `SameSite=None` without `Secure`

As with this author's other GTM container exports, all nine cookie tags here set `checkbox1SameSite: true` with `dropDownMenu1SameSite: "None"`, while `checkbox1Secure` is `false`. Modern browsers **require the `Secure` attribute whenever `SameSite=None` is used** — a cookie set this way will typically be **rejected outright**. You'll likely want to enable the Secure checkbox on all nine tags after import so these cookies actually get set in current browsers.

## Notes

- The container is moderate in scope: 9 tags, 9 triggers, 19 variables, 4 folders, 5 built-in variables, and 3 custom templates (all in active use).
- This is an unofficial, personal automation export and is not affiliated with or endorsed by Google or Meta/Facebook. Always review a container export's tags, triggers, and variables — and test thoroughly in a sandbox workspace — before merging it into a production GTM container.

## License

No license file is currently included in this repository. Check with the repository owner before reusing this container export in a commercial or redistributed context.
