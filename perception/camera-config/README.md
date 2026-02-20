# Camera Configuration & Image Acquisition

This module handles low-level camera setup and image acquisition.

The goal is to provide stable, clear frames for QR detection
under competition lighting conditions.

---

## 🎯 Objectives

- Initialize camera hardware (ESP32-CAM or equivalent)
- Configure resolution and frame rate
- Adjust exposure and brightness
- Ensure stable frame streaming
- Minimize latency

---

## 📷 Camera Parameters

Key configuration parameters:

- Resolution (e.g., 640x480)
- Frame rate (FPS)
- Exposure control
- White balance
- Gain adjustment
- JPEG quality (if streaming)

---

## ⚙ Architecture Options

### Option 1 — On-board Processing
- Camera captures frame
- QR decoded locally
- Robot moves autonomously

Pros:
- Fully autonomous
- No laptop dependency

Cons:
- Limited processing power

---

### Option 2 — Streaming to Laptop
- Camera streams frames
- QR decoded using Python/OpenCV
- Laptop sends movement commands

Pros:
- Strong processing power
- Better debugging

Cons:
- Requires external device

---

## 🔄 Data Flow

Camera → Frame Buffer → QR Module → Integration Layer

---

## ⚠ Challenges

- Lighting variation
- Motion blur
- Frame drops
- Overexposure

Mitigation strategies:
- Controlled lighting
- Reduced movement during scan
- Fixed exposure settings
