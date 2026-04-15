# Steganography: ML-Assisted Robust LSB Image Steganography

## Overview
This repository contains a machine-learning assisted Least Significant Bit (LSB) image steganography toolkit. A Random Forest classifier selects visually/ statistically safe pixels for embedding a secret message. A header (magic + length) enables blind decoding: a receiver needs only the stego image and the saved model to recover the payload.

## Key Features
- Intelligent pixel selection using per-pixel features (Brightness, Local Variance, Edge Strength)
- Header-based embedding (`MAGIC + 32-bit length + UTF-8 payload`) for blind extraction
- Classic LSB embedding fallback (requires length or stop token)
- Detection utilities (chi-square pairs, LSB entropy, monobit & runs tests)
- Command-line interface for encode, decode, and check operations
- Jupyter notebook demonstrating training, embedding, decoding, and detection

## Files
| File | Purpose |
|------|---------|
| `stegno.ipynb` | Notebook: training, feature extraction, encoding, decoding, detection demos. |
| `lsb_pixel_classifier.pkl` | Trained RandomForest model for pixel suitability. |
| `PROJECT_REPORT.md` | Detailed project report (title, abstract, implementation, code excerpts). |
| `steg_cli.py` | CLI for encode/decode/check operations. |
| `requirements-steg.txt` | Python dependencies list for toolkit. |
| `README.md` | This document. |

## Header Format
```
MAGIC: b"STEGv1"  (6 bytes)
LENGTH: 4 bytes (big-endian unsigned int, payload size in bytes)
PAYLOAD: UTF-8 encoded message
```
Total header overhead: 80 bits.

## Usage (CLI)
Install dependencies:
```
pip install -r requirements-steg.txt
```
Encode message:
```
python steg_cli.py encode -i cover.png -o stego.png -m "Secret message" -model lsb_pixel_classifier.pkl
```
Blind decode:
```
python steg_cli.py decode -i stego.png -model lsb_pixel_classifier.pkl
```
Check/detect (optionally reveal if present):
```
python steg_cli.py check -i stego.png -model lsb_pixel_classifier.pkl --reveal
```

## Notebook Workflow
1. Load/Train classifier.
2. Extract features from input image and predict good pixels.
3. Embed message with header.
4. Decode blindly using only stego + model.
5. Run detectors to evaluate stego likelihood.

## Detection Heuristics
- Chi-square pairs-of-values (2k, 2k+1) distribution shift
- LSB entropy approaching 1.0
- Monobit test (balance of 0/1)
- Runs test (bit sequence randomness)

## Capacity
Approximate payload capacity (bytes): `(len(good_pixels) - 80) / 8`.
Ensure you use a lossless format (PNG). JPEG recompression will likely corrupt LSBs.

## Edge Cases & Warnings
| Scenario | Handling |
|----------|----------|
| Insufficient capacity | Abort and report. |
| Model missing | Cannot reproduce pixel order -> decode fails. |
| MAGIC absent | Decoder returns encoded=False. |
| Truncated payload | Warning with message=None. |

## Extensibility Ideas
- Add encryption layer (e.g., AES) to payload before embedding.
- Multi-channel distribution for increased stealth.
- Error correction (Reed–Solomon) to resist minor transforms.
- Web service / API integration.

## License
Add a license of your choice (e.g., MIT) here.

## Contributing
Pull requests for improved models, detectors, or additional robustness features are welcome.

## Disclaimer
Steganography can be used for legitimate privacy purposes. Ensure compliance with local laws and organizational policies.

---
Generated README for initial GitHub publication.
