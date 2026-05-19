# Refious Security — Research

Public repository for finalized security research, exploit post-mortems, sanitized campaign writeups, and methodology summaries from Refious Security.

---

## What Lives Here

| Category | Description |
|---|---|
| **Post-mortems** | Analysis of historical exploits — root cause, exploit mechanics, mitigation patterns, broader class lessons |
| **Methodology summaries** | High-level descriptions of how Refious approaches a security domain — not internal tooling |
| **Campaign writeups** | Sanitized retrospectives of red-team engagements, bounty submissions, or research campaigns once disclosure windows close |
| **Class studies** | Bug-class taxonomies and recurrence patterns across the EVM and cross-chain ecosystem |
| **Adversarial notes** | Threat-model essays, attacker-perspective analyses, protocol-failure framings |

---

## What Does NOT Live Here

- Internal scanners, predicates, AST detection logic, or analyzer source.
- Operational tooling, campaign infrastructure, submission gates, or rulebook systems.
- Unpublished findings or pre-disclosure intelligence.
- Active engagement details.

Refious is a research collective. The output is the product. The toolchain is not.

---

## Repository Structure

```
research/
├── README.md                          ← this file
├── post-mortems/
│   └── YYYY-MM-<protocol>-<class>.md
├── methodology/
│   └── <topic>.md
├── campaigns/
│   └── YYYY-Q<n>-<codename>.md        ← sanitized only
├── class-studies/
│   └── <bug-class>.md
└── adversarial/
    └── <topic>.md
```

Naming convention is date-prefixed and class-tagged so the corpus self-organizes as it grows.

---

## Disclosure Discipline

- Campaign writeups are published only after every affected protocol has confirmed mitigation or explicit no-action.
- Post-mortems of public exploits are anchored to public sources (block explorers, post-incident statements from affected protocols) and never re-introduce live attack mechanics.
- Methodology summaries describe approach and reasoning surface, never internal predicates or implementation.

---

## Citation

Research published here is signed by Refious Security as the collective. Individual contributor attribution is internal.

Citation format:

> Refious Security. *<Title>*. <YYYY-MM>. https://github.com/refious-security/research

---

> Research credibility compounds. Every published artifact is one we stand behind in five years.
