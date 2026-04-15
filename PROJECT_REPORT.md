# Machine-Learning Assisted Robust LSB Image Steganography

## 1. Project Title
Machine-Learning Assisted Robust LSB Image Steganography with Blind Header-Based Decoding

## 2. Abstract
This project implements a hybrid steganography pipeline that combines classical Least Significant Bit (LSB) embedding with a machine learning model to intelligently select high-quality pixels for data hiding. A Random Forest classifier is trained on per-pixel statistical features (Brightness, Local Variance, Edge Strength) to predict which pixels can undergo an LSB flip with minimal perceptual impact and better stealth. To enable truly blind extraction (recipient needs only the stego image), a compact binary header (magic signature + 32-bit payload length) precedes the UTF-8 message. Multiple detection heuristics (chi-square pairs-of-values, LSB entropy, monobit and runs tests on classifier-selected pixels) are provided to assess whether an image likely contains hidden data. The result is a practical, extensible toolkit supporting encode, decode, and detection workflows via both a Jupyter notebook and a CLI script.

## 3. Objectives
- Reduce visual/statistical detectability of LSB steganography by selective pixel choice.
- Support blind decoding without the original cover image or prior message length.
- Provide detection metrics for stego forensics (entropy, chi-square, randomness tests).
- Deliver an easy-to-use command-line interface and reproducible notebook.

## 4. System Overview
| Component | Purpose |
|-----------|---------|
| Feature Extraction | Generates per-pixel statistics feeding the classifier. |
| Pixel Classifier (RandomForest) | Labels pixels as suitable (1) or unsuitable (0) for LSB embedding. |
| Encoder (Basic + Header-based) | Embeds plaintext bits into the LSB of the chosen channel (default Blue). |
| Header Format | `MAGIC (6 bytes) + LENGTH (4 bytes, big-endian) + PAYLOAD (UTF-8)` |
| Blind Decoder | Scans predicted good pixels, locates MAGIC, reads length, reconstructs message. |
| Detection Suite | Chi-square pairs-of-values, LSB entropy, monobit test, runs test. |
| CLI Toolkit | Simplifies encode, decode, and check operations. |

## 5. Data & Features
Each pixel (excluding a 1-pixel border) yields:
1. Brightness: grayscale intensity.
2. Local Variance: variance of 3×3 neighborhood (texture measure).
3. Edge Strength: Laplacian response (detail sensitivity).

These features inform whether flipping the least significant bit is less likely to trigger statistical anomalies.

## 6. Model Training
- Algorithm: RandomForestClassifier (n_estimators=150, random_state=42).
- Labels: `Label (LSB Flip)` from a pre-generated dataset (`structured_pixel_data.csv`).
- Split: 80/20 train/test.
- Output: Saved model `lsb_pixel_classifier.pkl` using joblib.

Training confirms predictive capacity for identifying pixels robust to LSB modification. The model's implicit ensemble nature reduces overfitting risk and handles mixed feature scales.

## 7. Embedding Strategies
### 7.1 Basic LSB Embedding
- Convert message to a bitstring (8 bits per character).
- For each chosen pixel coordinate `(x, y)` (in classifier-positive list), replace the least significant bit of the Blue channel with the next message bit: `b = (b & 0xFE) | bit`.
- Truncates if capacity is insufficient.

### 7.2 Header-Based Robust Embedding (Recommended)
Adds a structured header enabling blind decode:
```
MAGIC: b"STEGv1" (48 bits)
LENGTH: 4 bytes (32-bit big-endian unsigned integer, payload size in bytes)
PAYLOAD: UTF-8 encoded message
```
Bits of the concatenated packet are written sequentially into good pixels. Receiver recomputes good pixels with the same classifier and scans for MAGIC to recover the message length and payload, requiring only the stego image + model.

## 8. Decoding Approaches
| Approach | Requirements | Pros | Cons |
|----------|--------------|------|------|
| Basic (No Header) | Stego image + good pixel order + length or stop token | Minimal overhead | Not blind unless length known |
| Header-Based | Stego image + classifier model | Fully blind; robust length | Slight overhead (header bits) |

## 9. Detection Techniques
1. Chi-Square Pairs-of-Values: Tests uniformization of `(2k, 2k+1)` histogram pairs typical of LSB replacement.
2. LSB Entropy: Values near 1.0 may indicate randomized LSBs from embedding.
3. Monobit Test (on predicted good pixels): Evaluates balance of 0/1 bits.
4. Runs Test: Checks randomness of consecutive identical bits.

Combined heuristics flag likely stego when statistical patterns deviate from natural image noise distributions.

## 10. Core Algorithms & Code Excerpts
Below are key functions (abbreviated for clarity). Full versions are in the notebook.

```python
# Feature extraction (optimized Laplacian computation)
def extract_features_from_image(img_path):
    pil_img = Image.open(img_path).convert('RGB')
    img = cv2.cvtColor(np.array(pil_img), cv2.COLOR_RGB2BGR)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    lap_full = cv2.Laplacian(gray, cv2.CV_64F)
    h, w = gray.shape
    features, positions = [], []
    for i in range(1, h-1):
        for j in range(1, w-1):
            brightness = int(gray[i, j])
            local_patch = gray[i-1:i+2, j-1:j+2]
            local_variance = float(np.var(local_patch))
            edge_strength = float(lap_full[i, j])
            features.append([brightness, local_variance, edge_strength])
            positions.append((i, j))
    return np.array(features), positions, img
```
```python
# Pixel classification
def get_good_pixels(img_path, model_path='lsb_pixel_classifier.pkl'):
    model = joblib.load(model_path)
    features, positions, img = extract_features_from_image(img_path)
    preds = model.predict(features)
    good_pixels = [positions[i] for i, p in enumerate(preds) if p == 1]
    return good_pixels, img
```
```python
# Header-based hide
def hide_message_with_header(img, message, good_pixels, output_path, channel='B'):
    MAGIC = b'STEGv1'
    ch_index = {'B':0,'G':1,'R':2}[channel]
    payload = message.encode('utf-8')
    length_bytes = len(payload).to_bytes(4, 'big')
    bits = ''.join(f'{b:08b}' for b in MAGIC + length_bytes + payload)
    if len(bits) > len(good_pixels):
        raise ValueError('Not enough capacity')
    out = img.copy()
    for idx, (x,y) in enumerate(good_pixels):
        if idx >= len(bits): break
        b,g,r = out[x,y]
        val = (int((b,g,r)[ch_index]) & 0xFE) | int(bits[idx])
        if ch_index == 0: b = val
        elif ch_index == 1: g = val
        else: r = val
        out[x,y] = (b,g,r)
    cv2.imwrite(output_path, out.astype(np.uint8))
```
```python
# Blind detect + decode
def detect_and_decode_with_header(img_path, model_path='lsb_pixel_classifier.pkl', channel='B'):
    MAGIC = b'STEGv1'
    magic_bits = ''.join(f'{b:08b}' for b in MAGIC)
    good_pixels, img = get_good_pixels(img_path, model_path)
    ch_index = {'B':0,'G':1,'R':2}[channel]
    bits = []
    for (x,y) in good_pixels:
        b,g,r = img[x,y]
        val = (b,g,r)[ch_index]
        bits.append('1' if (int(val) & 1) else '0')
    bit_str = ''.join(bits)
    pos = bit_str.find(magic_bits)
    if pos == -1: return {'encoded': False}
    start = pos + len(magic_bits)
    length_bits = bit_str[start:start+32]
    payload_len = int(length_bits, 2)
    payload_bits = bit_str[start+32:start+32+payload_len*8]
    message = ''.join(chr(int(payload_bits[i:i+8],2)) for i in range(0,len(payload_bits),8))
    return {'encoded': True, 'length': payload_len, 'message': message}
```
```python
# Chi-square + entropy detection
from scipy.stats import chi2 as chi2_dist

def stego_lsb_chi2_detect(img_path, channel='B'):
    pil_img = Image.open(img_path).convert('RGB')
    img = cv2.cvtColor(np.array(pil_img), cv2.COLOR_RGB2BGR)
    ch_index = {'B':0,'G':1,'R':2}[channel]
    ch = img[:,:,ch_index].ravel().astype(np.uint8)
    lsb = ch & 1
    # Entropy
    p1 = lsb.mean(); p0 = 1.0 - p1
    import math
    ent = -(p0*math.log2(p0+1e-12) + p1*math.log2(p1+1e-12))
    # Chi-square pairs
    hist = np.bincount(ch, minlength=256)
    chi2 = 0.0; df = 0
    for k in range(0,256,2):
        h0, h1 = hist[k], hist[k+1]
        s = h0 + h1
        if s == 0: continue
        exp = s/2.0
        chi2 += ((h0-exp)**2)/exp + ((h1-exp)**2)/exp
        df += 1
    df = max(df-1,1)
    p_val = chi2_dist.sf(chi2, df)
    encoded = (p_val > 0.5) and (ent > 0.95)
    return {'entropy': ent, 'chi2': chi2, 'p_value': p_val, 'encoded': encoded}
```

## 11. Runtime Workflow
1. Train model (or load existing `lsb_pixel_classifier.pkl`).
2. Extract features and classify pixels from the cover image.
3. Embed message using header-based encoder.
4. Distribute only the stego image + model file.
5. Receiver runs blind decode to recover message.
6. Optionally run detection tests to confirm stego presence.

## 12. Capacity Considerations
Available bits ≈ number of classifier-approved pixels. Header consumes 48 (magic) + 32 (length) = 80 bits overhead. Maximum payload bytes ≈ (len(good_pixels) - 80) / 8.

## 13. Evaluation & Results
- Header-based demo successfully recovered: `"Hello Chirag, header-based blind detection works!"`.
- Good pixels example count: ~150k on sample image (ample capacity).
- Detection heuristics distinguish clean vs. encoded images with entropy and chi-square shifts.

## 14. Error Handling & Edge Cases
| Case | Handling |
|------|----------|
| Image not found | Graceful message, return empty results. |
| Insufficient capacity | Early warning and abort embedding. |
| MAGIC not found | Decoder returns `encoded: False`. |
| Truncated payload | Decoder returns warning with `message: None`. |
| Model missing | Returns error; cannot reproduce good pixel order. |

## 15. Limitations
- Classifier quality depends on training data diversity.
- Recompression (JPEG) may corrupt LSB payload.
- Using only one channel (Blue) concentrates modifications; multi-channel spreading could improve stealth but complicates decoding.
- Detection heuristics are heuristic—not formal proofs of absence.

## 16. Future Work
- Support encryption of payload prior to embedding.
- Adaptive multi-channel distribution.
- Robustness against common image transforms (resizing, slight blurring) via error-correction codes (e.g., Reed–Solomon).
- Web service / API endpoint for encode/decode.

## 17. CLI Usage (Summary)
Encoding:
```
python steg_cli.py encode -i input.png -o stego.png -m "Secret message" -model lsb_pixel_classifier.pkl
```
Blind Decode:
```
python steg_cli.py decode -i stego.png -model lsb_pixel_classifier.pkl
```
Check (detection heuristics):
```
python steg_cli.py check -i stego.png -model lsb_pixel_classifier.pkl --reveal
```

## 18. Conclusion
This project demonstrates that combining machine learning–driven pixel selection with a lightweight header yields an effective, blind-decoding LSB steganography system. The modular design (feature extraction, classification, embedding, detection) facilitates experimentation and extension while keeping operational complexity low for end users.

## 19. References & Influences
- Fridrich, J. et al. (LSB embedding detection via statistical analysis).
- Standard randomness tests (NIST SP 800-22 inspiration for monobit and runs concepts).
- Scikit-learn documentation for ensemble methods.

---
Report generated on: 2025-11-14
