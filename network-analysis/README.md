# Network Traffic Analysis — SYN Flood Attack Investigation

## Objective
Investigated a website connection-timeout incident by analyzing TCP/HTTP traffic logs
captured in Wireshark, to identify the attack type and explain its impact on the web
server.

## Approach
- Reviewed source/destination IPs, protocol, and packet info fields across the log
- Traced the standard TCP three-way handshake (SYN → SYN-ACK → ACK) to establish
  a baseline for normal traffic
- Compared normal employee traffic against a single IP address generating a high
  volume of repeated SYN requests
- Identified failed connections (504 Gateway Time-out, RST/ACK resets) affecting
  legitimate visitors as the attack progressed

## Findings
- The server received a sustained flood of SYN packets from a single source IP,
  consistent with a **direct Denial-of-Service (DoS) SYN flood attack**
- Because all malicious traffic originated from one IP (rather than many), this was
  identified as a direct DoS rather than a distributed DoS (DDoS)
- As the attack progressed, legitimate employee connections began failing —
  first intermittently (504 timeouts, RST/ACK resets), then completely, once the
  server's connection-handling resources were exhausted by the attacker's
  half-completed handshakes

## Root Cause
A SYN flood exploits the TCP handshake by sending SYN requests without completing
the final ACK step, forcing the server to hold reserved resources for each pending
connection. Once those resources are exhausted, the server can no longer respond to
genuine requests.

## Skills Demonstrated
Packet/traffic log analysis (Wireshark), TCP/IP protocol fundamentals, DoS/DDoS
attack identification, incident investigation and root-cause reporting