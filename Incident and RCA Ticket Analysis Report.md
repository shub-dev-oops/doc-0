# Incident and RCA Ticket Analysis Report

This report provides a structured analysis of the incident tickets, grouped by **Severity Level** and **Application**. It highlights root causes, actions taken, permanent fix status, and linked tickets.

## SEVERITY LEVEL: Sev 1

### Application: Swagit

| Ticket ID | Title | Status | Permanent Fix | RCA Included? |
| :--- | :--- | :--- | :--- | :--- |
| RUN-124252 | ["SwagIT"] GovMeetings - SwagIt Degradation Created: 18/Apr/... | In | No | Yes (Linked) |

#### Details for RUN-124252
- **Root Cause:** A Chef configuration update associated with Rocky Linux 9 / FedRAMP readiness unintentionally enforced SELinux policies on Swagitproduction servers.These policies prevented Apache from reading .htaccess and related content directories, resulting in access-denied errors across livestreaming, multi-view, and wiki services.Contributing factors: * Production servers lacked a lower-environment equivalent for validation
- **Action Taken:** Action to ResolveIdentified Chef-driven SELinux enforcement as the source of the issueRolled back the Chef change only for GAS MP production serversManually restored file ownership to ApacheCorrected SELinux contexts using restoreconValidated live streams, multi-view, and wiki functionalityUpdated customer-facing status page and closed incident
- **Linked Open Tickets:** RUN-124327, RUN-124254, RUN-124388, SRE-18389, SRE-18398

---

## SEVERITY LEVEL: Sev 2

### Application: OneMeeting

| Ticket ID | Title | Status | Permanent Fix | RCA Included? |
| :--- | :--- | :--- | :--- | :--- |
| RUN-123557 | govMeetings - OneMeeting Primegov - Content Control Function... | Resolved | Yes | Yes (Linked) |
| RUN-123738 | govMeetings OneMeeting - CANADA - Content Control Functional... | Resolved | Yes (Status Resolved) | Yes (Linked) |
| RUN-123738 | govMeetings OneMeeting - CANADA - Content Control Functional... | Resolved | Yes (Status Resolved) | Yes (Linked) |

#### Details for RUN-123557
- **Root Cause:** A CKEditor JavaScript regression introduced auto-formatting behavior that wrapped form-submitted text in <div> tags, causing content to falloutside document content controls and breaking automation logic.
- **Action Taken:** Who When (CT) Action
- **Linked Open Tickets:** RUN-123738, SRE-18327

#### Details for RUN-123738
- **Root Cause:** Not found
- **Action Taken:** The definitive corrective action was the deployment of the previously successful U.S. configuration fix to theCanadian servers, followed by a CDN cache purge. This decision was made collaboratively during the bridge, withexecution led by Anh Hoang, executed by Chitresh Das and validation confirmed by Sarah Kennedy at approximately2:55 PM CT. Once testing confirmed correct behavior, Chris Nigh declared the incident fully resolved.
- **Linked Open Tickets:** RUN-123557, SRE-18298

#### Details for RUN-123738
- **Root Cause:** Not found
- **Action Taken:** The definitive corrective action was the deployment of the previously successful U.S. configuration fix to theCanadian servers, followed by a CDN cache purge. This decision was made collaboratively during the bridge, withexecution led by Anh Hoang, executed by Chitresh Das and validation confirmed by Sarah Kennedy at approximately2:55 PM CT. Once testing confirmed correct behavior, Chris Nigh declared the incident fully resolved.
- **Linked Open Tickets:** RUN-123557, SRE-18298

### Application: Swagit

| Ticket ID | Title | Status | Permanent Fix | RCA Included? |
| :--- | :--- | :--- | :--- | :--- |
| RUN-124327 | Swagit FTP and Swagit web uploader (uploader.swagit.con)Crea... | Resolved | Yes (Status Resolved) | Yes (Linked) |

#### Details for RUN-124327
- **Root Cause:** Not found
- **Action Taken:** Leah Benner fixed the server permission issues that were fixed on other servers in RUN-124252.
- **Linked Open Tickets:** SRE-18409, RUN-124388, RUN-124252

### Application: Media Manager

| Ticket ID | Title | Status | Permanent Fix | RCA Included? |
| :--- | :--- | :--- | :--- | :--- |
| RUN-124336 | MediaManager issues - Latency & Errors on several sites whil... | Resolved | Yes (Status Resolved) | Yes (Linked) |

#### Details for RUN-124336
- **Root Cause:** Not found
- **Action Taken:** The issue self-resolved without any manual intervention. Trimming, captions, and related Media Managerfunctionality were successfully retested across all previously impacted customers, and the problem could no longerbe reproduced. Since the service recovered on its own and is operating normally, the incident is being closed.
- **Linked Open Tickets:** SRE-18366

### Application: Legistar

| Ticket ID | Title | Status | Permanent Fix | RCA Included? |
| :--- | :--- | :--- | :--- | :--- |
| RUN-124595 | govMeetings ["Legistar"] Legistar Execution Timeouts Created... | Open | No | Yes (Linked) |
| RUN-124595 | govMeetings ["Legistar"] Legistar Execution Timeouts Created... | Open | No | Yes (Linked) |

#### Details for RUN-124595
- **Root Cause:** The most likely cause is lock contention on the File / Matter / tblHistory / tblMeeting tables within affected agency databases, with asecondary contributor likely being the 2026-04-17 Legistar database release (DBA-10053 / 5.4.2604):
- **Action Taken:** Not found
- **Linked Open Tickets:** RUN-124370

#### Details for RUN-124595
- **Root Cause:** The most likely cause is lock contention on the File / Matter / tblHistory / tblMeeting tables within affected agency databases, with asecondary contributor likely being the 2026-04-17 Legistar database release (DBA-10053 / 5.4.2604):
- **Action Taken:** Not found
- **Linked Open Tickets:** RUN-124370

### Application: Speak Up

| Ticket ID | Title | Status | Permanent Fix | RCA Included? |
| :--- | :--- | :--- | :--- | :--- |
| RUN-124943 | DOWN alert: govM - SpeakUp - US (speakup-us-production.grani... | Resolved | Yes (Status Resolved) | Yes (Linked) |

#### Details for RUN-124943
- **Root Cause:** Not found
- **Action Taken:** The nodes were manually deregistered and reregistered, which restored the instances to a healthy state.
- **Linked Open Tickets:** RUN-124944

---

## SEVERITY LEVEL: Sev 3

### Application: OneMeeting

| Ticket ID | Title | Status | Permanent Fix | RCA Included? |
| :--- | :--- | :--- | :--- | :--- |
| RUN-123674 | govM - OneMeeting - Cowichan Valley RD (OM-CAN-WEB1/2) is DO... | Resolved | Yes (Status Resolved) | Yes (Linked) |

#### Details for RUN-123674
- **Root Cause:** Not found
- **Action Taken:** Auto Resolved
- **Linked Open Tickets:** None

### Application: Media Manager

| Ticket ID | Title | Status | Permanent Fix | RCA Included? |
| :--- | :--- | :--- | :--- | :--- |
| RUN-124370 | ["gasmp-mysqlp1"] gasmp-mysqlp1 (mysql) Storage Usage > 96% ... | Resolved | Yes (Status Resolved) | Yes (Linked) |

#### Details for RUN-124370
- **Root Cause:** Not found
- **Action Taken:** Queries from napacity ,client_flandreau executed on the database and the storage usage was stopped and for dreamiowe have killed the queries on the database post approval from dreamio team and the issue got resolved. As apreventive measure we have added 500GB of storage space on the data partition.
- **Linked Open Tickets:** SRE-18397, RUN-124595

---

## SEVERITY LEVEL: Sev 4

### Application: Land and Vitals

| Ticket ID | Title | Status | Permanent Fix | RCA Included? |
| :--- | :--- | :--- | :--- | :--- |
| RUN-124232 | ["Legistar"] govM - LandsRecords - US Remote Support Created... | Resolved | Yes (Status Resolved) | Yes (Linked) |

#### Details for RUN-124232
- **Root Cause:** Not found
- **Action Taken:** Verified ScreenConnect service status.Started all four ScreenConnect services that were found to be stopped.Confirmed restoration of connectivity to testing encoders and devices.No server restart was performed.
- **Linked Open Tickets:** RUN-124235

---

## SEVERITY LEVEL: Unknown

### Application: OneMeeting

| Ticket ID | Title | Status | Permanent Fix | RCA Included? |
| :--- | :--- | :--- | :--- | :--- |
| SRE-18298 | RUN-123738 - govMeetings OneMeeting - CANADA - Content Contr... | Open | No | No (Needs check) |
| SRE-18327 | RUN-123557 - govMeetings - OneMeeting Primegov - Content Con... | Open | No | No (Needs check) |

#### Details for SRE-18298
- **Root Cause:** Not found
- **Action Taken:** N/A
- **Linked Open Tickets:** RUN-123557, RUN-123738

#### Details for SRE-18327
- **Root Cause:** Not found
- **Action Taken:** N/A
- **Linked Open Tickets:** RUN-123557

### Application: Media Manager

| Ticket ID | Title | Status | Permanent Fix | RCA Included? |
| :--- | :--- | :--- | :--- | :--- |
| SRE-18366 | RUN-124336 - MediaManager issues - Latency & Errors on sever... | Open | No | No (Needs check) |

#### Details for SRE-18366
- **Root Cause:** Not found
- **Action Taken:** The issue self-resolved without any manual intervention. Trimming, captions, and related Media Managerfunctionality were successfully retested across all previously impacted customers, and the problem could no longerbe reproduced. Since the service recovered on its own and is operating normally, the incident is being closed.
- **Linked Open Tickets:** RUN-124336

### Application: Swagit

| Ticket ID | Title | Status | Permanent Fix | RCA Included? |
| :--- | :--- | :--- | :--- | :--- |
| SRE-18389 | Create Elastic Alert for SELinux AVC Denials Blocking Apache... | In | No | No (Needs check) |
| SRE-18409 | RUN-124327 - Swagit FTP and Swagit web uploader (uploader.sw... | Open | No | No (Needs check) |

#### Details for SRE-18389
- **Root Cause:** Not found
- **Action Taken:** Not found
- **Linked Open Tickets:** RUN-124252

#### Details for SRE-18409
- **Root Cause:** Not found
- **Action Taken:** Leah Benner fixed the server permission issues that were fixed on other servers in RUN-124252.
- **Linked Open Tickets:** RUN-124252, RUN-124327

### Application: Insite, Legistar, Media Manager, OneMeeting, Peak Agenda Management, Swagit, Video

| Ticket ID | Title | Status | Permanent Fix | RCA Included? |
| :--- | :--- | :--- | :--- | :--- |
| SRE-18398 | GovM: Enable Monitoring for Swagit SetupsCreated: 22/Apr/26 ... | Open | No | No (Needs check) |

#### Details for SRE-18398
- **Root Cause:** Not found
- **Action Taken:** Not found
- **Linked Open Tickets:** RUN-124252

---

