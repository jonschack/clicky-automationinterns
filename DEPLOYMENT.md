# Deployment Guide

How to go from this repo to a `.dmg` that clients download from automationinterns.com.

## Architecture

```
Client's Mac                    Cloudflare Worker              External APIs
┌─────────────┐                ┌─────────────────┐           ┌──────────────┐
│ Clicky.app  │───/chat───────▶│  clicky-proxy   │──────────▶│ Anthropic    │
│ (menu bar)  │───/tts────────▶│  (holds keys)   │──────────▶│ ElevenLabs   │
│             │───/transcribe─▶│                 │──────────▶│ AssemblyAI   │
└─────────────┘                └─────────────────┘           └──────────────┘
```

Clients never see API keys. They download the app, grant permissions, and go.

## Step 1: Get API Keys

You need accounts and keys from three services:

| Service | What It Does | Sign Up |
|---------|-------------|---------|
| Anthropic | Claude AI (sees screen, answers questions) | console.anthropic.com |
| ElevenLabs | Text-to-speech (Clicky speaks responses) | elevenlabs.io |
| AssemblyAI | Speech-to-text (push-to-talk transcription) | assemblyai.com |

## Step 2: Deploy the Cloudflare Worker

The Worker is a lightweight proxy that holds your API keys server-side.

```bash
# Install Wrangler (Cloudflare's CLI)
npm install -g wrangler

# Authenticate with Cloudflare
wrangler login

# Navigate to the worker directory
cd worker
npm install

# Add your API keys as encrypted secrets
npx wrangler secret put ANTHROPIC_API_KEY
npx wrangler secret put ASSEMBLYAI_API_KEY
npx wrangler secret put ELEVENLABS_API_KEY

# Deploy
npx wrangler deploy
```

After deploying, Wrangler prints your Worker URL. It looks like:
```
https://clicky-proxy.<your-subdomain>.workers.dev
```

Save this URL — you need it in the next step.

## Step 3: Set the Worker URL in the App

Open `leanring-buddy/Info.plist` and set the `WorkerBaseURL` value to your Worker URL:

```xml
<key>WorkerBaseURL</key>
<string>https://clicky-proxy.your-subdomain.workers.dev</string>
```

Or set it via Xcode: open the project, select the `leanring-buddy` target, go to the Info tab, and edit the `WorkerBaseURL` value.

## Step 4: Build and Distribute

### Option A: Quick Test (Xcode)

1. Open `leanring-buddy.xcodeproj` in Xcode
2. Select the `leanring-buddy` scheme
3. Set your signing team (Signing & Capabilities tab)
4. Cmd+R to build and run

**Important**: Do NOT use `xcodebuild` from the terminal for testing — it invalidates TCC permissions and the app will need to re-request screen recording, accessibility, etc.

### Option B: Release Build (.dmg for clients)

The release script automates the full pipeline: build, sign, notarize, create DMG, and publish to GitHub Releases.

**One-time setup:**
```bash
# Install tools
brew install create-dmg gh

# Authenticate GitHub CLI
gh auth login

# Store Apple notarization credentials
xcrun notarytool store-credentials "AC_PASSWORD"
# (Follow prompts for Apple ID, team ID, app-specific password)
```

**Release:**
```bash
# Auto-bumps version and build number
./scripts/release.sh

# Or specify version explicitly
./scripts/release.sh 1.0
```

This produces a signed, notarized `.dmg` that clients can download without Gatekeeper warnings.

## Step 5: Distribute to Clients

After the release script runs, the DMG is available at:
```
https://github.com/jonschack/clicky-automationinterns/releases/latest/download/Clicky.dmg
```

Link to this from automationinterns.com. Clients download, drag to Applications, and launch.

## What Clients Experience

1. **Download** the `.dmg` from your site
2. **Drag** Clicky to Applications
3. **Launch** — Clicky appears in the menu bar (top-right, blue triangle icon)
4. **Grant permissions** — the panel walks them through:
   - Microphone (for push-to-talk)
   - Accessibility (for the global keyboard shortcut)
   - Screen Recording (so Clicky can see their screen)
   - Screen Content (for detailed screen capture)
5. **Click Start** — Clicky demos by finding something on their screen and pointing at it
6. **Press Ctrl+Option** anytime they need help — Clicky takes a screenshot, listens to their question, and responds with voice + visual pointing

## Updating the App

Clicky uses Sparkle for auto-updates. When you release a new version:
1. Run `./scripts/release.sh`
2. The script updates `appcast.xml` and pushes it
3. Existing installs check the appcast feed and prompt to update automatically

## Configuration Reference

All configuration lives in `Info.plist`:

| Key | Purpose | Example |
|-----|---------|---------|
| `WorkerBaseURL` | Cloudflare Worker proxy URL | `https://clicky-proxy.example.workers.dev` |
| `VoiceTranscriptionProvider` | Speech-to-text backend | `assemblyai` (or `openai`, `applespeech`) |
| `SUFeedURL` | Sparkle auto-update feed | `https://raw.githubusercontent.com/.../appcast.xml` |

## Troubleshooting

**"API calls are failing"**: Check that `WorkerBaseURL` in Info.plist points to your deployed Worker, and that the Worker has all three secrets set (`ANTHROPIC_API_KEY`, `ASSEMBLYAI_API_KEY`, `ELEVENLABS_API_KEY`).

**"Screen recording not working"**: macOS requires a one-time manual grant. The user needs to go to System Settings > Privacy & Security > Screen Recording and enable Clicky. The app restart is required after granting this permission.

**"Keyboard shortcut not working"**: Accessibility permission must be granted. System Settings > Privacy & Security > Accessibility > enable Clicky.

**"App blocked by Gatekeeper"**: The `.dmg` must be notarized. Use `./scripts/release.sh` which handles notarization automatically. If testing a local build, right-click the app and select "Open" to bypass Gatekeeper.
