# QR Code Detection & Decoding Module

This module enables object identification using QR codes
during the autonomous pick-and-place mission.

---

## 🎯 Purpose

Each object in the competition arena contains a QR code.

The robot must:
- Detect QR location
- Decode the embedded data
- Map the decoded ID to a specific action

---

## 🔍 Detection Pipeline

1. Capture frame
2. Convert to grayscale
3. Apply thresholding (if required)
4. Detect QR bounding box
5. Decode data string
6. Validate data format
7. Send decoded ID to Integration

---

## 📦 Output Example

Decoded Data:
