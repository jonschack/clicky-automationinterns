# On-Device PII Redaction (Future)

> **Status**: Deferred — not in MVP. Document here for future implementation.

## Problem

An intern's screen will inevitably display PII, proprietary source code, internal network topologies, or confidential financial data. Relying on cloud-based redaction is insufficient because the raw data would still transit the network.

## Solution: Apple Vision Framework

Use Apple's native Vision framework to detect and redact sensitive text directly on-device before the screenshot is base64-encoded and transmitted. Runs on the Apple Silicon Neural Engine — no network required.

## Implementation

### 1. Text Detection with VNRecognizeTextRequest

```swift
let request = VNRecognizeTextRequest()
request.recognitionLevel = .accurate  // Use accurate path since we're not streaming video
request.usesLanguageCorrection = true

let handler = VNImageRequestHandler(cgImage: capturedScreenshot)
try handler.perform([request])

let observations = request.results as? [VNRecognizedTextObservation] ?? []
```

### 2. Pattern Matching Against Enterprise Regex

Iterate through detected text and match against sensitive patterns:

- Social Security Numbers: `\b\d{3}-\d{2}-\d{4}\b`
- API keys: `\b(sk|pk|api)[_-][a-zA-Z0-9]{20,}\b`
- Email addresses: Standard email regex
- Internal account IDs: Company-specific patterns
- Project codenames: Configurable blocked terms list

### 3. Coordinate Transformation for Redaction

Vision framework returns `boundingBox` as normalized coordinates (0.0-1.0) with **bottom-left origin**. To draw redaction rectangles on a CGImage (top-left origin):

```
Width_pixel  = boundingBox.width  * imageWidth
Height_pixel = boundingBox.height * imageHeight
X_pixel      = boundingBox.minX   * imageWidth
Y_pixel      = (1.0 - boundingBox.minY - boundingBox.height) * imageHeight
```

### 4. Flat Rendering

Draw solid black rectangles over the sensitive regions using Core Graphics. The redaction is baked into the image — irreversible before transmission.

## Processing Path Choice

| Path | Speed | Accuracy | Use Case |
|------|-------|----------|----------|
| `.fast` | ~50ms | Good | Real-time video streams |
| `.accurate` | ~200ms | Excellent | **Single screenshot capture (our case)** |

Since we only capture screenshots on hotkey press or idle timeout (not streaming 60fps), the `.accurate` path is appropriate.

## Privacy Guarantee

Only the sanitized, flattened image is serialized into the JSON payload. Raw screenshots with sensitive data never leave the device.
