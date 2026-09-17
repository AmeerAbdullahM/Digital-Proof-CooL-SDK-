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
*Note: CooL records what occurred, but it does not grade it. The receipt does not prove that an output was "correct" or "fair", only that it happened exactly as recorded.*

## ❓ The Problem We Solve

In modern regulated systems and AI workflows, answering forensic questions post-execution is critical:
> What software ran? Which model was used? In what environment? Has the record been tampered with? Can a third party verify this without accessing our raw data or internal logs?

Standard logs can be edited, and screenshots can be faked. CooL acts as a cryptographic evidence layer that transforms these questions into mathematically checkable statements.

## 🚀 Installation

Install via npm:
```sh
npm install cool-nwc
```
*Requires Node **≥ 20** and ESM. TypeScript declarations are included out of the box. There are no native build steps, network calls at runtime/install, or postinstall scripts.*

## ⚡ Interactive Hackathon Walkthrough

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

### Step 2: Run your script
In your terminal, execute the script by running:
```sh
npx tsx generate.ts
```
*You will now see a new file called `evidence.json` in your folder. If you open it, you'll see the complex cryptography inside, but you won't see your prompt or response!*

### Step 3: Verify it manually using the CLI
Now, pretend you are the hackathon judge checking the bounty hunter's proof. Run the built-in verifier against the file you just created:
```sh
npx tsx src/cli/index.ts verify evidence.json
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

## 🏗️ System Architecture

```text
                        APPLICATION
                            |
                            v
                    +---------------+
                    |   CooL SDK    |   new CooL({...}).record({...})
                    +-------+-------+
                            |  evidence event (metadata + optional payloads)
                            v
                    +-------+--------+
                    | Evidence Plane |   runs inside the enclave in production
                    +---+--------+---+
                       /          \
              commit  /            \  attest
                     v              v
              +----------+     +-----------+
              |  CRYPTO  |     |  dstack   |
              | sha256   |     | TDX quote |
              | ML-DSA65 |     | sealed key|
              | Ed25519  |     +-----+-----+
              +----+-----+           |
                   |  sign            |
                   v                  v
              +--------------------------+
              |     EVIDENCE RECORD      |   cool.evidence.v1
              +------------+-------------+
                           |  append
                           v
                  +------------------+
                  |  TRANSPARENCY    |   RFC 6962 log, signed tree head
                  |  (+ optional     |
                  |   OTS anchor)    |
                  +--------+---------+
                           |
                           v
                  +------------------+
                  |    VERIFIER      |   offline · 7 domains · never a bare bool
                  +------------------+
```

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
| **Signer Substitution** | Signatures are strictly bound to a `key_id` in the key directory | Key directory trust management |
| **Falsified TEE Claims** | Quote digest and enclave measurement are sealed inside the signed core | Hardware-level vulnerabilities (e.g., Intel TDX bugs) |
| **Quote Stapling (Re-use)** | Quote digest must match `runtime.tee_quote`, and `report_data` must commit to the signing key | — |
| **Transparency Log Tampering** | RFC 6962 inclusion and consistency proofs | Root trust (currently lacks external witnesses) |
| **Simulator Spoofing Hardware** | Receipts inject `mode: "simulated"` and fail hardware-required checks | Operator manually disabling `requireAttestation` |

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

CooL strictly typed. No public APIs utilize `any`. Type declarations (`.d.ts`) are shipped with the package and tested against `"nodenext"` and `"bundler"` module resolutions via CI (`npm run verify:package`).

## 🗺️ Project Roadmap

| Phase | Milestone |
|---|---|
| **Shipped** | Hybrid signatures, RFC 6962 log, offline verifier, evidence records, OpenTimestamps anchor, dstack HTTP/Simulator, TDX quote binding, `cool` CLI |
| **Experimental** | Bitcoin anchor confirmation, remote quote verification via Intel DCAP |
| **Planned** | JSON Schema publication for `cool.evidence.v1`, hosted verifier service, NVIDIA GPU attestation, external gossip/witnesses |
| **Research** | Formal verification of the canonicalization and core verifier |


