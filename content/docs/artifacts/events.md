---
description: These event artifacts stream monitoring events from the endpoint. We
  collect these events on the server.
linktitle: Windows Monitoring
menu:
  docs: {parent: Artifacts, weight: 25}
title: Windows Event Monitoring
toc: true

---
## Windows.Events.DNSQueries

Monitor all DNS Queries and responses.

This artifact monitors all DNS queries and their responses seen on
the endpoint. DNS is a critical source of information for intrusion
detection and the best place to collect it is on the endpoint itself
(Perimeter collection can only see DNS requests while the endpoint
or laptop is inside the enterprise network).

It is recommended to collect this artifact and just archive the
results. When threat intelligence emerges about a watering hole or a
bad C&C you can use this archive to confirm if any of your endpoints
have contacted this C&C.


Arg|Default|Description
---|------|-----------
whitelistRegex|wpad.home|We ignore DNS names that match this regex.


 <a href="javascript:void(0)" class="js-toggle dib w-100 link mid-gray hover-accent-color-light pl2 pr2 pv2 "
    data-target="#Windows_Events_DNSQueriesDetails">View Artifact</a>
 <div class="collapse dn" id="Windows_Events_DNSQueriesDetails" style="width: fit-content">


```
name: Windows.Events.DNSQueries
description: |
  Monitor all DNS Queries and responses.

  This artifact monitors all DNS queries and their responses seen on
  the endpoint. DNS is a critical source of information for intrusion
  detection and the best place to collect it is on the endpoint itself
  (Perimeter collection can only see DNS requests while the endpoint
  or laptop is inside the enterprise network).

  It is recommended to collect this artifact and just archive the
  results. When threat intelligence emerges about a watering hole or a
  bad C&C you can use this archive to confirm if any of your endpoints
  have contacted this C&C.

parameters:
  - name: whitelistRegex
    description: We ignore DNS names that match this regex.
    default: wpad.home

sources:
 - precondition:
     SELECT OS from info() where OS = "windows"

   queries:
      - |
        SELECT timestamp(epoch=Time) As Time, EventType, Name, CNAME, Answers
        FROM dns()
        WHERE not Name =~ whitelistRegex
```
   </div></a>

## Windows.Events.FailedLogBeforeSuccess

Sometimes attackers will brute force an local user's account's
password. If the account password is strong, brute force attacks are
not effective and might not represent a high value event in
themselves.

However, if the brute force attempt succeeds, then it is a very high
value event (since brute forcing a password is typically a
suspicious activity).

On the endpoint this looks like a bunch of failed logon attempts in
quick succession followed by a successful login.

NOTE: In order for this artifact to work we need Windows to be
logging failed account login. This is not on by default and should
be enabled via group policy.

https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/basic-audit-logon-events

You can set the policy in group policy managment console (gpmc):
Computer Configuration\Windows Settings\Security Settings\Local Policies\Audit Policy.


Arg|Default|Description
---|------|-----------
securityLogFile|C:/Windows/System32/Winevt/Logs/Security.evtx|
failureCount|3|Alert if there are this many failures before the successful logon.
failedLogonTimeWindow|3600|


 <a href="javascript:void(0)" class="js-toggle dib w-100 link mid-gray hover-accent-color-light pl2 pr2 pv2 "
    data-target="#Windows_Events_FailedLogBeforeSuccessDetails">View Artifact</a>
 <div class="collapse dn" id="Windows_Events_FailedLogBeforeSuccessDetails" style="width: fit-content">


```
name: Windows.Events.FailedLogBeforeSuccess
description: |
  Sometimes attackers will brute force an local user's account's
  password. If the account password is strong, brute force attacks are
  not effective and might not represent a high value event in
  themselves.

  However, if the brute force attempt succeeds, then it is a very high
  value event (since brute forcing a password is typically a
  suspicious activity).

  On the endpoint this looks like a bunch of failed logon attempts in
  quick succession followed by a successful login.

  NOTE: In order for this artifact to work we need Windows to be
  logging failed account login. This is not on by default and should
  be enabled via group policy.

  https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/basic-audit-logon-events

  You can set the policy in group policy managment console (gpmc):
  Computer Configuration\Windows Settings\Security Settings\Local Policies\Audit Policy.
type: EVENT

parameters:
  - name: securityLogFile
    default: >-
      C:/Windows/System32/Winevt/Logs/Security.evtx

  - name: failureCount
    description: Alert if there are this many failures before the successful logon.
    default: 3

  - name: failedLogonTimeWindow
    default: 3600

sources:
  - precondition:
      SELECT OS FROM info() where OS = 'windows'
    queries:
      - |
        LET failed_logon = SELECT EventData as FailedEventData,
           System as FailedSystem
        FROM watch_evtx(filename=securityLogFile)
        WHERE System.EventID.Value = 4625

      - |
        LET last_5_events = SELECT FailedEventData, FailedSystem
            FROM fifo(query=failed_logon,
                      max_rows=500,
                      max_age=atoi(string=failedLogonTimeWindow))

      # Force the fifo to materialize.
      - LET foo <= SELECT * FROM last_5_events

      - |
        LET success_logon = SELECT EventData as SuccessEventData,
           System as SuccessSystem
        FROM watch_evtx(filename=securityLogFile)
        WHERE System.EventID.Value = 4624

      - |
        SELECT * FROM foreach(
          row=success_logon,
          query={
           SELECT SuccessSystem.TimeCreated.SystemTime AS LogonTime,
                  SuccessSystem, SuccessEventData,
                  enumerate(items=FailedEventData) as FailedEventData,
                  FailedSystem, count(items=SuccessSystem) as Count
           FROM last_5_events
           WHERE FailedEventData.SubjectUserName = SuccessEventData.SubjectUserName
           GROUP BY LogonTime
          })  WHERE Count > atoi(string=failureCount)
```
   </div></a>

## Windows.Events.ProcessCreation

Collect all process creation events.


Arg|Default|Description
---|------|-----------
wmiQuery|SELECT * FROM __InstanceCreationEvent WITHIN 1 WHERE TargetInstance ISA 'Win32_Process'|
eventQuery|SELECT * FROM Win32_ProcessStartTrace|


 <a href="javascript:void(0)" class="js-toggle dib w-100 link mid-gray hover-accent-color-light pl2 pr2 pv2 "
    data-target="#Windows_Events_ProcessCreationDetails">View Artifact</a>
 <div class="collapse dn" id="Windows_Events_ProcessCreationDetails" style="width: fit-content">


```
name: Windows.Events.ProcessCreation
description: |
  Collect all process creation events.

type: EVENT

parameters:
  # This query will not see processes that complete within 1 second.
  - name: wmiQuery
    default: SELECT * FROM __InstanceCreationEvent WITHIN 1 WHERE
      TargetInstance ISA 'Win32_Process'

  # This query is faster but contains less data. If the process
  # terminates too quickly we miss its commandline.
  - name: eventQuery
    default: SELECT * FROM Win32_ProcessStartTrace

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    queries:
      - |
        // Convert the timestamp from WinFileTime to Epoch.
        SELECT timestamp(epoch=atoi(string=Parse.TIME_CREATED) / 10000000 - 11644473600 ) as Timestamp,
               Parse.ParentProcessID as PPID,
               Parse.ProcessID as PID,
               Parse.ProcessName as Name, {
                 SELECT CommandLine
                 FROM wmi(
                   query="SELECT * FROM Win32_Process WHERE ProcessID = " +
                    format(format="%v", args=Parse.ProcessID),
                   namespace="ROOT/CIMV2")
               } AS CommandLine,
               {
                 SELECT CommandLine
                 FROM wmi(
                   query="SELECT * FROM Win32_Process WHERE ProcessID = " +
                    format(format="%v", args=Parse.ParentProcessID),
                   namespace="ROOT/CIMV2")
               } AS ParentInfo
        FROM wmi_events(
           query=eventQuery,
           wait=5000000,   // Do not time out.
           namespace="ROOT/CIMV2")
```
   </div></a>

## Windows.Events.ServiceCreation

Monitor for creation of new services.

New services are typically created by installing new software or
kernel drivers. Attackers will sometimes install a new service to
either insert a malicious kernel driver or as a persistence
mechanism.

This event monitor extracts the service creation events from the
event log and records them on the server.


Arg|Default|Description
---|------|-----------
systemLogFile|C:/Windows/System32/Winevt/Logs/System.evtx|


 <a href="javascript:void(0)" class="js-toggle dib w-100 link mid-gray hover-accent-color-light pl2 pr2 pv2 "
    data-target="#Windows_Events_ServiceCreationDetails">View Artifact</a>
 <div class="collapse dn" id="Windows_Events_ServiceCreationDetails" style="width: fit-content">


```
name: Windows.Events.ServiceCreation
description: |
  Monitor for creation of new services.

  New services are typically created by installing new software or
  kernel drivers. Attackers will sometimes install a new service to
  either insert a malicious kernel driver or as a persistence
  mechanism.

  This event monitor extracts the service creation events from the
  event log and records them on the server.
type: EVENT

parameters:
  - name: systemLogFile
    default: >-
      C:/Windows/System32/Winevt/Logs/System.evtx

sources:
 - precondition:
     SELECT OS from info() where OS = "windows"

   queries:
      - |
        SELECT System.TimeCreated.SystemTime as Timestamp,
               System.EventID.Value as EventID,
               EventData.ImagePath as ImagePath,
               EventData.ServiceName as ServiceName,
               EventData.ServiceType as Type,
               System.Security.UserID as UserSID,
               EventData as _EventData,
               System as _System
        FROM watch_evtx(filename=systemLogFile) WHERE EventID = 7045
```
   </div></a>
