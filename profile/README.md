# KintaGen – Lab‑Assistant AI

**Focus:** biology / science workflows, IPFS hot‑storage (Pinata/Lighthouse), asynchronous job queues (Vercel/Redis/QStash), Nostr NIP-44 decentralized profiles and encryption, Flow Cadence Logs using the power of blockchain and NFTs

---

## Why KintaGen?

Modern life-science labs generate terabytes of high-value data—GC-MS chromatograms, NMR spectra, and mountains of supplemental PDFs. Managing this data is a significant challenge:

*   **Insecure & Ephemeral Sharing:** Critical data is shared via email or temporary links, leading to version control chaos (`Final_v4_reviewed_REALLYFINAL.csv`) and lost files.
*   **Lack of Reproducibility:** It's nearly impossible to prove which script or dataset produced a result six months ago, hindering validation and FAIR compliance.
*   **Data Silos & Security Risks:** Valuable intellectual property is often siloed, difficult to search, and at risk of being exposed before publication or patent filing.

KintaGen solves these challenges by integrating three core decentralized technologies:

| Technological Pillar | Core Capability | The Lab Benefit |
| :--- | :--- | :--- |
| **IPFS Storage (Pinata/Synapse)** | ⚡ **Fast & Decentralized Downloads.** All data is stored on decentralized networks and served globally via Pinata and Lighthouse. | Share multi-gigabyte datasets, like GC-MS bundles or raw datasets, as easily as a web link—no more shipping hard drives. |
| **Nostr Identities & NIP-44** | 🔐 **Decentralized Profiles & Encryption.** Files are encrypted client-side securely linking private data to public on-chain provenance. | Securely collaborate on pre-publication data with partner labs without giving up ownership or exposing sensitive IP, while maintaining a portable researcher profile. |
| **Flow (Cadence) Logbook** | 📜 **Verifiable Audit Trails.** Every analysis step (the agent used, timestamp, output CID) is appended to a project-specific **Flow Cadence NFT**, creating an immutable history. | Create a tamper-proof log for any project, ensuring reproducibility and simplifying compliance for grant reporting. |

On top of this robust data foundation, Kintagen layers a suite of AI and analysis services (powered by Vercel serverless, Redis, and QStash), transforming it into an autonomous research co-pilot that can:

- [✅] **Extract Metadata:** Automatically parse titles, authors, and keywords from scientific papers.
- [✅] **Run Analyses:** Execute complex calculations like LD50 dose-response curves, GC-MS metabolomics, and NMR deconvolution on demand via specialized asynchronous R-workers.
- [✅] **Answer Questions:** Perform Retrieval-Augmented Generation (RAG) on your project's documents to answer complex, domain-specific questions.

Every output is pinned back to IPFS (via Pinata or Lighthouse), ensuring that results are immediately available for downstream agents or external collaborators.

---

## 1. Data Lifecycle Overview
| Stage                           | Implementation in repo                                                                                                                                                                                                                               | Storage target                               |
| :------------------------------ | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------- |
| **Upload → (optional) Encrypt** | Front-end posts to Lighthouse/Pinata. File is streamed to IPFS. If encrypted, the browser uses **Nostr NIP-44** to wrap the file; the key is securely managed on the client side utilizing Nostr decentralized profiles. | IPFS via Pinata/Lighthouse |
| **Metadata capture**            | Application extracts metadata and captures analytical output hashes, linking them back to decentralized data structures and IPFS CIDs. | Storage & Application State |
| **Agent workflows**             | R-based serverless async agents run LD₅₀ / GC-MS / NMR etc.; artifacts are saved locally in the processing server, heavy files are processed, and output data like plots are saved on IPFS with their CIDs logged. | IPFS (heavy artefacts) + Blockchain Flow log |
| **User access**                 | React app lists data; Nostr integration decrypts data locally, checking keys in browser; provenance timeline pulled from the project’s **Flow NFT** via Cadence scripts.                                                                                           | Browser cache & Decentralized Node |


---

## 2. Scripts Catalogue

| Slug            | Status      | Runtime                                         | API / Entry-point        | Purpose (one-liner)                                                                             | Typical inputs                                  | Heavy outputs (IPFS)                                       |
| --------------------- | ----------- | ----------------------------------------------- | ------------------------ | ----------------------------------------------------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------------ |
| **`ld50-dose`**       | ✅ Live      | **Rscript** ([scripts/drc_analysis.R](cci:7://file:///home/henrique/projects/kintagen/kintagen-r-api/scripts/drc_analysis.R:0:0-0:0))          | `POST /process-job` (kintagen-r-api) | Classical LD₅₀ / ED₅₀ dose-response modelling with **drc** → JSON + plot base64. | CSV (`dose, response, total`) or sample data    | Plot PNG representation                               |
| **`gcms-xcms`**       | ✅ Live      | **Rscript** ([scripts/xcms_analysis.R](cci:7://file:///home/henrique/projects/kintagen/kintagen-r-api/scripts/xcms_analysis.R:0:0-0:0))         | `POST /process-job` (kintagen-r-api) | Full untargeted GC-MS pipeline using **xcms**. Includes peak extraction and retention alignment. | Directory of `mzML / CDF` (zipped)  | Stat data & intermediate results |
| **`nmr1d`**           | ✅ Live      | **Rscript** ([scripts/nmr1d_analysis.R](cci:7://file:///home/henrique/projects/kintagen/kintagen-r-api/scripts/nmr1d_analysis.R:0:0-0:0))        | `POST /process-job` (kintagen-r-api) | Quantitative 1H-NMR data processing, plotting, baseline correction and analysis. | Bruker dataset / raw FID files | Processed JSON & calibration/zoom plots |
| **`MoNA Identifier`** | ✅ Live      | **Node (fetch)** ([server.js](cci:7://file:///home/henrique/projects/kintagen/kintagen-r-api/server.js:0:0-0:0) identifying peaks)  | Internal to `process-job` | Identifies single GC-MS peaks against the MoNA similarity search API. | MS Spectrum string                       | None (JSON hit and compound records)                                            |


*(Note: Legacy theoretical pipelines such as 'bio-extractor', 'spectra-indexer', 'genome-annot', etc. can be created following this same async Vercel Worker paradigm).*

---

### 2.1. `ld50-dose` – Detailed Design

```mermaid
graph TD
    L1["CSV (dose,response,total)"] --> L2["drc::drm fit"]
    L2 --> L3["ED50 + SE + CI"]
    L2 --> L4["ggplot() curve PNG (base64)"]
    L3 & L4 --> L5[JSON emit back to Vercel/Redis]
