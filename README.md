# Windows Incident Investigation — Compromised Server Analysis

## Overview

A full investigation of a compromised Windows Server 2016 machine, conducted using the TryHackMe "Investigating Windows" lab environment. The goal wasn't just to answer the questions — it was to reconstruct the full attack chain, map the attacker's methodology, and produce a report that reads like actual SOC output rather than a lab writeup.

The investigation identified a multi-stage intrusion: initial access via a web shell uploaded to a misconfigured IIS server, followed by privilege escalation, three distinct persistence mechanisms, C2 communications, and DNS poisoning. The attacker showed deliberate operational awareness throughout — including creating a backup administrator account that was never logged into, naming a malicious scheduled task to sound like routine maintenance, and poisoning google.com specifically to intercept high-volume traffic.

## What's in this repo

- **[Windows_Incident_Investigation_Report.pdf](Windows_Incident_Investigation_Report.pdf)** — the full investigation report: executive summary, attack timeline reconstruction, technical analysis of each finding, IOC table, MITRE ATT&CK mapping across 10 techniques, and remediation recommendations.
- **[attack-chain.png](attack-chain.png)** — visual diagram of the full attack chain from initial access to C2 and DNS poisoning.

## The attack chain

| Stage | Finding | MITRE technique |
|---|---|---|
| Initial access | .jsp web shell uploaded via IIS file upload misconfiguration | T1505.003 |
| Privilege escalation | Jenny + Guest accounts added to local Administrators (Event ID 4732) | T1136.001 / T1098 |
| Persistence | Scheduled task "Clean file system" running nc.ps1 on port 1348 | T1053.005 |
| Persistence | Registry run key connecting to 10.34.2.3 on every boot | T1547.001 |
| Persistence | Attacker-created inbound firewall rule for port 1337 | T1562.004 |
| C2 | Communications to external IP 76.32.97.132 | T1071 |
| Defence evasion | Hosts file poisoning — google.com redirected to 76.32.97.132 | T1565.001 |
| Execution | PowerShell Netcat implementation (nc.ps1) | T1059.001 |

## Three things that stood out

**The Jenny account was created but never logged into.** At first glance that's easy to miss — the logon timestamp is null, so it doesn't show up in standard login review. But that's precisely the point. The attacker created a backup administrator account and held it in reserve, never touching it unless the primary access route was discovered. A null last-logon on an administrator account that didn't exist last week is the tell.

**The scheduled task was named "Clean file system".** That's deliberate obfuscation — the name is chosen to look like routine disk maintenance. The only way to catch it is to look at the action, not the name: `nc.ps1 -l 1348` has nothing to do with cleaning a file system. It's a Netcat listener waiting for incoming connections.

**Three independent persistence mechanisms.** The attacker wasn't relying on any single route back in. Remove the scheduled task — the registry run key still fires on boot. Block the registry connection — the firewall rule still allows inbound access on port 1337. This redundancy is a sign of operational discipline, not opportunism.

## What I learned about investigation methodology

The most useful thing this exercise reinforced: **evidence sources contradict each other, and that contradiction is the signal.** The scheduled task name says "maintenance tool". The action says "Netcat listener". The Jenny account looks dormant. The creation event and group membership change say otherwise. Following the contradiction — rather than accepting the surface-level label — is what turns a list of findings into an actual investigation.

Reading multiple evidence sources in combination also reveals scope that any single source would hide. The hosts file poisoning only makes sense once you know the C2 IP — suddenly a benign-looking text file becomes part of the attacker's traffic interception plan.

## Tools and methodology

Investigation conducted on a Windows Server 2016 VM provided by TryHackMe's "Investigating Windows" room. Evidence sources reviewed:

- Windows Event Log (Security, System, Application) — via Event Viewer and PowerShell Get-WinEvent
- Windows Task Scheduler — inspected action-level detail, not just task names
- Windows Registry — run key locations (HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run)
- Local user accounts — Get-LocalUser, Get-LocalGroupMember
- Windows Firewall rules — inbound rule audit
- Hosts file — C:\Windows\System32\drivers\etc\hosts

---

*Part of my ongoing hands-on cyber security portfolio — see my [GitHub profile](https://github.com/harigovindv1010) for other projects, including a [home SIEM lab](https://github.com/harigovindv1010/home-siem-lab) and [phishing email analysis](https://github.com/harigovindv1010/phishing-email-analysis).*
