# Architecture

```
Application
    │  new CooL({...}).record({ type, metadata, payloads })
    ▼
CooL (src/client.ts)            facade: lazy connect, config validation, typed errors
    │
    ▼
CoolTee (src/phala/client.ts)   wiring: attest → open queue → seal; nothing reaches
    │                           the caller's request path
    ├── CaptureQueue            bounded, fail-open, out-of-band. drop = counted, never silent
    ├── AttestedChannel         RA-TLS handshake against the evidence plane's quote
    ▼
EvidencePlane (src/phala/engine.ts)   runs INSIDE the enclave in production
    │  for each event:
    │   1. commit  — salted sha256 of metadata and payloads; plaintext discarded
    │   2. bind    — mh(canonicalCBOR(core))   (core includes measurement + quote digest)
    │   3. sign    — hybrid ML-DSA-65 + Ed25519 with the measurement-sealed key
    │   4. log     — append the binding digest to the RFC 6962 tree; take an STH + proof
    ▼
Receipt (cool.receipt.v2)  →  Verifier (src/phala/verify.ts)   offline, 7 domains
```

## Layers

| Layer | Files | Responsibility |
|---|---|---|
| Client facade | `src/client.ts`, `src/errors.ts` | small public API, config validation, environment detection, error typing |
| Verifier | `src/verify.ts`, `src/phala/verify.ts`, `src/phala/structure.ts` | offline verdict; structural validation; the human formatter |
| Evidence engine | `src/phala/engine.ts`, `src/phala/record.ts` | commit → bind → sign → log |
| Cryptography | `src/canonical.ts`, `src/hash.ts`, `src/multihash.ts`, `src/sign.ts`, `src/keys.ts`, `src/merkle.ts`, `src/log-memory.ts`, `src/record.ts` | CDE-CBOR, SHA-256, hybrid signatures, RFC 6962 |
| Attestation | `src/phala/dstack.ts`, `src/phala/ratls.ts`, `src/phala/quote.ts`, `src/phala/kms.ts` | dstack guest-agent client (HTTP + simulated), RA-TLS, TDX quote structure + binding, measurement-sealed keys |
| Transparency | `src/phala/log.ts`, `src/phala/log-file.ts`, `src/phala/witness.ts`, `src/phala/anchor.ts` | in-memory / file-backed logs, witness co-signatures, OpenTimestamps anchoring |
| Governance | `src/phala/policy.ts`, `src/phala/compliance.ts`, `src/phala/disclose.ts`, `src/phala/pack.ts`, `src/phala/query.ts` | policy engine, obligation mapping, selective disclosure, audit packs |
| CLI | `src/cli/*` | `cool` — verify, doctor, walkthrough, seal, records, pack, disclose |

## Trust boundaries

```
        untrusted                    trusted (in production)                 untrusted
  ┌──────────────────┐   RA-TLS   ┌──────────────────────────┐          ┌──────────────┐
  │  application code │──────────▶│  EvidencePlane in a TEE   │─receipt─▶│  the reader  │
  │  (CooL client)    │  quote    │  measurement-sealed key   │  (JSON)  │  verifies it │
  └──────────────────┘  checked   └──────────────────────────┘          └──────────────┘
```

- The **client** is untrusted: it hands plaintext across an attested channel and
  never sees the signing key.
- The **plane** is trusted only as far as its attestation goes — the reader
  re-checks the quote, the measurement, and the key binding.
- The **reader** trusts nothing: every domain is recomputed from the bytes.

## Deployment shapes

| Shape | `attestation.provider` | What's real |
|---|---|---|
| local dev / CI | `local` (default) | signatures, log, canonicalization. Attestation is `simulated`. |
| dstack simulator | `dstack` + `COOL_DSTACK_ENDPOINT` | quote *structure* and key binding, under a CooL-held root. Still `simulated`. |
| dstack on Intel TDX (self-hosted or Phala Cloud) | `dstack` + `/var/run/dstack.sock` | hardware quote, measurement-sealed key. `hardware` once a quote verifier confirms the vendor root. |
# Attestation

CooL's evidence plane can run in three modes. The mode is recorded in every
receipt and is never blurred.

| `mode` | What it means | Verifier verdict on `attestation` / `enclave` |
|---|---|---|
| `mock` | no attestation path at all | `mock` / `absent` |
| `simulated` | a structurally complete quote under a **CooL-held** root — real signature, real key binding, no vendor root | `simulated` / `simulated` |
| `hardware` | a vendor quote produced by real silicon (Intel TDX today) | `pass` / `pass` — **only** with a quote verifier and a real vendor root |

`simulated` is not a soft `pass`. It is exactly what it says: the shape and the
key binding are real; Intel/AMD/NVIDIA as the root of trust is not.

## What is bound to what

When the plane starts it:

1. reads the runtime measurement (`Info`),
2. derives a signing key **sealed to that measurement** (`GetKey`),
3. asks the hardware for a quote whose 64 bytes of `report_data` are
   `enclaveReportData(signingKey.publicKey)` — a commitment to the signing
   identity (`GetQuote`).

The quote's digest then goes **inside** the signed record core (`runtime.tee_quote`),
before the record is signed. So:

- the signature covers the attestation — a valid quote cannot be moved onto a
  record it did not attest;
- `report_data` ties the quote to the exact key that signed — "an attested
  enclave holds this key" and "this key signed this record" are one chain;
- the measurement is in the signed core — it cannot be relabelled after the fact.

The `enclave` verifier domain re-checks all of that from the bytes.

## Quote verification stages — do not conflate them

| Stage | Who does it | CooL status |
|---|---|---|
| quote **retrieved** | the plane, at startup | always, in `simulated` and `hardware` |
| quote **parsed / structurally checked** | `checkQuoteStructure` | always |
| quote **cryptographically verified against a vendor root** | a `QuoteVerifier` you supply (`remoteQuoteVerifier` → Intel DCAP collateral) | only `attestation: pass` |
| **measurement** matches the image you approved | your `expectedMeasurement` pin | only then is `enclave` pinned |
| **policy** accepts that measurement / vendor / signer | you | outside the verdict |

"We got a quote" is not "the quote is verified", and "the quote is verified" is
not "I accept this workload". CooL keeps these separate on purpose.

## Requiring hardware

```ts
const cool = new CooL({
  applicationId: "regulated",
  attestation: { provider: "dstack" },
  security: { requireAttestation: true },   // connect fails if the handshake isn't hardware-backed
});

// and at verification time:
await verifyEvidence(evidence, {
  requireHardware: true,
  quoteVerifier: remoteQuoteVerifier({ endpoint: DCAP_URL, root: "intel-dcap" }),
  expectedMeasurement: PINNED,
});
```

With `requireAttestation`, CooL never silently downgrades from hardware to
simulation — if the quote isn't real, it throws `AttestationRequiredError` at
connect and returns `ok: false` at verify.

## Keys

The signing and log keys are **derived inside the enclave** from the measurement
(dstack `GetKey`). There is no key to configure and none to leak into config.
A redeploy that changes the image rotates the keys automatically; historical
receipts stay verifiable because each carries its own `key_directory`.
# Development

```sh
git clone https://github.com/Northwind-Cipher/cool-sdk
cd cool-sdk
npm install
npm run verify:all
```

Node ≥ 20. No build step for development — tests run TypeScript through `tsx`.

## Layout

```
src/                 the SDK
  client.ts          the CooL facade  (the public API)
  verify.ts          verifyEvidence + formatVerdict
  errors.ts          typed errors
  index.ts           the "." entry;  tee.ts = kitchen sink
  canonical|hash|multihash|sign|keys|merkle|log-memory|record|codec|types.ts   primitives
  phala/             the advanced tier: engine, client (CoolTee), dstack, ratls,
                     quote, kms, structure, verify, anchor, witness, policy,
                     compliance, disclose, pack, query, gpu, types
  cli/               the `cool` command
scripts/             build.mjs, demo.ts, verify-package.mjs
tests/               node --test suites
examples/            runnable consumer projects
docs/                this
```

## Commands

| Command | Purpose |
|---|---|
| `npm test` | full suite (`node --test` + `tsx`) |
| `npm run typecheck` | `tsc --noEmit`, strict, `noUnusedLocals` |
| `npm run build` | `src/` → `dist/`, then patch ESM specifiers and copy `cli/ui/app.html` |
| `npm run demo` | valid → tamper → fail walkthrough |
| `npm run verify:package` | pack, install into a clean temp project, run, typecheck under `nodenext` + `bundler` |
| `npm run verify:all` | typecheck + tests |

## Testing conventions

- Every behavioural change gets a test that fails before and passes after.
- Adversarial tests assert the property, then attack it (`tests/evidence.test.ts`,
  `tests/verifier.test.ts`).
- One skipped test (`anchor.test.ts` → "public calendars accept a real head")
  is an opt-in network test; it is skipped offline by design.
- CLI tests spawn the `cool` binary in a temp directory — a CLI's contract is a
  process, an exit code, and bytes on stdout.

## Changing the evidence schema

The bytes that get hashed (`canonicalCbor(core)`) and signed
(`core ‖ binding_digest`) are frozen. To change them:

1. bump `record.schema` (`cool.evidence.vN`);
2. update `src/phala/structure.ts` (the structural validator);
3. teach `src/phala/verify.ts` the new `subject.kind` / fields;
4. add vectors and a migration note.

Never reinterpret a newer schema as an older one.

## Release

Maintainers: [`release.md`](release.md).
# Phala dstack

CooL talks to a [dstack](https://github.com/Dstack-TEE/dstack) guest agent over
its RPC surface — `Info`, `GetQuote(reportData)`, `GetKey(path)` — reached at
`/var/run/dstack.sock` inside a CVM, or an `http://` URL against the simulator.
CooL does not bundle `@phala/dstack-sdk`; it speaks the protocol directly through
`HttpDstackClient`. Pin a known-good dstack runtime for production rather than
tracking `latest`.

## 1. Local development — no dstack at all

```ts
const cool = new CooL({ applicationId: "dev" });   // provider defaults to "local"
```

The simulator runs the identical pipeline. Every receipt says `simulated`. Good
for tests, CI, and most of a hackathon.

## 2. The dstack simulator (hardware path, no hardware)

Install the Phala CLI and start the simulator, then point CooL at it:

```sh
npm install -g phala
phala simulator start          # serves the guest-agent RPC over a local endpoint
```

```sh
export COOL_DSTACK_ENDPOINT=http://127.0.0.1:8090   # use the address the simulator prints
node your-app.js
```

```ts
const cool = new CooL({
  applicationId: "refund-agent",
  attestation: { provider: "dstack" },   // endpoint picked up from COOL_DSTACK_ENDPOINT
});
```

The quote *structure* and key binding are exercised end to end; the root is still
CooL-held, so receipts remain `simulated`. Confirm the current simulator command
and port against the Phala docs — they have moved between releases.

## 3. Self-hosted Intel TDX

Run your app inside a dstack CVM with the guest-agent socket mounted:

```yaml
# docker-compose.yaml (inside the CVM)
services:
  app:
    image: your/app
    volumes:
      - /var/run/dstack.sock:/var/run/dstack.sock
    environment:
      - COOL_IMAGE_DIGEST=sha256:...   # the image you built and reviewed
```

```ts
const cool = new CooL({
  applicationId: "refund-agent",
  attestation: { provider: "dstack", endpoint: "/var/run/dstack.sock", vendor: "intel-tdx" },
  security: { requireAttestation: true, expectedMeasurement: PINNED },
});
```

To get `attestation: pass` (not just `hardware`), the *verifier* also needs a
quote verifier:

```ts
import { remoteQuoteVerifier } from "cool-nwc/phala";
await verifyEvidence(evidence, {
  quoteVerifier: remoteQuoteVerifier({ endpoint: process.env.QUOTE_VERIFIER_URL, root: "intel-dcap" }),
  expectedMeasurement: PINNED,
  requireHardware: true,
});
```

## 4. Phala Cloud

Deploy the same compose file through the Phala CLI:

```sh
phala auth login
phala deploy            # or `phala cvms create` — follow the current CLI help
```

Phala Cloud provides the TDX host and the guest agent; nothing in the CooL
integration changes. Pull the running instance's measurement and set it as
`expectedMeasurement` / `COOL_IMAGE_DIGEST`.

> Deployment to Phala Cloud requires a Phala account and TDX capacity, and has
> **not** been exercised as part of this repository's CI. The configuration
> above is the intended path; the external steps are yours to run.

## Compatibility

| CooL | dstack protocol | Environment | Status |
|---|---|---|---|
| 3.x | `Info` / `GetQuote` / `GetKey` | local simulator (built-in) | tested in CI |
| 3.x | same | dstack simulator binary | supported; run it yourself |
| 3.x | same | self-hosted Intel TDX | supported; not in CI |
| 3.x | same | Phala Cloud | supported; not in CI |

## What dstack contributes — precisely

- a **confidential VM** (Intel TDX) to run the evidence plane in;
- an **attested workload identity** — the MRTD + RTMR measurement set;
- a **quote** binding that identity to CooL's signing key via `report_data`;
- a **measurement-sealed key** so the signer is the code, not an operator.

It does **not** make CooL "secure" on its own, and CooL does not create the TEE —
it uses the one dstack provides.
# Evidence format

Version: `cool.receipt.v2` envelope carrying a `cool.evidence.v1` record.
Structural validator: `validateReceiptV2Shape` (`cool-nwc/phala`) — hand-written,
names the exact failing field. A published JSON Schema is on the roadmap.

## Encodings

| Form | Grammar | Used for |
|---|---|---|
| multihash | `mh:sha256:<64 lowercase hex>` | every digest |
| hex field | `hex:<lowercase hex>` | 16-byte salts |
| base64 field | `base64:<standard base64>` | keys, signatures, raw quote bytes |
| ULID | Crockford base32, 26 chars | `record_id` |
| time | RFC 3339 / `date-time`, UTC | `issued_at`, `timestamp` |

## The record core (`cool.evidence.v1`)

This is the object that is canonicalized and hashed. The signature is **not**
part of it.

| Field | Type | Meaning |
|---|---|---|
| `schema` | `"cool.evidence.v1"` | |
| `record_id` | ULID | unique per record; sorts by time |
| `time.issued_at` | RFC 3339 | when the plane sealed it |
| `time.seq` | int ≥ 0 | monotone within one plane instance |
| `event.type` | string | dotted, e.g. `model.execution`, `agent.action`, `artifact.created`, `policy.decision` |
| `event.application_id` | string | the app that produced it |
| `event.execution_id` | string | the run / session / request this belongs to |
| `event.metadata_hash` | multihash | `mh:sha256(metadata_salt_bytes ‖ canonicalCBOR(metadata))` |
| `event.metadata_salt` | hex field | 16 random bytes |
| `event.commitments.{input,output,state}` | multihash \| null | `mh:sha256(salt ‖ bytes)` of a payload, or null |
| `event.commitments.{input,output,state}_salt` | hex field \| null | paired salt; null iff the commitment is null |
| `event.software` | `{ name, version, digest: multihash\|null }` \| null | workload identity (cleartext by design) |
| `runtime.tee_vendor` | `none \| intel-tdx \| amd-sev-snp \| nvidia-cc` | |
| `runtime.mode` | `mock \| simulated \| hardware` | |
| `runtime.enclave_measurement` | `{ mrtd, rtmr0..3 }` (hex fields) \| null | |
| `runtime.tee_quote` | multihash \| null | digest of the quote in the envelope; `hardware` requires it |
| `runtime.gpu` | GPU attestation ref \| null | |

## The signature

```
message      = canonicalCBOR(core) ‖ multihashDigest(binding_hash)   // 32-byte digest appended
signature    = { alg: "ml-dsa-65+ed25519", key_id, ml_dsa: base64, ed25519: base64 }
```

Both `ml_dsa` (FIPS 204 ML-DSA-65) and `ed25519` are computed over the same
message. Verification requires **both** to pass.

## The envelope (`cool.receipt.v2`)

| Field | Meaning |
|---|---|
| `record` | the core above, plus `signature` |
| `binding_hash` | `mh:sha256(canonicalCBOR(core))` — recompute and compare |
| `inclusion` | `{ leaf_index, tree_size, audit_path: multihash[] }` — RFC 6962 audit path, or null |
| `sth` | signed tree head: `{ log_id, tree_size, root_hash, timestamp, signature, witnesses[] }`, or null |
| `attestation` | `{ mode, note, quote, expected_measurement, key_binding }` |
| `anchor` | OpenTimestamps proof `{ kind, chain, target, tree_size, proof(base64), calendars[], submitted_at, heights[] }`, or null |
| `key_directory` | `{ [key_id]: { ml_dsa_pub: base64, ed25519_pub: base64 } }` — everything needed to verify offline |

`inclusion` and `sth` are present together or absent together.

## Canonicalization

`canonicalCbor` = RFC 8949 §4.2 Core Deterministic Encoding, via `cbor2`. The
same logical value always encodes to the same bytes regardless of property
insertion order, run, or machine. This is what makes `binding_hash` and the
signature reproducible across implementations. Never hash or sign the JSON form —
only `canonicalCBOR(core)`.

## Replay and freshness

Uniqueness is `(application_id, execution_id, record_id)`. `record_id` is a ULID
(time-ordered), `time.seq` is monotone within a plane instance, and
`time.issued_at` is UTC. Timestamps are **not** the sole replay defence — a
consumer that cares about freshness compares `issued_at` and `execution_id`
against its own expectations.

## Versioning

`record.schema` and the envelope `schema` are explicit. A verifier accepts the
versions it knows and rejects unknown ones with a reason — it never interprets a
newer schema as an older one. Changing what bytes get hashed or signed is a
schema-version change.
# Getting started

## Install

```sh
npm install cool-nwc
```

Node **≥ 20**, ESM. TypeScript types are included. No native build, no postinstall
script, no network access at install time.

## Record a piece of evidence

```ts
import { CooL, verifyEvidence } from "cool-nwc";

const cool = new CooL({ applicationId: "my-app" });

const { evidence, recordId, digest } = await cool.record({
  type: "model.execution",
  metadata: { model: "my-model", version: "1.0.0" },
});

const verdict = await verifyEvidence(evidence);
console.log(verdict.ok); // true
```

`new CooL()` does no I/O. The first `record()` connects the evidence plane. With
no `attestation.provider`, that plane is the built-in **simulator**: identical
code path, every receipt labelled `simulated`.

## Commit to sensitive data without storing it

```ts
await cool.record({
  type: "model.execution",
  metadata: { route: "/score" },
  payloads: {
    input: JSON.stringify(request.body),   // hashed with a random salt, then discarded
    output: JSON.stringify(response),
    state: undefined,
  },
});
```

The receipt carries `mh:sha256(salt ‖ bytes)` and the salt — never the bytes. An
auditor you later choose to show the plaintext to can check it against the
commitment (`cool disclose`, or `disclose()` from `cool-nwc/phala`).

## Verify — anywhere, offline

```ts
import { verifyEvidence, formatVerdict } from "cool-nwc";

const verdict = await verifyEvidence(JSON.parse(fileContents));
console.log(formatVerdict(verdict));
```

or from the CLI:

```sh
npx cool-nwc verify evidence.json   # exit 0 = verified, non-zero = failed
```

## Shut down cleanly

```ts
process.on("SIGTERM", async () => {
  await cool.flush();   // drain queued async events
  await cool.close();
});
```

## Next

- [configuration and the `CooL` options](../README.md#api)
- [the evidence record, field by field](evidence-format.md)
- [what the verifier checks](verification.md)
- [running inside Phala dstack](dstack.md)
- [troubleshooting](troubleshooting.md)
# Migration — inference capture → `record()`

`cool-nwc` 3.0 removes the AI-inference capture surface. If you used
`Cool.complete()`, `CoolTee.complete()` / `completeSealed()`, a `backend` option,
`PhalaPrivateLLM`, or the v1 `cool.receipt.v1` client, this is the map.

## Why

The SDK now records **generic execution evidence**: an event type plus a salted
commitment to your metadata, with optional commitments to input/output/state. It
never takes a raw prompt or wraps a model call. Everything the old inference path
committed, `record()` still commits — you just say what the event was.

## API changes

| Removed | Replacement |
|---|---|
| `new Cool({ signing, backend })` + `cool.complete({ model, prompt })` (v1) | `new CooL({ applicationId })` + `cool.record({ type, metadata, payloads })` |
| `CoolTee.connect({ backend })` | drop `backend` — you call your model yourself |
| `cool.complete({ model, prompt })` | `cool.recordAsync({ type, metadata, payloads })` (fire-and-forget) |
| `cool.completeSealed({ model, prompt })` → `{ output, receipt }` | `await cool.record({ type, metadata, payloads })` → `{ evidence, recordId, ... }` |
| `PhalaPrivateLLM` / `ConfidentialCompletion` | call the model with your own HTTP client; pass `simulatedGpu(...)` / `gpuRefFromReport(...)` as `record({ gpu })` |
| `verifyReceipt` (v1, `cool.receipt.v1`) | `verifyEvidence` (`cool.receipt.v2`) |
| `import { ... } from "cool-nwc/v1"` | primitives are on the root: `import { canonicalCbor, hybridSign, ... } from "cool-nwc"` |

## Before

```ts
import { CoolTee } from "cool-nwc";

const cool = await CoolTee.connect({
  app: { name: "refund-agent", imageDigest: process.env.IMAGE_DIGEST! },
  backend: async ({ model, prompt, params }) => ({ output: await yourModel(model, prompt, params) }),
});

const { output, receipt } = await cool.completeSealed({
  model: "phala/deepseek-v4-pro@2026.07",
  prompt: "Assess application A-40182",
  params: { temperature: 0.2 },
});
```

## After

```ts
import { CooL } from "cool-nwc";

const cool = new CooL({ applicationId: "refund-agent" });

const prompt = "Assess application A-40182";
const output = await yourModel("phala/deepseek-v4-pro@2026.07", prompt, { temperature: 0.2 });

const { evidence } = await cool.record({
  type: "model.execution",
  metadata: { model: "phala/deepseek-v4-pro@2026.07", params: { temperature: 0.2 } },
  payloads: { input: prompt, output },
});
```

`await verifyReceipt(receipt)` → `await verifyEvidence(evidence)`. The verdict
shape is the same seven domains; `subject.kind` is now `"evidence"` (or
`"change"`), never `"inference"`.

## `change()` is unchanged

`cool.change({ kind, ref, before, after, actor, approval })` — recording a change
to an AI system itself — works exactly as before, on `CoolTee` (`cool-nwc/phala`).

## Old receipts

`cool.receipt.v1` receipts are not readable by the 3.0 verifier. Re-verify them
with `cool-nwc@2.5.x`, or re-seal the underlying events as `cool.evidence.v1`.
# Release (maintainers)

Publishing is done by `.github/workflows/release.yml` on a version tag, using
**npm trusted publishing (OIDC)** — there are no npm tokens in the repo or in CI
secrets. The workflow generates provenance automatically.

## Checklist

1. `main` is green (`ci.yml`).
2. Bump `version` in `package.json` (semver). Breaking public-API or
   evidence-schema changes → major.
3. Update `CHANGELOG.md`: move `## [Unreleased]` to `## [x.y.z] — YYYY-MM-DD`.
4. Locally:
   ```sh
   npm run verify:all
   npm run verify:package      # pack → clean install → run → typecheck
   npm pack --dry-run          # inspect file list and size
   ```
5. Commit, then tag:
   ```sh
   git tag vX.Y.Z
   git push origin main --tags
   ```
6. `release.yml` runs: checkout → install → verify:all → build → `npm publish`
   (with `--provenance`, via OIDC) → create the GitHub Release with the
   changelog section.
7. Verify: `npm view cool-nwc@X.Y.Z` shows the version and a provenance link;
   install it in a scratch dir and run the quickstart.

## First publish

The npm package name and the GitHub repo must be linked for trusted publishing:
configure the `cool-nwc` package on npm to trust
`Northwind-Cipher/cool-sdk` → `.github/workflows/release.yml`
(npm: *Settings → Publishing access → Trusted publisher*). Until that is done the
release job's publish step will fail closed — which is the intended default.

## Evidence schema versions

`cool.evidence.v1` / `cool.receipt.v2` are versioned independently of the package.
A schema change ships its own structural-validator update and a `docs/migration.md`
entry, and bumps the package major.
# Security model

## What a receipt proves

When `verifyEvidence` returns `ok: true`:

- the record's contents match its `binding_hash` (nothing was edited);
- **both** signatures verify against the embedded public key (it was sealed by
  the holder of that hybrid key);
- if a log proof is present, the record is included under a validly signed tree
  head (append-only);
- if a hardware quote is present and a verifier was supplied, the quote chains
  to a vendor root, its measurement matches, and its `report_data` commits to the
  signing key (it was produced in *that* attested environment).

## What it does not prove

- that the output was correct, fair, safe, or policy-compliant — CooL records
  what happened, it does not grade it;
- anything about a `key_id` beyond "this key signed this" — key ids are
  operator-chosen labels, not certified identities; bring your own allow-list;
- independent witnessing or public availability of the log — no external
  witnesses in this build;
- confidentiality, when `mode` is `simulated`.

## Trust boundaries

| Party | Trusted for | Checked by |
|---|---|---|
| CooL client (in the app) | nothing — it holds no key | — |
| Evidence plane (in a TEE) | producing a correct record for the event it was handed | the reader re-checks the quote, measurement, and key binding |
| dstack guest agent | issuing a genuine quote and a measurement-sealed key | a `QuoteVerifier` against the vendor root |
| TEE hardware + vendor attestation service | the measurement reflects the code; the quote is authentic | outside CooL's scope |
| Transparency log operator | not removing or reordering entries | RFC 6962 consistency proofs (external witnesses would strengthen this) |
| The reader | nothing | recomputes every domain from the bytes |

## Cryptographic assumptions

- SHA-256 collision resistance (commitments, Merkle tree, digests).
- Ed25519 EUF-CMA **or** ML-DSA-65 (FIPS 204) EUF-CMA — the hybrid holds if
  *either* survives.
- RFC 8949 CDE gives a single encoding per logical value (canonicalization).
- The platform CSPRNG (`crypto.getRandomValues`) for salts and keys.

## Failure semantics

**Fail-open toward the application.** A dead transparency endpoint, a closed
RA-TLS channel, a full capture queue — none of these throw into the caller's
request path. Loss is counted (`cool.stats()`), never silent.

**Fail-closed toward verification.** The verifier never reports success on a
check it could not perform. `requireAttestation` (connect) and `requireHardware`
(verify) turn "no real quote" into a hard failure instead of a silent downgrade.

## Key management

Signing and log keys are derived inside the enclave from the measurement (dstack
`GetKey`) — there is nothing to configure or rotate manually, and a redeploy
rotates them. Each receipt carries its own `key_directory`, so historical
receipts verify after rotation. Keys are never logged, never persisted outside
the enclave, and never appear in a receipt (only public halves do).

## Privacy

Metadata and every `payloads` value are committed as `mh:sha256(salt ‖ bytes)`
and discarded. Salts are stored beside the commitments so a value can be
selectively disclosed later and checked. `software` identity (name/version/digest)
is cleartext by design. The SDK makes no network calls of its own and carries no
telemetry.

## Supply chain

- runtime dependencies: `@noble/curves`, `@noble/hashes`, `@noble/post-quantum`,
  `cbor2`, `ulid` — audited, minimal, no native code;
- no `postinstall` / install scripts; no downloads at install time;
- the published tarball is an explicit `files` allowlist (`dist`, `src`, README,
  LICENSE, CHANGELOG);
- releases are published by a tagged GitHub Actions workflow using npm trusted
  publishing (OIDC) with provenance — no long-lived tokens.

## Known limitations

- No external transparency witnesses — log integrity rests on the operator plus
  consistency proofs.
- Bitcoin anchoring is implemented but confirmation against block headers is
  opt-in and `pending` until aggregated.
- Remote quote verification against live Intel DCAP collateral is exercised only
  against a mock in CI.
- Phala Cloud / real TDX deployment is documented, not CI-tested.
# Threat model

Assets: the integrity and authenticity of evidence records, and the privacy of
the data they commit to. Adversary: anyone who can obtain a receipt and wants it
to say something false — or wants to read the data behind it.

| # | Threat | CooL defense | Residual risk |
|---|---|---|---|
| 1 | Edit a field of a stored receipt | `binding_hash` = `mh(canonicalCBOR(core))`; the signature covers `core ‖ binding_digest`. Any change breaks both. | Compromise of the enclave signing key. |
| 2 | Re-encode / reorder JSON to change bytes | Hash and signature are over RFC 8949 CDE CBOR, not JSON. Key order is irrelevant. | A canonicalization bug — mitigated by cross-run/cross-machine tests; formal verification is on the roadmap. |
| 3 | Replay an old record as new | `(application_id, execution_id, record_id)` uniqueness; ULID `record_id`; monotone `seq`; UTC `issued_at`. | Freshness policy is the consumer's — CooL supplies the fields, not a clock authority. |
| 4 | Swap the signer (present a different key) | The signature names a `key_id`; verification uses `key_directory[key_id]`; `report_data` in a hardware quote commits to that exact key. | Trusting the wrong `key_id` — bring an allow-list; key-directory management. |
| 5 | Staple a valid quote from another enclave onto this record | `enclave` domain: quote digest must equal `runtime.tee_quote` (inside the signature), measurement must match, `report_data` must commit to the signing key. | TEE hardware / vendor attestation-service compromise. |
| 6 | Claim a simulated run was hardware | `mode: "simulated"` is in the signed core; verifier returns `simulated`, never `pass`; `requireHardware` makes it `ok: false`. | Operator turns the policy off. |
| 7 | Remove or reorder entries in the transparency log | RFC 6962 inclusion + consistency proofs; the STH root commits to every leaf. | No external witnesses in this build — a malicious operator with the log key could fork it; consistency proofs bound the damage. |
| 8 | Backdate a tree head | Optional OpenTimestamps anchor: the root is committed into a Bitcoin block, which cannot be pre-dated. | Anchor is optional and `pending` for ~1h after submission. |
| 9 | Read the prompt / output / metadata from a receipt | Only salted commitments are stored; plaintext is hashed and discarded inside the plane. | Weak/low-entropy payloads are still guessable from a commitment despite the salt — commit to high-entropy representations where it matters. |
| 10 | Man-in-the-middle the client → plane channel | RA-TLS: the client verifies the plane's quote against policy **before** transmitting; a mismatched endpoint fails the handshake and the caller is unaffected. | Client host compromise (the client holds no key, but it holds the plaintext before it is sent). |
| 11 | Tamper with the SDK itself (supply chain) | Minimal audited deps, no install scripts, `files` allowlist, OIDC trusted publishing + provenance. | A compromised upstream dependency; verify provenance on install. |
| 12 | DoS the evidence pipeline | Bounded, fail-open capture queue: oldest events dropped and **counted**, never blocking the request. | Under sustained overload, evidence is lost — visible in `cool.stats()`, not silent. |

## Explicitly out of scope

CooL does not protect against:

- application logic that is compromised or simply wrong, producing valid
  evidence of the wrong thing;
- TEE hardware vulnerabilities or a compromised vendor attestation service;
- an operator who configures incorrect trust roots or disables the security
  policy;
- a malicious but *approved* workload;
- traffic analysis / metadata leakage at the network layer.
# Troubleshooting

Run `npx cool-nwc doctor` first — it checks Node, web crypto, the dstack socket,
a quote verifier, and a seal→verify round trip.

---

### `npm install` fails compiling something

CooL has **no native code and no build step**. If a compiler runs, it's another
dependency in your tree. `cool-nwc` itself needs only Node ≥ 20 and pure-JS deps
(`@noble/*`, `cbor2`, `ulid`).

### `Cannot find module 'cool-nwc'` after install / ESM errors

`cool-nwc` is ESM-only. Your project needs `"type": "module"` (or `.mjs` files),
or a bundler configured for ESM. There is no CommonJS build; a `require("cool-nwc")`
will not work.

### Types don't resolve / `moduleResolution` errors

The package ships `.d.ts` and is tested under `moduleResolution: "nodenext"` and
`"bundler"`. If you're on `"node10"`/`"node"`, switch to `"bundler"` or
`"nodenext"`. `npm run verify:package` in this repo reproduces the consumer setup.

### `COOL_DSTACK_UNAVAILABLE`

```
CooL could not reach the dstack guest agent

  run the application inside a dstack CVM with /var/run/dstack.sock mounted,
  start the dstack simulator, or use attestation.provider: 'local'
```

You set `attestation.provider: "dstack"` (or `$COOL_DSTACK_ENDPOINT`) but nothing
is listening. For local work, drop the provider (defaults to `local`). For the
hardware path without a CVM, run the dstack simulator and set
`COOL_DSTACK_ENDPOINT`. See [dstack.md](dstack.md).

### `COOL_ATTESTATION_REQUIRED`

`security.requireAttestation` is set and the RA-TLS handshake did not produce a
verified hardware quote. Either deploy inside attested hardware, or remove
`requireAttestation` for development. CooL will not silently downgrade to the
simulator when you asked for hardware.

### `ConfigurationError: requireAttestation is set but provider is 'local'`

These two contradict. Use `provider: "dstack"` with `requireAttestation`, or
neither.

### Verification returns `ok: false` with `attestation`/`enclave` = `simulated`

Expected in local / simulator mode. To *require* hardware at verify time pass
`{ requireHardware: true }` — and understand it will fail until you run against
real dstack with a `quoteVerifier`.

### Verification fails: "recomputed binding_hash does not match"

The record was modified after signing (or serialised through something that
mutated numbers/strings). This is the tamper-detection working. If it happens on
an untouched receipt, check that whatever transported it preserved the JSON
exactly (no float re-formatting, no key re-casing).

### `cool verify` exits non-zero in CI but the receipt "looks fine"

That's the contract — any failed domain is a non-zero exit. Run
`cool verify <file>` locally to see the boxed report and the `reasons`.

### The transparency endpoint / anchor calendar is down

The SDK is fail-open: your app keeps producing signed, logged evidence. The
`anchor` domain stays `absent`/`pending`; nothing else is affected.

### Large metadata

Metadata is CBOR-encoded and hashed; very large objects cost CPU and memory on
the seal path. Commit a digest or a summary rather than a multi-megabyte blob.

### Browser / edge runtime

`cool-nwc` targets Node (it uses `node:*` for the filesystem log and unix-socket
transport via `cool-nwc/node`). The core client + verifier need only WebCrypto
and `btoa`/`atob`, but browser/edge use is **not** currently tested — don't rely
on it.
# Verification

```ts
import { verifyEvidence, formatVerdict } from "cool-nwc";

const verdict = await verifyEvidence(evidence, options);
```

`verifyEvidence` never throws on bad input — a malformed or tampered receipt
comes back as a verdict with failed domains and `reasons`. No network is needed
for any domain except chaining a hardware quote to its vendor root
(`options.quoteVerifier`) or confirming a Bitcoin anchor (`options.blockHeaders`).

## The verdict

```ts
interface Verdict {
  ok: boolean;                     // never a bare boolean elsewhere
  schema: "cool.receipt.v2";
  subject: { kind: "evidence" | "change"; subject; issued_at; record_id; key_id; tee } | null;
  checks: {
    binding; signature; inclusion; witnesses; attestation; enclave; anchor;
  };                               // each: { status, detail }
  reasons: string[];
}
```

`status` is one of `pass | fail | absent | mock | simulated | pending`.

## The seven domains

| Domain | Passes when | Notes |
|---|---|---|
| `binding` | `mh(canonicalCBOR(core))` recomputes to `binding_hash` | pure maths |
| `signature` | **both** ML-DSA-65 and Ed25519 verify over `core ‖ binding_digest`, against `key_directory[key_id]` | pure maths |
| `inclusion` | the audit path reconstructs `sth.root_hash` and the STH signature verifies | `absent` if the receipt carries no log proof |
| `witnesses` | ≥ `witnessThreshold` **external** co-signatures verify | a CooL self-signature is shown, never counted; `absent` in this build |
| `attestation` | a real quote verifier chains the quote to a real vendor root | `simulated` for the CooL sim root; `absent` if a hardware quote is present but no verifier was supplied; `mock` if no quote |
| `enclave` | quote digest == `runtime.tee_quote`, measurements match, `report_data` == commitment to the signing key, and the pinned measurement (if any) matches | this is the domain that ties a quote to *this* record |
| `anchor` | the recomputed commitment equals a real Bitcoin block's merkle root | `pending` between submission and aggregation; `absent` if never anchored |

## `ok`

```
ok  =  binding == pass
   &&  signature == pass
   &&  inclusion ∈ { pass, absent }
   &&  enclave != fail
   &&  attestation != fail
   &&  (not options.requireHardware  ||  attestation == pass)
```

`witnesses` and `anchor` never make `ok` true on their own, and `simulated` /
`mock` / `pending` are never `pass`.

## Options

| Option | Effect |
|---|---|
| `expectedMeasurement` | pin the image you approved; any mismatch fails `enclave` |
| `requireHardware` | a receipt that is not backed by a verified hardware quote cannot be `ok` |
| `witnessThreshold` | minimum independent witness co-signatures for `witnesses` to pass |
| `quoteVerifier` | chains a hardware quote to Intel DCAP / AMD KDS / NVIDIA NRAS (`remoteQuoteVerifier` from `cool-nwc/phala`) |
| `blockHeaders` | a Bitcoin block-header source for confirming an anchor |

## Cryptographic validity vs. policy

The verdict answers "is this receipt authentic and internally consistent?". It
does **not** answer "do I accept this measurement / this signer / this vendor?".
A receipt can be cryptographically valid and still policy-rejected — pin a
measurement, require hardware, or check `subject.key_id` against your own
allow-list.

## CLI

```sh
cool verify evidence.json               # one file
cool verify all                         # every receipt in .cool/receipts
cool verify last --require-hardware     # fail unless a real quote backs it
```

Exit codes: `0` verified · `1` verification failed · `2` bad configuration ·
`3` environment unavailable · `4` usage error.
