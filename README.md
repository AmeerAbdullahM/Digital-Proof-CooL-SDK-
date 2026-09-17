## 📖 Introduction to CooL

CooL provides a developer SDK to generate **independently verifiable evidence** regarding software execution.
By calling `record()` within your application, CooL produces a self-contained, offline-verifiable receipt. This receipt requires no central trust or user account, and it proves:

* **The exact software** that executed (including its name, version, and content digest).
* **The specific event** that occurred (represented by a dotted type and a salted cryptographic commitment to your metadata).
* **Authenticity and Integrity**, guaranteed by a hybrid signature combining classical Ed25519 and post-quantum ML-DSA-65 over deterministic data.
* **Log Inclusion**, via an RFC 6962 inclusion proof under a signed tree head ensuring append-only properties.
* **Hardware Attestation (TEE)**, confirming exactly where the code ran using enclave measurements and a TDX quote whose `report_data` commits to the signing key.

Crucially, **your private data, prompts, and outputs are never recorded**. CooL commits to sensitive values using salted hashes and immediately discards the plaintext.

```text
OBSERVE → COMMIT → SIGN → ATTEST → ANCHOR → VERIFY
```

> ⚠️ **What CooL does *not* prove.** A receipt establishes that a specific piece of software ran, did a specific thing, in a specific place, and that the record hasn't been altered since. It does **not** grade the output — a receipt can prove a biased, wrong, or harmful response was generated exactly as claimed, just as easily as it proves a good one was. Treat CooL as an evidence layer, not a correctness or fairness guarantee. See [Threat Model](#️-security-posture--threat-model) for what's explicitly out of scope.

## ❓ The Problem We Solve

In modern regulated systems and AI workflows, answering forensic questions post-execution is critical:
> What software ran? Which model was used? In what environment? Has the record been tampered with? Can a third party verify this without accessing our raw data or internal logs?

Standard logs can be edited, and screenshots can be faked. CooL acts as a cryptographic evidence layer that transforms these questions into mathematically checkable statements.

📎 New to the project? See [ABOUT.md](ABOUT.md) for the motivation, design philosophy, and what CooL is deliberately *not* trying to be.

## 🚀 Installation

Install via npm:
```sh
npm install cool-nwc
```
*Requires Node **≥ 20** and ESM. TypeScript declarations are included out of the box. There are no native build steps, network calls at runtime/install, or postinstall scripts.*

For the CLI commands used below (`cool verify`, `cool doctor`, …), install the CLI globally as well:
```sh
npm install -g cool-nwc
```

## ⚡ Interactive Hackathon Walkthrough

This walkthrough uses two commands throughout: your own `generate.ts` script (run with `npx tsx`) to produce evidence, and the globally-installed `cool` CLI to verify it. If you'd rather not install the CLI globally, substitute `npx tsx src/cli/index.ts` for every `cool` command below — the two are equivalent.

### Step 1: Create a generation script
Create a new file in your folder called `generate.ts` and paste this code into it:

```typescript
import { CooL } from "./src/index.js";
import * as fs from "node:fs/promises";

async function run() {
  // 1. Initialize the CooL SDK for your app
  const cool = new CooL({ applicationId: "bias-bounty-demo" });

  console.log("Generating cryptographic receipt for AI execution...");

  // 2. Record an event (like an AI model responding to a prompt)
  const { evidence } = await cool.record({
    type: "model.execution",
    metadata: {
      model: "gpt-4",
      status: "completed"
    },
    // Payloads are mathematically hashed. The actual text NEVER goes into the receipt.
    payloads: {
      input: "Tell me a joke",
      output: "Here is a biased joke..."
    }
  });

  // 3. Save the receipt to a file
  await fs.writeFile("evidence.json", JSON.stringify(evidence, null, 2));
  console.log("Evidence generated and saved to 'evidence.json'!");
}

run();
```

> Note on payload confidentiality: salts are stored *inside* the receipt next to each commitment, so they don't stop a low-entropy payload (short phrases, categorical labels, yes/no answers) from being guessed by an attacker who already has the receipt. They only defeat precomputed dictionary attacks across *many* records. If a payload's plaintext space is small, don't rely on the commitment alone to keep it secret — see [Security Considerations](#-security-considerations).

### Step 2: Run your script
In your terminal, execute the script by running:
```sh
npx tsx generate.ts
```
*You will now see a new file called `evidence.json` in your folder. If you open it, you'll see the complex cryptography inside, but you won't see your prompt or response!*

### Step 3: Verify it manually using the CLI
Now, pretend you are the hackathon judge checking the bounty hunter's proof. Run the built-in verifier against the file you just created:
```sh
cool verify evidence.json
```
*It should output a green table saying **VERIFIED**!*

### Step 4: The Heist! (Tamper with the evidence)
1. Open `evidence.json` in VSCode.
2. Find the line that says `"metadata_hash": "mh:sha256:..."`
3. Change just **one single letter or number** in that hash. Save the file.
4. Run the verifier command from Step 3 again.

It will immediately throw a **FAILED** alert because the cryptographic signatures no longer match the altered hash. The math caught the lie!

**Bonus Tip:** If you ever just want to run an interactive 3-minute tutorial in your terminal that explains all of this conceptually, you can run:
```sh
npx tsx src/cli/index.ts walkthrough
```

### Which mode am I in?

By default (no `dstack` socket configured), CooL runs in **simulator mode** — you get real cryptographic signatures over a simulated hardware quote, and the verifier explicitly reports `mode: "simulated"` for anything hardware-dependent. This is the right mode if you just want a tamper-evident audit trail and don't need to prove *where* the code ran. See [☁️ Phala dstack Integration](#️-phala-dstack-integration) for turning on real TEE attestation.

## 🏗️ System Architecture

Every call to `record()` moves through the same six stages, whether or not real TEE hardware is present:

<p align="center">
<svg viewBox="0 0 860 110" width="100%" role="img" aria-label="Six stage pipeline: observe, commit, sign, attest, anchor, verify">
  <defs>
    <marker id="cool-arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="#57606a" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>
  <rect x="20" y="20" width="120" height="60" rx="10" fill="#f6f8fa" stroke="#57606a"/>
  <text x="80" y="45" text-anchor="middle" font-size="13" font-weight="600" fill="#1f2328">Observe</text>
  <text x="80" y="62" text-anchor="middle" font-size="10" fill="#57606a">SDK call</text>
  <line x1="140" y1="50" x2="158" y2="50" stroke="#57606a" stroke-width="1.5" marker-end="url(#cool-arrow)"/>

  <rect x="160" y="20" width="120" height="60" rx="10" fill="#ddf4ff" stroke="#0969da"/>
  <text x="220" y="45" text-anchor="middle" font-size="13" font-weight="600" fill="#0969da">Commit</text>
  <text x="220" y="62" text-anchor="middle" font-size="10" fill="#0969da">salted hash</text>
  <line x1="280" y1="50" x2="298" y2="50" stroke="#57606a" stroke-width="1.5" marker-end="url(#cool-arrow)"/>

  <rect x="300" y="20" width="120" height="60" rx="10" fill="#ddf4ff" stroke="#0969da"/>
  <text x="360" y="45" text-anchor="middle" font-size="13" font-weight="600" fill="#0969da">Sign</text>
  <text x="360" y="62" text-anchor="middle" font-size="10" fill="#0969da">hybrid signature</text>
  <line x1="420" y1="50" x2="438" y2="50" stroke="#57606a" stroke-width="1.5" marker-end="url(#cool-arrow)"/>

  <rect x="440" y="20" width="120" height="60" rx="10" fill="#fbefff" stroke="#8250df"/>
  <text x="500" y="45" text-anchor="middle" font-size="13" font-weight="600" fill="#8250df">Attest</text>
  <text x="500" y="62" text-anchor="middle" font-size="10" fill="#8250df">TDX or simulated</text>
  <line x1="560" y1="50" x2="578" y2="50" stroke="#57606a" stroke-width="1.5" marker-end="url(#cool-arrow)"/>

  <rect x="580" y="20" width="120" height="60" rx="10" fill="#fbefff" stroke="#8250df"/>
  <text x="640" y="45" text-anchor="middle" font-size="13" font-weight="600" fill="#8250df">Anchor</text>
  <text x="640" y="62" text-anchor="middle" font-size="10" fill="#8250df">RFC 6962 log</text>
  <line x1="700" y1="50" x2="718" y2="50" stroke="#57606a" stroke-width="1.5" marker-end="url(#cool-arrow)"/>

  <rect x="720" y="20" width="120" height="60" rx="10" fill="#dafbe1" stroke="#1a7f37"/>
  <text x="780" y="45" text-anchor="middle" font-size="13" font-weight="600" fill="#1a7f37">Verify</text>
  <text x="780" y="62" text-anchor="middle" font-size="10" fill="#1a7f37">offline verdict</text>
</svg>
</p>

<sub>Rendered as static SVG so it displays inline on GitHub without JavaScript. `Verify` runs completely offline against the receipt alone — nothing above needs to be re-contacted.</sub>

## 📄 Evidence Record Format

The `cool.evidence.v1` schema is delivered within a `cool.receipt.v2` envelope:

```jsonc
{
  "schema": "cool.receipt.v2",
  "record": {
    "schema": "cool.evidence.v1",
    "record_id": "01J…",                 // ULID
    "time": { "issued_at": "…Z", "seq": 0 },
    "event": {
      "type": "model.execution",
      "application_id": "my-app",
      "execution_id": "01J…",
      "metadata_hash": "mh:sha256:…",     // salted commitment to canonical(metadata)
      "metadata_salt": "hex:…",
      "commitments": {                    // optional salted commitments; plaintext discarded
        "input": "mh:sha256:…",  "input_salt": "hex:…",
        "output": "mh:sha256:…", "output_salt": "hex:…",
        "state": null, "state_salt": null
      },
      "software": { "name": "my-service", "version": "2.3.1", "digest": null }
    },
    "runtime": { "tee_vendor": "intel-tdx", "mode": "simulated", "enclave_measurement": {…}, "tee_quote": "mh:sha256:…", "gpu": null },
    "signature": { "alg": "ml-dsa-65+ed25519", "key_id": "…", "ml_dsa": "base64:…", "ed25519": "base64:…" }
  },
  "binding_hash": "mh:sha256:…",          // mh(canonicalCBOR(core))
  "inclusion": { "leaf_index": 0, "tree_size": 1, "audit_path": [] },
  "sth": {…},                             // RFC 6962 signed tree head
  "attestation": {…},                     // quote + pinned measurement + key binding
  "anchor": null,                         // optional OpenTimestamps proof
  "key_directory": { "…": { "ml_dsa_pub": "base64:…", "ed25519_pub": "base64:…" } }
}
```

**Reading the envelope visually** — the blue block is the signed evidence record; everything in gray around it is what lets a verifier check that record without calling home:

<p align="center">
<svg viewBox="0 0 860 340" width="100%" role="img" aria-label="Nested structure of the receipt envelope">
  <rect x="20" y="20" width="820" height="300" rx="14" fill="#fbefff" stroke="#8250df"/>
  <text x="40" y="44" font-size="13" font-weight="600" fill="#8250df">cool.receipt.v2</text>
  <text x="40" y="60" font-size="10" fill="#8250df">the outermost, fully self-contained envelope</text>

  <rect x="40" y="76" width="380" height="220" rx="10" fill="#ddf4ff" stroke="#0969da"/>
  <text x="56" y="98" font-size="12" font-weight="600" fill="#0969da">record — cool.evidence.v1</text>

  <rect x="56" y="110" width="348" height="40" rx="8" fill="#f6f8fa" stroke="#57606a"/>
  <text x="70" y="126" font-size="11" fill="#1f2328">event</text>
  <text x="70" y="141" font-size="9.5" fill="#57606a">type, application_id, metadata_hash + salt</text>

  <rect x="56" y="156" width="348" height="40" rx="8" fill="#f6f8fa" stroke="#57606a"/>
  <text x="70" y="172" font-size="11" fill="#1f2328">commitments</text>
  <text x="70" y="187" font-size="9.5" fill="#57606a">salted hashes of input / output / state</text>

  <rect x="56" y="202" width="348" height="40" rx="8" fill="#f6f8fa" stroke="#57606a"/>
  <text x="70" y="218" font-size="11" fill="#1f2328">runtime</text>
  <text x="70" y="233" font-size="9.5" fill="#57606a">tee_vendor, mode, enclave_measurement</text>

  <rect x="56" y="248" width="348" height="34" rx="8" fill="#f6f8fa" stroke="#57606a"/>
  <text x="70" y="269" font-size="11" fill="#1f2328">signature — ml-dsa-65 + ed25519</text>

  <rect x="440" y="76" width="380" height="50" rx="10" fill="#f6f8fa" stroke="#57606a"/>
  <text x="456" y="97" font-size="11" fill="#1f2328">binding_hash</text>
  <text x="456" y="112" font-size="9.5" fill="#57606a">mh(canonicalCBOR(core))</text>

  <rect x="440" y="136" width="380" height="50" rx="10" fill="#f6f8fa" stroke="#57606a"/>
  <text x="456" y="157" font-size="11" fill="#1f2328">inclusion + sth</text>
  <text x="456" y="172" font-size="9.5" fill="#57606a">RFC 6962 proof under a signed tree head</text>

  <rect x="440" y="196" width="380" height="50" rx="10" fill="#f6f8fa" stroke="#57606a"/>
  <text x="456" y="217" font-size="11" fill="#1f2328">attestation</text>
  <text x="456" y="232" font-size="9.5" fill="#57606a">TDX quote + pinned measurement (or "simulated")</text>

  <rect x="440" y="256" width="380" height="40" rx="10" fill="#f6f8fa" stroke="#57606a"/>
  <text x="456" y="272" font-size="11" fill="#1f2328">key_directory</text>
  <text x="456" y="287" font-size="9.5" fill="#57606a">public keys needed to verify offline</text>
</svg>
</p>

**Replay protection.** `execution_id`, the monotone `time.seq`, and `time.issued_at` together are what a verifier should check to detect replay — `record_id` alone (a ULID) only guarantees uniqueness, not freshness. There's no built-in freshness *policy* (e.g. "reject anything older than 5 minutes"); that's left to your application, since acceptable staleness is domain-specific. If you don't set one, a captured-and-replayed evidence event will still verify.

For a detailed breakdown of fields, refer to the [Full Documentation](docs/full_documentation.md).

## 🔐 Cryptography Implementation

| Primitive | Function | Notes |
|---|---|---|
| **SHA-256** | Merkle trees, commitments, quote/report-data digests | Implemented via `@noble/hashes` |
| **Salted Commitments** | `mh:sha256(salt ‖ bytes)` for metadata & payloads | Uses 16-byte CSPRNG salt stored alongside the commitment for future disclosure |
| **Canonical CBOR** | The precise byte array that is hashed and signed | RFC 8949 CDE; deterministic across implementations |
| **Ed25519** | Classical cryptographic signature | Implemented via `@noble/curves` |
| **ML-DSA-65 (FIPS 204)** | Post-quantum cryptographic signature | Implemented via `@noble/post-quantum` |
| **Hybrid Verification** | Requires **both** signature schemes to pass | Maintains security even if one algorithm is compromised in the future |
| **RFC 6962 Merkle** | Consistency and inclusion proofs | Modifying any log entry will immediately alter the root |

*Important:* The ML-DSA/Ed25519 hybrid signatures protect individual *records* against future cryptographic breaks. However, this does **not** classify the entire CooL system as a "quantum-safe system."

*Size note:* ML-DSA-65 signatures and public keys are considerably larger than Ed25519's (roughly 3.3 KB and 2 KB respectively, vs. ~64 bytes and ~32 bytes for Ed25519). At high record volumes this affects storage and transfer cost — budget for it, or consider batching records and deferring the PQ signature to an anchor/checkpoint step rather than signing every single record twice.

## 🛡 Security Considerations

A few properties that are easy to assume the receipt gives you, but which need explicit handling:

- **Salt disclosure ≠ payload secrecy.** Salts ship inside the receipt, not behind it. They stop precomputed rainbow-table attacks across many records, but not a targeted guess against one record with a small plaintext space (a single word, a category, a yes/no). If a payload's true entropy is low, pad it, blind it, or don't commit it directly — the commitment alone won't keep it confidential from someone holding the receipt.
- **Key validity windows.** The key directory tells a verifier *which* key signed a record, but the current schema doesn't carry an explicit validity period for that key. If you rotate signing keys (which dstack does automatically on image changes), keep an out-of-band record of when each `key_id` was valid so a verifier can tell a legitimately-old signature from one made with a since-revoked key.
- **No external witnesses yet.** The RFC 6962 log gives you consistency and inclusion proofs, but a single log operator could in principle show two different, both-internally-consistent views of the log to two different parties (a "split-view" attack). This is listed as a known residual risk in the threat model below; if your use case needs resistance to a dishonest log operator, don't rely on this alone until external gossip/witnessing ships.
- **Freshness is your job.** See the note under [Evidence Record Format](#-evidence-record-format) — CooL gives you the fields to detect replay, not a default policy for how stale is too stale.

## ☁️ Phala dstack Integration

Without `dstack`, CooL defaults to simulator mode, generating real signatures over simulated quote structures. The verifier will explicitly report `simulated` for hardware-dependent domains.

When running with `dstack` in an Intel TDX confidential VM, it provides:
* **Attested Workload Identity**: Includes MRTD + RTMR measurements inside the signed record.
* **TEE Quote**: A quote containing a `report_data` block that commits to the signing key (proving "this key is held by this enclave").
* **Measurement-Sealed Signing Key**: Automatically derived inside the enclave, rotating when the image changes.

```ts
const cool = new CooL({
  applicationId: "refund-agent",
  attestation: { provider: "dstack", endpoint: "/var/run/dstack.sock", vendor: "intel-tdx" },
  security: { requireAttestation: true }, // Verification and execution will fail without a genuine quote
});
```
*CooL utilizes the `@phala/dstack` guest-agent protocol (`/var/run/dstack.sock`) but does not create the TEE itself or bundle the full dstack SDK.*

## 🛡️ Security Posture & Threat Model

CooL is designed to be **fail-open for your app, and fail-closed for verification**. If the transparency service goes offline, your application continues to function, but a verifier will never pass a check it cannot validate.

| Threat Scenario | CooL Mitigation | Remaining Risk |
|---|---|---|
| **Post-facto Evidence Modification** | Canonical CBOR binding hash + hybrid signatures | Signing key compromise |
| **Record Replay Attacks** | `record_id` (ULID) + `execution_id` + `issued_at` + monotone `seq` | Dependent on application policy freshness windows |
| **Signer Substitution** | Signatures are strictly bound to a `key_id` in the key directory | Key directory trust management; no built-in key validity window (see Security Considerations) |
| **Falsified TEE Claims** | Quote digest and enclave measurement are sealed inside the signed core | Hardware-level vulnerabilities (e.g., Intel TDX bugs) |
| **Quote Stapling (Re-use)** | Quote digest must match `runtime.tee_quote`, and `report_data` must commit to the signing key | — |
| **Transparency Log Tampering** | RFC 6962 inclusion and consistency proofs | Root trust (currently lacks external witnesses) |
| **Simulator Spoofing Hardware** | Receipts inject `mode: "simulated"` and fail hardware-required checks | Operator manually disabling `requireAttestation` |
| **Low-entropy Payload Guessing** | Salted commitments discard plaintext | Salt is disclosed in the receipt, so small plaintext spaces remain guessable (see Security Considerations) |

*Note: CooL does not protect against flawed application logic generating valid evidence of incorrect actions, compromised dependencies, or TEE hardware bugs.*

## 📦 SDK API Modules

| Import Path | Description |
|---|---|
| `cool-nwc` | Core `CooL` class, `verifyEvidence`, typed errors, `formatVerdict`, and primitives |
| `cool-nwc/verify` | Lightweight standalone verifier |
| `cool-nwc/phala` | Advanced toolkit (`CoolTee`, dstack clients, policies, anchors, quotes) |
| `cool-nwc/node` | Unix-socket transport and filesystem-backed transparency logs |
| `cool-nwc/tee` | Comprehensive single import encompassing all modules |

### Initialization: `new CooL(options?)`

| Option | Type | Default Value |
|---|---|---|
| `applicationId` | `string` | `"cool-app"` |
| `attestation.provider` | `"local" \| "dstack"` | `"local"` (switches to `"dstack"` if `$COOL_DSTACK_ENDPOINT` is present) |
| `attestation.endpoint` | `string` | Falls back to `/var/run/dstack.sock` |
| `security.requireAttestation`| `boolean` | `false` |
| `security.expectedMeasurement`| `Measurement` | — |

### Recording Evidence

```ts
// cool.record(input) → { evidence, recordId, executionId, digest }
```
Expects `{ type, executionId?, metadata?, payloads?, software?, gpu? }`. Plaintext payloads and metadata are replaced with salted hashes and discarded.

### Verifying Evidence

```ts
// cool.verify(evidence, options?) → Verdict
// verifyEvidence(evidence, options?)
```
Returns a comprehensive Verdict object rather than a simple boolean:
```ts
{ ok: false,
  checks: { binding, signature, inclusion, witnesses, attestation, enclave, anchor },
  reasons: ["signature: ML-DSA-65 did not verify …"] }
```

**A "degraded but valid" example.** Not every failing check means the record is fraudulent — a receipt made in simulator mode, for instance, will legitimately fail hardware-only checks:
```ts
{ ok: true,           // binding, signature, and inclusion all check out
  checks: {
    binding:     { ok: true },
    signature:   { ok: true },
    inclusion:   { ok: true },
    witnesses:   { ok: true,  detail: "no external witnesses configured" },
    attestation: { ok: false, detail: "mode: simulated — no hardware quote to verify" },
    enclave:     { ok: false, detail: "mode: simulated" },
    anchor:      { ok: null,  detail: "no OpenTimestamps anchor present on this record" }
  },
  reasons: [] }
```
Here `ok: true` at the top level reflects that everything the record *claims* to provide checks out cryptographically — it simply never claimed hardware attestation. Always read `checks` alongside `ok`, not just the top-level boolean, to know exactly what was and wasn't proven.

## 💻 The Command-Line Interface (`cool`)

Access the global CLI by running `npm install -g cool-nwc`:

```sh
cool verify evidence.json     # Verify evidence offline without an enclave or account
cool doctor                   # Diagnostics: checks Node version, web crypto, dstack socket
cool walkthrough              # Interactive 3-minute tutorial
cool seal …                   # Advanced CLI tools
```
*`cool verify` outputs a non-zero exit code upon failure, perfect for gating CI/CD jobs.*

## 📂 Example Projects

Check out the [`examples/`](examples) directory for runnable implementations (`basic`, `verification`, `express`, `agent`, `dstack`). Run `npm install && npm start` inside any folder.

## 📚 Documentation

All comprehensive documentation has been aggregated into one easy-to-read document:
👉 **[View Full Documentation](docs/full_documentation.md)**

## ⌨️ TypeScript Support

CooL is strictly typed. No public APIs utilize `any`. Type declarations (`.d.ts`) are shipped with the package and tested against `"nodenext"` and `"bundler"` module resolutions via CI (`npm run verify:package`).

## 🗺️ Project Roadmap

| Phase | Milestone |
|---|---|
| **Shipped** | Hybrid signatures, RFC 6962 log, offline verifier, evidence records, OpenTimestamps anchor, dstack HTTP/Simulator, TDX quote binding, `cool` CLI |
| **In progress** | JSON Schema publication for `cool.evidence.v1` (enables independent verifier reimplementations), key validity-window metadata |
| **Experimental** | Bitcoin anchor confirmation, remote quote verification via Intel DCAP |
| **Planned** | Hosted verifier service, NVIDIA GPU attestation, external gossip/witnesses for the transparency log |
| **Research** | Formal verification of the canonicalization and core verifier |

---
See [ABOUT.md](ABOUT.md) for the project's motivation and design philosophy.
