# Incident 001 --- SSH Authentication Investigation

**Status:** Closed --- Controlled Lab Simulation\
**Date:** 29 September 2026\
**Environment:** Kali Linux + Wazuh\
**Wazuh Agent:** `001` (`kali`)\
**Log Source:** `/var/log/auth.log`

------------------------------------------------------------------------

## 1. Incident Summary

A controlled SSH authentication scenario was generated in the lab to
learn how Wazuh detects and represents failed authentication activity.

The monitored Kali endpoint recorded **nine failed password-based SSH
authentication attempts** from source IP `100.93.87.80` against the
`clown` account.

During one SSH session, two failed password attempts were followed by a
successful authentication. The authenticated session was opened and
later closed.

Because this was an authorized lab simulation, the activity was known to
be benign. In an unknown environment, the same pattern would warrant
further investigation because repeated authentication failures followed
by a successful login can be consistent with password-guessing activity.

------------------------------------------------------------------------

## 2. Initial Wazuh Alert

The primary Wazuh event contained:

  Field              Value
  ------------------ --------------------------------
  Agent ID           `001`
  Agent name         `kali`
  Agent IP           `100.98.45.127`
  Source IP          `100.93.87.80`
  Target user        `clown`
  Decoder            `sshd`
  Rule ID            `5760`
  Rule level         `5`
  Rule description   `sshd: authentication failed.`
  Log location       `/var/log/auth.log`
  Source port        `51946`
  Protocol           SSH

### Initial interpretation

The evidence showed that an SSH authentication attempt against the
`clown` account failed.

At this stage, the alert alone did **not** establish that the source was
malicious or that a compromise had occurred.

------------------------------------------------------------------------

## 3. Source Investigation

The source IP was initially treated as unknown.

The address:

``` text
100.93.87.80
```

was investigated within the lab network environment and identified as an
address belonging to the authorized VPN environment.

This established the network context of the source, but the IP address
alone did not establish the identity or intent of the person/device
using it.

### Investigation principle

> Identify the source before assigning intent.

------------------------------------------------------------------------

## 4. Raw Log Validation

The Wazuh alert was validated against the endpoint's raw authentication
log.

Command used:

``` bash
sudo grep 'Failed password' /var/log/auth.log | grep '100.93.87.80'
```

The endpoint returned nine failed SSH password authentication records:

``` text
00:15:01.268027  Failed password for clown from 100.93.87.80 port 43396 ssh2
00:15:08.965357  Failed password for clown from 100.93.87.80 port 43396 ssh2
00:15:42.413395  Failed password for clown from 100.93.87.80 port 43396 ssh2

00:16:02.864594  Failed password for clown from 100.93.87.80 port 51274 ssh2
00:16:11.877576  Failed password for clown from 100.93.87.80 port 51274 ssh2
00:16:29.933855  Failed password for clown from 100.93.87.80 port 51274 ssh2

00:16:39.859892  Failed password for clown from 100.93.87.80 port 41708 ssh2
00:16:46.056536  Failed password for clown from 100.93.87.80 port 41708 ssh2

00:20:35.508501  Failed password for clown from 100.93.87.80 port 51946 ssh2
```

### Finding

There were **9 actual failed password authentication records**.

This was established from the endpoint log rather than relying on the
Wazuh `rule.firedtimes` field.

------------------------------------------------------------------------

## 5. Why Multiple Wazuh Events Were Generated

A single SSH authentication attempt can produce multiple Linux
authentication events.

During the investigation, events were observed from:

-   `unix_chkpwd`
-   `pam`
-   `sshd`

For example:

``` text
unix_chkpwd: Password check failed
        ↓
PAM: User login failed
        ↓
sshd: authentication failed
```

Therefore:

> A count of Wazuh alerts or `rule.firedtimes` should not automatically
> be interpreted as the number of individual attack attempts.

The raw endpoint log was used to determine the actual number of failed
SSH password records.

------------------------------------------------------------------------

## 6. Successful Authentication

The investigation searched for both successful and failed SSH
authentication from the source:

``` bash
sudo grep -E 'Accepted|Failed password' /var/log/auth.log | grep '100.93.87.80'
```

A successful authentication was found:

``` text
2026-09-29T00:16:53.736185+05:30 kali sshd-session[17046]:
Accepted password for clown from 100.93.87.80 port 41708 ssh2
```

This was significant because it followed two failed attempts within the
same SSH session.

------------------------------------------------------------------------

## 7. Session Reconstruction

The SSH process/session ID was `17046`.

The following records were found:

``` text
00:16:37.907685
pam_unix(sshd:auth): authentication failure
user=clown
rhost=100.93.87.80

00:16:39.859892
Failed password for clown from 100.93.87.80 port 41708 ssh2

00:16:46.056536
Failed password for clown from 100.93.87.80 port 41708 ssh2

00:16:53.736185
Accepted password for clown from 100.93.87.80 port 41708 ssh2

00:16:53.743772
pam_unix(sshd:session): session opened for user clown(uid=1000)

00:17:26.970556
pam_unix(sshd:session): session closed for user clown
```

### Session timeline

``` text
00:16:37  Authentication failure
00:16:39  Failed password
00:16:46  Failed password
00:16:53  Password accepted
00:16:53  Session opened
           |
           | ~33 seconds
           |
00:17:26  Session closed
```

The authenticated session therefore lasted approximately **33 seconds**.

------------------------------------------------------------------------

## 8. `last` Verification

The command:

``` bash
last -ai | head -20
```

also showed:

``` text
clown  pts/4  ...  00:16 - 00:17  100.93.87.80
```

This provided additional historical session evidence consistent with the
SSH authentication records.

------------------------------------------------------------------------

## 9. Additional Timeline

The nine failed authentication attempts occurred between:

``` text
00:15:01
```

and

``` text
00:20:35
```

The successful session occurred between approximately:

``` text
00:16:53
```

and

``` text
00:17:26
```

The observed pattern was:

``` text
Multiple failed attempts
        ↓
Two failures within one SSH session
        ↓
Successful authentication
        ↓
Session opened
        ↓
Session closed ~33 seconds later
```

------------------------------------------------------------------------

## 10. Investigation Assessment

### Confirmed

-   An SSH authentication activity occurred.
-   The source was `100.93.87.80`.
-   The target endpoint was Kali.
-   The target account was `clown`.
-   Nine failed password authentication records were observed.
-   A successful authentication subsequently occurred.
-   An SSH session was opened.
-   The session was later closed.
-   Wazuh detected the authentication failures using rule `5760`.

### Not established by the available evidence

The logs examined did **not** establish:

-   That the activity was malicious.
-   That an attacker compromised the system.
-   What commands were executed during the authenticated session.
-   What files were accessed or modified during the session.
-   Whether privilege escalation occurred.
-   Whether any additional malicious activity occurred after
    authentication.

Because this was an authorized lab simulation, the activity was
intentionally generated.

------------------------------------------------------------------------

## 11. Telemetry Limitation

The investigation attempted to inspect the broader system journal during
the authenticated session:

``` bash
sudo journalctl --since "2026-09-29 00:16:50" --until "2026-09-29 00:17:30"
```

No entries were returned for that query.

This does **not** mean that nothing happened during the session. It
means that the queried journal did not provide additional telemetry for
that time range.

The available `auth.log` data provided authentication and session
lifecycle information, but not a reliable record of every command
executed during the SSH session.

### Key lesson

> A SIEM can only investigate telemetry that the underlying systems
> actually collect.

------------------------------------------------------------------------

## 12. SOC Lessons Learned

### 1. An alert is a starting point

A Wazuh alert should initiate investigation rather than automatically
determine the conclusion.

### 2. Separate facts from assumptions

The evidence established failed and successful authentication. It did
not by itself establish malicious intent.

### 3. Validate SIEM alerts with raw logs

The endpoint's `/var/log/auth.log` was used to establish the actual
authentication sequence.

### 4. Correlate events

One authentication attempt can generate multiple related log records
through SSH, PAM, and password-checking components.

### 5. Build a timeline

The sequence of failures, success, session opening, and session closure
provided much more context than a single alert.

### 6. Authentication is not the same as activity

``` text
Authentication
    ↓
Did the credentials work?

Authorization
    ↓
What can the account do?

Activity
    ↓
What did the account actually do?
```

This investigation primarily covered authentication and session
lifecycle.

------------------------------------------------------------------------

## 13. Detection Improvement --- Future Work

The next stage of the lab will focus on post-authentication visibility.

Potential improvements include:

-   Process execution telemetry
-   Linux audit logging
-   Privilege escalation monitoring
-   File Integrity Monitoring
-   Additional Wazuh rules
-   Better command/activity visibility
-   Correlation of authentication with subsequent endpoint activity

The goal is to move from:

``` text
"Someone logged in."
```

to:

``` text
"Someone logged in → these actions occurred → these detections fired → this evidence supports the assessment."
```

------------------------------------------------------------------------

## 14. Final Lab Conclusion

This incident demonstrated a complete beginner-level SOC investigation
using Wazuh and Linux authentication telemetry.

The investigation progressed from:

``` text
Wazuh Alert
    ↓
Source Identification
    ↓
Raw Log Validation
    ↓
Event Correlation
    ↓
Authentication Timeline
    ↓
Successful Login Detection
    ↓
Session Reconstruction
    ↓
Telemetry Gap Identification
```

The activity was a controlled and authorized lab simulation.

No conclusion of compromise was made because the available evidence did
not support such a conclusion.
