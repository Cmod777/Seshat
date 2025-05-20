# README.verify.md — Verification Instructions for Seshat Repository  
**Date:** 2025-05-20  
**Author:** Cmod777  
**Repository:** [https://github.com/Cmod777/Seshat](https://github.com/Cmod777/Seshat)  
**PGP Fingerprint:** `358B 4463 4943 B02C A075 D877 ED68 61BD DFB2 63C2`

---

## Purpose

This file provides complete instructions for verifying the **authenticity**, **authorship**, and **integrity** of the licensing materials and public key associated with this repository. All materials have been signed using the PGP key of **Cmod777** and are cryptographically linked to the commit date `2025-05-20`.

---

## Folder Structure: `/signatures/2025-05-20/`

| File name         | Description                                        |
|-------------------|----------------------------------------------------|
| `LICENSE.md`      | Complete licensing file                            |
| `LICENSE.sha1`    | SHA-1 hash of `LICENSE.md`                         |
| `LICENSE.sha256`  | SHA-256 hash of `LICENSE.md`                       |
| `KEYS.md`         | Public key declaration and metadata                |
| `KEYS.sha1`       | SHA-1 hash of `KEYS.md`                            |
| `KEYS.sha256`     | SHA-256 hash of `KEYS.md`                          |
| `proof.json`      | Machine-verifiable authorship + hash declaration   |
| `proof.json.asc`  | Detached PGP signature for `proof.json`            |

---

## Step 1 – Hash Verification

Make sure you are in the `2025-05-20` directory. Then run:

```bash
sha256sum -c LICENSE.sha256
sha1sum -c LICENSE.sha1
sha256sum -c KEYS.sha256
sha1sum -c KEYS.sha1
```

Expected output:

```
LICENSE.md: OK
KEYS.md: OK
```

---

## Step 2 – Signature Verification (proof.json)

To verify the authorship and integrity of `proof.json`, use:

```bash
gpg --verify proof.json.asc proof.json
```

Expected result:

```
gpg: Good signature from "Cmod777 (GitHub public key) <an0n1mu5@protonmail.com>" [ultimate]
```

---

## Notes

- Make sure the `proof.json` file was not modified after signature generation.
- Verify that the PGP key used matches the published fingerprint.
- These checks ensure that both the license terms and associated key metadata are authentic and legally bound as of the date above.
