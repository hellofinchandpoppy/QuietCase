# QuietCase — iOS App Development Specification

**Version:** 1.0  
**Date:** March 23, 2026  
**Target Build Time:** 3 hours with Claude Opus 4.6  
**App Store Price:** $2.99 (one-time purchase, no subscriptions, no ads)  
**Bundle ID suggestion:** com.yourcompany.quietcase

---

## 1. App Summary

**QuietCase** is a noise disturbance logging app for iOS. It helps renters, homeowners, and anyone dealing with noise issues document disturbances with timestamps, dB readings, descriptions, and optional audio recordings — then export everything as a professional PDF report suitable for landlords, property managers, HOAs, or legal proceedings.

**Tagline:** "Build your case for quiet."

**Category:** Utilities / Lifestyle  
**Minimum iOS:** 17.0  
**Frameworks:** SwiftUI, SwiftData, AVFoundation (AVAudioEngine for dB, AVAudioRecorder for clips), PDFKit, Charts  
**Permissions Required:** Microphone (for dB measurement and optional audio clips)

---

## 2. Why This Wins

- **Weak competition:** "The Noise App" requires UK housing providers to sign up; "LoudLog" is web-based; generic decibel meters don't log or export. No clean, native, US-market iOS app exists for this specific purpose.
- **Emotional buyer:** People suffering from noise issues are desperate and highly motivated to pay. $2.99 feels like nothing when you're losing sleep.
- **Zero backend:** Everything is on-device. SwiftData + PDFKit. No API keys, no server costs, no ongoing expenses.
- **Clear Apple Review story:** Legitimate utility that helps people document disturbances. Not spammy, not a clone, not a gimmick.
- **Virality angle:** People share their noise problems online constantly (Reddit r/neighbors, r/legaladvice, Nextdoor). "I used QuietCase to document 47 disturbances over 3 weeks and my landlord finally acted" is the kind of post that drives organic downloads.

---

## 3. Data Model (SwiftData)

### NoiseIncident
```swift
@Model
class NoiseIncident {
    var id: UUID
    var timestamp: Date              // when the disturbance started
    var endTimestamp: Date?           // when it ended (optional, can be nil)
    var peakDecibels: Double          // highest dB reading captured
    var avgDecibels: Double           // average dB during measurement
    var noiseType: String             // "Music", "Voices/Yelling", "Construction", "Pets", "Vehicles", "Footsteps/Impact", "Other"
    var noiseDescription: String      // free-text notes
    var audioClipPath: String?        // relative path to saved .m4a file (optional)
    var audioClipDuration: Int?       // seconds
    var location: String              // "From above", "From below", "Adjacent left", "Adjacent right", "Outside", "Unknown"
    var severity: Int                 // 1-5 scale (1 = minor annoyance, 5 = unbearable)
    
    init() {
        self.id = UUID()
        self.timestamp = Date()
        self.peakDecibels = 0
        self.avgDecibels = 0
        self.noiseType = "Other"
        self.noiseDescription = ""
        self.severity = 3
        self.location = "Unknown"
    }
}
```

### UserProfile (stored in @AppStorage / UserDefaults)
```
- userName: String = ""              // for PDF report header
- userAddress: String = ""           // for PDF report header
- unitNumber: String = ""            // apartment/unit number
- landlordName: String = ""          // recipient of reports
- landlordEmail: String = ""         // pre-fills share sheet
- localQuietHoursStart: String = "10:00 PM"  // for flagging violations
- localQuietHoursEnd: String = "7:00 AM"
- hasSeenOnboarding: Bool = false
```

---

## 4. Screen-by-Screen Specification

### 4.1 — Onboarding (shown once on first launch)

Three simple swipe-through pages. No account creation. No permissions yet.

**Page 1: "Document the noise"**
- Icon: waveform with a pencil
- Body: "Log each disturbance with a timestamp, dB reading, and description. Build an irrefutable record."

**Page 2: "Measure the decibels"**  
- Icon: microphone with a meter
- Body: "Use your phone's microphone to capture peak and average noise levels. Optional: record a short audio clip as evidence."

**Page 3: "Export your case"**
- Icon: document with a checkmark
- Body: "Generate a professional PDF report with every incident logged. Share it with your landlord, HOA, or attorney."

**CTA Button:** "Get Started" → navigates to main screen and requests microphone permission.

---

### 4.2 — Main Screen (Incident Log)

This is the home screen. A chronological list of all logged incidents.

**Layout:**

1. **Nav bar:** "QuietCase" title (left), gear icon for Settings (right)

2. **Summary Card (top, always visible):**
   - Left side: Total incidents logged (big number), with "incidents" label below
   - Center: Incidents during quiet hours (with a moon icon), labeled "quiet hours"
   - Right side: Days since first incident, labeled "days tracked"
   - Background: subtle secondary color, rounded corners
   - If zero incidents, show encouraging empty state instead

3. **Incident List:**
   - Sorted reverse-chronological (newest first)
   - Each row shows:
     - **Time and date** (e.g., "11:47 PM · Tue, Mar 18") — bold if during quiet hours, with a small moon icon
     - **Noise type pill** (e.g., "Music" in a colored pill badge)
     - **Peak dB** (e.g., "78 dB") with color coding:
       - < 60 dB: green
       - 60-75 dB: amber
       - 75-85 dB: orange
       - > 85 dB: red (with "LOUD" badge)
     - **Severity dots** (1-5 filled circles)
     - **Audio clip indicator** (small waveform icon if recording attached)
     - **Chevron** for detail view
   - Swipe to delete individual incidents

4. **Floating Action Button (bottom right):**
   - Large, circular, accent-colored "+" button
   - Tapping opens the New Incident screen
   - This is THE primary action — it must be impossible to miss

5. **Bottom Toolbar:**
   - Two items: "Log" (list icon, selected by default) and "Report" (document icon)
   - Tapping "Report" navigates to the Report Generation screen

**Empty State (zero incidents):**
- Illustration: A simple line drawing of a peaceful sleeping person with a "zzz"
- Title: "No incidents logged yet"
- Body: "Tap the + button whenever a noise disturbance occurs. The more you document, the stronger your case."

---

### 4.3 — New Incident Screen (Modal Sheet)

Presented as a full-screen modal when tapping the "+" button.

**Layout (scrollable form):**

1. **dB Meter Section (top, prominent):**
   - A live-updating dB display using AVAudioEngine
   - **Current dB** shown as a large number (e.g., "73 dB") with a color-coded circular gauge behind it
   - **Peak dB** shown smaller below (auto-captured, updates to highest reading seen)
   - **Average dB** shown smaller (running average since screen opened)
   - The meter starts automatically when the screen appears (mic permission already granted)
   - A thin animated waveform visualization below the number for visual feedback

2. **Record Audio Button (optional):**
   - A "Record 30s Clip" button (secondary style)
   - When tapped, records up to 30 seconds of audio as .m4a
   - Shows recording progress with a countdown timer
   - When done, shows a small playback preview with play/delete options
   - Label below: "Optional: attach a short audio clip as evidence"

3. **Noise Type Selector:**
   - Label: "What type of noise?"
   - Horizontally scrolling row of selectable pills:
     - Music, Voices/Yelling, Construction, Pets, Vehicles, Footsteps/Impact, Other
   - Single select, defaults to none (required field)

4. **Direction/Location Selector:**
   - Label: "Where is it coming from?"
   - Horizontally scrolling pills:
     - From above, From below, Adjacent left, Adjacent right, Outside, Unknown
   - Single select, defaults to "Unknown"

5. **Severity Slider:**
   - Label: "How disruptive? (1-5)"
   - A slider or 5 tappable icons that go from 😐 to 🤬 (or simple numbered circles 1-5)
   - Each level has a label: "Minor", "Noticeable", "Disruptive", "Severe", "Unbearable"

6. **Notes Field:**
   - Label: "Description (optional)"
   - Multi-line text field, 3-4 lines visible
   - Placeholder: "e.g., Loud bass music vibrating through walls, started around 11 PM..."

7. **Timestamp:**
   - Auto-populated with current date/time
   - A small "Adjust" link that opens a date/time picker (for logging a disturbance after the fact)

8. **Action Buttons (bottom):**
   - "Save Incident" — primary, full-width, accent-colored. Validates that noise type is selected, saves to SwiftData, dismisses modal with success haptic
   - "Cancel" — text button at top left of modal nav bar

---

### 4.4 — Incident Detail Screen

Pushed from tapping a row in the incident list.

Shows all fields from the incident in a clean, read-only layout:
- Date/time (with quiet hours badge if applicable)
- Peak dB / Avg dB (with color coding)
- Noise type
- Direction/location
- Severity (visual dots)
- Description
- Audio playback (if clip attached) — a simple waveform with play button
- "Edit" button in nav bar (opens same form as New Incident, pre-populated)
- "Delete Incident" button at bottom (destructive, with confirmation alert)

---

### 4.5 — Report Generation Screen

Accessed via the "Report" tab in the bottom toolbar.

**Layout:**

1. **Report Preview Section:**
   - A scaled-down preview of the first page of the PDF report
   - Shows how many incidents will be included

2. **Date Range Filter:**
   - "All Time" by default
   - Tappable to select: "Last 7 Days", "Last 30 Days", "Last 90 Days", "Custom Range" (opens date pickers)

3. **Include Options (toggles):**
   - Include quiet hours analysis: ON by default
   - Include severity breakdown: ON by default
   - Include dB statistics: ON by default
   - Include incident descriptions: ON by default
   - Include your contact info: OFF by default (uses UserProfile data)

4. **Action Buttons:**
   - "Generate PDF Report" — primary, full-width. Generates the PDF and presents a share sheet
   - "Preview Report" — secondary, opens a full-screen PDF preview

**PDF Report Layout:**

```
========================================
          NOISE DISTURBANCE REPORT
               QuietCase
========================================

Prepared by: [User Name]
Address: [User Address], Unit [Unit Number]
Report Date: March 23, 2026
Reporting Period: Feb 21, 2026 – Mar 23, 2026

----------------------------------------
SUMMARY
----------------------------------------
Total incidents documented: 47
Incidents during quiet hours (10 PM – 7 AM): 31
Average incident severity: 3.8 / 5
Average peak decibel level: 74 dB
Highest recorded peak: 92 dB (March 14 at 1:23 AM)

----------------------------------------
QUIET HOURS ANALYSIS
----------------------------------------
[Bar chart showing incidents by hour of day]
[Highlights that 66% of incidents occurred during 
designated quiet hours]

----------------------------------------
INCIDENT LOG
----------------------------------------

#1 — March 22, 2026, 11:47 PM ⚠️ QUIET HOURS
Type: Music
Direction: From above
Severity: ●●●●○ (4/5)
Peak dB: 78 | Avg dB: 72
Duration: ~45 minutes
Notes: "Loud bass music vibrating through ceiling. 
Could feel vibrations in bed. Went on until past 
midnight."
Audio Evidence: Yes (30 sec clip attached)

#2 — March 21, 2026, 12:15 AM ⚠️ QUIET HOURS
Type: Voices/Yelling
...

[continues for all incidents in date range]

----------------------------------------
This report was generated by QuietCase 
(quietcase.app) and contains [X] documented 
incidents. All timestamps reflect the device's 
local time zone at the time of logging.
========================================
```

---

### 4.6 — Settings Screen

Accessed via the gear icon.

**Sections:**

**Your Information (for reports)**
- Name — text field
- Address — text field
- Unit/Apartment Number — text field
- Landlord/Manager Name — text field
- Landlord/Manager Email — text field (pre-fills share sheet)

**Quiet Hours**
- Start Time — time picker (default 10:00 PM)
- End Time — time picker (default 7:00 AM)
- Note: "Incidents during quiet hours will be flagged in your reports"

**App Settings**
- Audio Recording Quality — High / Standard (High = better evidence, larger files)
- Max Recording Duration — 15s / 30s / 60s (default 30s)

**Data**
- Export All Data (CSV) — exports raw incident data as CSV
- Clear All Data — destructive button with "Type DELETE to confirm" prompt
- Incident count shown as "47 incidents logged since Feb 21, 2026"

**About**
- App version
- Rate QuietCase — App Store link
- Share QuietCase — share sheet
- Privacy Policy — NavigationLink to in-app PrivacyPolicyView (NOT a URL — see Section 16)
- Terms of Use — NavigationLink to in-app TermsOfUseView (NOT a URL — see Section 16)
- "QuietCase does not collect, transmit, or store any data outside your device."

---

## 5. App Architecture

### File Structure
```
QuietCase/
├── QuietCaseApp.swift               // @main, SwiftData modelContainer
├── Models/
│   └── NoiseIncident.swift
├── ViewModels/
│   ├── IncidentLogViewModel.swift   // list management, filtering, stats
│   ├── DecibelMeterViewModel.swift  // AVAudioEngine dB measurement
│   └── ReportViewModel.swift        // PDF generation, date filtering
├── Views/
│   ├── MainTabView.swift            // Bottom tab bar (Log + Report)
│   ├── OnboardingView.swift         // First-launch walkthrough
│   ├── IncidentListView.swift       // Main log screen
│   ├── NewIncidentView.swift        // Modal form for logging
│   ├── IncidentDetailView.swift     // Read-only detail
│   ├── ReportView.swift             // Report generation screen
│   ├── SettingsView.swift           // Settings
│   ├── PrivacyPolicyView.swift      // In-app privacy policy (native SwiftUI text)
│   ├── TermsOfUseView.swift         // In-app terms of use (native SwiftUI text)
│   └── Components/
│       ├── DecibelGaugeView.swift   // Animated dB display with gauge
│       ├── SeverityDotsView.swift   // 1-5 severity indicator
│       ├── NoiseTypePill.swift      // Selectable pill for noise type
│       ├── SummaryCardView.swift    // Stats card at top of log
│       ├── IncidentRowView.swift    // Individual list row
│       └── AudioPlayerView.swift    // Compact audio playback
├── Utilities/
│   ├── DecibelMeter.swift           // AVAudioEngine wrapper for dB
│   ├── AudioRecorder.swift          // AVAudioRecorder wrapper
│   ├── PDFReportGenerator.swift     // Builds the PDF report
│   ├── DateFormatters.swift         // Shared formatters
│   └── HapticManager.swift          // Centralized haptics
├── Resources/
│   └── Assets.xcassets              // App icon, colors
└── Info.plist                       // NSMicrophoneUsageDescription
```

---

## 6. Decibel Measurement (Core Technical Detail)

### DecibelMeter.swift

Uses AVAudioEngine with an input node tap to get real-time dB readings.

```swift
import AVFoundation

@Observable
class DecibelMeter {
    var currentDb: Double = 0
    var peakDb: Double = 0
    var avgDb: Double = 0
    var isRunning = false
    
    private var audioEngine = AVAudioEngine()
    private var readings: [Double] = []
    
    func start() {
        let session = AVAudioSession.sharedInstance()
        try? session.setCategory(.record, mode: .measurement)
        try? session.setActive(true)
        
        let inputNode = audioEngine.inputNode
        let format = inputNode.outputFormat(forBus: 0)
        
        inputNode.installTap(onBus: 0, bufferSize: 1024, format: format) { [weak self] buffer, _ in
            guard let self else { return }
            
            let channelData = buffer.floatChannelData?[0]
            let frameLength = UInt(buffer.frameLength)
            
            var rms: Float = 0
            vDSP_measqv(channelData!, 1, &rms, frameLength)
            rms = sqrtf(rms)
            
            // Convert to dB (referenced to typical full-scale)
            let db = 20 * log10(rms) + 100  // +100 offset for iPhone mic calibration approximation
            let clampedDb = max(20, min(Double(db), 130))  // Clamp to realistic range
            
            DispatchQueue.main.async {
                self.currentDb = clampedDb
                self.peakDb = max(self.peakDb, clampedDb)
                self.readings.append(clampedDb)
                self.avgDb = self.readings.reduce(0, +) / Double(self.readings.count)
            }
        }
        
        try? audioEngine.start()
        isRunning = true
    }
    
    func stop() {
        audioEngine.inputNode.removeTap(onBus: 0)
        audioEngine.stop()
        isRunning = false
    }
    
    func reset() {
        peakDb = 0
        avgDb = 0
        readings.removeAll()
    }
}
```

**Important calibration note:** iPhone microphones are not calibrated instruments. The +100 offset is an approximation. Include a disclaimer in Settings: "Decibel readings are approximate and may not match professional equipment. They are suitable for documenting relative noise levels and patterns."

---

## 7. Audio Recording

### AudioRecorder.swift

Records short audio clips as evidence. Files saved to the app's documents directory.

```swift
import AVFoundation

@Observable
class AudioRecorder {
    var isRecording = false
    var recordingDuration: TimeInterval = 0
    var audioFilePath: URL?
    
    private var audioRecorder: AVAudioRecorder?
    private var timer: Timer?
    private var maxDuration: TimeInterval = 30
    
    func startRecording(maxSeconds: Int = 30) {
        maxDuration = TimeInterval(maxSeconds)
        let filename = UUID().uuidString + ".m4a"
        let path = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
            .appendingPathComponent(filename)
        
        let settings: [String: Any] = [
            AVFormatIDKey: Int(kAudioFormatMPEG4AAC),
            AVSampleRateKey: 44100,
            AVNumberOfChannelsKey: 1,
            AVEncoderAudioQualityKey: AVAudioQuality.high.rawValue
        ]
        
        audioRecorder = try? AVAudioRecorder(url: path, settings: settings)
        audioRecorder?.record(forDuration: maxDuration)
        audioFilePath = path
        isRecording = true
        recordingDuration = 0
        
        timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
            guard let self else { return }
            self.recordingDuration += 1
            if self.recordingDuration >= self.maxDuration {
                self.stopRecording()
            }
        }
    }
    
    func stopRecording() {
        audioRecorder?.stop()
        timer?.invalidate()
        isRecording = false
    }
    
    func deleteRecording() {
        if let path = audioFilePath {
            try? FileManager.default.removeItem(at: path)
        }
        audioFilePath = nil
        recordingDuration = 0
    }
}
```

---

## 8. PDF Report Generation

### PDFReportGenerator.swift

Uses UIGraphicsPDFRenderer to create a professional multi-page PDF.

The PDF should include:
1. **Header page:** Report title, user info, date range, summary statistics
2. **Analysis page:** Bar chart of incidents by hour of day (highlight quiet hours in red), breakdown by noise type (pie or bar), severity distribution
3. **Incident log pages:** Every incident in chronological order with all details

For charts in the PDF, draw them manually using Core Graphics (UIBezierPath, CGContext). Keep it simple — horizontal bar charts are easiest to render and most readable.

**Key function signature:**
```swift
static func generateReport(
    incidents: [NoiseIncident],
    dateRange: ClosedRange<Date>,
    userProfile: UserProfile,
    options: ReportOptions
) -> Data  // PDF data
```

**ReportOptions:**
```swift
struct ReportOptions {
    var includeQuietHoursAnalysis: Bool = true
    var includeSeverityBreakdown: Bool = true
    var includeDbStatistics: Bool = true
    var includeDescriptions: Bool = true
    var includeContactInfo: Bool = false
}
```

---

## 9. Color System

| Name | Light Mode | Dark Mode | Usage |
|------|-----------|-----------|-------|
| AccentColor | #5B4FCF | #7B6FE8 | Primary buttons, selected tabs |
| DecibelGreen | #34C759 | #30D158 | dB < 60 |
| DecibelAmber | #FF9500 | #FFB340 | dB 60-75 |
| DecibelOrange | #FF6B35 | #FF8555 | dB 75-85 |
| DecibelRed | #FF3B30 | #FF453A | dB > 85 |
| QuietHoursBadge | #5856D6 | #7D7AFF | Moon icon, quiet hours indicators |
| SeverityLow | #A8D5BA | #6BBF8A | Severity 1-2 |
| SeverityMid | #FFD966 | #FFCA28 | Severity 3 |
| SeverityHigh | #FF6B6B | #FF4757 | Severity 4-5 |

Use semantic system colors for backgrounds, text, and separators. Define the above as named colors in Assets.xcassets for automatic light/dark mode support.

---

## 10. Info.plist Keys

```xml
<key>NSMicrophoneUsageDescription</key>
<string>QuietCase uses the microphone to measure noise levels and optionally record short audio clips as evidence of disturbances.</string>
```

No other permissions needed. No location, no contacts, no camera, no network.

---

## 11. App Store Metadata

**App Name:** QuietCase — Noise Log  
**Subtitle:** Document disturbances. Build your case.

**Keywords:** noise complaint, neighbor noise, noise log, disturbance diary, decibel meter, noise tracker, noise report, landlord, HOA, apartment noise, quiet hours

**Description (first paragraph, most important for search):**
"Dealing with noisy neighbors? QuietCase helps you document noise disturbances with timestamps, decibel readings, and optional audio recordings. Generate professional PDF reports to share with your landlord, property manager, HOA, or attorney. Build an irrefutable record — because one complaint can be ignored, but a documented pattern cannot."

**Privacy:** No data collected. All data stays on device. No analytics, no tracking.

**App Review Notes:** "QuietCase is a documentation and measurement tool. It uses the device microphone solely to measure ambient noise levels (decibel readings) and to optionally record short audio clips that the user saves locally. No data is transmitted off-device. The app generates PDF reports that users can share via the standard iOS share sheet."

---

## 12. Monetization & Growth Strategy

**Price:** $2.99 one-time purchase. No free tier, no subscriptions, no ads. The paid-upfront model signals quality and eliminates App Review concerns about subscription dark patterns.

**Why people will pay:**
- They're in pain (noise is disrupting their life)
- The alternative is doing nothing or trying to track with Notes app
- $2.99 is trivial compared to the problem they're solving
- The PDF report alone justifies the price (it looks professional and saves hours of manual documentation)

**Growth channels:**
- Reddit: r/legaladvice, r/neighbors, r/ApartmentLiving, r/homeowners, r/HOA
- Nextdoor (noise complaint threads are constant)
- TikTok/Reels: "I documented 47 noise disturbances and my landlord finally did something" — the report is inherently visual and shareable
- SEO: "how to document noise complaints" lands on your app's website

---

## 13. Build Priority Order

When building in 3 hours, prioritize in this exact order:

1. **Data model + SwiftData container** (15 min)
2. **DecibelMeter class** (20 min) — core technical feature
3. **NewIncidentView with dB gauge** (45 min) — THE primary user flow
4. **IncidentListView with rows** (30 min) — where users see their data
5. **IncidentDetailView** (15 min) — simple read view
6. **PDFReportGenerator** (30 min) — the monetization justifier
7. **ReportView with share sheet** (15 min)
8. **SettingsView** (15 min)
9. **SummaryCard + polish** (15 min)
10. **Onboarding + empty states** (10 min)
11. **Audio recording** (if time allows — this is a "nice to have")

**Cut list if running behind:** Audio recording, onboarding carousel, Charts in PDF (just use text stats), incident editing (just allow delete and re-create).

---

## 14. App Icon Direction

A stylized waveform (sound wave) combined with a shield or gavel shape, in deep purple/indigo tones. Should feel authoritative but approachable — like a tool that means business but doesn't look like a legal app. Avoid: generic microphone icons, angry faces, noise/speaker icons with X marks.

---

## 15. Post-Launch Enhancements (v1.1+)

These are NOT in scope for v1.0 but should be considered for updates:
- **Widgets:** Home screen widget showing "X incidents this week" and a quick-log button
- **Shortcuts integration:** "Hey Siri, log a noise incident" 
- **iCloud sync:** Sync incidents across devices
- **Photo attachments:** Capture photos of damage or the source
- **Location tagging:** Optional GPS for incidents away from home
- **Noise type auto-detection:** Use ML to classify noise type (music vs. construction vs. pets)
- **Landlord email template:** Pre-written email template with the PDF attached

---

## 16. In-App Legal Views & Hosting Strategy

### Strategy

Apple requires a Privacy Policy URL in App Store Connect — there's no way around that field. But the in-app links in SettingsView should navigate to native SwiftUI views (not Safari). This gives you:

- **Instant loading** — no network request, no web view, no loading spinner
- **Automatic dark mode** — matches the system theme
- **Automatic Dynamic Type** — respects the user's font size settings
- **No embedded browser concerns** — Apple reviewers sometimes flag WKWebView usage

**Two-part approach:**
1. **In-app:** PrivacyPolicyView.swift and TermsOfUseView.swift (native SwiftUI, shown below)
2. **App Store Connect URL field:** Point to `https://hellofinchandpoppy.github.io/QuietCase/privacy` — a static GitHub Pages site (see Section 17 for the files)

### SettingsView.swift — Legal Navigation Links

Replace the old URL-based links with NavigationLinks:

```swift
// In SettingsView.swift, inside the "About" section:

Section("About") {
    // ... app version, rate, share rows ...
    
    NavigationLink {
        PrivacyPolicyView()
    } label: {
        Label("Privacy Policy", systemImage: "hand.raised")
    }
    
    NavigationLink {
        TermsOfUseView()
    } label: {
        Label("Terms of Use", systemImage: "doc.text")
    }
    
    Text("QuietCase does not collect, transmit, or store any data outside your device.")
        .font(.caption)
        .foregroundStyle(.secondary)
}
```

### PrivacyPolicyView.swift

```swift
import SwiftUI

struct PrivacyPolicyView: View {
    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 24) {
                
                // Header
                VStack(alignment: .leading, spacing: 8) {
                    Text("Privacy Policy")
                        .font(.largeTitle.bold())
                    Text("Effective March 23, 2026")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                
                // Summary callout
                HStack(spacing: 12) {
                    Image(systemName: "checkmark.shield.fill")
                        .font(.title2)
                        .foregroundStyle(.green)
                    Text("QuietCase collects no data. Everything stays on your device. No analytics, no tracking, no accounts.")
                        .font(.subheadline)
                }
                .padding()
                .background(.green.opacity(0.08))
                .clipShape(RoundedRectangle(cornerRadius: 12))
                
                // Sections
                legalSection("1. Who we are",
                    "QuietCase is a mobile application published on the Apple App Store. This Privacy Policy explains how we handle information when you use the QuietCase app."
                )
                
                legalSection("2. Information we collect",
                    """
                    We collect nothing.
                    
                    QuietCase operates entirely on your device. We do not collect, receive, transmit, or have access to any personal information, usage data, or content you create within the app.
                    
                    • We do not require an account, login, or registration.
                    • We do not collect your name, email, phone number, or any personal identifier.
                    • We do not collect device identifiers, IP addresses, or advertising IDs.
                    • We do not use analytics frameworks, crash reporting, or tracking SDKs.
                    • We do not use cookies or similar tracking technologies.
                    • The app makes no network requests. It operates fully offline.
                    """
                )
                
                legalSection("3. Data stored on your device",
                    """
                    QuietCase stores the following data locally on your device:
                    
                    • Noise incident logs: Timestamps, decibel readings, noise type, severity ratings, location direction, and descriptions you enter.
                    • Audio recordings: Short audio clips (if you choose to record them) saved as .m4a files in the app's sandboxed storage.
                    • Your profile information: Name, address, unit number, and landlord/manager details you optionally enter for PDF reports.
                    • App preferences: Quiet hours settings, recording quality, and configuration choices.
                    
                    This data exists only on your device and in your personal iCloud backup if you have device backups enabled. We cannot access, view, or retrieve this data under any circumstances.
                    """
                )
                
                legalSection("4. Microphone usage",
                    """
                    QuietCase requests microphone access for two purposes:
                    
                    • Decibel measurement: The app uses the microphone to measure ambient sound levels in real time. Audio data is processed on-device to calculate decibel readings and is not recorded or stored unless you explicitly choose to record a clip.
                    • Audio evidence recording: If you tap the Record button, the app records a short audio clip and saves it locally. This recording is never transmitted anywhere.
                    
                    You may revoke microphone access at any time through Settings. Without it, dB measurement and recording will be unavailable, but you can still log incidents manually.
                    """
                )
                
                legalSection("5. PDF reports",
                    "When you generate a PDF report, it is created entirely on your device. The report may contain incident data and personal information you have entered. When you share a report using the iOS share sheet, it is sent through whatever channel you choose. We have no involvement in or visibility into this sharing."
                )
                
                legalSection("6. Third-party services",
                    "QuietCase uses zero third-party services. No analytics. No advertising networks. No crash reporting. No social media SDKs. The app is entirely self-contained."
                )
                
                legalSection("7. Children's privacy",
                    "QuietCase does not knowingly collect any information from anyone, including children under 13. Since the app collects no data whatsoever, there is no risk of inadvertent collection of children's information."
                )
                
                legalSection("8. Data deletion",
                    """
                    Since all data resides on your device, you have complete control:
                    
                    • Delete individual incidents from within the app.
                    • Clear all data using the "Clear All Data" option in Settings.
                    • Delete the app entirely, which removes all associated data.
                    
                    We cannot delete your data because we do not have it.
                    """
                )
                
                legalSection("9. Data security",
                    "Your data is protected by iOS security measures including your device passcode, Face ID/Touch ID, and Apple's app sandboxing. Data stored by QuietCase is accessible only to the app and is encrypted at rest when your device is locked."
                )
                
                legalSection("10. Changes to this policy",
                    "If we change this Privacy Policy, we will update this page and the effective date. If a future version were to collect any data, we would notify you through an in-app notice before any collection begins and require your explicit consent."
                )
                
                legalSection("11. Contact us",
                    "If you have questions about this Privacy Policy, contact us at privacy@quietcase.app"
                )
                
                Text("© 2026 QuietCase. All rights reserved.")
                    .font(.caption)
                    .foregroundStyle(.tertiary)
                    .frame(maxWidth: .infinity)
                    .padding(.top, 16)
                
            }
            .padding()
        }
        .navigationTitle("Privacy Policy")
        .navigationBarTitleDisplayMode(.inline)
    }
    
    @ViewBuilder
    private func legalSection(_ title: String, _ body: String) -> some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(title)
                .font(.headline)
            Text(body)
                .font(.subheadline)
                .foregroundStyle(.secondary)
                .fixedSize(horizontal: false, vertical: true)
        }
    }
}
```

### TermsOfUseView.swift

```swift
import SwiftUI

struct TermsOfUseView: View {
    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 24) {
                
                // Header
                VStack(alignment: .leading, spacing: 8) {
                    Text("Terms of Use")
                        .font(.largeTitle.bold())
                    Text("Effective March 23, 2026")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                
                // Summary callout
                HStack(spacing: 12) {
                    Image(systemName: "info.circle.fill")
                        .font(.title2)
                        .foregroundStyle(.orange)
                    Text("QuietCase is a documentation tool — not a legal service. Decibel readings are approximate. You are responsible for complying with local recording laws.")
                        .font(.subheadline)
                }
                .padding()
                .background(.orange.opacity(0.08))
                .clipShape(RoundedRectangle(cornerRadius: 12))
                
                // Sections
                legalSection("1. Acceptance of terms",
                    "By downloading, installing, or using QuietCase, you agree to these Terms of Use. If you do not agree, do not use the app."
                )
                
                legalSection("2. Description of service",
                    """
                    QuietCase allows you to:
                    
                    • Log noise disturbance incidents with timestamps, descriptions, and severity ratings.
                    • Measure approximate ambient sound levels using your device's microphone.
                    • Optionally record short audio clips as supporting evidence.
                    • Generate PDF reports summarizing your logged incidents.
                    • Share those reports with third parties of your choosing.
                    
                    The app is a documentation and personal record-keeping tool. It is not a legal service, a professional noise measurement instrument, or a substitute for professional advice.
                    """
                )
                
                legalSection("3. License",
                    """
                    We grant you a limited, non-exclusive, non-transferable, revocable license to use the app on Apple devices you own or control, as permitted by the App Store Terms of Service. You may not:
                    
                    • Modify, reverse-engineer, decompile, or disassemble the app.
                    • Redistribute, sublicense, or make the app available to third parties.
                    • Use the app for any unlawful purpose.
                    • Remove or alter any copyright or proprietary notices.
                    """
                )
                
                legalSection("4. Decibel measurement disclaimer",
                    """
                    IMPORTANT: QuietCase uses your device's built-in microphone to estimate sound levels. These readings are approximate and should not be treated as calibrated, scientific, or legally authoritative measurements.
                    
                    Consumer smartphone microphones are not calibrated instruments. Readings may vary based on device model, case, microphone condition, orientation, and environmental factors. They are useful for documenting relative noise levels and patterns, but should not be presented as precise measurements in legal proceedings without independent professional verification.
                    """
                )
                
                legalSection("5. Audio recording and legal compliance",
                    """
                    You are solely responsible for ensuring your use of the audio recording feature complies with all applicable local, state, and federal laws.
                    
                    • Recording laws vary by jurisdiction. Some require consent of all parties ("two-party consent"). Others permit recording if at least one party consents ("one-party consent").
                    • Recording ambient noise (music, construction) is generally treated differently from recording private conversations, but the legal distinction can be nuanced.
                    • We strongly recommend researching the recording laws in your jurisdiction before using this feature.
                    • We are not responsible for any legal consequences arising from your use of the audio recording feature.
                    """
                )
                
                legalSection("6. Not legal advice",
                    """
                    Nothing in the app, its reports, or associated materials constitutes legal advice.
                    
                    • We make no representation about the legal admissibility or weight of any report.
                    • We make no guarantees about how third parties will respond to your reports.
                    • We do not provide guidance on noise ordinances, tenant rights, or dispute resolution.
                    • If involved in a legal dispute, consult a licensed attorney in your jurisdiction.
                    """
                )
                
                legalSection("7. Your content and data",
                    "All data you create within the app remains yours. We do not collect, access, or have any rights to your content. You are responsible for the accuracy and truthfulness of the information you enter. Fabricating or exaggerating incident records and presenting them as factual documentation may have legal consequences."
                )
                
                legalSection("8. Intellectual property",
                    "The app, including its design, code, icons, and documentation, is protected by copyright and intellectual property laws. The QuietCase name and logo are trademarks. All rights not expressly granted are reserved."
                )
                
                legalSection("9. Limitation of liability",
                    """
                    To the maximum extent permitted by law:
                    
                    • The app is provided "as is" and "as available" without warranties of any kind.
                    • We do not warrant the accuracy of decibel measurements or any data produced by the app.
                    • We shall not be liable for any indirect, incidental, special, consequential, or punitive damages.
                    • Our total liability shall not exceed the amount you paid for the app.
                    """
                )
                
                legalSection("10. Indemnification",
                    "You agree to indemnify and hold harmless QuietCase and its officers, directors, employees, and agents from any claims, damages, losses, and expenses arising from your use of the app, violation of these terms, or violation of any applicable law."
                )
                
                legalSection("11. Termination",
                    "We may terminate your license at any time. Upon termination, cease all use and delete the app. Sections 6, 9, 10, and 12 survive termination."
                )
                
                legalSection("12. Governing law",
                    "These Terms are governed by the laws of the Commonwealth of Kentucky, United States. Disputes shall be resolved in Kentucky state or federal courts."
                )
                
                legalSection("13. Apple-specific terms",
                    "These Terms are between you and QuietCase, not Apple Inc. Apple has no obligation to provide maintenance or support. Apple is not responsible for claims relating to the app. Apple and its subsidiaries are third-party beneficiaries of these Terms."
                )
                
                legalSection("14. Severability",
                    "If any provision is found unenforceable, it shall be limited to the minimum extent necessary so remaining provisions remain in effect."
                )
                
                legalSection("15. Changes to these terms",
                    "We may modify these Terms at any time. Material changes will be reflected on this page with an updated effective date. Continued use constitutes acceptance."
                )
                
                legalSection("16. Contact us",
                    "Questions about these Terms? Contact us at legal@quietcase.app"
                )
                
                Text("© 2026 QuietCase. All rights reserved.")
                    .font(.caption)
                    .foregroundStyle(.tertiary)
                    .frame(maxWidth: .infinity)
                    .padding(.top, 16)
                
            }
            .padding()
        }
        .navigationTitle("Terms of Use")
        .navigationBarTitleDisplayMode(.inline)
    }
    
    @ViewBuilder
    private func legalSection(_ title: String, _ body: String) -> some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(title)
                .font(.headline)
            Text(body)
                .font(.subheadline)
                .foregroundStyle(.secondary)
                .fixedSize(horizontal: false, vertical: true)
        }
    }
}
```

---

## 17. GitHub Pages Hosting (App Store Connect Requirement)

App Store Connect has a required "Privacy Policy URL" field that must be a web URL — you cannot skip it. The simplest solution is a static GitHub Pages site hosted from your existing repo.

**Repository:** `https://github.com/hellofinchandpoppy/QuietCase/`
**Pages URL:** `https://hellofinchandpoppy.github.io/QuietCase/`

Create a `/docs` folder in the repo root with three files:
- `docs/index.html` — minimal landing page
- `docs/privacy.html` — privacy policy (web version)
- `docs/terms.html` — terms of use (web version)

Then in GitHub repo Settings → Pages, set Source to "Deploy from a branch", branch `main`, folder `/docs`.

**App Store Connect URLs to enter:**
- Privacy Policy URL: `https://hellofinchandpoppy.github.io/QuietCase/privacy`  
- Marketing URL (optional): `https://hellofinchandpoppy.github.io/QuietCase/`

If you later purchase the `quietcase.app` domain, add a CNAME file and update the URLs.

See the companion deliverable files for the complete HTML for all three pages.
