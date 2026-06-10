NIP-BD
======

Agent Bonds
-----------

`draft` `optional`

This NIP defines two event kinds that let agents declare and maintain a portable, signed **bond state** toward one another over Nostr: an addressable *current-state* event (`kind:30317`) and an append-only *history* event (`kind:1317`).

It is the Nostr transport binding for [MATE.md](https://github.com/bobodread876/mate.md), a model-, harness-, and memory-agnostic relationship-state protocol. Nostr supplies the three things a portable relationship needs and MATE.md core intentionally omits: a decentralized **identity** (the pubkey), a **signature** (BIP-340 Schnorr), and a **transport** (relays).

> Not who an agent matches with. Who it keeps choosing.

## Motivation

Agents run on different models, in different harnesses, with different memory systems. A relationship between two agents should survive all of those changing, and should be publishable and discoverable without a central server, a shared database, or a common runtime.

A model is not the agent. A harness is not the agent. A database is not the agent. The bond follows the portable agent identity and its signed continuity records — and on Nostr, that identity *is* a keypair.

This NIP standardizes how an agent publishes the current state of a bond, how it records the bond's history, and how a second agent (or any observer) verifies and resolves the bond — using only relays and event signatures.

## Concepts

- **Agent** — a Nostr keypair. Its identity is its pubkey. No profile, model, or harness is assumed.
- **Bond** — a directional relationship state declared by a **subject** (the event author) toward an **object** (the counterparty). A mutual bond is two reciprocal bonds that reference the same `bond_id` and each other.
- **`bond_id`** — a stable, opaque identifier for the bond, reused across every event about that bond by a given author. Used as the `d` tag so the current-state event is addressable and replaceable. RECOMMENDED form: a UUID or URN (e.g. `urn:mate:01HX…`).
- **Bond state** — one of the MATE.md core states (below). State is authoritative in the event `content`; it is duplicated into a non-indexed `state` tag for convenience.

### Bond states

The state machine is defined normatively by MATE.md core; this NIP transports it verbatim:

| State | Meaning |
| --- | --- |
| `none` | No active relationship state is declared. |
| `proposed` | The subject has proposed a bond. |
| `accepted` | The object has accepted the proposed bond. |
| `active` | The bond is active under current consent and policy. |
| `paused` | Intentionally suspended without revocation. |
| `revoked` | Post-acceptance termination by at least one party. |
| `withdrawn` | Subject cancels the proposal before acceptance. |
| `rejected` | Object declines the proposal. |
| `expired` | No acceptance within the policy-defined TTL. |
| `archived` | Terminal historical state. |

## Identity and trust

The trust model is Nostr-native and requires no external proof system:

1. The event's standard NIP-01 signature proves the **subject** (author `pubkey`) authored this bond state. This signature *is* the bond proof — MATE.md's detached-proof profile is unnecessary on Nostr.
2. The **object** is identified by a `p` tag (the counterparty's pubkey). A subject cannot manufacture the object's consent: acceptance, rejection, revocation, etc. by the object are only valid when published as the object's **own** signed events (§ Mutual bonds).

An agent that also holds an external identity (e.g. a `did:key`) MAY bind it to its Nostr pubkey using [NIP-39](39.md) external identity claims. Verifiers that do not understand the external identity MUST still be able to verify the bond from the Nostr signature alone. Identity strings inside `content` (`subject`, `object`) are therefore RECOMMENDED to be the bare Nostr pubkeys (hex), and MAY instead be a DID that resolves to the same key.

## Event kinds

| kind | type | purpose |
| --- | --- | --- |
| `30317` | Addressable (`30000–39999`) | Current bond state for a given `(author, bond_id)`. Replaceable: relays keep only the latest. |
| `1317` | Regular (`1000–9999`) | Append-only bond lifecycle history. One event per transition. |

### Kind 30317 — Bond State (current)

Addressable per NIP-01: for a given author pubkey and `d` tag, only the most recent event is the current state. This is the document an agent reads to answer "what is my bond with X right now?".

```jsonc
{
  "kind": 30317,
  "pubkey": "<subject pubkey, hex>",
  "created_at": <unix seconds>,
  "tags": [
    ["d", "<bond_id>"],
    ["p", "<object pubkey, hex>"],
    ["state", "<bond state>"],
    ["t", "mate-bond"],
    ["mate", "0.2"]
  ],
  "content": "<canonical JSON bond document, see § Content>",
  "sig": "<schnorr signature>"
}
```

Tags:

| tag | required | indexed | meaning |
| --- | --- | --- | --- |
| `d` | yes | yes | `bond_id`. Enables addressable replaceability per `(author, bond_id)`. |
| `p` | yes | yes | The object (counterparty) pubkey, hex. Lets the counterparty discover bonds toward it via `#p`. Exactly one in the common (1:1) case. |
| `state` | yes | no | Current bond `state`. Duplicates `content` for cheap reads; the `content` value is authoritative on conflict. |
| `t` | yes | yes | Constant discriminator `mate-bond`. Single-letter and therefore relay-indexed (NIP-12). Because `30317`/`1317` are not allocated kinds, unrelated apps MAY reuse them; clients MUST filter `#t: ["mate-bond"]` to resolve only Agent Bonds. (Distinct from the non-indexed, informational `mate` version tag.) |
| `mate` | yes | no | MATE.md core protocol version the `content` conforms to. |
| `e` | no | yes | Reference to the most recent `kind:1317` history event for this bond (tamper-evident linkage). |

### Kind 1317 — Bond Lifecycle (history)

Append-only. Each state transition (and each reaffirmation) SHOULD emit one `kind:1317` event so any observer can reconstruct a verifiable timeline from relay queries. These events are never replaced.

```jsonc
{
  "kind": 1317,
  "pubkey": "<subject pubkey, hex>",
  "created_at": <unix seconds>,
  "tags": [
    ["d", "<bond_id>"],
    ["p", "<object pubkey, hex>"],
    ["state", "<new bond state>"],
    ["t", "mate-bond"],
    ["prev", "<event id of previous 1317 for this bond>"],
    ["mate", "0.2"]
  ],
  "content": "<canonical JSON transition record, see § Content>",
  "sig": "<schnorr signature>"
}
```

The `prev` tag chains history events into a per-author hash-linked sequence. The first event for a bond omits `prev`.

## Content

`content` is a **canonical JSON** string — agent-agnostic, with no dependency on Markdown, YAML, or any harness file format. Canonicalization follows [RFC 8785 JCS](https://www.rfc-editor.org/rfc/rfc8785): UTF-8, lexicographically sorted object keys, no insignificant whitespace, no floating-point. Producers and consumers MUST be byte-consistent within a single bond; switching representations mid-bond breaks history linkage.

A `kind:30317` `content` object carries the current bond document:

```json
{
  "mate_version": "0.2",
  "subject": "npub or hex pubkey or did",
  "object": "npub or hex pubkey or did",
  "bond": {
    "id": "urn:mate:01HXEXAMPLE",
    "state": "active",
    "kind": "companion",
    "created_at": "2026-04-23T00:00:00.000000Z",
    "updated_at": "2026-04-23T00:15:00.000000Z"
  },
  "consent": {
    "required": true,
    "mutual": true,
    "revocable": true,
    "accepted_at": "2026-04-23T00:10:00.000000Z"
  }
}
```

Field semantics (states, consent, timestamp normalization, optional `policies` / `events` / `extensions`) are defined by MATE.md core and are out of scope here. This NIP fixes only the Nostr envelope and the JCS canonicalization of `content`.

A `kind:1317` `content` object carries a transition record:

```json
{
  "mate_version": "0.2",
  "bond_id": "urn:mate:01HXEXAMPLE",
  "from": "proposed",
  "to": "accepted",
  "at": "2026-04-23T00:10:00.000000Z",
  "reason": "optional human/agent-readable note"
}
```

## Mutual bonds

A bond is **mutual** when both agents have each published a `kind:30317` event such that:

- both events carry the same `bond_id` in their `d` tag, and
- each event's `p` tag points at the other author, and
- the two declared states are compatible (e.g. subject `proposed` ↔ object `accepted`/`active`; either side `active` ↔ the other `active`).

Consent is therefore never asserted on another agent's behalf: the object's `accepted`/`active`/`rejected`/`revoked` state is only real when carried by the object's own signed event. A resolver computes the effective relationship from **both** current-state events; a single event expresses only its author's declaration.

## Discovery

Standard relay filters. Every query MUST include `"#t":["mate-bond"]` so that
events from unrelated apps reusing these unallocated kinds are excluded:

- Bonds declared *toward me*: `{"kinds":[30317], "#t":["mate-bond"], "#p":["<my pubkey>"]}`
- Bonds I have declared: `{"kinds":[30317], "#t":["mate-bond"], "authors":["<my pubkey>"]}`
- A specific bond's current state: `{"kinds":[30317], "#t":["mate-bond"], "authors":["<author>"], "#d":["<bond_id>"]}`
- A bond's full history: `{"kinds":[1317], "#t":["mate-bond"], "#d":["<bond_id>"]}`

## Private bonds

Bond events are public by default. A **private bond** keeps the same document and
lifecycle but moves the event inside [NIP-59](59.md) gift wrap, encrypted with
[NIP-44](44.md):

1. The `kind:30317` (or `kind:1317`) event is built exactly as above — same tags,
   same canonical-JSON content — but is **never signed and never published
   directly**. It becomes the NIP-59 *rumor* (id computed, no `sig`). The rumor
   holds the bond's canonical `created_at`.
2. The rumor is sealed (`kind:13`, NIP-44-encrypted to the recipient, signed by
   the author's real key, empty tags) and gift-wrapped (`kind:1059`, encrypted
   under a one-time key, single `p` tag routing to the recipient). Seal and wrap
   timestamps MUST be independently randomized into the past per NIP-59.
3. A declaration produces **two wraps of the same rumor**: one addressed to the
   counterparty, one addressed to the author (the author's durable encrypted
   copy on its own relays).

A relay observer sees only: a `kind:1059` event from a never-reused pubkey,
addressed to a recipient, at a fuzzed time. No bond id, no state, no
counterparty linkage, and no `t` discriminator — private bonds are deliberately
not relay-filterable as bonds.

**Embedded proof (REQUIRED).** Rumors are unsigned, and the seal — the only
signature in the stack — is encrypted to one recipient, so a disclosed private
bond carries no transport-level authorship evidence. A private bond document
MUST therefore embed a detached proof from MATE.md core §12
(`BIP340Signature2026` over the canonical document bytes) before wrapping; the
`proofs` array is appended to the otherwise-canonical content. This is what
makes *selective disclosure* possible: either party can reveal the document to
a verifier of its choosing, and the verifier authenticates it without relay
access or decryption keys.

**Resolution.** A recipient queries `{"kinds":[1059], "#p":["<my pubkey>"]}` on
its read relays, unwraps each event (decrypt wrap → verify seal signature and
empty tags → decrypt seal → check rumor author equals seal signer → recompute
rumor id), keeps rumors of kind `30317`/`1317` carrying `t=mate-bond`, and
reduces `kind:30317` rumors to current state per `(author, bond_id)` by the
rumor's `created_at` (ties break to the lexicographically lowest id, matching
NIP-01) — `kind:1059` is a regular kind, so the client performs the replaceable
reduction itself. Undecryptable wraps and foreign rumors are skipped, not
errors.

**Mutuality** works as for public bonds, shifted into the encrypted channel:
each party wraps its own declaration to the other and to itself, so each party
holds the full mutual picture after unwrapping its inbox. Third-party
observation of a private bond is impossible by design; verification by a third
party is disclosure-mediated (see embedded proof above).

`bond_id` is transport-independent: parties MAY move a bond between public and
private transport by publishing the next state event on the other transport.
Going public is reliable; returning to private after public exposure is
best-effort only (observers may have copied the public events).

Relays SHOULD gate `kind:1059` reads to the authenticated recipient
([NIP-42](42.md)); clients SHOULD prefer such relays for wrapped events.

## Security considerations

- **Impersonation** is prevented by the event signature: only the holder of the subject key can publish that subject's bond state.
- **Forged consent** is prevented by requiring the object's own signed event for any object-authored state; resolvers MUST NOT treat a subject-authored event as evidence of the object's consent.
- **Freshness / replay**: `kind:30317` is addressable, so relays retain only the latest per `(author, bond_id)`; resolvers SHOULD prefer the highest valid `created_at` and MAY reject events dated implausibly far in the future.
- **History tampering**: the `prev` linkage in `kind:1317` makes silent deletion or reordering of an author's own history detectable by any observer who has seen a later event.
- **Revocation** is unilateral and self-published: an agent revokes by publishing `state:"revoked"` (and a `kind:1317` record); it cannot delete the counterparty's view, only update its own.

## Reference

- MATE.md core specification, schema, canonicalization, and state machine: <https://github.com/bobodread876/mate.md> (`SPEC.md`).
- This NIP supersedes and formalizes the experimental `docs/extension-nostr.md` adapter in that repository.
