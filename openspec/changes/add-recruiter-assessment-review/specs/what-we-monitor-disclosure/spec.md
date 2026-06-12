# What We Monitor Disclosure

## ADDED Requirements

### Requirement: Disclosure page accessible from dashboard footer (D-R8)

The system SHALL provide a "What we monitor" page in the platform-shell dashboard, accessible via a footer link from every dashboard page. The page sets honest expectations for recruiters about what the integrity capture system CAN and CANNOT detect.

Route: `/help/what-we-monitor` (or similar).

#### Scenario: Footer link is present on dashboard pages
- **WHEN** a recruiter is on any of: campaign list, campaign overview, session detail, applicant detail
- **THEN** the dashboard footer includes a "What we monitor" link
- **AND** clicking it navigates to the disclosure page

### Requirement: Page lists capabilities AND limitations honestly

The page SHALL include the following sections, each with a specific, factual description:

**Capabilities (what we capture):**
- **Face presence** — face-api.js detects whether the applicant's face is visible in the webcam feed throughout the assessment
- **Face identity matching** — comparison against the Persona-verified enrollment photo at intervals; flags low-confidence matches (debounced to suppress false positives)
- **Attention tracking** — head-pose estimation flags sustained off-screen attention (e.g., looking down at a phone)
- **Audio voice activity** — detects whether human speech is occurring (no audio is recorded or stored)
- **Browser visibility / focus** — captures when the applicant switches tabs or browser windows
- **Paste / copy events** — captures copies and pastes in the short-answer textarea
- **Picture-in-Picture activity** — captures when the browser enters or exits PiP
- **Media-device fingerprinting** — detects screen-share software installed on the applicant's device (OBS, Snap Camera, etc.)
- **Multi-monitor detection** — limited; based on browser-reported screen dimensions, not the Window Management API
- **Windows screenshot detection** — captures PrintScreen keystrokes (Windows only)
- **rrweb session replay** — pixel-faithful recording of the assessment screen for post-session review

**Limitations (what we CANNOT detect):**
- **External screen capture via hardware** — HDMI splitter, capture card, or other physical screen-recording devices are invisible to the browser
- **Phone camera pointed at the screen** — a candidate photographing the screen with another device cannot be detected
- **macOS screenshots** — Cmd+Shift+3/4/5 is intercepted by the OS before the browser sees it
- **Voice-AI assistance with eye contact** — a candidate who uses voice-only ChatGPT/Claude while maintaining eye contact with the screen produces no clear off-screen-attention signal
- **Sophisticated screen-share software** — applications that disguise as legitimate camera devices may not match our fingerprint patterns
- **Browser extensions** — extension activity is best-effort detected; many extensions are invisible to detection

The disclosure page wording SHALL be reviewed by legal before going live (action item, not a code requirement).

#### Scenario: Page contains all capabilities + limitations sections
- **WHEN** the "What we monitor" page renders
- **THEN** the page contains both a "What we monitor" section (capabilities) and a "What we can't detect" section (limitations)
- **AND** every capability listed in this spec is present in the rendered page
- **AND** every limitation listed in this spec is present in the rendered page

### Requirement: Disclosure includes retention + deletion process

The page SHALL include a "How we handle this data" section covering:
- **Retention:** biometric data (Persona selfie, face embedding, rrweb recording) is retained for 180 days; audit logs are retained for 180 days; the access trail of who-viewed-what is retained for 180 days even after biometric data is deleted (operational records vs biometric records)
- **Audio:** explicitly NOT stored; only voice-activity detection runs in-memory during the assessment
- **Right-to-deletion:** applicants can request deletion by contacting their recruiter; the recruiter triggers the deletion via the dashboard

#### Scenario: Retention + deletion section is present
- **WHEN** the page renders
- **THEN** the page contains a section explaining the 180-day retention + deletion process
- **AND** the audio-not-stored disclaimer is prominently visible

### Requirement: Page is informational only — no actions

The disclosure page SHALL have no interactive controls beyond the dashboard's standard navigation footer. No forms, no buttons that perform actions, no settings to toggle. It is purely an information page.

#### Scenario: No actions on the page
- **WHEN** the page renders
- **THEN** the page body contains no buttons, no input fields, no toggles
- **AND** the only interactive elements are standard navigation (back, home, dashboard nav)
