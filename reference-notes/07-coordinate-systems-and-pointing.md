# Coordinate Systems and Pointing Architecture

## Three Coordinate Systems in Play

The app must translate between three different coordinate systems. Getting this wrong means the cursor points at the wrong place.

### 1. Screenshot Pixel Coordinates (Top-Left Origin)

- Origin (0,0) at top-left of the captured image
- X increases rightward, Y increases downward
- Dimensions match the actual captured image pixels (e.g., 1280x831)
- This is the coordinate space Claude sees and generates `[POINT:x,y]` tags in

### 2. AppKit Global Coordinates (Bottom-Left Origin)

- Origin (0,0) at bottom-left of the primary display
- X increases rightward, Y increases **upward**
- Used by `NSEvent.mouseLocation`, `NSScreen.frame`, overlay window positioning
- Multi-monitor: secondary displays have offsets based on arrangement

### 3. SwiftUI Screen-Local Coordinates (Top-Left Origin)

- Origin (0,0) at top-left of each screen
- Used for rendering in `BlueCursorView`
- Each overlay window has its own local coordinate space

## Conversion: Claude's Pixels to AppKit Global

```swift
// Claude provides: (pointX, pointY) in screenshot pixel space
// We have: screenshotWidth, screenshotHeight, displayWidth, displayHeight, displayFrame

// 1. Clamp to screenshot bounds
let clampedX = max(0, min(pointX, screenshotWidth))
let clampedY = max(0, min(pointY, screenshotHeight))

// 2. Scale from screenshot pixels to display points
let displayLocalX = clampedX * (displayWidth / screenshotWidth)
let displayLocalY = clampedY * (displayHeight / screenshotHeight)

// 3. Flip Y-axis (screenshot top-left → AppKit bottom-left)
let appKitY = displayHeight - displayLocalY

// 4. Offset to global screen coordinates
let globalX = displayLocalX + displayFrame.origin.x
let globalY = appKitY + displayFrame.origin.y
```

## Conversion: AppKit Global to SwiftUI Local

```swift
// For rendering in BlueCursorView on a specific screen
let swiftUIX = appKitPoint.x - screenFrame.origin.x
let swiftUIY = (screenFrame.origin.y + screenFrame.height) - appKitPoint.y
```

## Window-Isolated Capture Adjustment

When using frontmost-window-only capture (see 01-frontmost-window-isolation.md), Claude's coordinates are relative to the **window content area**, not the full display. To convert to global screen coordinates, add the window's absolute position:

```swift
let globalX = windowLocalX + windowFrame.origin.x
let globalY = windowLocalY + windowFrame.origin.y  // after Y-flip
```

## The Overlay Window

- Full-screen transparent `NSPanel` per connected display
- Window level: `.screenSaver` (always above everything)
- `ignoresMouseEvents = true` — fully click-through
- `canJoinAllSpaces = true` — visible on all desktops
- Non-activating — never steals focus

## Bezier Arc Animation

The cursor doesn't teleport — it flies along a bezier arc:
- Duration: ~0.6 seconds
- Mid-arc scale boost: 1.0x → 1.3x → 1.0x
- Triangle rotates to face direction of travel (tangent to curve)
- Glow radius intensifies during flight
- On arrival: speech bubble pops in with spring animation
- Return flight can be cancelled if user moves cursor >100px
