# askVEPai: AI chatbot interface for Ensembl VEP web

<img src="vep_logo.png" alt="Ensembl VEP logo" width="220" />

**Contributor:** David Gao ([Uninterpretable-Evolving-Blackbox](https://github.com/Uninterpretable-Evolving-Blackbox))  
**Mentors:** Likhitha (mentor), Aine (co-mentor), Nakib (co-mentor)  
**Organisation:** EMBL-EBI (Ensembl, Genome Assembly and Annotation)  
**Programme:** Google Summer of Code 2026  

- **Full report:** [`GSOC_2026.md`](https://github.com/Uninterpretable-Evolving-Blackbox/askVEPai/blob/main/GSOC_2026.md), covering how it works, every result, and how to extend it
- **Repository:** [askVEPai](https://github.com/Uninterpretable-Evolving-Blackbox/askVEPai), with the code, how to install and run it, and the evidence behind every figure

---

## Project summary

Ensembl VEP's web interface offers dozens of configuration options for variant annotation, which overwhelm new users and generate recurring helpdesk queries. askVEPai takes a plain-English description of an analysis and returns the options to tick on the VEP web form: RECOMMENDED options, OPTIONAL add-ons, and what the form already ticks. It runs on an open-source model that EBI can host, so queries never go to a commercial AI service. Our system performs much faster and more accurately than the commercial LLMs tested.

---

## What was built

### What it does, at a high level

- **Plain English in, configuration out.** Three lists: RECOMMENDED (tick these), OPTIONAL (add-ons worth considering) and ALREADY ON (the form's defaults), each option with where it sits on the form.
- **Missing facts: asked or assumed, never silent.** A fact the description leaves out is asked about when the answer would change the recommendation; otherwise the tool uses a safe default and says what it assumed.
- **Species-aware.** It recognises any organism in Ensembl's species list and offers only the options Ensembl provides for it.
- **States its limits.** A request for a gene list or a class of consequences gets instructions for filtering the results page instead. Questions that are off-topic, or about more than choosing VEP options, get a short note saying so.
- **Main Modes:** `--explain` adds why each option is there and Ensembl's own description of it; `--cli` prints the configuration as a VEP command; `--minimal` shows only what must be ticked. Any fact can be stated with a flag (`--species`, `--organism`, `--goal`, ...) instead of left to the model.
- **Self-hosted and private.** An open-source model that EBI can host, so queries stay within EBI.

### How it works

- **One model call.** [Gemma 4](https://ai.google.dev/gemma) (`gemma4:26b`, through [Ollama](https://ollama.com)) reads the description into five facts, plus the organism when one is named:

  | fact | values |
  |---|---|
  | species | human, non-human |
  | variant origin | germline, somatic |
  | variant size | small, structural (or both) |
  | region | coding, regulatory (or both) |
  | analysis goal | basic consequences, clinical interpretation, population frequency (one or more) |
  | organism | any name in Ensembl's species list ("pig", "Sus scrofa", "zebra finch"), checked against that list |

- **A priority table**, written for this project and reviewed with the mentors, says for each VEP option and each fact value whether the option is recommended, optional or not applicable. Everything after the model call is deterministic, so the same facts always give the same configuration.
- **A checker** removes options Ensembl does not offer for that organism or genome build (SIFT exists for 13 species; CADD for human, pig, turkey and the Red Jungle fowl), resolves conflicts between options, adds prerequisites, and never recommends a filter that deletes result rows.

### The knowledge base

- 70 VEP options as the release-116 web form offers them, each fact cited to the Ensembl file and line it came from (form code, plugin configuration, options and plugins pages).
- Ensembl's species list from its REST API, so the organism the model names is checked against what Ensembl serves.

---

## Stats

The model's one job is reading the description, so the tests measure that. All with `gemma4:26b`, run three or more times (first run shown):

- **150 test cases written to mislead the model**, such as "not a mouse study" or a tool name that sounds like a fact: all five facts read right on **145**
- **31 example scenarios reviewed and corrected by the mentors:** the same RECOMMENDED options as the correct facts give on **30**
- **754 organism names from Ensembl's species list** (scientific, common and breed names): the right organism on **746**, and on **749** when the text also mentions a second organism the data is not from
- **78 scenarios with one fact removed:** the tool assumed a safe value and said so, or asked, on **73**
- **Against commercial chat models**, on 20 scenarios: askVEPai recommended **92** of the priority table's 103 options; the best, Claude Opus 5.5 given the VEP documentation, recommended **53**

---

## Code

- **The tool:** [`vep_ai_demo/`](https://github.com/Uninterpretable-Evolving-Blackbox/askVEPai/tree/main/vep_ai_demo) (every flag and environment variable in its README)
- **Every measurement:** [`evidence/current_evidence/README.md`](https://github.com/Uninterpretable-Evolving-Blackbox/askVEPai/blob/main/evidence/current_evidence/README.md)

No code was merged into an Ensembl repository: integration into VEP web waits until the new VEP in Ensembl's redesigned website ([ensembl-client](https://github.com/Ensembl/ensembl-client)) is in place.

---

## What's left / future work

- Integration into the new VEP web interface, through a JSON output or an API if the web team wants one
- The priority table stays under review as the tool is used and VEP is updated
- Testing with real user questions
- Catalogue citations: most options still cite the release-115 copies of the form files; their facts were checked against release 116, so only the citations need moving
- An unstated analysis goal is sometimes filled in by the model; the configuration is unchanged, but the user misses the line saying it was assumed

---

## Challenges and learnings

- Very few real user questions existed to build and test on, so the test scenarios had to be constructed. Real questions are probably scarce because no tool this convenient existed before: forum replies take time, so people rarely ask how to configure VEP there.
- "Simplicity is prerequisite for reliability" (Edsger W. Dijkstra). The design began as a RAG pipeline with two model calls; it ended as one model call plus deterministic code, which was faster and more accurate.
- The decisions outside the model, the priority table and the fallbacks for missing facts, took as much care as the system design and decide what the user sees.
- Species support had to follow Ensembl's own per-species and per-plugin lists, which differ from option to option and from the web form's own help text.
