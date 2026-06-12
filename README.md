# Chiron

An open coaching system for endurance athletes. Three documents that together 
define how an AI coach should think, speak, and prescribe.

Designed for use with Claude and the [intervals-mcp](https://github.com/funkylarma/intervals.mcp) 
Cloudflare Worker, but the documents are plain markdown — use them with any 
AI platform that supports a system prompt and tool access to your training data.

---

## The documents

### CHIRON.md — coaching persona

The system prompt. Defines who Chiron is: a firm, direct, compassionate 
accountability partner named for the wounded-healer mentor of myth. Covers 
voice, coaching posture, how to use the MCP tools, when to push and when to 
ease, and the two cognitive modes the athlete operates in (dawn execution vs 
deliberate planning).

Paste this into any Claude Project (or equivalent) as the custom instructions.

### PROCESS_W.md — Workout Prescription Ontology

The single grounding document for workout creation. Defines from first 
principles: effort blocks, training zones, structural patterns, stimulus 
specificity, the grey zone problem, acute and chronic adaptation, session 
outcomes, and the full ladder progression system (seeding, advancement tiers, 
cross-system weight matrix, phase transitions).

The prescription layer that consumes this is `workout-builder.js` in 
intervals-mcp, plus the LLM coach reasoning over it.

### SECTION_11.md — AI Coach Protocol (Chiron fork)

A fork of the [Section 11](https://github.com/CrankAddict/section-11) 
AI coaching protocol by CrankAddict, reframed for this deployment:

- References to `sync.py` / `pull.py` repointed to the Node.js sync pipeline 
  in intervals-mcp
- `DOSSIER.md` athlete-store repointed to live Intervals.icu data via the MCP
- Schema fields aligned to this project's naming conventions

The canonical upstream is CrankAddict's repository. This fork diverges only 
where the deployment differs; the coaching science and protocol structure are 
unchanged.

---

## How the documents relate

PROCESS_W and SECTION_11 are companion documents with a clear ownership 
boundary:

- **Measurement and readiness** — load, TSB, TID, durability, P0–P3 decisions: 
  **Section 11 is authoritative**
- **Prescription and progression** — what to prescribe, how to structure it, 
  the ladder mechanic: **PROCESS_W is authoritative**

Where they use the same terms differently (zone numbering, readiness 
vocabulary), the crosswalk is documented in PROCESS_W's 
*Relationship to Section 11* section.

---

## Licence

`CHIRON.md` and `PROCESS_W.md` are original works by Adam Chamberlin, released under the [MIT License](LICENSE).

`SECTION_11.md` is derived from [Section 11](https://github.com/CrankAddict/section-11) by CrankAddict, used and modified under the MIT License. Full attribution in [LICENSE](LICENSE).

---

*Chiron was the wounded healer of myth — the centaur who trained Achilles, Jason, and Asclepius, and who coached through his own incurable wound. The name fits: a coach who reasons from evidence, names the gaps honestly, and does not let struggle become an excuse to stop.*