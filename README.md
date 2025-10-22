# **Vigil & Cofactor: A Schema-Driven Framework for Verifiable Computational Science**

## Abstract

Reproducibility remains the cornerstone of scientific progress, yet computational research today suffers from a crisis of provenance: results are often irreproducible, unverifiable, or untraceable.
Code evolves, data shifts, and environments decay — leaving most published computational results unverifiable within months.

**Vigil** is a schema-driven reproducibility protocol and proof engine designed to solve this problem at the source.
It introduces the concept of a **cryptographically verifiable receipt** for every computation, providing an immutable, machine-verifiable record of *what was run, on what data, with which environment, by whom, and when*.

**Cofactor** is the broader scientific infrastructure and platform that operationalizes Vigil — integrating secure provenance, collaborative sharing, and version-controlled data workflows into a unified environment for computational research and AI.

Together, they form the foundation of **Verifiable Computational Science**: a world where every experiment can be independently reproduced and proven, not merely claimed.

---

## 1. Introduction

### 1.1 The Reproducibility Problem

Modern science increasingly depends on computation.
Yet, reproducibility in computational research lags far behind theoretical or experimental standards.
Key challenges include:

* **Ephemeral environments** — versions of code, dependencies, and models vanish.
* **Hidden data lineage** — datasets evolve without traceable histories.
* **Opaque results** — published results lack cryptographic provenance.
* **Broken workflows** — recomputation is complex and non-deterministic.

Existing tools (Git, MLflow, DVC, etc.) capture fragments of this problem — version control, artifact management, or pipeline orchestration — but none solve **verifiable provenance**: proving that a given result truly came from a given computation.

---

## 2. The Vigil Model

Vigil introduces a new primitive for science: the **receipt** —
a signed, canonical, cryptographically verifiable record of a computation.

A receipt answers five questions:

1. **Who** performed the computation?
2. **What** code and data were used?
3. **Where** and **when** was it run?
4. **How** was the environment configured?
5. **Why** (or under what experiment/project) was it executed?

### 2.1 Schemas as Law, Receipts as Evidence

At the heart of Vigil is a simple philosophy:

> **Schemas are law. Receipts are evidence.**

All Vigil data — projects, runs, artifacts, datasets, and receipts — are defined by **canonical JSON Schemas**, versioned under semantic versioning (v1, v2, …).
Every layer of the stack — from database to API to client SDK — is generated from or validated against these schemas.
This ensures structural and semantic consistency across all systems, languages, and users.

### 2.2 The Receipt Loop

Vigil provides an end-to-end reproducibility loop:

```
vigil run       → capture inputs, outputs, environment
vigil promote   → canonicalize JSON, sign receipt (Ed25519)
vigil verify    → re-hash outputs, verify signature
vigil push      → upload to backend (API)
vigil pull      → re-download and re-verify anywhere
```

This loop can run entirely offline and produces portable, verifiable receipts that can be validated decades later, independent of Vigil itself.

---

## 3. Cofactor: The Scientific Platform

While Vigil defines the *protocol*, **Cofactor** provides the *infrastructure*.

Cofactor is a distributed research platform that implements Vigil as its core proof layer.
It combines secure data management, computational orchestration, and AI-assisted research tooling into one cohesive environment.

### 3.1 Purpose

Cofactor’s mission is to make *every scientific result verifiable*.
It provides researchers and institutions with:

* A **backend registry** for verified computational receipts.
* A **frontend workspace** for browsing, comparing, and validating results.
* A **CLI and SDK suite** (via Vigil) for experiment capture and proof submission.
* Integration with object storage and compute backends for large-scale reproducible science.

### 3.2 Layers of the Cofactor Stack

| Layer                   | Component                                        | Description                                          |
| ----------------------- | ------------------------------------------------ | ---------------------------------------------------- |
| **Proof layer**         | Vigil                                            | Cryptographic receipts and schemas                   |
| **Storage layer**       | Postgres + S3                                    | Provenance graph + artifact persistence              |
| **Application layer**   | Fastify API + Next.js UI                         | OpenAPI interface + researcher dashboard             |
| **Compute layer**       | External orchestrators (Slurm, Kubernetes, etc.) | Optional reproducible execution                      |
| **Collaboration layer** | Cofactor Cloud                                   | Authentication, permissions, sharing, AI integration |

---

## 4. System Architecture

Cofactor’s architecture follows a **schema-driven, contract-verified** model.

```
User CLI (vigil-science)
│
├── vigil-core      → Offline proof engine (run, promote, verify)
├── vigil-client    → Network bridge (push, pull, login)
│
▼
apps/api            → Fastify backend (Prisma + Postgres)
│                    → Validates receipts via JSON Schema
│                    → Exposes OpenAPI 3.1
│
▼
apps/app            → Next.js frontend
                     → Uses autogenerated TypeScript SDK
                     → Browse + verify receipts and artifacts
│
▼
packages/vigil-schemas → Canonical JSON Schemas + fixtures + OpenAPI
```

Each layer is *schema-aware* and *version-controlled*.
If any component drifts from the canonical spec, CI fails.

---

## 5. The Vigil Components

### 5.1 vigil-core — Offline Proof Engine

* Executes experiments and captures metadata.
* Canonicalizes JSON receipts (JCS).
* Signs receipts with Ed25519 keypairs.
* Verifies reproducibility via re-hashing and schema validation.
* Operates offline; no network dependencies.

**Example:**

```bash
vigil run python train.py
vigil promote
vigil verify
```

Output:

```
.vigil/receipts/rcpt-8d3f.json
```

### 5.2 vigil-client — Network Bridge

* Authenticates via OAuth (Clerk) or service tokens.
* Uploads receipts, artifacts, and datasets to the Cofactor backend.
* Downloads verified results and receipts for local validation.
* Uses autogenerated Python SDK (from OpenAPI 3.1).
* Ensures end-to-end integrity between client and backend.

**Example:**

```bash
vigil login
vigil push
vigil pull receipt rcpt-8d3f
```

---

## 6. Schema-Driven Contract Model

### 6.1 Canonical JSON Schemas

All entities (Project, Run, Artifact, Receipt) are defined as versioned JSON Schemas.

```json
{
  "$id": "https://vigil.dev/schemas/v1/Receipt.schema.json",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$version": "1.0.0",
  "type": "object",
  "required": ["id", "project_id", "created_at", "signature"],
  "properties": {
    "id": { "type": "string" },
    "project_id": { "type": "string" },
    "created_at": { "type": "string", "format": "date-time" },
    "signature": { "type": "object" }
  }
}
```

### 6.2 OpenAPI Contract

The backend exports a fully schema-referenced OpenAPI 3.1 spec, which serves as the contract for SDK generation.

Example endpoint:

```yaml
post:
  summary: Upload a signed receipt
  operationId: uploadReceipt
  requestBody:
    content:
      application/json:
        schema:
          $ref: "./v1/Receipt.schema.json"
  responses:
    201:
      description: Receipt uploaded successfully
      content:
        application/json:
          schema:
            $ref: "./v1/Receipt.schema.json"
```

### 6.3 Prisma Implementation

Prisma defines the database models mirroring those same schemas:

```prisma
model Receipt {
  id          String   @id @default(cuid())
  projectId   String
  body        Json
  sha256      String
  createdAt   DateTime @default(now())
  project     Project  @relation(fields: [projectId], references: [id])
}
```

CI checks Prisma → JSON Schema → OpenAPI consistency.
If drift occurs, builds fail.

---

## 7. The Proof Lifecycle

| Stage                | Tool           | Description                                     |
| -------------------- | -------------- | ----------------------------------------------- |
| **Run**              | `vigil-core`   | Capture command, inputs, outputs, and env.      |
| **Promote**          | `vigil-core`   | Canonicalize and sign receipt.                  |
| **Verify (local)**   | `vigil-core`   | Rehash, verify signature, validate schema.      |
| **Push**             | `vigil-client` | Upload via OpenAPI → backend (validated again). |
| **Persist**          | `apps/api`     | Store in Postgres JSONB + S3 artifact store.    |
| **View / Verify**    | `apps/app`     | Display and re-verify client-side.              |
| **Pull / Reproduce** | `vigil-client` | Re-download + re-validate receipt locally.      |

This chain provides *provable reproducibility* — the computational analog of a laboratory chain-of-custody.

---

## 8. Security & Integrity Model

| Mechanism                                 | Purpose                                                   |
| ----------------------------------------- | --------------------------------------------------------- |
| **JSON Canonicalization (JCS)**           | Ensures deterministic byte order before hashing/signing.  |
| **Hashing (SHA-256)**                     | Produces stable digests for receipts and artifacts.       |
| **Signing (Ed25519)**                     | Provides cryptographic proof of authorship and integrity. |
| **Anchoring (Merkle / Transparency Log)** | Optional on-chain or off-chain timestamping.              |
| **Schema Validation**                     | Prevents structural tampering.                            |
| **Immutable Audit Log**                   | Local `.vigil/audit.log` tracks all actions.              |

---

## 9. End-to-End Example

1. **Researcher runs a local experiment:**

   ```bash
   vigil run python train_model.py
   vigil promote
   ```

2. **A signed receipt is generated:**

   ```json
   {
     "id": "rcpt_01J7B9M5JT",
     "project_id": "proj_snowchild",
     "created_at": "2025-10-22T04:01:09Z",
     "signature": { "alg": "Ed25519", "sig": "..." }
   }
   ```

3. **Researcher uploads it:**

   ```bash
   vigil login
   vigil push
   ```

4. **Backend verifies & stores:**

   * Schema validation via Fastify
   * Prisma writes to Postgres
   * Artifact metadata stored in S3

5. **UI displays verifiable record:**

   * Project page lists receipt
   * “Verify” button re-validates signature client-side

6. **Any user can pull & verify:**

   ```bash
   vigil pull receipt rcpt_01J7B9M5JT
   vigil verify rcpt_01J7B9M5JT.json
   ```

✅ The receipt remains verifiable indefinitely, even without access to the Cofactor platform.

---

## 10. Scientific Impact

Vigil + Cofactor aim to create a new layer of accountability for computational science —
a **verifiable research substrate** that operates across institutions, domains, and time.

### Benefits

| Stakeholder       | Value                                               |
| ----------------- | --------------------------------------------------- |
| **Researchers**   | Portable, cryptographically provable results.       |
| **Institutions**  | Transparent provenance for publications and grants. |
| **Journals**      | Verified computational appendices for peer review.  |
| **AI Developers** | Reproducible model training and benchmarking.       |
| **Public**        | Trustworthy, open, verifiable science.              |

---

## 11. Future Work

* **Policy engine:** Fine-grained reproducibility and compliance rules.
* **Graph provenance:** Full DAG visualization of receipts and dependencies.
* **Chunked artifacts:** Large-scale dataset streaming and partial verification.
* **Blockchain anchoring:** Public timestamping via transparency log.
* **Multi-language SDKs:** JS, Go, and Rust bindings.
* **Collaborative notebooks:** Live verified experiments within Cofactor Cloud.

---

## 12. Conclusion

Vigil and Cofactor redefine computational science as a provable discipline.

By coupling canonical schemas, deterministic signing, and schema-aware infrastructure, they transform computational results from *claims* into **cryptographic evidence**.

Science becomes not only open, but **verifiable**.

---

## References

1. Blair, W. et al. *Vigil: Schema-Driven Provenance for Computational Reproducibility*, Science Abundance Coalition (2025).
2. Kelsey, J., Schneier, B. *The JSON Canonicalization Scheme (JCS)*, RFC 8785, IETF (2020).
3. Ed25519 – *High-speed High-security Signatures*, Bernstein et al. (2012).
4. FAIR Principles for Scientific Data Management, Wilkinson et al. *Scientific Data* (2016).

---

## Contact

**Science Abundance Coalition**
📍 Toronto / San Francisco / London
🌐 [vigil.dev](https://vigil.dev) · [cofactor.science](https://cofactor.science)
📧 [contact@vigil.dev](mailto:contact@vigil.dev)

---

**Vigil + Cofactor** — *Reproducibility made verifiable.*
