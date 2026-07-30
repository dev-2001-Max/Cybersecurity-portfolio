# Incident Handler's Journal — Ransomware Incident

## Objective
Documented a simulated ransomware security incident at a healthcare organization
using the 5 W's framework (Who, What, When, Where, Why), to practice structured
incident logging as part of incident response fundamentals.

## Journal Entry

**Date:** [use today's date] | **Entry:** #1
**Description:** Documenting a ransomware security incident at a healthcare clinic
**Tools used:** None (documentation exercise)

### The 5 W's
- **Who:** An organized threat actor group targeting healthcare and transportation
  sector organizations
- **What:** A ransomware attack that encrypted critical files, including patient
  records, halting business operations
- **When:** Tuesday morning, approximately 9:00 a.m.
- **Where:** A small U.S. healthcare clinic providing primary-care services
- **Why:** Attackers gained initial access via a targeted phishing campaign —
  employees received emails with a malicious attachment that installed malware upon
  download. Once inside the network, attackers deployed ransomware to encrypt files
  and demanded payment for the decryption key, indicating a financially motivated
  attack.

### Additional Notes / Analysis
- **Prevention:** This incident traces back to a single successful phishing email —
  reinforcing that security awareness training and phishing simulation (the same
  work covered in my SaxonPenny experience) are frontline defenses against
  ransomware. Email filtering/attachment sandboxing and endpoint detection could
  have interrupted the attack chain before encryption occurred.
- **Ransom payment consideration:** Paying does not guarantee file recovery and can
  fund further attacks; the stronger response is restoring from clean backups and
  reporting to relevant authorities (e.g. CISA, and HHS given the healthcare/HIPAA
  context) — reinforcing why disaster recovery planning and backups are critical
  controls, as also identified in my Botium Toys security audit.

## Skills Demonstrated
Incident documentation, structured incident analysis (5 W's), ransomware attack
chain understanding, phishing as an initial access vector, incident response
fundamentals