---
name: pg-ddl
description: How to do data modeling, i.e. decide what the facts are, before and while writing PostgreSQL DDL in the house style. Use when designing or changing tables, keys, columns, constraints, or schema/*.sql files. The PostgreSQL conventions themselves are in conventions.md beside this file.
---
<!-- Model-output: Claude Mythos 5.1 -->
<!-- Model-output: ChatGPT 6 Astra -->

## The job

The hard part of a schema is deciding what the facts are: which things exist for our purposes, what makes two of them one, what a row asserts, and what people can actually know and maintain. Representation comes after, and conventions.md settles it.

Modeling is design, not description (Simsion). Several models fit any domain; you and the operators choose, and requirements are negotiable. Kent listed 36 record layouts for the one fact "a salesman serves a territory" and left room for more; some of them change which populations and operations are possible. So no uniquely correct ontology awaits discovery, yet every choice has consequences, and a model people already depend on is costly to change. Iterate hard while the model is disposable.

## The questions

Ask them in this order, and return to an earlier one whenever a counterexample changes its answer.

1. **What work must become possible?** Get concrete lookups, reports, edits, imports, and deletions, including an awkward case. "Query positions with no plants" made positions a table of their own rather than a column on plants. Try "render an order months later": rgp copies the price into the order item at order time instead of looking it up. "We need a table for X" distinguishes nothing.

2. **What does one row assert?** Say it as a sentence with named participants, a time where one matters, and how it is known. Each row is a true proposition of its table's predicate (Date); verbalize elementary facts before drawing anything (Halpin). Keep apart: a thing which persists (plant, accession, file), an occurrence (a graft attached, a probe, a payment), a relationship (accession on plant), an observation (video looked alive at 12:03 by method 2), a derived summary (current liveness), and maintained state (stock). "Accession A is on plant P" and "Q scions of A are available from P" are different sentences; rgp replaced a table asserting the second when operators needed the first, and a row now survives at quantity zero. Say what a missing row asserts, too: no probe recorded is not "dead".

3. **What is one thing, and what is the same thing tomorrow?** (Kent: oneness, sameness, naming.) Test every proposed key with two things whose descriptions coincide: two accessions both named "Seedling" killed a plausible natural key, and a name turned out not to be an identity. Test the same relationship twice between the same participants: two grafts of one accession on one tree needed a discriminator; two subtitle tracks in one language needed an identifier. Test re-description: a capture regenerated from the same files is the same capture, so its stored video file became the key and the generated capture id was dropped. Test one upstream thing with several local records: one YouTube video has several captures, so video id is an attribute, not the key. An identifier verifiable in the world beats a generated one (Celko), but a generated one is right when descriptions cannot distinguish things.

4. **Whose fact is this?** A value describes the thing, an occurrence, the relationship, a transaction, or the source. Common name moved from plants to accessions; plants then got expected_common_name, a different fact, intention rather than actuality. Origin moved from plants to accessions and then returned to plants as well, presumably because a tree's provenance and a cultivar's provenance differ. Classify each table (Codd): an independent thing; a description which cannot exist without its subject, so its reference points at the subject itself (capture_descriptions references captures, not files); or a relationship between independent things. Test the empty cases: no attachment, no occupant, no credentials. Test the doubled cases.

5. **What can people actually know and maintain?** Establish where each value comes from and whether the workflow collects it. A ledger of cuttings, singly or in batches, gave way to a count per accession because operators do not record which tree a stick came from; the attachment's quantity became "available at the start of the sale". Separate zero, unknown, not applicable, and not yet observed: a hidden dislike count stays NULL, not 0. Sources which disagree stay as separate columns with provenance (channel, channel_from_folder), yet provenance must earn its keep: tubekit later folded two extraction paths into one column. A constraint which rejects legitimate but incomplete data was a claim about our knowledge, not about the world.

6. **What happens over time?** Separate when something happened from when the database learned it (Snodgrass: valid time versus transaction time). Paid, packed, shipped, and delivered are event times; periods row history is database time; an upload table dropped row history and kept an explicit put_time when the history cost more in writes than its question was worth. Decide what survives change: a persisting thing survives re-description; an event does not change when its record is corrected; a probe is one observation row and the current conclusion is derived from all of them, retaining the last observed death even after revival; a price agreed in an order does not follow today's price (Hickey: a dated fact is a value). For each fact choose: editable in place, versioned, appended as a new observation, or snapshotted.

7. **Which rules are true of the domain, and which merely typical?** State uniqueness, mandatory participation, and permitted changes in domain language, then attack each with a neighboring population (Halpin's population check): Seedlings, shared thumbnails, repeated grafts, oversold stock going negative, season number zero, dislike counts hidden since 2021, a capture with no metadata file. A sample can refute a rule; a tidy sample cannot prove one. When a legitimate case refutes a CHECK, relax it and leave the counterexample beside the rule.

8. **Which distinctions and structures earn their cost?** Remove identities with no independent meaning: exastash folded chunk sequences into the storage row for lack of a clear independent use. Combine what the operations cannot tell apart; keep what a real question needs. Compare a few meaningfully different shapes and say what each loses, permits, and costs (Kent, "The Many Forms of a Single Fact"). A physical split, such as moving descriptions which were 57% of a table into a 1:1 table, is not an ontological discovery. When a surviving choice changes what users must record or can later ask, show them the competing sentences with examples and let them choose.

9. **Can the operator use and explain it?** Walk the examples through the queries and the edits, including direct editing in psql or a table editor: a default of '' for the discriminator and an updatable view exist for that reason. A model operators cannot maintain is not finished.

For a candidate column this collapses to: whose fact, at what grain, for which time, known how, independent or derived, distinguishable from which neighbor, maintained by which operation. For a candidate table add: what a row's existence asserts, what identifies it, what may repeat or be absent, and why this group needs a boundary.

## One model, five rewrites

The rgp scion model between November 2023 and February 2024, each version as its row sentence and the case which broke it:

1. accessions carried a plant_id: "cultivar C is grafted on plant P". Broken by the same accession on several trees and several accessions on one tree. Accessions became independent things; a new table carried the relation.
2. scion_availability, quantity at least 1, row deleted at zero: "Q scions of A are available from P". Broken by operators needing to know that A is on P when nothing is for sale. The row came to assert presence, quantity may be zero, and the table was renamed plant_accessions.
3. plant_accessions keyed by plant and accession: "A is on P". Broken by one accession grafted twice on one tree, each graft with its own date. The key gained a disambiguator; the row became an attachment, not a pair.
4. cuttings: "a batch of Q cuttings of A was taken, from P when known". Broken by a workflow which records no source tree and counts stock per accession. Replaced by inventory_scions: "Q scions of A remain for sale"; the attachment's quantity became the count at the start of the sale.
5. inventory quantity at least 0. Broken by overselling. Negative allowed: the number is a balance, not a count of sticks.

Three of the five rewrites changed what a row means; every one came from a concrete case rather than from a diagram.

## The loops

Three loops, not to be confused:

- **Conceptual.** Revise the sentences and examples. When an example fails, revisit identity and predicate before adding an optional column to rescue the first guess; sometimes the right move is to replace the table.
- **Executable.** Install the proposal into a throwaway database with fixtures and deliberately awkward example rows, and run the real queries and edits. A clean install proves syntax, not fit.
- **Operational.** Revise after measurement and use: scan cost, write volume, regeneration, how stock is really counted. After each finding ask whether the facts changed or only their representation.

Once records and consumers depend on meanings, preserve them and plan conversions explicitly; one collation change reached a domain, history tables, views, and clients. Prefer additive changes when they are honest. Do not keep a disproven ontology alive with workarounds, and do not silently change what old rows assert.

## Signals the model is wrong

- Two legitimate things collide under the key: identity was inferred from description.
- The same participants need a second row: the relationship needs an occurrence discriminator.
- Something operators still talk about vanishes at quantity zero or with no relationship: existence was confused with participation.
- Updating a value changes the meaning of an old transaction: current state and historical fact were conflated.
- Operators cannot say where a detailed value comes from: the grain exceeds the evidence.
- Regenerating derived data needs an id preserved for its own sake: an implementation artifact acquired identity.
- A constraint fails on legitimate incomplete data: domain truth was confused with knowledge.
- A name keeps inviting the wrong lookup: the referent is misnamed (root_dir_id named a root, not a dir).
- A summary cannot explain disagreement, deletion, or out-of-order evidence: it lost its evidential basis.
- Maintaining a distinction costs more than any operation gains: justify the abstraction or remove it.

## Reading

- William Kent, *Data and Reality* (1978): §1.1 "One Thing", §1.2 "How Many Things Is It?", §1.3 "Change", §1.5 "Categories (What Is It?)"; ch. 2 §2.4 existence tests; ch. 3 naming, with §3.9 "Sameness"; §5.2 "Attribute vs. Relationship" and §5.3 "Are Attributes Entities?". "Data models are tools. They do not contain in themselves the 'true' structure of information." Excerpts at bkent.net/Doc/darxrp.htm.
- Kent, "The Many Forms of a Single Fact" (1989), bkent.net/Doc/manyform.htm, and "Choices in Practical Data Design" (VLDB 1982): even the set of data elements is a design choice.
- Terry Halpin, "Object-Role Modeling: An Overview", orm.net/pdf/ORMwhitePaper.pdf: the conceptual schema design procedure; take the verbalization and population checks, not the tooling.
- Graeme Simsion, Simon Milton, Graeme Shanks, "Data modeling: Description or design?", *Information & Management* 49 (2012); Simsion, *Data Modeling: Theory and Practice* (2007).
- E. F. Codd, "Extending the Database Relational Model to Capture More Meaning" (1979), §6: kernel, characteristic, associative. Take the questions, not the surrogate machinery.
- C. J. Date and Hugh Darwen, *The Third Manifesto*, 3rd ed., ch. 2: relvar predicates and constraints. Reject its ban on nulls; incomplete observations are nullable here.
- Christian Jensen and Richard Snodgrass, "Semantics of Time-Varying Information" (1996), §2: valid time versus transaction time. Take the distinction, not a bitemporal architecture for every table.
- Martin Fowler, *Analysis Patterns* (1997), ch. 3: Quantity, Observation, Dual Time Record; bliki "Contradictory Observations".
- Joe Celko, "A Tale of Identifiers" (Simple Talk, 2011): validation versus verification of a key. Take the verification test, not the ban on generated ids.
- Rich Hickey, "The Value of Values" (2012): facts are values with a time. Not a mandate for append-only storage; rgp edits stock and descriptions in place.
- Barry Smith et al., *Basic Formal Ontology 2.0* (2015), §2.13: continuants versus occurrents, as a diagnostic only.

Then write the DDL with conventions.md.
