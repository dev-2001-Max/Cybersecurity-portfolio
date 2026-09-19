# SOC / Blue Team Lab

Small security lab built in VirtualBox: Wazuh SIEM monitoring a Windows 10
endpoint, with a simulated RDP brute-force attack from Kali Linux to
validate detection.

## Setup
- Wazuh manager + dashboard (4.14.6)
- Windows 10 endpoint with Wazuh agent
- Kali Linux as attacker
- Isolated internal network (192.168.100.0/24)

## What I did
See [incident-01-rdp-bruteforce.md](./incident-01-rdp-bruteforce.md) for
the full writeup, or browse `/screenshots` for raw evidence.

## Key finding
Wazuh's correlation engine escalated repeated low-severity failed-login
events (rule 60122) into a high-severity "Multiple Windows Logon
Failures" alert (rule 60204, MITRE T1110) once the failure threshold was
crossed — demonstrating real brute-force detection, not just logging.