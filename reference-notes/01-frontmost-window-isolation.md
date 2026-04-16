# Frontmost Window Isolation

## Problem

Capturing the full desktop sends excessive visual noise to the LLM — background wallpapers, notification banners, the dock, irrelevant apps on secondary monitors, and potentially sensitive content from personal apps. This degrades pointing accuracy, inflates token costs, and creates privacy risk.

## Solution: Window-Isolated Screen Capture

Instead of `SCContentFilter(display:)`, use `SCContentFilter(desktopIndependentWindow:)` to capture only the active window the intern is working in.

## Implementation Steps

### 1. Retrieve shareable content inventory

```swift
let content = try await SCShareableContent.excludingDesktopWindows(false, onScreenWindowsOnly: true)
// Returns arrays of SCDisplay, SCRunningApplication, and SCWindow
```

### 2. Identify the frontmost window via Accessibility API

ScreenCaptureKit does not know which window is "frontmost." Query the Accessibility API to find it:

```swift
let systemWide = AXUIElementCreateSystemWide()
// Use AXUIElementCopyAttributeValue with kAXFocusedApplicationAttribute
// Then drill into the app's kAXFocusedWindowAttribute
// Match the resulting window against the SCWindow array by windowID or title
```

### 3. Apply a window-specific content filter

```swift
let filter = SCContentFilter(desktopIndependentWindow: targetWindow)
```

## Comparison

| Capture Method | Privacy Risk | LLM Accuracy | Token Cost |
|----------------|-------------|--------------|------------|
| Full Desktop | High — ingests all visible apps | Low — hallucinations from noise | High |
| App-Excluded | Medium — still captures backgrounds | Moderate | Medium |
| **Window-Isolated** | **Low — bounded to active task** | **High — clean context** | **Low** |

## Coordinate System Note

When using window-isolated capture, the screenshot dimensions map to the window's content area, not the full display. The LLM's `[POINT:x,y]` coordinates will be relative to the window, so the overlay must offset them by the window's absolute screen position when rendering the pointing animation.

## When to Fall Back to Full Capture

Keep full multi-monitor capture as a fallback for cases where the intern explicitly references something on another screen ("what about that terminal on my other monitor?"). The current `CompanionScreenCaptureUtility.captureAllScreensAsJPEG()` should remain available alongside the new window-isolated mode.
