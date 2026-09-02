# PROJECT REFERENCE DOCUMENT
## Threat Capture and Custom Detection Engineering: A Honeypot-Driven Approach to SIEM Alerting, Malware Analysis, and Rule Development

**Document purpose:** This is the canonical reference for this Final Year Project. It is written to be read by both humans and AI coding/research assistants as full context before doing any work on this project. Anyone or anything reading this should come away with an unambiguous understanding of what the project is, why it exists, who is doing what, what tools are involved, and what "done" looks like for each piece.

**Institution:** AUPP (American University of Phnom Penh), BSc Cybersecurity
**Type:** Final Year Project (FYP), two-person team, single semester (18 weeks)
**Status as of this document:** Proposal approved, entering execution phase

---

## 1. PROJECT IDENTITY

### 1.1 One-line summary
Deploy a real honeypot to capture genuine attacker behavior, feed that data into a SIEM for visibility, reverse-engineer any captured malware, author original detection rules (Sigma for behavior, YARA for files) directly from first-hand observation, and empirically prove whether those custom rules outperform generic public detection rules.

### 1.2 The core research claim being tested
Publicly available detection rules (default Sigma rules from SigmaHQ, default Wazuh rules, generic YARA rules) are written reactively and generically, by people who did not personally observe the specific attack traffic they're meant to catch. This project tests whether rules written from **first-hand, self-collected evidence** achieve measurably better detection performance (precision, recall, false-positive rate) than those generic rules, when tested against the same real-world data.

### 1.3 Why this project is methodologically sound (not just a demo)
- It does **not** rely on a static, pre-existing, potentially outdated public dataset (e.g., CICIDS2017, NSL-KDD). All primary evidence is generated live by the system itself.
- It goes beyond "deploy and observe" (where most tutorial-level honeypot projects stop) into full detection-engineering: analysis → rule authoring → **quantitative comparative evaluation** against a real baseline.
- The evaluation uses standard information-retrieval metrics (precision, recall, F1, false-positive rate) so results are objectively comparable and defensible, not anecdotal.

### 1.4 Three research questions (RQs) driving the project
- **RQ1:** What categories of attacker behavior (reconnaissance, credential attack, payload deployment, persistence, evasion) are observable against a low/medium-interaction honeypot over a defined collection window?
- **RQ2:** Do detection rules authored directly from reverse-engineered, self-captured samples achieve higher precision and recall against the same traffic than generic, publicly available default rules?
- **RQ3:** What is the practical evidentiary value, and what are the limitations, of a single, low-cost, student-operated honeypot as a source of original threat intelligence, relative to large-scale commercial telemetry?

---

## 2. TEAM AND ROLE BOUNDARIES

Two students. Work is split along a natural data-flow boundary: **Person A's output is Person B's input.** Both jointly own evaluation, ethics documentation, and thesis synthesis.

### 2.1 Person A — Infrastructure & Detection Operations *(this document's primary reader is Person A)*
Owns everything from the network/system layer up through behavior-based detection:
- **Feature A1 — Honeypot Deployment & Containment:** cloud VM provisioning, Cowrie deployment, service emulation believability, network-level containment (egress restriction, isolated segment, non-default admin access).
- **Feature A2 — SIEM Integration & Live Dashboard:** Wazuh install/config, log-ingestion pipeline from the honeypot, dashboards (session volume, geolocation, command frequency, file-drop alerting).
- **Feature A3 — Sigma Rule Authoring:** translating observed attacker behavior into Sigma rules, integrating into Wazuh, tuning for false positives.

### 2.2 Person B — Malware Analysis & Detection Engineering
Owns everything from captured-file triage through file-based detection:
- **Feature B1 — Sample Triage & Dynamic Analysis:** retrieving captured files, hashing/entropy/file-type triage, optional Cuckoo Sandbox dynamic execution.
- **Feature B2 — Static Reverse Engineering:** Ghidra disassembly/decompilation, documenting control flow/strings/IOCs/persistence, MITRE ATT&CK technique mapping.
- **Feature B3 — YARA Rule Authoring & Validation:** encoding findings into YARA rules, testing against self-collected corpus + MalwareBazaar supplementary set.

### 2.3 Shared / cross-cutting (both team members)
- Comparative evaluation of Sigma vs. default rules AND YARA vs. default rules against a common baseline methodology.
- Risk, ethics, and containment documentation.
- Thesis write-up and defense preparation.

### 2.4 Person A's specific background (relevant for calibrating explanations/assistance)
- Strong CTF background: Active Directory attacks, Kerberos delegation chains, container escapes, heap exploitation (tcache poisoning, fastbin dup, use-after-free).
- Comfortable with Linux and networking generally. Primary working environment: Kali Linux on Parallels.
- **No prior reverse-engineering experience** (no Ghidra/malware-analysis background) — this is Person B's lane, but Person A wants end-to-end conceptual understanding of it, not hands-on ownership.
- **No prior SIEM or detection-engineering background** — this is new territory being learned during the project itself.
- Learning preference: concept explained in plain language first, before commands/technical specifics; no assumed prior knowledge of SIEM/detection-engineering terms; incremental, not front-loaded.

---

## 3. SYSTEM ARCHITECTURE

### 3.1 Data flow (strictly left to right)

```
[ Internet-facing attackers ]
          |  (unsolicited scans, brute-force, exploitation attempts)
          v
[ LAYER 1: CAPTURE ]        Cowrie (SSH/Telnet honeypot) on isolated cloud VM
          |  (session logs, commands typed, dropped/downloaded files)
          v
[ LAYER 2: VISIBILITY ]     Wazuh (log ingestion, dashboards, alerting)
          |                              \
          v                               v
[ LAYER 3a: DETECTION ]        [ LAYER 3b: ANALYSIS ]
  Sigma rules (behavior)          Ghidra (static) / Cuckoo Sandbox (dynamic)
  — Person A —                    — Person B —
          |                               |
          |                               v
          |                      [ LAYER 3c: DETECTION ]
          |                        YARA rules (file-level)
          |                        — Person B —
           \_____________   ______________/
                        v  v
             [ LAYER 4: EVALUATION ]
      Precision / Recall / FP-rate vs. default rulesets
      (Sigma-vs-default AND YARA-vs-default, both benchmarked)
```

### 3.2 Layer-by-layer definitions (plain language)

**Layer 1 — Capture (Honeypot).** A deliberately exposed, non-real computer that mimics a normal SSH-accessible Linux server. It exists purely to be attacked, and every interaction is logged. It is "low/medium-interaction," meaning it fakes command execution and filesystem contents convincingly, without ever running a real, fully-working system that an attacker could genuinely compromise or pivot from.

**Layer 2 — Visibility (SIEM).** Security Information and Event Management. Centralizes raw honeypot logs, structures them into searchable fields, visualizes them as dashboards, and raises alerts automatically on defined conditions (most importantly: file-drop events).

**Layer 3a — Detection, behavior-based (Sigma).** A vendor-neutral rule format describing a *sequence or pattern of log events* that indicates malicious behavior (e.g., failed logins → success → file download → execution). Translatable into native queries for many SIEM backends.

**Layer 3b — Analysis (Reverse Engineering).** When the honeypot captures an actual file (malware, script) dropped by an attacker, it is studied in an isolated environment to understand what it actually does: static analysis (reading disassembled/decompiled code without running it, via Ghidra) and optionally dynamic analysis (actually running it inside a fully isolated sandbox, via Cuckoo, to observe real-time behavior).

**Layer 3c — Detection, file-based (YARA).** A pattern-matching rule format describing *byte sequences, strings, or structural patterns within a file* that identify it as belonging to a specific malware family.

**Layer 4 — Evaluation.** Both custom rule types are run against the self-collected corpus (and, for YARA, a supplementary MalwareBazaar sample set) and compared quantitatively against generic/default rulesets using precision, recall, false-positive rate, F1 score, coverage delta, and time-to-detection.

### 3.3 Hard architectural constraint: containment
Containment is enforced at Layer 1 as a non-negotiable design requirement, not an afterthought:
- **Egress (outbound) traffic from the honeypot VM is restricted at the network/firewall level**, so that even a fully "compromised" honeypot instance cannot be used to pivot into or attack other systems.
- **Captured samples are moved to an offline/isolated analysis environment** before any reverse engineering or dynamic execution occurs — never analyzed on a live, network-connected, production host.

---

## 4. TOOLS AND TECHNOLOGY STACK

| Component | Tool(s) | License/Cost | Pipeline Role | Owner |
|---|---|---|---|---|
| Honeypot | Cowrie (baseline); T-Pot (optional extension) | Free/open source (GPL/Apache) | Layer 1 — Capture | Person A |
| SIEM | Wazuh (self-hosted) | Free/open source | Layer 2 — Visibility & Alerting | Person A |
| Behavior detection | Sigma / SigmaHQ | Free/open source | Layer 3a — Detection | Person A |
| Static reverse engineering | Ghidra (NSA) | Free/open source (Apache 2.0) | Layer 3b — Analysis | Person B |
| Dynamic analysis | Cuckoo Sandbox (optional) | Free/open source | Layer 3b — Analysis | Person B |
| File detection | YARA | Free/open source (BSD) | Layer 3c — Detection | Person B |
| Supplementary malware corpus | MalwareBazaar (abuse.ch) | Free, registration required | Layer 4 — Evaluation | Person B |
| Primary hosting | Oracle Cloud Always Free | Free, no expiry | Infrastructure | Person A |
| Secondary hosting | Azure for Students (GitHub Student Pack) | Free credit (~US$100) | Infrastructure/backup | Both |
| Version control | Git + GitHub (private repo) | Free | Collaboration | Both |

**Total project cost: US$0.** No paid licenses, subscriptions, or infrastructure required for any objective.

**Base OS:** Ubuntu 22.04 LTS for all cloud VMs.

---

## 5. OBJECTIVES (SMART-aligned, with owner and week)

| ID | Objective | Owner | Target |
|---|---|---|---|
| O1 | Deploy a contained, internet-facing honeypot verified to log inbound sessions and dropped payloads | Person A | Weeks 3–4 |
| O2 | Integrate honeypot telemetry with Wazuh; produce ≥4 dashboard views (session volume, geolocation, command frequency, file-drop events) | Person A | Week 6 |
| O3 | Maintain continuous honeypot uptime for a 3-week data-collection window, yielding a documented attacker-session corpus | Person A | Weeks 7–9 |
| O4 | Perform static (and where feasible dynamic) reverse engineering on every captured executable/script, producing a structured analysis note per sample (behavior summary, IOCs, MITRE ATT&CK mapping) | Person B | Week 11 |
| O5 | Author ≥1 original YARA rule per distinct captured malware family and ≥3 original Sigma rules covering observed attacker behavior | Both | Week 13 |
| O6 | Quantitatively evaluate all custom rules (precision/recall/FP-rate) against self-collected corpus + MalwareBazaar, benchmarked against Wazuh/SigmaHQ default rulesets | Both | Week 15 |

---

## 6. DATA STRATEGY

### 6.1 Primary evidence base
Self-generated. Live attacker sessions and any malware/scripts captured directly by the honeypot during the deployment window. This is a deliberate methodological choice — it avoids the staleness and representativeness problems of reused public datasets, and constructing this evidence base is itself part of the project's contribution.

### 6.2 Supplementary evidence
A small, clearly delineated sample set from **MalwareBazaar** (abuse.ch), used **exclusively** to stress-test how well custom YARA rules generalize beyond the exact samples the honeypot happened to capture. Always reported **separately** from the self-collected corpus in evaluation results, to preserve methodological transparency.

### 6.3 Ground truth
Established manually by the team — both members review sessions/samples and jointly confirm malicious vs. benign classification. This is standard practice for small-scale student detection-engineering research, and is explicitly documented as a limitation (potential labeling bias), mitigated by cross-checking between both team members.

---

## 7. EVALUATION METHODOLOGY (Layer 4 in detail)

### 7.1 Metrics used

| Metric | Formula | Meaning |
|---|---|---|
| Precision | TP / (TP + FP) | Of everything flagged, what % was a genuine threat |
| Recall (Sensitivity) | TP / (TP + FN) | Of all genuine threats, what % was caught |
| False Positive Rate | FP / (FP + TN) | Rate of benign activity wrongly flagged |
| F1 Score | 2×(Precision×Recall)/(Precision+Recall) | Balanced combination of precision and recall |
| Coverage Delta | TP(custom) − TP(default) | Net improvement in real detections attributable to custom rules |
| Time-to-Detection | t(alert) − t(first malicious action) | Latency between attack onset and alert |

### 7.2 Comparison structure
- **Sigma track:** Person A's custom Sigma rules vs. SigmaHQ's public default rules, run against the same Wazuh-ingested honeypot corpus.
- **YARA track:** Person B's custom YARA rules vs. Wazuh's/generic default YARA rulesets, run against both the self-collected corpus and the MalwareBazaar supplementary set (reported separately).
- Both tracks feed into a joint results/discussion chapter.

---

## 8. ETHICS, LEGAL, AND SAFETY REQUIREMENTS (non-negotiable constraints)

Any work on this project — infrastructure, code, or documentation — must respect these constraints without exception:

1. **Authorization:** the honeypot is deployed exclusively on cloud infrastructure the team owns or is explicitly authorized to use (personal cloud accounts). Never deployed on institutional or third-party networks without prior written permission.
2. **Non-participation in harm:** outbound (egress) network access from the honeypot is restricted at the network/firewall level so the system cannot be leveraged to attack, scan, or pivot into third-party infrastructure, even if an automated payload attempts to do so.
3. **Safe malware handling:** all captured samples are analyzed only within an offline or network-isolated environment. No sample is ever executed on a network-connected or production host.
4. **Data minimization and privacy:** attacker IP addresses/geolocation are reported only in aggregate/statistical form in published results. No attempt is made to identify individual attackers; no personal data beyond what is inherent to standard network logs is collected or retained.
5. **Responsible disclosure posture:** any genuinely novel and broadly useful rule may optionally be submitted to the public SigmaHQ repository, subject to supervisor approval — an optional stretch outcome, not a requirement.
6. **Institutional review:** a written risk-acceptance and containment plan must be reviewed and signed off by the project supervisor **before** the honeypot is exposed to the public internet. This is a hard gate, not a formality.

---

## 9. RISK REGISTER

| Risk | Likelihood/Impact | Mitigation |
|---|---|---|
| Low attacker traffic volume during collection window | Medium/Medium | Deploy as early as possible (Week 3); consider T-Pot's broader service suite to widen attack surface |
| Captured samples are low-value, generic commodity malware | Medium/Low | Supplement evaluation with MalwareBazaar; frame commodity-malware findings as representative, not a shortfall |
| Free-tier hosting capacity or credit expiry | Low/Medium | Two independent free-hosting accounts (one per student) as redundancy |
| Reverse engineering proves too time-intensive for a sample | Medium/Medium | Scope RE depth to what's needed for rule-writing; prioritize simpler samples first, defer complex ones |
| Honeypot fingerprinted and avoided by sophisticated attackers | High/Low | Explicitly scope expected outcomes to commodity/opportunistic attack traffic — stated as expected behavior, not failure |
| Team coordination/integration failure between workstreams | Low/Medium | Shared GitHub repo with clearly separated module folders; weekly sync at each phase handoff point |

---

## 10. KNOWN LIMITATIONS (acknowledged up front, not discovered later)

- The honeypot's evidence base reflects **opportunistic, automated internet-wide scanning and commodity malware** — it is not expected to, and does not claim to, capture sophisticated, targeted, or nation-state-level attacker activity.
- Ground truth is manually established by the research team rather than drawn from an independently verified external source — standard for small-scale student research, but introduces potential labeling bias (mitigated by cross-checking between both members).
- Results are specific to the deployment window, geographic hosting region, and honeypot configuration used, and may not generalize to all internet-facing environments.

---

## 11. PROJECT TIMELINE (18 weeks)

| Phase | Weeks | Person A | Person B | Milestone |
|---|---|---|---|---|
| 1. Planning | 1–2 | Cloud accounts, network/VM architecture plan | Literature review; local analysis VM setup | Approved proposal, signed containment plan |
| 2. Deployment | 3–4 | Deploy Cowrie; configure logging & egress lockdown | Set up isolated sandbox environment | Live, contained honeypot |
| 3. SIEM Integration | 5–6 | Install Wazuh; build dashboards & alerting | Prepare YARA/Sigma templates; review MalwareBazaar | Working SIEM dashboard |
| 4. Data Collection | 7–9 | Monitor uptime/traffic; flag notable sessions | Begin triaging captured files as they arrive | Documented attack corpus |
| 5. Malware Analysis | 10–11 | Support log correlation per flagged sample | Static (Ghidra) + dynamic analysis of samples | Analysis notes + IOC list |
| 6. Rule Development | 12–13 | Author/tune Sigma rules; integrate into Wazuh | Author/tune YARA rules | Rule set (YARA + Sigma) |
| 7. Evaluation | 14–15 | Benchmark Sigma vs. default rules | Benchmark YARA vs. generic rules + MalwareBazaar | Evaluation results |
| 8. Write-up | 16–17 | Architecture, containment, ethics chapters | Analysis, detection-engineering chapters | Draft thesis |
| 9. Final Review | 18 | Joint revision, demo rehearsal | Joint revision, demo rehearsal | Submission-ready thesis |

---

## 12. EXPECTED OUTCOMES AND CONTRIBUTION

1. A working, fully documented honeypot-to-SIEM pipeline with live dashboards, reproducible by future students from the project's GitHub repository and written methodology.
2. An original set of YARA and Sigma detection rules derived from independently observed, first-hand attacker activity — not adapted from existing public rule repositories.
3. A quantitative, metric-based evaluation demonstrating whether and to what degree evidence-derived custom rules outperform generic detection baselines.
4. A characterization of real-world opportunistic attacker behavior (command patterns, malware families, TTPs) mapped to the MITRE ATT&CK framework.
5. Optionally, external validation via submission of a qualifying rule to the public SigmaHQ repository.

---

## 13. TERMINOLOGY GLOSSARY (for disambiguation)

| Term | Definition in this project's context |
|---|---|
| Honeypot | A deliberately exposed, non-functional decoy system instrumented purely for observing attacker behavior |
| Low/medium-interaction | Honeypot design where attacker commands are faked/simulated rather than genuinely executed on a real system |
| Cowrie | The specific honeypot software used; emulates SSH/Telnet services |
| T-Pot | Optional multi-honeypot platform extension (Docker-based, 20+ emulated services) — not required for baseline scope |
| Egress | Outbound network traffic (leaving the honeypot VM toward the internet) |
| SIEM | Security Information and Event Management — centralizes, structures, visualizes, and alerts on log data |
| Wazuh | The specific open-source SIEM platform used in this project |
| Decoder (Wazuh) | Pattern that tells Wazuh how to parse raw log text into structured, labeled fields |
| Rule (Wazuh) | Logic that tells Wazuh what decoded field patterns should trigger an alert/classification |
| Sigma | A vendor-neutral, YAML-based rule format for describing detectable behavior patterns in log data |
| YARA | A pattern-matching rule format for identifying malware by byte/string/structural patterns in files |
| Static analysis | Studying a file's code/structure without executing it (e.g., via Ghidra) |
| Dynamic analysis | Executing a file inside an isolated sandbox to observe real-time behavior (e.g., via Cuckoo Sandbox) |
| IOC | Indicator of Compromise — a specific artifact (file hash, IP, domain, string) associated with malicious activity |
| MITRE ATT&CK | A standardized public reference framework of known attacker tactics/techniques, used to classify observed behavior |
| Ground truth | The manually verified, authoritative labeling of which events/samples are actually malicious vs. benign, used to score detection accuracy |
| Precision / Recall / F1 / FP-rate | Standard information-retrieval metrics used to quantitatively score detection rule performance (see Section 7.1) |
| MalwareBazaar | Public, free malware sample repository (abuse.ch), used only as a supplementary evaluation set, never as the primary evidence base |

---

## 14. HOW TO USE THIS DOCUMENT

- This document describes **what the project is and why**, at the level of scope, architecture, roles, constraints, and definitions.
- For **step-by-step build instructions and commands** (Person A's side: VM hardening, Cowrie install, Wazuh install, decoder/rule XML, Sigma rule authoring workflow), refer to the companion file: `PersonA_Full_Technical_Workflow.md`.
- For a **plain-language conceptual walkthrough** of every layer plus a phase-by-phase task plan, refer to: `FYP_Understanding_and_PersonA_Plan.md`.
- Anything not explicitly covered by these three documents should be treated as **undecided/to be determined by the team**, not assumed — check with Person A/Person B or the original proposal before making architectural decisions on their behalf.
