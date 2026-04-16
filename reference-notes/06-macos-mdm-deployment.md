# macOS MDM Deployment and Permissions

> **Status**: Reference for IT ops when ready to deploy fleet-wide.

## The TCC Problem

macOS requires explicit user consent for Screen Recording, Accessibility, and Microphone access. If an intern clicks "Deny" out of caution, the assistant silently fails and generates IT tickets.

## Solution: PPPC Profiles via MDM

Deploy Privacy Preferences Policy Control (PPPC) configuration profiles through an MDM solution (Jamf Pro, Addigy, Microsoft Intune) to pre-authorize the app.

## Required TCC Services

| TCC Service | PPPC Can Force? | What It Enables |
|-------------|----------------|-----------------|
| Accessibility | Yes (Allow) | Global hotkey monitoring (CGEvent tap), UI coordinate mapping |
| Microphone | Yes (Allow) | Push-to-talk voice capture |
| Screen Capture | **Partial** | MDM can pre-approve so standard users can enable it themselves, but cannot force silent "Yes" |

### Screen Recording Limitation

Apple prevents MDM from silently forcing Screen Recording approval. The intern must check the box in System Settings at least once. The PPPC profile eliminates the need for admin credentials to do this.

## macOS 15 Sequoia Considerations

### The Monthly Re-Authorization Problem

macOS 15.0 introduced aggressive "Allow for One Month" dialogs for screen capture apps. This would cause the assistant to break monthly with confusing OS popups.

### The Fix (macOS 15.1+)

Apple added an MDM restriction key to suppress these recurring alerts:

```xml
<key>forceBypassScreenCaptureAlert</key>
<true/>
```

Deploy this as a system restrictions profile alongside the PPPC payload. This allows continuous screen monitoring without monthly re-authorization popups.

## App Distribution Requirements

To distribute internally without Gatekeeper warnings:

1. **Code sign** with an Apple Developer ID certificate
2. **Enable hardened runtime** in build settings
3. **Submit to Apple Notary** service via `notarytool`
4. **Staple** the notarization ticket to the app bundle
5. **Package** as a signed `.pkg` or `.dmg` for MDM distribution

```bash
# Notarize
xcrun notarytool submit Clicky.dmg --apple-id $APPLE_ID --team-id $TEAM_ID --password $APP_PASSWORD --wait

# Staple
xcrun stapler staple Clicky.dmg
```

## Pre-Seeding Configuration via MDM

UserDefaults can be pre-configured via MDM managed preferences or `defaults write`:

```bash
# Set custom workflow context for the intern's team
defaults write com.yourteam.leanring-buddy customWorkflowContext "Your company context..."

# Pre-configure approved apps for Tutor Mode
defaults write com.yourteam.leanring-buddy approvedBundleIdentifiers -array \
    "com.apple.Terminal" \
    "com.microsoft.VSCode" \
    "com.googlecode.iterm2"

# Set the Cloudflare Worker URL
defaults write com.yourteam.leanring-buddy workerBaseURL "https://your-worker.workers.dev"
```
