# The ENS Start Screen

## Architecture Design Overview

![Landing Page](../img/ENS_UI_StartScreen.png)

### Overview

In critical incident management, the "Initial Notification" is the most volatile action an operator can take. In a Managed Service Provider (MSP) environment, a single outage can trigger a "Fog of War" where multiple responders attempt to coordinate across different time zones. Without a structured entry point, this leads to **Communication Sprawl**-duplicate alerts, conflicting incident numbers, and fragmented telemetry.

The **Start Screen** of the Executive Notification System (ENS) was designed not merely as a landing page, but as a strategic **Operational Landing Page** to enforce state reconciliation before a single byte of data is transmitted.

### 1\. The Design Logic: Why a Landing Page?

Landing an operator directly on a communication form in a high-pressure environment is a known UX anti-pattern. The ENS Start Screen enforces a mandatory **State Check** based on two primary design goals:

#### A. Duplicate Prevention & Shared Awareness

By integrating a live **Incident Gallery** on the entry screen, the system provides immediate situational awareness.

- **The Goal:** Before a manager clicks "New," they are forced to see if a colleague has already initiated a session for the current outage.
- **The Result:** Teams transition from "Creation" to "Synchronization," ensuring stakeholders receive a single, continuous narrative rather than competing alerts.

#### B. Intent-Based Provisioning

The UI separates the **Provisioning** of a new session from the **Maintenance** of an existing one. This clears cached data from previous sessions, preventing "Ghost Data" (leftover INC numbers or bridge links) from accidentally being sent in a new notification.

### 2\. Core Feature: Omnichannel State Synchronization (Deep Linking)

The most critical logic of the Start Screen is its ability to **disappear** when not needed. To maximize efficiency for returning users, the system utilizes a sophisticated Deep Link Protocol.

#### The "Skip-Logic" Architecture

When an executive or manager clicks an "Update" button inside a **Teams Adaptive Card** or an **Outlook Email**, they are not sent to the Start Screen. Instead, the application performs an instantaneous **State Handshake**:

- **Intercept:** The app intercepts a SessionID parameter from the URL.
- **Validate:** It queries the IncidentNotificationLogger (SharePoint) to verify the session is still active.
- **Bypass:** The app logic evaluates the presence of the ID and automatically routes the user directly to the **Control Panel**, pre-loading the exact context of that incident.

This creates a seamless loop: **Automation initiates the context, and the UI provides the surgical interface to update it.**

### 3\. Visual Hierarchy & Intent

The visual design prioritizes authority and clarity to reduce cognitive load:

- **Compliance Banner:** A prominent "Authorized Use Only" section establishes the gravity of the tool, reminding operators that they are handling executive-level data.
- **Dual-Path Interaction:** The screen is split between a large, high-contrast CTA for "New" events and a dense, data-rich gallery for "Existing" events.
- **Active Highlighting:** Using dynamic TemplateFill logic, the gallery visually "anchors" the selected incident, providing immediate visual confirmation of which session is being loaded into memory.

### 4\. Technical Implementation (Power Fx)

The following implementation logic governs the transition from the "Landing Page" to the active workspace.

#### App-Level Routing (Deep Link Bypass)

This logic is placed in the StartScreen property of the application object to determine the entry point.
```
// If a SessionID is passed via URL parameters, skip the Home Screen.  
If(  
!IsBlank(Param("SessionID")),  
Screen_ControlPannel,  
Screen_Home  
)
```
#### Provisioning Logic (Start New)

Ensures a clean state for the new SessionID.
```
// Clear session cache and set mode to Provisioning  
Set(varSelectedRecord, Blank());  
UpdateContext({ varMode: "New" });  
<br/>Navigate(Screen_ControlPannel);
```
#### Synchronization Logic (Gallery Selection)

This complex routine ensures the SessionID remains the immutable anchor even if metadata (like INC numbers) is missing or TBD.
```
// 1. Clear session overrides to prevent data leakage  
Set(varCurrentAction, Blank());  
Set(varCurrentSessionID, Blank());  
Set(varTempINC, Blank());  
<br/>// 2. Activate "TBD Trap" logic if the incident is in early investigation  
Set(varForceTBDReset, ThisItem.Priority = "TBD");  
<br/>// 3. Synchronize record context and navigate  
Set(varSelectedRecord, ThisItem);  
UpdateContext({ varMode: "Existing" });  
<br/>Navigate(Screen_ControlPannel, ScreenTransition.None);
```
### 5\. Data Architecture Requirements

To maintain the integrity of this design, the Start Screen relies on a high-availability connection to:

- **Incident Tracker List:** The primary telemetry source for the live gallery.
- **Notification Logger:** The authoritative source for SessionID validation and deep-link reconciliation.
