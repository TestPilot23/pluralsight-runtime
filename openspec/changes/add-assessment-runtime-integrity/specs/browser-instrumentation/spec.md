# Browser Instrumentation

## ADDED Requirements

### Requirement: Tab visibility + window focus events are silent (D-SM2-pre)

The system SHALL capture browser visibility and focus events as silent audit-log events. The events SHALL NOT trigger any applicant-facing UX (no toaster, no warning, no pause).

**Events:**
- `tab.visibility_lost` — fired when document.visibilityState transitions to "hidden" (visibilitychange API)
- `tab.visibility_restored` — fired when document.visibilityState returns to "visible"
- `window.focus_lost` — fired on the window's blur event (OS-level focus loss to another app)
- `window.focus_restored` — fired on the window's focus event

Both `tab.*` and `window.*` events are captured because they signal different behaviors. Switching tabs in the same browser fires `tab.visibility_lost` without `window.focus_lost`. Alt-tabbing to another desktop app fires both. The analyzer interprets the combined pattern.

#### Scenario: Tab switch fires visibility event without focus event
- **WHEN** an applicant Cmd+T to a new tab within the same browser
- **THEN** `tab.visibility_lost` is emitted
- **AND** `window.focus_lost` is NOT emitted (browser window itself retains focus)

#### Scenario: Cmd+Tab to another app fires both
- **WHEN** an applicant Cmd+Tab to Slack
- **THEN** both `tab.visibility_lost` AND `window.focus_lost` are emitted
- **WHEN** the applicant returns
- **THEN** both `tab.visibility_restored` AND `window.focus_restored` are emitted

#### Scenario: No applicant UX on visibility/focus loss
- **WHEN** an applicant switches away from the assessment for any reason
- **THEN** no toaster, warning, or pause is triggered
- **AND** the assessment continues normally (timer keeps running, etc.)

### Requirement: Picture-in-Picture events are silent

The system SHALL emit silent events on Picture-in-Picture lifecycle:
- `picture_in_picture.entered` on the `enterpictureinpicture` event
- `picture_in_picture.exited` on the `leavepictureinpicture` event

These indicate some screen-recording workflows that pop video into PiP. Best-effort signal for the analyzer.

#### Scenario: PiP entry emits event
- **WHEN** any video element on the page transitions to Picture-in-Picture mode
- **THEN** `picture_in_picture.entered` is emitted with payload `{ video_element_id?, item_id_at_time? }`

### Requirement: Media device enumeration changes detect screen-share software

The system SHALL listen for `navigator.mediaDevices.devicechange` events. On each change (and once at session start):
- Enumerate devices via `enumerateDevices()`
- Diff against the previous device list
- Emit `media_devices.changed` with payload `{ added: DeviceInfo[], removed: DeviceInfo[] }`

The system SHALL ALSO match device labels against a regex of known screen-share software fingerprints:
- `OBS`, `Snap`, `ManyCam`, `XSplit`, `Loopback`, `BlackHole`, `Aggregate`, `NDI`

When any device matches one of these fingerprints (either at session start OR on a subsequent change), emit `media_devices.suspicious_present` with payload `{ matched_device_label, matched_pattern }`. Silent.

#### Scenario: Plugging in a USB headset emits media_devices.changed
- **WHEN** an applicant plugs in a USB headset mid-session
- **THEN** `media_devices.changed` is emitted with the headset in the `added` array

#### Scenario: OBS Virtual Camera triggers suspicious_present
- **WHEN** an applicant has OBS Virtual Camera installed (device label contains "OBS Virtual Camera")
- **AND** the session starts and enumerates devices
- **THEN** `media_devices.suspicious_present` is emitted with `{ matched_device_label: "OBS Virtual Camera", matched_pattern: "OBS" }`

#### Scenario: Hardware capture is undetectable (acknowledged limitation)
- **WHEN** an applicant uses an external HDMI capture card to record their screen
- **THEN** no `media_devices.suspicious_present` event fires (capture cards don't appear in `enumerateDevices()`)
- **AND** the system makes no false claim of detecting this attack vector

### Requirement: Secondary display detection without permission prompt

The system SHALL detect secondary displays via the basic `window.screen` API (no Window Management API permission request per D-DC1). At session start:
- If `window.screen.isExtended === true` OR screen dimensions exceed typical primary-monitor bounds → emit `display.secondary_present` with payload `{ primary_width, primary_height, total_width, total_height }`

If the applicant denies the Window Management API permission (in a future version that prompts), emit `display.permission_denied` — the denial itself is a signal.

#### Scenario: Multi-monitor setup emits display.secondary_present
- **WHEN** a session starts on a desktop with `window.screen.isExtended: true`
- **THEN** `display.secondary_present` is emitted once at session start
- **AND** the payload includes the screen dimensions

#### Scenario: Single-monitor setup emits nothing
- **WHEN** a session starts on a single-monitor laptop
- **THEN** no `display.secondary_present` event is emitted

### Requirement: Copy + paste detection on short-answer textareas

The system SHALL listen for `copy` and `paste` events on the document. When such events fire while a short-answer textarea is focused, emit:
- `copy.detected` with payload `{ text, item_id_at_time, timestamp }` — the text being copied
- `paste.detected` with payload `{ text, item_id_at_time, timestamp }` — the text being pasted

Phase 1 partially wired these events for the short-answer renderer; this phase ensures full coverage including the document-level listener that catches paste events that the React component's onChange might miss.

#### Scenario: Paste into short-answer fires event
- **WHEN** an applicant pastes "binary search" into a short-answer textarea on item I7
- **THEN** `paste.detected` is emitted with `{ text: "binary search", item_id_at_time: "I7", timestamp }`

#### Scenario: Copy from short-answer fires event
- **WHEN** an applicant selects and copies text from their short-answer textarea
- **THEN** `copy.detected` is emitted with the copied content as payload

### Requirement: IP-change detection mid-session

The system SHALL monitor the applicant's public IP and emit `ip.changed` events when it changes mid-session. Implementation: poll the runtime's `/whoami` HTTPS endpoint at 5-minute intervals + on `tab.visibility_restored`. Compare to the prior known IP; emit on difference.

The telemetry-service stamps the per-event server-side IP separately (per integrity-event-taxonomy spec) — this client-side module's job is to surface IP CHANGES as an explicit signal.

Payload: `{ previous_ip, current_ip, item_id_at_time? }`.

#### Scenario: IP change emits event
- **WHEN** an applicant's mobile hotspot reconnects mid-session and assigns a different IP
- **THEN** `ip.changed` is emitted on the next /whoami poll with the new IP

#### Scenario: Stable IP emits nothing
- **WHEN** an applicant's IP remains stable throughout the session
- **THEN** no `ip.changed` events are emitted

### Requirement: Screenshot detection on Windows (D-SM2-pre — partial coverage acknowledged)

The system SHALL listen for `PrintScreen` keydown events on the document. On detection, emit `screenshot.print_screen_detected` with payload `{ item_id_at_time?, timestamp }`. Silent event.

**Limitations (honestly acknowledged in the spec):**
- Windows: PrintScreen is reliably detected
- macOS: Cmd+Shift+3/4/5 is INTERCEPTED by the OS before the browser sees the keydown — detection fails entirely
- Mobile/touch screenshot gestures: undetectable
- Phone camera pointed at screen: undetectable

The recruiter dashboard's "what we monitor" page (Phase 3) MUST be honest about these limitations.

#### Scenario: Windows PrintScreen emits event
- **WHEN** an applicant on Windows Chrome presses PrintScreen
- **THEN** `screenshot.print_screen_detected` is emitted

#### Scenario: macOS screenshot is not detected (acknowledged)
- **WHEN** an applicant on macOS presses Cmd+Shift+4
- **THEN** no `screenshot.print_screen_detected` event is emitted (browser cannot detect)

### Requirement: Extension activity detection (best-effort)

The system SHALL use a MutationObserver on document.body to detect unexpected DOM mutations that did not originate from the React app's render cycle. Combined with checks for known extension footprints (e.g., presence of `window.Grammarly`, `window._copilot`, etc.), emit `extension.suspicious_activity` with payload `{ detection_method, evidence }` when signals are seen. Silent.

This is BEST-EFFORT. Many extensions are undetectable. The spec acknowledges this; the analyzer treats the signal as a hint, not proof.

#### Scenario: Grammarly extension detected
- **WHEN** an applicant has Grammarly enabled and `window.Grammarly` is present at session start
- **THEN** `extension.suspicious_activity` is emitted with `{ detection_method: "window_footprint", evidence: "Grammarly" }`

#### Scenario: Unknown extension is not detected (acknowledged)
- **WHEN** an applicant has a niche extension installed without window-level footprints
- **THEN** no `extension.suspicious_activity` event may fire — best-effort acknowledged
