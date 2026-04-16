# Proactive Tutor Mode (Idle Detection)

## Concept

The defining feature that elevates the assistant from a reactive utility to a pedagogical tutor. Instead of waiting for the intern to press the hotkey, the app detects when they've been idle (stuck) and proactively offers a guiding hint.

## Idle Detection Mechanism

Use `CGEventSource.secondsSinceLastEventType` to track system-wide input idleness at the hardware level. This is hardware-agnostic and doesn't require injecting code into target apps.

```swift
let idleTimeInSeconds = CGEventSource.secondsSinceLastEventType(
    CGEventSourceStateID.hidSystemState,
    eventType: CGEventType(rawValue: ~0)!
)
```

This returns a `CFTimeInterval` of the exact elapsed time since the last hardware input event (keyboard, mouse, trackpad).

## Trigger Conditions

Idle time alone is not sufficient — the intern might be reading documentation or stepped away. Apply a logical AND:

1. **Idle threshold exceeded**: 45-60 seconds of no keyboard/mouse input
2. **Approved app in focus**: The frontmost app's `CFBundleIdentifier` matches a list of approved internal tools (IDE, deployment dashboard, terminal, etc.)
3. **Not already intervening**: No active TTS playback or pending response

## Implementation Pattern

```swift
// Lightweight background timer (every 5-10 seconds)
Timer.scheduledTimer(withTimeInterval: 10.0, repeats: true) { _ in
    let idleTime = CGEventSource.secondsSinceLastEventType(
        .hidSystemState,
        eventType: CGEventType(rawValue: ~0)!
    )
    
    let frontmostBundleID = NSWorkspace.shared.frontmostApplication?.bundleIdentifier
    let isApprovedApp = approvedBundleIdentifiers.contains(frontmostBundleID ?? "")
    
    if idleTime > 45.0 && isApprovedApp && !isAlreadyIntervening {
        triggerProactiveIntervention()
    }
}
```

## Proactive Intervention Flow

1. Silently capture a window-isolated screenshot (no user interaction)
2. Send to Claude with a hidden system trigger appended to the prompt:
   ```
   "System Trigger: The user has been idle on the provided screen state 
   for 60 seconds without advancing the workflow. Initiate a proactive, 
   low-pressure intervention offering guidance. Do not solve the problem. 
   Ask a guiding question based on the visual context to unblock them."
   ```
3. Claude responds with a brief spoken hint via TTS
4. The cursor overlay fades in transiently (if hidden) for the interaction
5. After TTS completes, fade out and resume idle monitoring

## Cooldown

After a proactive intervention, enforce a cooldown period (3-5 minutes) before the next one. Repeated unprompted interruptions would be annoying and counterproductive.

## User Control

- The intern should be able to toggle Tutor Mode on/off from the menu bar panel
- A "Don't help me with this" dismissal should suppress interventions for the current app/window for the rest of the session

## Approved App List

Store the list of approved bundle identifiers in UserDefaults so it can be configured per-deployment:

```swift
// Example approved apps
let approvedBundleIdentifiers: [String] = [
    "com.apple.Terminal",
    "com.microsoft.VSCode",
    "com.googlecode.iterm2",
    "com.apple.dt.Xcode",
    // Add company-specific internal tools here
]
```
