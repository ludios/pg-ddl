---
name: pg-ddl
description: How to do data modeling, i.e. decide what the facts are, before and while writing PostgreSQL DDL in the house style. Use when designing or changing tables, keys, columns, constraints, or schema/*.sql files. The PostgreSQL conventions themselves are in conventions.md beside this file.
---
<!-- Model-output: Claude Mythos 5.1 -->
<!-- Model-output: ChatGPT 6 Astra -->

## Designing the model

The hard part of schema design is deciding what the facts are. Decide which things exist for the system's purposes, when two things count as one, what a row asserts, and what people can actually know and maintain. Then choose a representation using conventions.md.

Simsion treats modeling as a design task rather than a descriptive one. Several models can fit a domain, and you and the operators must choose among them. Requirements are negotiable. Kent listed 36 record layouts for the single fact "a salesman serves a territory" and left room for more. Some of those layouts change which populations and operations are possible. There is no uniquely correct ontology to discover, but the choices have consequences. Once people depend on a model, changing it is costly. Revise and test it thoroughly while it is still easy to replace.

## Modeling questions

Work through these questions in order. Return to an earlier question whenever a counterexample changes its answer.

1. **What work must become possible?** Ask for concrete examples of lookups, reports, edits, imports, and deletions, including an awkward case. The requirement "Query positions with no plants" led to a separate positions table instead of a column on plants. Another useful example is "render an order months later": rgp copies the price into the order item at order time instead of looking it up. A request such as "We need a table for X" gives no basis for choosing a model.

2. **What does one row assert?** Write a sentence naming the participants, the time if it matters, and how the fact is known. Each row is a true proposition of its table's predicate (Date). Verbalize elementary facts before drawing anything (Halpin). Distinguish a persistent thing (plant, accession, file), an occurrence (a graft attached, a probe, a payment), a relationship (accession on plant), an observation (video looked alive at 12:03 by method 2), a derived summary (current liveness), and maintained state (stock). "Accession A is on plant P" and "Q scions of A are available from P" assert different facts. When operators needed the first, rgp replaced the table that asserted the second. A row now survives at quantity zero. Explain what a missing row asserts as well: the absence of a recorded probe does not mean "dead".

3. **What counts as one thing, and how do we recognize it tomorrow?** Use Kent's questions about oneness, sameness, and naming. Test every proposed key with two things whose descriptions coincide. Two accessions both named "Seedling" ruled out a plausible natural key because the name did not establish identity. Test repeated relationships between the same participants. Two grafts of one accession on one tree needed a discriminator, and two subtitle tracks in one language needed an identifier. Test what happens when a thing is described again. A capture regenerated from the same files is the same capture, so its stored video file became the key and the generated capture id was dropped. Also test an upstream thing with several local records. One YouTube video has several captures, so video id is an attribute rather than the key. Prefer an identifier that can be verified in the world (Celko). Use a generated one when descriptions cannot distinguish things.

4. **Whose fact is this?** Determine whether a value describes the thing, an occurrence, the relationship, a transaction, or the source. Common name moved from plants to accessions. Plants then got expected_common_name to record a different fact: intention rather than actuality. Origin moved from plants to accessions and later returned to plants as well, presumably because a tree's provenance differs from a cultivar's. Classify each table using Codd's categories: an independent thing, a description that cannot exist without its subject, or a relationship between independent things. A dependent description should reference its subject directly, as capture_descriptions references captures rather than files. Test empty cases, such as no attachment, no occupant, or no credentials. Test doubled cases as well.

5. **What can people actually know and maintain?** Establish where each value comes from and whether the workflow collects it. A ledger of cuttings, recorded singly or in batches, gave way to a count per accession because operators do not record which tree a stick came from. The attachment's quantity became "available at the start of the sale". Distinguish zero, unknown, not applicable, and not yet observed. A hidden dislike count stays NULL rather than 0. Keep disagreeing sources in separate columns with provenance, as in channel and channel_from_folder. Retain that provenance only while it is useful: tubekit later combined two extraction paths into one column. A constraint that rejects legitimate but incomplete data assumes more knowledge than we have, rather than expressing a rule about the world.

6. **What happens over time?** Separate when something happened from when the database learned it (Snodgrass: valid time versus transaction time). Paid, packed, shipped, and delivered are event times. Row history kept by the `periods` extension records database time. An upload table dropped row history and kept an explicit put_time when the cost of the extra writes outweighed the value of the question that history answered. Decide what survives a change. A persistent thing remains the same thing when its description changes. Correcting an event's record does not change the event. A probe is one observation row, and the current conclusion is derived from all the probe rows, retaining the last observed death even after revival. A price agreed in an order remains independent of today's price (Hickey: a dated fact is a value). For each fact, choose whether to edit it in place, version it, append a new observation, or take a snapshot.

7. **Which rules are true of the domain, and which describe typical cases?** State uniqueness, mandatory participation, and permitted changes in domain language. Test each rule against a neighboring population (Halpin's population check): Seedlings, shared thumbnails, repeated grafts, oversold stock going negative, season number zero, dislike counts hidden since 2021, and a capture with no metadata file. A sample can refute a rule. Even a tidy sample cannot prove one. When a legitimate case refutes a CHECK, relax it and leave the counterexample beside the rule.

8. **Which distinctions and structures justify their cost?** Remove identities with no independent meaning. For example, exastash folded chunk sequences into the storage row because they had no clear independent use. Combine things that the operations cannot distinguish, and retain distinctions needed to answer a real question. Compare a few meaningfully different structures and explain what each loses, permits, and costs (Kent, "The Many Forms of a Single Fact"). A physical split changes the representation without establishing a new distinction in the domain. One such change moved descriptions that occupied 57% of a table into a 1:1 table. If the remaining alternatives change what users must record or what they can ask later, show them the sentences each model would assert, with examples, and let them choose.

9. **Can the operator use and explain it?** Work through the examples using the queries and edits, including direct editing in psql or a table editor. A default of '' for the discriminator and an updatable view support that use. Continue revising the model until operators can maintain it.

For each proposed column, establish whose fact it holds, its grain, the time it applies to, how it is known, whether it is independent or derived, which neighboring facts it must be distinguished from, and which operation maintains it. For each proposed table, also establish what a row's existence asserts, what identifies it, what may repeat or be absent, and why these facts need to be grouped in a separate table.

## Changes to the rgp scion model

Between November 2023 and February 2024, the rgp scion model went through five revisions. Each began with a concrete case that the existing model could not handle:

1. The accessions table carried a plant_id, asserting "cultivar C is grafted on plant P". It could not represent the same accession on several trees or several accessions on one tree. Accessions became independent things, with a new table recording the relationship.
2. The scion_availability table asserted "Q scions of A are available from P". It required quantity to be at least 1, and the row was deleted at zero. Operators still needed to know that A was on P when nothing was for sale. The row came to assert presence, quantity was allowed to be zero, and the table was renamed plant_accessions.
3. The plant_accessions table was keyed by plant and accession, asserting "A is on P". It could not represent one accession grafted twice on one tree, with a separate date for each graft. A disambiguator was added to the key so that each row identified an attachment rather than just a pair.
4. The cuttings table asserted "a batch of Q cuttings of A was taken, from P when known". The workflow recorded no source tree and counted stock per accession. The table was replaced by inventory_scions, asserting "Q scions of A remain for sale". The attachment's quantity became the count at the start of the sale.
5. Inventory quantity had to be at least 0, but overselling required negative values. Allowing them made the number a balance rather than a count of sticks.

Three of these five revisions changed what a row meant. All five were prompted by concrete cases rather than diagrams.

## Iteration and testing

Revise and test the model in three distinct ways:

- **Conceptual review.** Revise the sentences and examples. When an example fails, reconsider identity and predicate before adding an optional column to make the original model work. Sometimes the table needs to be replaced.
- **Executable testing.** Install the proposal in a throwaway database with fixtures and deliberately awkward example rows. Run the real queries and edits. A successful installation establishes that the syntax is valid, but does not establish that the model fits the work.
- **Operational review.** Revise the model based on measurements and use, including scan cost, write volume, regeneration, and how stock is actually counted. After each finding, ask whether the facts changed or only their representation.

Once records and consumers depend on the model's meanings, preserve those meanings and plan conversions explicitly. One collation change affected a domain, history tables, views, and clients. Prefer additive changes when they still describe the facts accurately. Do not use workarounds to preserve a model that has been disproven, or silently change what old rows assert.

## Signs of a modeling problem

- Two legitimate things collide under the key. The key treats a description as an identity.
- The same participants need a second row. The relationship needs a discriminator for each occurrence.
- Something operators still talk about disappears at quantity zero or when it has no relationship. The model confuses existence with participation.
- Updating a value changes the meaning of an old transaction. The model conflates current state with historical fact.
- Operators cannot say where a detailed value comes from. The model requires a finer grain than the evidence supports.
- Regenerating derived data requires preserving an id solely for its own sake. An implementation artifact has acquired an identity.
- A constraint fails on legitimate incomplete data. It confuses what is true of the domain with what is known about it.
- A name repeatedly suggests the wrong lookup. It misnames its referent, as root_dir_id named a root rather than a dir.
- A summary cannot explain disagreement, deletion, or evidence that arrives out of order. It has lost the evidence needed to support it.
- Maintaining a distinction costs more than any operation gains from it. Justify the abstraction or remove it.

## References

- William Kent, *Data and Reality* (1978): §1.1 "One Thing", §1.2 "How Many Things Is It?", §1.3 "Change", and §1.5 "Categories (What Is It?)". See also ch. 2 §2.4 on existence tests, ch. 3 on naming (including §3.9 "Sameness"), and §5.2 "Attribute vs. Relationship" and §5.3 "Are Attributes Entities?". "Data models are tools. They do not contain in themselves the 'true' structure of information." Excerpts at bkent.net/Doc/darxrp.htm.
- Kent, "The Many Forms of a Single Fact" (1989), bkent.net/Doc/manyform.htm, and "Choices in Practical Data Design" (VLDB 1982): even the set of data elements is a design choice.
- Terry Halpin, "Object-Role Modeling: An Overview", orm.net/pdf/ORMwhitePaper.pdf: the conceptual schema design procedure. Use the verbalization and population checks without adopting the tooling.
- Graeme Simsion, Simon Milton, Graeme Shanks, "Data modeling: Description or design?", *Information & Management* 49 (2012), and Simsion, *Data Modeling: Theory and Practice* (2007).
- E. F. Codd, "Extending the Database Relational Model to Capture More Meaning" (1979), §6: kernel, characteristic, associative. Use the questions without adopting the surrogate machinery.
- C. J. Date and Hugh Darwen, *The Third Manifesto*, 3rd ed., ch. 2: relvar predicates and constraints. Its ban on nulls does not apply here. Incomplete observations are nullable.
- Christian Jensen and Richard Snodgrass, "Semantics of Time-Varying Information" (1996), §2: valid time versus transaction time. Use this distinction without requiring a bitemporal architecture for every table.
- Martin Fowler, *Analysis Patterns* (1997), ch. 3: Quantity, Observation, Dual Time Record. See also the bliki entry "Contradictory Observations".
- Joe Celko, "A Tale of Identifiers" (Simple Talk, 2011): validation versus verification of a key. Use the verification test without adopting the ban on generated ids.
- Rich Hickey, "The Value of Values" (2012): facts are values with a time. This does not require append-only storage. rgp edits stock and descriptions in place.
- Barry Smith et al., *Basic Formal Ontology 2.0* (2015), §2.13: continuants versus occurrents, as a diagnostic only.

Once the model is ready, write the DDL using conventions.md.
