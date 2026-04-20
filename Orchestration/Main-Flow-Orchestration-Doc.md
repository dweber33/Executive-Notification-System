# Orchestration Layer
## Executive Notification System (ENS) · Power Automate Backend Architecture

This document is a detailed technical reference for the three Power Automate Cloud Flows that form the orchestration backbone of the ENS. It covers the design logic, execution sequence, inter-flow data contracts, unique coding patterns, and the architectural decisions behind every non-trivial step, sourced directly from the production flow JSON definitions.

---

## Table of Contents

1. [System Overview: Three-Flow Architecture](#1-system-overview-three-flow-architecture)
2. [Flow 1 - Main Notification Engine (ExecutiveNotificationAppCollector)](#2-flow-1--main-notification-engine)
3. [Flow 2 - External Telemetry Listener (Provider Alert Tracker)](#3-flow-2--external-telemetry-listener)
4. [Flow 3 - Monthly Governance Audit (Audit-ExecutiveContacts)](#4-flow-3--monthly-governance-audit)
5. [Inter-Flow Interoperability & Shared Data Contracts](#5-inter-flow-interoperability--shared-data-contracts)
6. [Cross-Cutting Patterns & Unique Techniques](#6-cross-cutting-patterns--unique-techniques)
7. [Failure Architecture & Resiliency](#7-failure-architecture--resiliency)
8. [Connection Reference Model](#8-connection-reference-model)

---

## 1. System Overview: Three-Flow Architecture

The ENS orchestration layer is composed of three independently deployed Cloud Flows, each with a distinct trigger model, runtime boundary, and responsibility domain. They do not call each other directly, they are decoupled through a shared SharePoint data layer that functions as the system's integration bus.

```
┌──────────────────────────────────────────────────────────────────────┐
│                         TRIGGER SURFACE                              │
│                                                                      │
│  Power Apps Button ──► Flow 1 (Main Engine)                                    │
│  Incoming Email    ──► Flow 2 (External Telemetry Listener)  [Async / Passive]  │
│  Monthly Timer     ──► Flow 3 (Governance Audit)  [Scheduled]        │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    SHARED DATA LAYER (SharePoint)                    │
│                                                                      │
│  ┌─────────────────────┐   ┌────────────────────┐                    │
│  │  ExecNotif Tracker  │   │  External Telemetry  │  ◄── Written by    │
│  │  (Session History)  │   │  (Provider Alerts)   │      Flow 2        │
│  └──────────┬──────────┘   └──────────┬─────────┘                    │
│             │ Read/Write by            │ Read by                      │
│             │ Flow 1                   │ Flow 1                       │
│  ┌──────────▼──────────────────────────▼─────────┐                    │
│  │           StakeholderContacts List             │  ◄── Read by      │
│  │   (Name, Email, Email2Text, Twilio, Carrier)   │      Flow 1 & 3   │
│  └────────────────────────────────────────────────┘                    │
└──────────────────────────────────────────────────────────────────────┘
```

**Key architectural principle:** Flow 2 is entirely passive. It runs continuously in the background, listening for incoming vendor emails and writing parsed data into the `External Telemetry` SharePoint list. Flow 1, when triggered, reads from that same list. This is a deliberate **event-sourcing pattern**, the two flows are temporally decoupled but share a common data substrate.

---

## 2. Flow 1 - Main Notification Engine

**File:** `ExecutiveNotificationAppCollector-470C24A1-D1E5-F011-8544-7C1E520AF318.json`  
**Trigger:** `PowerAppV2` (manual, synchronous)  
**Connectors:** `shared_office365`, `shared_sharepointonline` (invoker), `shared_teams` (invoker), `shared_twilio`

This is the primary orchestration engine. It is triggered synchronously by the Power App and must return a `{ "status": "Complete" }` response to release the UI lockout. Every other action in the flow is sequenced within that contract.

---

### Phase 1: Data Ingestion & Initialization

#### Payload Parsing - The Single-String Workaround

The Power Apps V2 trigger contract accepts a single typed input. Rather than defining individual parameters for each field (Priority, INC, Status, etc.), the entire incident context is serialized into a **single JSON string** in Power Apps using the `JSON()` function, then sent as one `text` field:

```json
// Trigger schema - only one input field
"inputs": {
  "schema": {
    "type": "object",
    "properties": {
      "text": {
        "title": "AppData",
        "type": "string",
        "x-ms-content-hint": "TEXT"
      }
    }
  }
}
```

The first action, `Parse_JSON_From_PowerApp`, immediately deserializes this string back into a strongly typed object:

```
Parse_JSON_From_PowerApp
  Content: @triggerBody()?['text']
  Schema: { Priority, Status, INC, Title, Notification,
            Bridge, InternalChannel, Executives, TechStatus,
            SessionID, ResponderName, ResponderEmail }
```

**Why this matters:** This pattern bypasses the 16-parameter limit on Power Apps V2 triggers and allows the app-side data contract to evolve independently without re-establishing the connector. It also means the entire payload is atomic, all fields arrive together or not at all.

#### SessionID Initialization

Immediately after parsing, the `SessionID` string is extracted from the payload into a dedicated flow variable:

```
Initialize_Variable_SessionID
  Value: @body('Parse_JSON_From_PowerApp')?['SessionID']
```

This value is either `"NEW"` (first notification) or an existing GUID (subsequent updates). The variable is mutable, it will be overwritten with a real GUID if the flow determines this is a new session. This design allows all downstream actions to reference `variables('SessionID')` without branching logic.

#### Internal Channel Map - Static Routing Table as a Compose Block

One of the most elegant patterns in the flow is the internal channel routing logic. Rather than storing channel URLs in SharePoint or environment variables, the flow uses a single `Compose` action as an **inline lookup dictionary**:

```json
// Internal_Channel_Map (Compose action)
{
  "ChannelAlpha": "https://teams.microsoft.com/l/chat/19:{thread-id-alpha}.../conversations?...",
  "ChannelBravo": "https://teams.microsoft.com/l/chat/19:{thread-id-bravo}.../conversations?...",
  "ChannelCharlie": "https://teams.microsoft.com/l/chat/19:{thread-id-charlie}.../conversations?...",
  "ChannelDelta": "https://teams.microsoft.com/l/chat/19:{thread-id-delta}.../conversations?...",
  "None": ""
}
```

The downstream variable is then set using a **dynamic key lookup** against this object:

```
Initialize_and_Set_Variable_Internal_Chat_Link
  Value: @outputs('Internal_Channel_Map')?[json(triggerBody()?['text'])?['InternalChannel']]
```

The `InternalChannel` value from the payload (e.g., `"ChannelAlpha"`) is used as the property key to index directly into the compose output. This is a zero-overhead routing mechanism, no additional SharePoint lookups, no HTTP calls, no switch statements. The URL resolution happens in a single expression.

---

### Phase 2: Contextual Enrichment (External Telemetry Audit)

Before dispatching any notifications, the flow performs a real-time audit of the External Telemetry SharePoint list.

#### The OData Filter Query

```
Get_items_TechStatus
  $filter: INC eq '@{body('Parse_JSON_From_PowerApp')?['INC']}'
  $orderby: ID desc
```

The `$orderby: ID desc` is intentional, the most recent external telemetry entry for this INC number is sorted to the top of the result set, which is critical for the `first()` function used downstream.

#### The Boolean Gate Pattern

The result count is evaluated using a simple but powerful length check:

```
Initialize_and_Set_Variable_TechStatus_Check
  // Description from flow: "It counts the rows found. If the count is
  //  greater than 0, the variable becomes True. Otherwise, it becomes False."
  Value: @greater(length(outputs('Get_items_TechStatus')?['body/value']), 0)
```

This boolean (`varCalculatedTechStatus`) serves dual duty: it gates the HTML construction block, and it is injected directly into the email body and SMS message body as a human-readable status field (`Was an External Telemetry Communication Sent?`).

#### Dynamic HTML Construction via Loop

When `varCalculatedTechStatus = true`, the flow executes a three-step HTML construction sequence:

**Step 1 - Capture the most recent record:**
```
Set_variable_as_Object
  Value: @first(outputs('Get_items_TechStatus')?['body/value'])
```

**Step 2 - Build the communication history rows:**
The `Apply_to_each_1` loop iterates over *all* matching external telemetry records and appends a formatted HTML table row for each entry into `varHTML_HistoryRows`:

```html
<!-- Row template (per iteration) -->
<tr>
  <td style="font-size: 12px; color: #666;">
    @{formatDateTime(item()?['Created'], 'MM/dd/yy h:mm tt')}
  </td>
  <td style="font-size: 12px;">
    <strong>@{item()?['Status']}</strong>: @{item()?['BusinessStatus']}
  </td>
</tr>
```

**Step 3 - Assemble the complete section:**
`Set_variable_as_HTML` wraps the accumulated `varHTML_HistoryRows` string inside a fully structured HTML table, referencing `varTechStatus_Record` for top-level metadata (title, status, business segment, start/end time, and a direct link to the TechStatus record).

If `varCalculatedTechStatus = false`, both `varHTML_TechStatusSection` and `varHTML_HistoryRows` remain empty strings, and the section simply doesn't render in the email, because the email template references the variable placeholder directly.

---

### Phase 3: Contact Retrieval

```
Get_Exec_Contacts
  $top: 5000
  view: {your-sharepoint-view-guid}
```

The `$top: 5000` ceiling prevents pagination issues. More critically, the flow queries a **specific SharePoint View** by its GUID rather than the default list view. This means the active distribution scope (who actually receives notifications) can be modified entirely in SharePoint without touching the flow, a clean separation between data governance and execution logic.

The contact list schema contains four delivery-relevant fields used in Phase 4:
- `Email` - for the HTML executive email
- `Email2Text` - for the carrier gateway SMS path  
- `Twilio` - for the Twilio API SMS path (may be empty for some contacts)
- *(The `Carrier` field is used by the governance flow, not the main flow)*

---

### Phase 4: Omnichannel Dispatch

The `Production_Flow` condition block is structurally an `If(1 == 1)` - always true. This is a **scaffolding pattern**: the condition wrapper exists to provide a named, collapsible scope in the Power Automate designer, not to perform actual branching. The "true" branch contains the entire dispatch logic.

Inside, `Email_Notification` and `Text_Notification` are parallel `Scope` blocks, they execute concurrently, not sequentially.

#### Email Notification Scope

**Recipient List Construction:**

```
Select_Emails   → extracts Email field from each contact record
Join_emails_together → joins with ";" delimiter
```

The resulting semicolon-delimited string is passed directly to `emailMessage/To`. Office 365 accepts this format natively.

**New Notification Entry Formatting:**

Before the `First_Notification` condition, the flow composes a timestamped HTML block that will become the newest entry in the cumulative history log:

```html
<div style="border-left: 4px solid #0078d4; background-color: #f4f4f4; padding: 10px;">
  <div style="font-weight: bold; font-size: 12px;">
    UPDATE: @{formatDateTime(convertFromUtc(utcNow(), 'Eastern Standard Time'), 'MM/dd/yyyy h:mm tt')}
  </div>
  <div style="margin-top: 5px;">
    @{body('Parse_JSON_From_PowerApp')?['Notification']}
  </div>
</div>
```

The `convertFromUtc()` call wraps `utcNow()` to display Eastern Standard Time, an important operational detail, since all stakeholders and logs are aligned to the same timezone.

#### The SessionID Branching Gate

```
First_Notification condition:
  @equals(variables('SessionID'), 'NEW')
```

**Branch A - First Notification (SessionID == "NEW"):**

1. `Set_variable_SessionID_NEW` - overwrites the `SessionID` variable with a fresh `@guid()`. From this point forward, every reference to `variables('SessionID')` carries the new GUID.
2. `Send_First_Email` - dispatches the initial HTML notification email.
3. `Create_item_Tracker_List` - persists the session to SharePoint, writing the GUID as the `SessionID` field and the formatted notification entry as the initial `History` value.

The first email template differs from the update template: it uses `INCIDENT NOTIFICATION` as the section header (no history log section), because there is no history yet.

**Branch B - Update/Resolve/Disengage (SessionID != "NEW"):**

1. `Get_items_Notification_History` - retrieves the existing SharePoint record using an OData filter:
   ```
   $filter: SessionID eq '@{body('Parse_JSON_From_PowerApp')?['SessionID']}'
   ```
   Note: this uses the **payload's** `SessionID`, not the variable, because the variable was already set in Phase 1 and hasn't been overwritten in this branch.

2. `For_each_1` - loops over the returned records (expected: exactly 1). Inside the loop:
   - `Send_Update_email` - dispatches the update email. This template includes a `LATEST SITUATION UPDATE` callout box at the top (the newest entry, visually distinct in blue) and the full `INCIDENT ALERT HISTORY` section below it, rendered from `item()?['History']`.
   - `Update_item` - writes back to SharePoint, **prepending** the new entry to the existing history:
     ```
     item/History: "<p>@{outputs('Format_New_Notification_Entry')}</p>
                   <p>@{item()?['History']}</p>"
     ```
     This is the **Recursive History Logger** - each update prepends to the accumulated string. An executive reading the latest email sees the complete chronological timeline without needing to cross-reference previous messages.

#### Teams Adaptive Card (Post-Branch, Always Executes)

After the `First_Notification` condition resolves (either branch), `Post_card_in_a_chat_or_channel` fires unconditionally:

```json
// Simplified card action buttons
{
  "type": "Action.OpenUrl",
  "title": "📝 Update",
  "url": "https://apps.powerapps.com/play/{AppID}?tenantId={TenantID}
          &SessionID=@{variables('SessionID')}&Action=Update&source=Teams"
},
{
  "type": "Action.OpenUrl",
  "title": "✅ Resolve",
  "url": "...&SessionID=@{variables('SessionID')}&Action=Resolve&source=Teams"
},
{
  "type": "Action.OpenUrl",
  "title": "👋 Disengage",
  "url": "...&SessionID=@{variables('SessionID')}&Action=Disengage&source=Teams"
}
```

The `SessionID` injected into each URL is the **resolved GUID**, whether newly generated or retrieved from the existing session. This means the card, from the moment it posts, carries the correct session DNA. Any manager who clicks a button in that card will launch the Power App with the exact session context pre-loaded.

The card also embeds the live `Bridge` link and the `InternalChatLink` variable, providing direct-dial navigation to both the incident bridge and the internal war room.

#### Text Notification Scope (Parallel)

The `Text_Notification` scope executes concurrently with `Email_Notification` and contains two parallel SMS delivery paths:

**Path A - Email-to-Text (Carrier Gateway):**

```
SelectEmail2Text  → extracts Email2Text field from each contact
Join_Email2Text   → joins with ";"
Send_Email2text   → sends via Office 365 to carrier gateway addresses
```

Email-to-Text relies on each carrier's SMTP-to-SMS gateway (e.g., `5550101@vtext.com` for Verizon). The message body is deliberately minimal, carrier gateways impose strict character limits and strip HTML:

```
Subject: @{Priority} @{INC} - @{Status}
Body:    @{Notification}
         External Telemetry Sent: @{varCalculatedTechStatus}
         Executives: @{Executives}
         Coordinator: @{ResponderName}
         Bridge: @{Bridge}
```

**Path B - Twilio API:**

The Twilio path adds a filtering step not present in the Email-to-Text path:

```
Filter_array
  where: @not(empty(item()?['Twilio']))
```

This filters the contact array down to only records where the `Twilio` field is populated, because not all contacts have a registered Twilio number. The filtered array feeds directly into `Apply_to_each`, which calls `Send_Text_Message_(SMS)` for each record.

The Twilio message body mirrors the Email-to-Text content but uses the registered sender number `+1{your-10DLC-number}` (the A2P 10DLC registered long code). This number is hardcoded in the flow, changing the sender number requires a flow edit, which is an intentional governance control.

---

### Phase 5: Lifecycle & Response

```
Respond_to_a_Power_App_or_flow
  statusCode: 200
  body: { "status": "Complete" }
```

This response is the unlock signal for the Power App's `locProcessing` context variable. The UI lockout modal that prevents double-submission only releases when this response is received. The flow's success path terminates exactly here.

---

## 3. Flow 2 - External Telemetry Listener

**File:** `ExternalTelemetry-CommunicationTracker-[ID].json`  
**Trigger:** `OnNewEmailV3` - fires on each email received from the external provider's automated alerting system  
**Connectors:** `shared_conversionservice`, `shared_sharepointonline`, `shared_office365`

This flow is the ENS's passive intelligence layer. It runs continuously in the background without any human intervention and feeds the data that Flow 1 uses for its External Telemetry enrichment section. The external provider operates their own independent incident notification system, this flow is the integration bridge that parses their automated email broadcasts and surfaces that data within the ENS context.

---

### Trigger Model: Email-Scoped Filtering

```json
"triggers": {
  "When_a_new_email_arrives_(V3)": {
    "splitOn": "@triggerOutputs()?['body/value']",
    "inputs": {
      "parameters": {
        "from": "provider.techstatus@[external-domain].com"
      }
    }
  }
}
```

The `from` filter ensures only emails from the designated external provider system address trigger the flow. The `splitOn` property enables **parallel branching**, if multiple emails arrive simultaneously, Power Automate creates a separate concurrent run for each one rather than processing them sequentially.

### HTML-to-Text Stripping

The incoming external provider email is an HTML-formatted message. Before any data extraction can occur, the `Html_to_text` action (using the `shared_conversionservice` connector) strips all markup, leaving clean plaintext that can be reliably parsed with string functions.

```
Html_to_text
  Content: <p>@{triggerOutputs()?['body/body']}</p>
```

This is a necessary pre-processing step, the downstream `split()` and `trim()` expressions depend on predictable text boundaries that HTML tags would corrupt.

### Data Extraction: String Parsing as a Pipeline

The `Extract_Data_From_E-mail` Scope contains seven sequential Compose actions, each extracting one data point from the email body using chained string functions. Each step builds on the previous one's output. The parsing strategies vary by field:

**Email Subject Parsing (Status & INC):**

The external provider's email subject follows the format: `*Resolved* Subject Text (INC0001234)`

```
// Extract Status, grabs text between the two * delimiters
Compose_Extract_Status:
  @trim(split(triggerOutputs()?['body/subject'], '*')[1])

// Extract INC, grabs text between the last ( and )
Compose_Extract_Incident:
  @first(split(last(split(triggerOutputs()?['body/subject'], '(')), ')'))
```

**Email Body Parsing (Segment, Status, Times):**

Body fields are extracted by identifying their label as a boundary and splitting on it:

```
// Business Segment, text between "Business Segment" and "Incident"
@trim(first(split(last(split(outputs('Html_to_text')?['body'], 'Business Segment')), 'Incident')))

// Business Status, text between "Business Status" and "Start Time"
@trim(first(split(last(split(outputs('Html_to_text')?['body'], 'Business Status')), 'Start Time')))

// Start Time, text after "Start Time" up to the first newline
@trim(first(split(trim(last(split(outputs('Html_to_text')?['body'], 'Start Time'))), decodeUriComponent('%0A'))))
```

The `decodeUriComponent('%0A')` is a notable technique, it decodes to a newline character `\n`, which cannot be typed directly into a Power Automate expression. This is the standard workaround for literal newline splitting.

**End Time / Next Update, Conditional Extraction:**

The End Time field uses a multi-level conditional because the external provider's email format changes depending on whether the incident is resolved or still active:

```
// If resolved: "End Time" exists → extract it
// If active:   "Next Update" exists → extract it instead
// Fallback:    null

@if(
  contains(outputs('Html_to_text')?['body'], 'End Time'),
  trim(first(split(trim(last(split(body, 'End Time'))), decodeUriComponent('%0A')))),
  if(
    contains(outputs('Html_to_text')?['body'], 'Next Update'),
    trim(first(split(trim(last(split(body, 'Next Update'))), decodeUriComponent('%0A')))),
    null
  )
)
```

**URL Extraction (TechStatus & Bridge Links):**

Links embedded in the HTML-to-text output appear in Markdown-like format: `[link text](url)`. The flow extracts them by splitting on the known URL prefix:

```
// TechStatus Link
@concat(
  'https://techstatus.disney.com/e-tech/events/',
  split(split(body, 'https://techstatus.disney.com/e-tech/events/')[1], ']')[0]
)

// Bridge Link, text between [ and ] after "Join the Incident Bridge in Teams"
@split(split(last(split(body, 'Join the Incident Bridge in Teams')), '[')[1], ']')[0]
```

### SharePoint Write

After all seven fields are extracted, `Create_item` writes a new record to the DTOC TechStatus list:

```
item/INC:             @outputs('Compose_-_Extract_Incident')
item/Status:          @outputs('Compose_-_Extract_Status')
item/BusinessSegment: @outputs('Compose_-_Extract_Business_Segment')
item/BusinessStatus:  @outputs('Compose_-_Extract_Business_Status')
item/StartTime:       @outputs('Compose_-_Extract_Start_Time')
item/EndTime:         @outputs('Compose_-_Extract_End_Time')
item/TechStatusLink:  @outputs('Compose_-_Extract_Tech_Status_Link')
item/BridgeLink:      @outputs('Compose_-_Extract_Incident_Bridge_Link')
item/DialByNumber:    @outputs('Compose_-_Extract_Dial-in_Number')
```

Each new DTOC email creates a **new row**, the list accumulates a complete communication history for every incident. Flow 1 retrieves all matching rows (`$orderby: ID desc`) and renders the full history table in the email.

---

## 4. Flow 3 - Monthly Governance Audit

**File:** `Audit-ExecutiveContacts-7868141A-ABF3-F011-8406-7C1E520AF318.json`  
**Trigger:** Recurrence - 1st of each month, 09:00 AM EST  
**Connectors:** `shared_sharepointonline`, `shared_office365`

This flow is the simplest in structure but serves a critical governance function: it forces human review of the contact roster that drives all three delivery channels. The flow has zero dependencies on the other two flows and operates entirely as a standalone scheduled process.

### Execution Sequence

```
Recurrence → Get_items → Create_HTML_table → Compose_Format_HTML_Table
          → Select_Emails_From_List → Join_E-mails_Together → Send_email
```

### Native HTML Table Generation

The `Create_HTML_table` action uses Power Automate's native `Table` type (rather than a manual loop) to generate a base HTML table from the SharePoint items array:

```json
// Create_HTML_table action
{
  "type": "Table",
  "inputs": {
    "from": "@outputs('Get_items')?['body/value']",
    "format": "HTML",
    "columns": [
      { "header": "Name",         "value": "@item()?['Title']"       },
      { "header": "E-mail",       "value": "@item()?['Email']"        },
      { "header": "Phone Number", "value": "@item()?['PhoneNumber']"  },
      { "header": "Carrier",      "value": "@item()?['Carrier']"      }
    ]
  }
}
```

This outputs a raw, unstyled HTML table. The `Compose_Format_HTML_Table` step then wraps it with a complete `<style>` block using inline CSS:

```html
<style>
  table  { width: 100%; border-collapse: collapse; font-family: Segoe UI; }
  th     { background-color: #00478f; color: white; }
  td     { padding: 4px 6px; border: 1px solid #ccc; }
  tr:nth-child(even) { background-color: #f4f4f4; }
</style>
@{body('Create_HTML_table')}
```

The `#00478f` header color is the corporate brand blue, ensuring the audit report is visually consistent with other enterprise communications.

### Self-Targeting Recipient List

A notable design choice: the audit email is not sent to a hardcoded distribution list. Instead, the recipient list is **dynamically built from the same data being audited**:

```
Select_Emails_From_List  → extracts Email from each contact record
Join_E-mails_Together    → joins with ";"
Send_email → To: @body('Join_E-mails_Together')
```

This means the audit is delivered to exactly the people currently on the list, including any new additions and excluding any removals that happened since the last audit. It's self-referential: the list governs who reviews the list.

### Email Configuration

```
Subject: Situation Manager Alert Contact List – @{formatDateTime(utcNow(),'MMMM yyyy')} – Audit
From:    CORP.SitMan.Notification@disney.com
ReplyTo: corp.dl-disney-sitman@disney.com
```

The dynamic subject line (`formatDateTime(utcNow(),'MMMM yyyy')`) auto-generates the correct month/year on every execution, making the email self-describing and easily searchable in archive.

---

## 5. Inter-Flow Interoperability & Shared Data Contracts

The three flows share two SharePoint lists as their integration layer. Neither flow calls the other directly. The coupling is entirely data-driven.

### Shared List 1: StakeholderContacts

| Consumer | Access Type | Purpose |
|---|---|---|
| Flow 1 (Main Engine) | Read | Build Email/SMS/Twilio recipient lists |
| Flow 3 (Audit) | Read | Build audit report table + self-targeting recipient list |

Both flows query the same list but access different fields. Flow 1 uses `Email`, `Email2Text`, and `Twilio`. Flow 3 uses `Title` (Name), `Email`, `PhoneNumber`, and `Carrier`. Neither flow writes to this list, it is a human-managed data source.

### Shared List 2: DTOC TechStatus

| Consumer | Access Type | Purpose |
|---|---|---|
| Flow 2 (DTOC Listener) | Write | Parse and persist incoming vendor alert emails |
| Flow 1 (Main Engine) | Read | Enrich email body with external telemetry section |

The INC number (`item/INC`) is the join key. Flow 1 filters by `INC eq '@{payload.INC}'` to find all matching records. Multiple DTOC emails for the same INC accumulate as separate rows, and Flow 1 renders all of them in the communication history table.

### Shared List 3: ExecutiveNotification Tracker

| Consumer | Access Type | Purpose |
|---|---|---|
| Flow 1 (Main Engine) | Read + Write | Session persistence (create on NEW, patch on update) |
| Power App | Read | Populate the home screen gallery + deep link validation |

This list is the session state store. The `SessionID` GUID is the primary key used for all lookups. Only Flow 1 writes to it, no other flow touches it.

---

## 6. Cross-Cutting Patterns & Unique Techniques

### Pattern 1: Inline Dictionary Routing (Compose as Lookup Table)

Used in Flow 1's `Internal_Channel_Map`. A Compose action is given a JSON object with named keys. A downstream variable uses the payload field value as the dynamic property key to index into that object. This eliminates a database lookup, a switch statement, or a condition chain, the entire routing table resolves in a single expression.

### Pattern 2: Boolean Gate via Length Check

Used in Flow 1's `varCalculatedTechStatus`. The expression `@greater(length(array), 0)` converts a collection result into a boolean in one step. This boolean is then used both to gate the HTML construction block and as a human-readable value injected into the notification body (`True` / `False`).

### Pattern 3: Recursive HTML Prepending

Used in Flow 1's `Update_item`. The History field is updated as:
```
"<p>@{new_entry}</p><p>@{item()?['History']}</p>"
```
This creates a self-accumulating log. Each dispatch adds one entry at the top. The email template reads the full field verbatim, rendering a complete timestamped timeline to the recipient without any aggregation logic.

### Pattern 4: Decode-Escaped Newline for String Splitting

Used in Flow 2's Start/End Time extraction:
```
split(text, decodeUriComponent('%0A'))
```
`decodeUriComponent('%0A')` produces `\n`. Power Automate expression editors do not accept raw newline characters in string literals, so this decode function is the standard workaround for splitting on line breaks.

### Pattern 5: Email-as-SMS via SMTP Gateway

Used in Flow 1's `Email_to_Text` scope. Rather than requiring all recipients to have a Twilio number, the flow maintains a secondary `Email2Text` field per contact containing the carrier's SMTP-to-SMS gateway address (e.g., `5550101@vtext.com`). The Office 365 connector sends a plain-text email to these addresses, which the carrier automatically converts to an SMS. This gives the system a no-cost, no-registration SMS fallback path for contacts not enrolled in Twilio.

### Pattern 6: Invoker vs. Embedded Connection Runtime

Flow 1 uses two different connection runtime modes:
- `shared_office365` and `shared_twilio`, `runtimeSource: "embedded"` (the flow runs as the connection owner)
- `shared_sharepointonline` and `shared_teams`, `runtimeSource: "invoker"` (the flow runs as the user who triggered it from Power Apps)

The invoker model for SharePoint ensures that SharePoint audit logs attribute data writes to the actual operator who sent the notification, not to the flow's service account. This is a deliberate governance decision that preserves a complete, attributable audit trail.

### Pattern 7: Dynamic App URL with Runtime Parameters

The failure notification card in Flow 1 constructs a direct deep link to the specific failed run in Power Automate:

```
@{concat(
  'https://make.powerautomate.com/environments/',
  workflow()['tags']['environmentName'],
  '/flows/',
  workflow()['name'],
  '/runs/',
  workflow()['run']['name']
)}
```

`workflow()` is a system function that exposes runtime metadata about the executing flow. This link takes an admin directly to the exact failed run in the Power Automate portal, zero navigation required.

---

## 7. Failure Architecture & Resiliency

### Synchronous Handshake with the UI

The `Respond_to_a_Power_App_or_flow` action only runs after `Production_Flow` succeeds. If `Production_Flow` fails or times out, the response action is skipped, and the UI remains in its lockout state (the modal stays open). This prevents the operator from thinking the notification was sent when it wasn't.

### The Failure Notification Card

The `Post_card_with_flow_bot_on_Failure` action is configured to trigger on `TimedOut`, `Skipped`, and `Failed` states of the response action:

```json
"runAfter": {
  "Respond_to_a_Power_App_or_flow": ["TimedOut", "Skipped", "Failed"]
}
```

This posts an Adaptive Card to the admin monitoring channel containing:
- The failure timestamp (`formatDateTime(utcNow(), ...)`)
- The flow's display name (`workflow()['tags']['flowDisplayName']`)
- A direct link to the failed run in Power Automate

An admin can respond to a notification failure in seconds without navigating the Power Automate portal manually.

### Parallel SMS Paths as Redundancy

The dual SMS delivery architecture (Email-to-Text + Twilio) means that even if one delivery path degrades, the other continues independently. They execute in parallel within the same scope, so a failure in the Twilio path does not block the Email-to-Text path, and vice versa.

---

## 8. Connection Reference Model

| Connection Reference | Runtime Source | Used By | Purpose |
|---|---|---|---|
| `shared_office365` | embedded | Flow 1, 2, 3 | Send HTML emails, Email-to-Text SMS |
| `shared_sharepointonline` | invoker (Flow 1) / embedded (Flow 2, 3) | All | All SharePoint read/write operations |
| `shared_teams` | invoker | Flow 1 | Post Adaptive Cards to group chats |
| `shared_twilio` | embedded | Flow 1 | Send A2P SMS via Twilio REST API |
| `shared_conversionservice` | embedded | Flow 2 | Strip HTML from incoming vendor emails |

The mixed invoker/embedded model on `shared_sharepointonline` is intentional. Flow 1 writes to SharePoint as the invoking operator (governance traceability). Flow 2 and 3, which run on automated triggers with no human invoker, use the embedded (service account) model.

---

*This document reflects the production state of the orchestration layer as implemented in the ENS Power Automate environment. All SharePoint table GUIDs, Teams chat thread IDs, and App/Tenant IDs referenced in the flow definitions should be updated to match your target environment during deployment.*
