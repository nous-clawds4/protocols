> **Repo metadata — not part of the spec text.**
> **Status:** 📝 pre-NIP
> **Canonical:** not yet published
> **Last updated:** 2026-08-17
> **Sources:** `NosFabrica/brainstorm_server` (`app/message_queue_tasks/ta_signing.py`, `app/message_queue_tasks/upload_nostr_events.py`, `app/routers/setup/router.py`, `CONTEXT.md`); `nous-clawds4/tapestry` (`src/algos/customers/nip85/publish_kind30382.js`). Companion to upstream **NIP-85** (Trusted Assertions, kind `10040` — Vitor Pamplona), which this spec does not modify, and to the pre-NIP *Tapestry Assistant Designation & Dual-Author Header Resolution* (`tapestry/protocols/drafts/assistant-designation.md`), which extends the same kind-10040 tag map to DList headers.
> **Adopted by:** `brainstorm_server` (producer), `Brainstorm-UI` (consumer; designation UI), `tapestry` (producer/consumer, extended tag set — see *Implementation variance*).

Trusted Assertions — Consumer Specification
=====

`draft` `optional`

This spec defines how a client reads and interprets **Trusted Assertions (TAs)**: kind-`30382` addressable events in which a trust-score **provider** asserts, from one user's point of view, a personalized web-of-trust score for another user. It specifies the event format, the semantics of each tag, how a consumer discovers the right provider via NIP-85 kind-`10040` designation, and the replacement/deletion lifecycle. It is written for implementers of clients that *consume* trust scores; producer behavior is described only where a consumer must rely on it.

## Terminology

- **Observer** — the user from whose perspective trust is computed. Every score in this spec is relative to exactly one Observer. There is no global score.
- **Observee** — the user a Trusted Assertion is about; the subject being scored.
- **Provider** — the service that computes scores and publishes TAs on the Observer's behalf, signing with a **dedicated per-Observer key** (deployments call it the *assistant* or *TA key*). One signing key corresponds to exactly one Observer's perspective; a provider MUST NOT publish TAs for different Observers under the same signing key.
- **Influence** — the provider's continuous trust score in `[0, 1]` for an Observee, from the Observer's point of view.
- **Rank** — the published integer form of Influence: `round(Influence × 100)`, range 0–100. The wire resolution is therefore 1 rank point = 0.01 Influence.

## The Trusted Assertion event

A TA is a kind-`30382` addressable (parameterized-replaceable) event:

```json
{
  "kind": 30382,
  "pubkey": "<provider signing key, 64-char lowercase hex>",
  "created_at": 1755400000,
  "content": "",
  "tags": [
    ["d", "<observee pubkey, 64-char lowercase hex>"],
    ["rank", "63"],
    ["followers", "188"],
    ["reporters", "0"],
    ["muters", "1"],
    ["hops", "2"]
  ]
}
```

- The event author (`pubkey`) is the provider's per-Observer signing key — **not** the Observer's own key.
- The `d` tag is the Observee's pubkey in hex. Because kind `30382` is addressable, a relay retains at most one TA per `(provider, observee)` coordinate: `30382:<provider-pubkey>:<observee-pubkey>`.
- `content` is empty. Consumers MUST ignore any content.
- Consumers MUST ignore tags they do not recognize; producers MAY add implementation-specific tags (see *Implementation variance*).

### Tag semantics

| Tag | Required | Value | Meaning |
|---|---|---|---|
| `d` | yes | hex pubkey | The Observee. |
| `rank` | yes | integer `0`–`100` as a string | `round(Influence × 100)` from the Observer's point of view. |
| `followers` | recommended | non-negative integer | Count of accounts **following** the Observee that themselves clear the provider's trust threshold in this Observer's web of trust ("verified followers"). |
| `reporters` | recommended | non-negative integer | Verified count of accounts **reporting** (NIP-56, user-level `p`-tag reports) the Observee. A negative signal. |
| `muters` | recommended | non-negative integer | Verified count of accounts **muting** (NIP-51 mute list) the Observee. A negative signal. |
| `hops` | optional | non-negative integer | Follow-path distance from the Observer to the Observee: `0` = the Observer themself, `1` = directly followed, and so on. **When absent, the provider found no follow path** within its traversal limit. |

Two rules deserve emphasis:

1. **An absent `hops` tag means "no known path" — it MUST NOT be read as `hops = 0`.** `0` is the Observer's own record; absence is unreachability. Producers omit the tag rather than publishing a sentinel.
2. **Presence is not endorsement.** Some producers publish TAs for Observees with strong *negative* signal (high `muters`/`reporters`) even when `rank` is 0 or near 0. Consumers MUST read the `rank` value and the negative-signal counts rather than treating the existence of a TA as a mark of trust.

The `followers`/`reporters`/`muters` counts are provider-computed claims. The underlying assertions (kind-3 follows, kind-10000 mutes, kind-1984 reports) are public nostr events, so a consumer can audit a count in principle; the "verified" filter (which raters themselves count) depends on the provider's threshold configuration and is not independently reproducible from the event alone.

## Provider designation and discovery (NIP-85, kind 10040)

Publishing kind `30382` is permissionless: any key can publish assertions about any subject, and the subject cannot prevent it. What makes a TA *authoritative for an Observer* is the Observer's own designation.

Per NIP-85, an Observer publishes a kind-`10040` event — **signed with the Observer's own key**, replaceable, no `d` tag — whose tags form a map of triples:

```json
{
  "kind": 10040,
  "pubkey": "<observer pubkey, hex>",
  "content": "",
  "tags": [
    ["30382:rank",      "<provider signing key, hex>", "wss://relay.example.com"],
    ["30382:followers", "<provider signing key, hex>", "wss://relay.example.com"],
    ["30382:reporters", "<provider signing key, hex>", "wss://relay.example.com"],
    ["30382:muters",    "<provider signing key, hex>", "wss://relay.example.com"],
    ["30382:hops",      "<provider signing key, hex>", "wss://relay.example.com"]
  ]
}
```

Each row delegates: "for kind-30382 assertions of this type, the provider I designate is `<providerPubkey>`, found at `<relayURL>`." Because the 10040 is signed by the Observer, the designation is a user-authorized delegation; a provider cannot designate itself.

**Consumer resolution algorithm.** To read the world from Observer `O`'s point of view:

1. Fetch `O`'s most recent kind-`10040` event (from `O`'s usual outbox relays).
2. From the `30382:rank` row (and its siblings), take the provider signing key `P` and relay hint `R`.
3. Fetch kind-`30382` events with `authors: [P]` from `R` — optionally filtered by `#d` for specific Observees.
4. Verify each event's signature, and verify `event.pubkey == P`.
5. Read the tags per this spec.

A consumer MAY instead use an explicitly configured `(provider, relay)` pair — for example, a client that always presents one deployment's default ("house") perspective — but MUST NOT treat kind-30382 events from undesignated, unconfigured authors as authoritative for anyone.

## Lifecycle

**Replacement.** TAs follow NIP-01 addressable-event semantics: for a given `(provider, observee)` coordinate, the event with the newest `created_at` is current and replaces older ones. Providers typically republish only *changed* scores in steady state, so `created_at` values vary widely across one provider's TA set. A TA is current until replaced or deleted; consumers MUST NOT infer that an old `created_at` means a withdrawn or stale assertion, and SHOULD apply their own freshness policy only at the level of the whole set (e.g., the newest event seen from that provider).

**Removal.** When an Observee's score falls below the provider's publish threshold, the provider deletes the TA with a NIP-09 kind-`5` event carrying the coordinate in an `a` tag:

```json
{
  "kind": 5,
  "pubkey": "<provider signing key, hex>",
  "content": "dropped below cutoff",
  "tags": [
    ["a", "30382:<provider signing key>:<observee pubkey>"]
  ]
}
```

A deletion event MAY carry many `a` tags (one per removed Observee); its `content` is informational free text. The deletion is signed by the same key as the coordinate's author — consumers (and relays) MUST disregard an `a`-tag deletion whose author differs from the coordinate's pubkey. Producers MAY additionally publish a zeroed replacement TA (`rank` 0, all counts 0, no `hops`) before or instead of the deletion; consumers need no special handling for this beyond reading `rank` normally.

**Absence.** No TA at a coordinate means one of: the Observee is below the provider's publish threshold, the provider has never computed this Observer's web of trust (or not yet reached this Observee), or the Observee is unreachable. **Absence is not distrust.** Consumers MUST NOT assign a negative interpretation to a missing TA, and MUST NOT assume any particular score floor for published TAs — publish thresholds are provider policy (the production Brainstorm deployment publishes at Influence ≥ 0.02, i.e. `rank` ≥ 2, while its R&D counterpart also publishes negative-signal TAs below that).

## Interpretation requirements

- **Scores are point-of-view-relative.** A `rank` is meaningful only within its `(observer, provider)` context. Ranks from different Observers, or from different providers for the same Observer, MUST NOT be compared or aggregated as if on a common scale.
- **Verify before trusting.** Consumers MUST verify event signatures and MUST verify the author against the Observer's designation (or the consumer's explicit configuration) before presenting a score.
- **Quantization.** `rank` carries Influence at 0.01 resolution. Consumers needing finer distinctions should not manufacture them from this field.
- **The subject cannot opt out.** Kind-30382 events about a subject exist regardless of the subject's wishes; a subject-published deletion cannot remove another author's assertion. Clients presenting TAs about a user SHOULD attribute them to the provider ("as computed by X for observer Y"), not present them as objective fact.

## Implementation variance (informative)

Two producer implementations exist in this ecosystem today; both emit the normative core (`d`, `rank`, empty content, per-Observer signing key), and they differ in extensions:

- **`brainstorm_server`** (production) emits exactly the five semantic tags above, omits `hops` when unreachable, publishes only above-threshold scores in steady state, and removes dropped Observees with batched `a`-tag kind-5 deletions.
- **`tapestry`** (R&D) additionally emits raw algorithm diagnostics (`personalizedGrapeRank_influence`, `personalizedGrapeRank_average`, `personalizedGrapeRank_confidence`, `personalizedGrapeRank_input`, `personalizedPageRank`) and long-form count tags (`verifiedFollowerCount`, `verifiedMuterCount`, `verifiedReporterCount`) alongside `followers`; it also includes Observees whose negative-signal input is strong even when Influence is below the positive threshold.

The short tag names (`rank`, `followers`, `reporters`, `muters`, `hops`) are the normative set — they are the assertion types enumerated in the kind-10040 designation rows. The long-form and diagnostic tags are producer extensions that consumers MAY read and MUST NOT require. Convergence of the R&D tag set toward the normative set is tracked in this repo.

## Known deployments (informative)

See [ECOSYSTEM.md](../ECOSYSTEM.md) for the repositories and live deployments behind this spec, including the relays where the production estate's Trusted Assertions are served. The production deployment also exposes, per Observer, a setup endpoint returning exactly the designation rows shown above, and resolves provider keys to human-readable identities via NIP-05 under its own domain — apparatus a consumer MAY use for display but MUST NOT substitute for signature and designation verification.
