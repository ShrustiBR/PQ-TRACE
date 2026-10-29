# PQ-TRACE

Live Demo : https://tubular-banoffee-ab79c5.netlify.app/

**Every copy carries a secret name tag.**
Post-quantum decryption provenance and leak attribution for multi-recipient encrypted documents.

> **Status:** in development. This README describes the design and the planned demo. File names and commands below are the planned layout, so adjust them to match your final code.

---

## The problem

A confidential document is sent to many people. Everyone receives the same file, so if it leaks:

- there are many suspects,
- there is no proof of who did it,
- and today's signatures may not stay trustworthy once quantum computers arrive.

## Our solution

PQ-TRACE makes every copy unique and every opening provable.

1. **Lock once.** The document is encrypted a single time. Each recipient gets their own key.
2. **Everyone opens their own copy.** Decryption happens on the recipient's device. At that moment a hidden ID is embedded in their copy.
3. **Sign and record.** The recipient signs a receipt ("I opened this document, at this time, with this ID"). The receipt is saved in a tamper-evident, hash-chained logbook.
4. **When it leaks, read the tag.** Extract the hidden ID from the leaked copy, find it in the logbook, verify the signature, and one name comes out, with proof.

A mark alone never accuses anyone. A signed record must back it up.

## What makes it different

- The ID is added **at the moment of opening**, so it is unique per person and per session.
- Signatures and key exchange use **NIST post-quantum standards**, so the evidence stays trustworthy for years.
- It works **fully offline**: no cloud, no public blockchain.

---

## How it works

```
                 ┌──────────────┐
  Document ────► │ Encrypt once │ ──► one ciphertext
                 │  (AES-GCM)   │
                 └──────┬───────┘
                        │ file key wrapped separately per recipient (ML-KEM)
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
       Alice          Bob          Charlie
          │             │             │
   decrypt on own device + embed hidden ID (watermark + error-correcting code)
          │             │             │
          ▼             ▼             ▼
   signed receipt (ML-DSA) ──► hash-chained logbook (SQLite)
                                        │
   Leaked copy ──► extract ID ──► look up in logbook ──► verify signature
                                        │
                                        ▼
                               "Bob, with signed proof"
```

## Tech stack

| Part | Tool | What it does |
|---|---|---|
| Language | Python 3 | Everything is written in it |
| File encryption | AES-256-GCM (`cryptography`) | Locks the document |
| Per-recipient keys | ML-KEM / FIPS 203 (`liboqs-python`) | Quantum-safe key wrapping |
| Signed receipts | ML-DSA / FIPS 204 (`liboqs-python`) | Quantum-safe signatures, so an opening cannot be denied |
| Hidden ID | Watermarking: zero-width characters for text, `invisible-watermark` / OpenCV for images | Embeds the ID in each copy |
| Damage tolerance | Reed-Solomon codes (`reedsolo`) | ID stays readable if part of it is damaged |
| Logbook | SQLite + hash chain | Tamper-evident record of every opening |
| Demo UI | Streamlit | Simple web page for the live demo |

## Planned project layout

```
pq-trace/
├── README.md
├── requirements.txt
├── app.py                # Streamlit demo
├── pqtrace/
│   ├── crypto.py         # AES-GCM, ML-KEM key wrapping, ML-DSA signatures
│   ├── watermark.py      # embed and extract the hidden ID
│   ├── ledger.py         # hash-chained logbook (SQLite)
│   └── trace.py          # leak tracing and verification
├── demo/
│   ├── sample_document.txt
│   └── run_demo.py       # scripted 1 document, 3 recipients, 1 leak
└── tests/
```

## Getting started

**Requirements:** Python 3.10+. Installing `liboqs` needs a C compiler and CMake, plus internet for the install step only.

```bash
git clone <your-repo-url>
cd pq-trace
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

`requirements.txt` (starting point):

```
cryptography
liboqs-python
reedsolo
streamlit
opencv-python
invisible-watermark
```

> If `liboqs` will not install on your machine, switch the code to Ed25519 signatures and X25519 key exchange as a **stand-in** and label them clearly as such. The flow stays identical.

Run the demo:

```bash
python demo/run_demo.py        # scripted walkthrough
streamlit run app.py           # interactive version
```

## Demo walkthrough

Runs with the network turned off.

1. Create **1 document** and **3 recipients** (Alice, Bob, Charlie), each with their own key pair.
2. Lock the document once and wrap the key for each recipient.
3. Each recipient opens it. Each gets a **uniquely marked copy** and a signed receipt in the logbook.
4. Pick one copy and treat it as **leaked**.
5. Run the trace. The tool reads the ID, checks the logbook, verifies the signature and names the person.
6. **Tamper test:** edit one old logbook row by hand. Verification fails, which shows the logbook cannot be quietly changed.

## Security notes and limitations

- **The watermark is the main research risk.** Screenshots, print-and-scan and photos can damage it. We will test each leak channel and report honestly what survives.
- **Colluding recipients** can compare copies to find and strip marks. Collusion-resistant codes (Boneh–Shaw style) are planned as a next step, along with varying marks per session.
- **A mark is a pointer, not a verdict.** Attribution needs the signed logbook entry as well.
- Private keys stay in a local key store on each recipient's device. The logbook root hash (Merkle root) can be shared for audit.
- This is a hackathon prototype and has not had a professional security audit.

## Roadmap

- [ ] Crypto core: encrypt once, wrap keys per recipient
- [ ] Hash-chained logbook with signature verification
- [ ] Watermark embed and extract with error correction
- [ ] Trace tool and tamper test
- [ ] Streamlit demo and test report on leak channels
- [ ] Collusion-resistant codes (stretch)

## References

- NIST FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) — https://csrc.nist.gov/pubs/fips/203/final
- NIST FIPS 204: Module-Lattice-Based Digital Signature Standard (ML-DSA) — https://csrc.nist.gov/pubs/fips/204/final
- Open Quantum Safe project (liboqs) — https://openquantumsafe.org
- NIST SP 800-38D: AES in Galois/Counter Mode (GCM) — https://csrc.nist.gov/pubs/sp/800/38/d/final
- RFC 6962: Certificate Transparency (Merkle-tree append-only logs) — https://www.rfc-editor.org/rfc/rfc6962
- RFC 9180: Hybrid Public Key Encryption (HPKE) — https://www.rfc-editor.org/rfc/rfc9180
- Boneh & Shaw, "Collusion-secure fingerprinting for digital data" (1998)
- Cox et al., "Digital Watermarking and Steganography"
