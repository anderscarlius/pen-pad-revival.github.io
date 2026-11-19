# Arne's function, in any language — PEN&PAD on the Arne scenario
Nordic Hackathon 2026, PEN&PAD-26 bonus track · Nimloth (Anders Carlius)

**Live demo:** https://penpad.nimloth.se (clinician mode) · https://penpad.nimloth.se/patient/ (patient mode). The demo sits behind Cloudflare Access; reviewers are invited by e-mail on request.
**Contact:** TODO(Anders): GitHub handle or contact page

## The gap we aimed at
Arne (83, fictional) breaks his hip on the stairs in Copenhagen and is discharged from a Danish hospital to home care in Sweden. The organisers' medical comments name three gaps: **functional status is missing and nobody notices**, **his son ends up as the interpreter**, and **nobody confirms the hand-over**.

We used the same coded record for both sides of the border:
- **Clinician mode** (discharge, Rigshospitalet) follows the track's three steps. *Orientation:* body map (hip, leg, chest), topic list (transfer, walking, stairs, breathing, pain, wound), or free text / speech. *Refinement:* only follow-ups that make sense, as short questions (fracture → left or right? operated? · walking → aid? how far? · breathing → breathless at rest, walking or stairs?). *Confirmation:* "Is this what you meant?" as a live sentence.
- **Patient mode** (at home in Sweden): plain Swedish and big buttons, plus a **3D body** to point at (where it hurts, where the leg ulcer is). The 3D view is only the default in patient mode; the clinician keeps a fast 2D map and list, because a model you must rotate slows a clinician down (Rector: "exhaustive is the enemy of the useful"). Clinicians can switch on the same 3D view ("Visa i 3D"); it uses the same component, geometry, FMA→SNOMED table and refinement engine. A list next to the 3D view does the same thing, and replaces it when WebGL or the geometry is missing. Arne answers himself, or his son does it for him. The result is saved as a **draft marked patient-reported**, with "reported by: patient / relative". The home-care nurse opens the draft in clinician mode, refines it and confirms it. We show provenance; we do not try to solve proxy consent.

## Three things worth looking at
1. **The track's three steps in both modes**: orientation, refinement and confirmation.
2. **The sentence generator removes the need for an interpreting son.** One coded entry is rendered deterministically in **Danish, Swedish, plain Swedish (for Arne) and English** side by side. The templates are hand-written and visible (`generator/templates/{da,sv,sv-enkel,en}.yaml`). No LLM generates text. A concept without a template is shown as SNOMED term + code + "no translation"; it never fails silently.
   - sv: *Opererad fraktur på vänster lårbenshals. Går med rollator inomhus cirka 10 meter. Klarar inte trappor. Andfådd vid gång.*
   - da: *Opereret fraktur af venstre lårbenshals. Går med rollator indendørs ca. 10 meter. Kan ikke gå på trapper. Åndenød ved gang.*
3. **Functional status lands in an EPS document, with the validation result.** The functional-status findings become `Condition`s in the EPS section *Functional status* (LOINC 47420-5, read from `composition-eu-eps` 1.0.0-ballot, which allows Condition / ClinicalImpression / DocumentReference there). They are added to a copy of the hackathon's Arne EPS bundle; the original is never overwritten. The copy is published as [`Bundle/24785`](https://fhir.nimloth.se/fhir/Bundle/24785) (identifier `nimloth-eps-arne-penpad`) and validates against `bundle-eu-eps` 1.0.0-ballot (matchbox) with **0 errors** (the original fixture has 0 errors / 118 warnings; ours 127, the extra ones are an unavailable eHDSI value set). A **receipt button** ("Mottaget och läst") writes a FHIR `Task` with status *completed* ([`Task/24786`](https://fhir.nimloth.se/fhir/Task/24786), focus `Bundle/24785`): a sent document is not a confirmed hand-over.

## Rector's recipe, applied
| Rector (interviews part 2 and 4) | What we built for Arne |
|---|---|
| Separate the concept model, the information model and inference; coordinate, don't merge | SNOMED CT (Snowstorm, Swedish edition) sanctions what can be said; openEHR (EHRbase, template `nimloth_penpad26_symptom.v0`) stores it; FHIR R4 / EPS carries it across the border |
| The concept model *sanctions*: query it, don't invent concepts | Every option is checked at runtime: ECL membership, or the MRCM attribute range (laterality, severity). Unsanctioned options are dropped, or shown dotted when Snowstorm is down |
| Assemble forms from fragments hung on the concept model | `model/arne-scenario.json` holds the fragments (topics, questions, candidate codes); no form rows are hard-coded in the UI |
| Big toggle buttons, a visible "more", no death by a thousand clicks | Big toggles, at most three short questions per topic, one click to undo |
| "Is this what you meant?": show back a processed result, easy to correct | Live four-language sentence and SNOMED CT expression on every click; optional local LLM (on our own box, never the cloud) may only pick among offered values (JSON schema) and its picks are marked until the clinician accepts them |
| Different language for clinicians and patients | `sv` vs `sv-enkel` from the same record; patient mode in plain Swedish |
| Be tolerant of broken parts | Snowstorm down → offline snapshot or unsanctioned + warning; EHRbase down → queued in the browser, "ej sparad"; FHIR down → resource shown, "ej skickad". Tested by pointing each at a wrong URL |
| Show that options exist; let the user turn away at any time | Patient points on a 3D body (BodyParts3D meshes); a hand-written FMA→SNOMED table (`mapping/fma-to-snomed-arne.yaml`, codes confirmed in Snowstorm, laterality as its own field) turns the tap into a site; unknown meshes resolve to the nearest listed region or "unknown place – choose in the list", never a guessed code |
| Cheap cardboard prototypes first | The engine was first built against recorded Snowstorm fixtures (the Clinergy thumb-pain walkthrough is kept as a regression test) |

## The actor test
TODO(Anders): keep the actor-test section from the v0.1 draft (Rector, part 2: was the clinician attending to the patient or to the computer?).

## Honest limitations
- FMA→SNOMED is a hand-written table for this one scenario (hip, thigh, lower leg, foot, chest, lumbar back), not a general mapping. 3D is deliberately the default only in patient mode (see the actor test and Rector's "exhaustive is the enemy of the useful").
- Danish and plain Swedish are hand-written templates for this one scenario, not a general translator.
- Danish SNOMED CT terms are not loaded in our Snowstorm (only the international and Swedish editions), so the Danish text comes from our templates only.
- Proxy consent for relatives is not solved; we only show provenance.
- openEHR uses a small template generated for the demo (`nimloth_penpad26_symptom.v0`: encounter → story → symptom_sign + anatomical_location, plus exam), pruned from CKM archetypes; a validated functional-status template (e.g. Barthel) would be better.
- Local LLM suggestions run on CPU and take about a minute.
- EHDS requirements apply from 2029–2031; this is a prototype on synthetic data (Arne is fictional).

## 3D model attribution
The 3D body in patient mode uses **BodyParts3D 4.0** (© The Database Center for Life Science, [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/), https://dbarchive.biosciencedbc.jp/en/bodyparts3d/) as packaged in **[Human Atlas](https://github.com/ashemag/human-atlas) by ashemag (MIT)**. We load only 64 skeleton meshes plus the skin (1.6 MB gzip instead of ~33 MB); meshes are copied unchanged. Human Atlas itself is not part of this pull request; we link to it.

## Thanks
To the PEN&PAD revival team for the [Clinergy extractor and notes](https://pen-pad-revival.github.io/docs/clinergy/) and the [thumb-pain walkthrough](https://pen-pad-revival.github.io/docs/clinergy-thumb-pain-demo/), and to the Nordic Hackathon organisers for the Arne scenario. We built on the ideas; we did not copy code or content from the Clinergy installation.

## Screenshots
| | |
|---|---|
| ![Orientation](01-inriktning.png) Orientation: body map + topic list | ![Refinement](02-forfining.png) Refinement as short questions |
| ![Four languages](03-fyra-sprak.png) One record, four languages | ![EPS validation](04-eps-validering.png) openEHR saved, EPS validated |
| ![Receipt](06-kvittens.png) Published EPS + "Mottaget och läst" receipt | |
| ![Clinician 3D](08-kliniker-3d.png) Optional 3D in clinician mode: left hip → pain card, side pre-filled | |
| ![Patient mode](05-patient-lage.png) Patient mode (relative answering) | ![3D body](07-patient-3d-hoft.png) Arne points at his left hip in 3D → "Du har ont i vänstra höften." / "Smerter i venstre hofte." |
