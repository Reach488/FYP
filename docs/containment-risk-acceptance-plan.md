# Containment and Risk-Acceptance Plan

**Project:** Threat Capture and Custom Detection Engineering: A Honeypot-Driven Approach to SIEM Alerting, Malware Analysis, and Rule Development
**Institution:** AUPP, BSc Cybersecurity — Final Year Project
**Prepared by:** Person A (infrastructure/detection operations), reviewed jointly with Person B
**Status:** DRAFT — pending completion of Phase 1 network build-out and supervisor sign-off

> This document is the hard gate referenced in `FYP.md` Section 8, point 6: the honeypot must not be exposed to the public internet until this plan is reviewed and signed off by the project supervisor. Sections marked `[TO FILL — Phase 1]` are placeholders to be completed once the VM and network are actually provisioned; everything else reflects committed project policy and can be reviewed now.

---

## 1. Purpose and Scope

This plan documents how the project satisfies the six non-negotiable constraints in `FYP.md` Section 8 (Ethics, Legal, and Safety Requirements), and describes the concrete technical controls — built by Person A per `Person-A.md` Phases 1–2 — that enforce them. It covers the honeypot VM and its immediate network environment only; malware handling controls on Person B's isolated analysis environment are documented separately by Person B and referenced here where relevant.

---

## 2. Authorization

- The honeypot is deployed exclusively on cloud infrastructure the team personally owns and controls: a personal Oracle Cloud Always Free tenancy (primary) and a personal Azure for Students subscription (secondary/backup), both created under individual student accounts — not any AUPP-owned or third-party network.
- No institutional network, shared lab infrastructure, or third-party system is used at any point in the pipeline.
- Account holder(s): `[TO FILL — Oracle Cloud tenancy owner name/email, Azure subscription owner name/email]`

## 3. Non-Participation in Harm — Egress Restriction

The honeypot must never be usable as a launchpad against third-party infrastructure, even if an automated payload captured by Cowrie attempts outbound activity.

- Default-deny outbound policy enforced at the host firewall (UFW) per `Person-A.md` Section 1.5: `ufw default deny outgoing`, with only narrow, explicit exceptions.
- During initial setup only: outbound DNS (53), HTTP (80), HTTPS (443) permitted for package installation and Cowrie/Wazuh setup.
- After setup is verified working (end of Phase 2): HTTP/HTTPS exceptions are removed or restricted to specific package-mirror IPs, since Cowrie itself requires no outbound internet access to function (all command output is faked locally).
- The one exception retained long-term: outbound traffic from the honeypot to the Wazuh manager's IP on ports 1514/1515 (agent communication), added deliberately and narrowly per `Person-A.md` Section 3.4 — not a general internet allowance.
- This host-level policy is mirrored at the cloud level by a dedicated Security List on an isolated subnet (see Section 6), so the restriction is enforced independently at two layers.
- Final firewall ruleset as deployed: `[TO FILL — Phase 1/2 — paste final `ufw status verbose` output and cloud Security List export]`

## 4. Safe Malware Handling

- Any file captured by Cowrie (`var/lib/cowrie/downloads/`) is treated as live malware from the moment it is detected.
- Captured files are handed to Person B, together with their SHA-256 hash, and are analyzed only within Person B's offline/network-isolated analysis environment — never executed or opened on the honeypot VM, on Person A's working machine, or on any network-connected host.
- Person B's isolated analysis environment setup is documented separately (Person B's own technical workflow); this plan defers to that document for analysis-side controls and records only the handoff boundary: Person A never executes a captured sample under any circumstance.

## 5. Data Minimization and Privacy

- Attacker IP addresses and session-level detail are visible on the operational Wazuh dashboard (Person A's own monitoring use, per `Person-A.md` Phase 5) but are never published, screenshotted, or included in the thesis/report at the individual level.
- Any geolocation or attacker-activity figures that appear in the thesis, defense slides, or any external-facing document are aggregated to country/region level only.
- No attempt is made to identify individual attackers. No personal data beyond what is inherent to standard network/session logs (source IP, timestamps, commands typed) is collected or retained.

## 6. Responsible Disclosure Posture

- Any Sigma or YARA rule judged genuinely novel and broadly useful may optionally be submitted to the public SigmaHQ repository (or equivalent), but only with prior supervisor approval. This is an optional stretch outcome, not a project requirement, and no submission will occur without that approval.

## 7. Network-Level Containment Design (Isolated Segment + Non-Default Admin Access)

Per `Person-A.md` Section 1.1a and Sections 1.3–1.4:

- **Isolated segment:** the honeypot VM sits in a dedicated VCN/subnet, separate from any other cloud resource the team controls, with its own cloud-level Security List independent of host firewall rules. If Wazuh is hosted on a second VM, it sits in a different subnet, reachable from the honeypot only on the specific agent-communication ports.
- **Non-default admin access:** real administrative SSH access uses key-based authentication only (no password auth), a non-default port, a non-root dedicated admin user, and is never exposed on the honeypot's bait port (22 — which is redirected to Cowrie, not to a real shell).
- Network diagram / Security List export / subnet CIDR layout: `[TO FILL — Phase 1 — attach diagram or console screenshot]`
- Honeypot VM public IP, region, and subnet: `[TO FILL — Phase 1]`

## 8. Institutional Review

- This plan requires review and written sign-off by the project supervisor **before** Phase 1 provisioning exposes any service to the public internet. Verbal approval is not sufficient.
- Supervisor review status: `[TO FILL]`

---

## 9. Risk Acknowledgment

The following risks (full detail in `FYP.md` Section 9) are acknowledged and accepted as inherent to the project's scope, with the stated mitigations in place:

| Risk | Mitigation in place |
|---|---|
| Low attacker traffic volume | Early deployment (Week 3–4); T-Pot contingency evaluated by Week 6 if volume is low (`Person-A.md` Section 2.10) |
| Low-value/generic captured malware | MalwareBazaar supplementary set for YARA evaluation; framed as representative, not a shortfall |
| Free-tier hosting capacity/credit expiry | Two independent free-hosting accounts (Oracle + Azure) as redundancy |
| Honeypot fingerprinted by sophisticated attackers | Explicitly scoped to opportunistic/commodity attack traffic as expected behavior |
| Team coordination failure | Shared GitHub repo, weekly sync at each phase handoff (see Section 10) |

## 10. Team Coordination

- Weekly sync cadence between Person A and Person B: `[TO FILL — day/time and format]`
- Shared repository: https://github.com/Reach488/FYP — created, currently empty; `/honeypot/`, `/siem/`, `/sigma-rules/`, `/evaluation/`, `/docs/` structure to be pushed (in progress).

---

## Sign-off

| Role | Name | Date | Signature |
|---|---|---|---|
| Person A (student) | | | |
| Person B (student) | | | |
| Project Supervisor | | | |

**Gate:** the honeypot VM is not exposed to the public internet (i.e., no inbound rule is opened beyond the admin SSH port used for setup) until this section is signed.
