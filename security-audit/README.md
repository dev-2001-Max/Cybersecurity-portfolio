# Security Audit — Botium Toys (Course Scenario)

## Objective
Conducted a security audit for a fictional small business (Botium Toys) as part of the
Google Cybersecurity Professional Certificate. The audit assessed the company's assets,
existing controls, and compliance posture against NIST CSF, PCI DSS, GDPR, and SOC
frameworks.

## Approach
- Reviewed the organization's asset inventory (employee devices, internal network,
  systems, data storage, and legacy systems)
- Assessed existing administrative, technical, and physical controls against control-type
  categories (preventative, detective, corrective, deterrent)
- Completed a controls and compliance checklist to identify gaps against PCI DSS, GDPR,
  and SOC 1/2 requirements
- Assigned a risk score and documented rationale for the assessment

## Key Findings
- Absence of least-privilege access controls — all employees had access to sensitive
  customer data (PII/SPII, cardholder data)
- No encryption in place for stored/transmitted credit card information
- No intrusion detection system (IDS) or disaster recovery plan
- Existing firewall and antivirus controls were adequate
- Password policy existed but did not meet modern complexity standards, and no
  centralized password management system was in place

## Recommendations
Prioritized implementing least privilege, separation of duties, encryption, an IDS, and a
disaster recovery plan — mapped each recommendation to the specific compliance gap
it addresses (PCI DSS, GDPR, SOC).

## Skills Demonstrated
Risk assessment, control categorization (administrative/technical/physical), compliance
gap analysis (PCI DSS, GDPR, SOC 1/2), NIST CSF application, security reporting