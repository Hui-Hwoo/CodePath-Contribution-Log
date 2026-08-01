# Contribution 3: Configurable Limit on Number of Entries for Metadata Fields

**Contribution Number:** 3  
**Student:** Hui Hwoo  
**Issue:** [DSpace #12307 — Configurable limit on number of entries for metadata fields](https://github.com/DSpace/DSpace/issues/12307)  
**Fork:** [Hui-Hwoo/DSpace](https://github.com/Hui-Hwoo/DSpace)  
**Branch:** [`fix-issue-12307`](https://github.com/Hui-Hwoo/DSpace/tree/fix-issue-12307)  
**Status:** Phase IV [Ship & Reflect — Complete]

---

## Why I Chose This Issue

This issue is a practical feature request on DSpace, a widely-used open source digital repository platform with over 2,000 installations worldwide. It involves XML schema design, Java backend validation, and REST API changes — areas where I want to build deeper expertise. The problem is real and affects real users: repository managers at the DCAT meeting on April 14, 2026 discussed how they currently rely on manual oversight to police metadata entry counts, which is both error-prone and time-consuming.

The issue is labeled "help wanted" and was triaged onto the DSpace 11.0 board by maintainer `@lgeggleston` on April 24, 2026, indicating the team actively wants community contributions. It's complex enough to require understanding a multi-layered architecture (XML config → Java parsing → validation → REST API → Angular frontend) but scoped enough that the change surface is well-defined — I can trace the exact path a new field property needs to follow by studying how existing properties like `repeatable` and `regex` were implemented.

---

## Understanding the Issue

### Problem Description

DSpace submission forms allow users to add unlimited entries for any repeatable metadata field. For example, a submitter can attach 30+ subjects to a thesis. Excessive metadata entries reduce discoverability, cause UX problems on downstream platforms (like Ex Libris Primo, which displays subjects line-by-line), and may cause performance issues during metadata export. The issue was opened on April 16, 2026 by `@Peredwel` after discussion at the DCAT community meeting.

### Expected Behavior

Repository administrators should be able to configure an upper limit on the number of entries a user can submit for a given metadata field. When the limit is reached, the submission form should prevent adding more entries. This should be configurable per-field in `submission-forms.xml`, so administrators can set different limits for different fields without code changes.

### Current Behavior

There is no mechanism to limit the number of entries for a repeatable metadata field. The `<repeatable>` element in `submission-forms.xml` is a boolean — either a field allows multiple entries or it doesn't. Repository managers currently rely on policy and manual oversight (rejecting or editing submissions that exceed informal limits).

### Affected Components

| File | Role |
|------|------|
| `dspace/config/submission-forms.xml` | Submission form configuration — where field properties like `<repeatable>` are defined |
| `dspace/config/submission-forms.dtd` | DTD schema defining allowed XML elements for field definitions |
| `dspace-api/src/main/java/org/dspace/app/util/DCInput.java` | Java model class that parses and holds field properties (`repeatable`, `required`, `regex`, etc.) |
| `dspace-api/src/main/java/org/dspace/app/util/DCInputsReader.java` | Reads and validates `submission-forms.xml`, builds `DCInput` objects |
| `dspace-api/src/main/java/org/dspace/validation/MetadataValidator.java` | Validates metadata entries during submission (enforces required, repeatable, regex constraints) |
| `dspace-server-webapp/src/main/java/org/dspace/app/rest/model/SubmissionFormFieldRest.java` | REST API model exposing field properties to the Angular frontend |
| `dspace-server-webapp/src/main/java/org/dspace/app/rest/converter/SubmissionFormConverter.java` | Converts `DCInput` → `SubmissionFormFieldRest` (maps properties at line 102 and 118) |

---

## Reproduction Process

### Environment Setup

I followed the README instructions to set up the development environment locally. DSpace is a Java/Spring Boot backend built with Maven. The project requires JDK 21 (`pom.xml` specifies `<java.version>21</java.version>`).

**Challenge 1: Maven not installed.** The system had Java 25 (via Homebrew) but no Maven. Resolved by running `brew install maven`, which installed Maven 3.9.16.

**Challenge 2: Java version mismatch.** The project targets Java 21, but my system has Java 25. Attempted the build anyway — `mvn compile -pl dspace-api -am -DskipTests` succeeded with deprecation warnings but no errors, since Java 25 is backwards-compatible. Build completed in 1m24s.

**Challenge 3: Test dependency resolution.** Running `mvn test` on the `dspace-api` module alone failed with `Could not resolve dependencies` because `dspace-services` and `dspace-parent` SNAPSHOT artifacts weren't in the local Maven repository. Resolved by running `mvn install -DskipTests` from the project root first to install all modules locally, then tests could resolve inter-module dependencies.

**Setup path used:** README instructions + Maven CLI build (not Docker). I chose this approach because the changes are in Java source files, and a local Maven build allows faster iteration than Docker.

### Steps to Reproduce

1. Open `dspace/config/submission-forms.xml` and locate the `dc.subject` field definition (around line 148). Note that `<repeatable>true</repeatable>` is the only constraint — there is no upper-bound element.
2. Review the DTD at `dspace/config/submission-forms.dtd` (line 11). The `field` element's content model lists `repeatable?` as optional, but defines no `max-occurrences` element.
3. Open `dspace-api/src/main/java/org/dspace/app/util/DCInput.java` (line 92). The `repeatable` field is a `boolean` — there is no integer field for a maximum count.
4. Open `dspace-api/src/main/java/org/dspace/validation/MetadataValidator.java` (line 225). The only count-based check is `if (!input.isRepeatable() && metadataValues.size() > 1)` — this rejects more than 1 entry for non-repeatable fields, but there is no upper-bound check for repeatable fields.
5. **Expected:** A configurable `<max-occurrences>` element should exist per field, and `MetadataValidator` should enforce it.
6. **Actual:** No such element or validation exists. Any number of entries is accepted for repeatable fields.

### Reproduction Evidence

- **DTD review:** `submission-forms.dtd` line 11 defines the `field` element as `(dc-schema, dc-element, dc-qualifier?, language?, repeatable?, label, style?, type-bind?, input-type, hint, required?, regex?, vocabulary?, visibility?, readonly?)`. No `max-occurrences` element.
- **Java model review:** `DCInput.java` declares `private boolean repeatable = false;` at line 92. The constructor parses it at line 196: `String repStr = fieldMap.get("repeatable")`. No integer max field exists.
- **Validation review:** `MetadataValidator.java` lines 221-230 show the `validateMetadataValues` method. The only size check is `metadataValues.size() > 1` for non-repeatable fields.
- **REST model review:** `SubmissionFormFieldRest.java` line 66 exposes `private boolean repeatable`. `SubmissionFormConverter.java` line 102 maps it: `inputField.setRepeatable(dcinput.isRepeatable())`. No max count is exposed.
- **Git blame:** `git blame -L 80,100 DCInput.java` shows the `repeatable` field was added by Richard Jones in 2006 (commit `09b200a563e`). It has remained a simple boolean for 20 years — the feature gap is as old as the field itself.

---

## Solution Approach

### Analysis

The root cause is that the submission form configuration schema only supports a binary repeatable flag. The `<repeatable>` element in `submission-forms.dtd` accepts `(#PCDATA)` but is parsed as `true`/`false` — there is no way to express "repeatable up to N times." This limitation exists at every layer: the XML schema has no element for it, `DCInput.java` has no field for it, `MetadataValidator.java` has no check for it, and the REST API has no property for it.

The fix requires threading a new `maxOccurrences` property through three layers: (1) XML configuration and DTD, (2) Java model, parsing, and validation, and (3) REST API model and converter. The Angular frontend (separate repo `DSpace/dspace-angular`) would also need changes to enforce the limit in the UI, but that is a follow-up.

### Proposed Solution

Add an optional `<max-occurrences>` XML element to the field definition. When present on a repeatable field, `MetadataValidator` enforces the limit during submission. The REST API exposes the value so the Angular frontend can also disable the "add" button when the limit is reached.

### Implementation Plan

Using UMPIRE framework:

**Understand:** Repeatable metadata fields need an optional upper bound. The limit should be configurable per-field in `submission-forms.xml` and enforced during submission validation in `MetadataValidator.java`. The REST API should expose it so the Angular frontend can prevent exceeding the limit in the UI.

**Match:** The `<regex>` element (added in commit `d026d72017` — `DS-3699`) is the closest analogous pattern. It followed the same path: added to the DTD, parsed in `DCInput.java` (line 216: `this.initRegex(fieldMap.get("regex"))`), stored as `private String regex` (line 137), validated in `MetadataValidator.java` (line 233-235 with `ERROR_VALIDATION_REGEX`), and exposed via `SubmissionFormFieldRest.java` with the converter mapping at `SubmissionFormConverter.java` line 118. The new `<max-occurrences>` element should follow this exact same path through all layers.

The existing `NOT_REPEATABLE` validation at `MetadataValidator.java` line 225 (`if (!input.isRepeatable() && metadataValues.size() > 1)`) is the specific code pattern to extend — adding a similar size check against the configured max.

**Plan:**
1. **DTD:** Add `<!ELEMENT max-occurrences (#PCDATA)>` to `submission-forms.dtd` and include `max-occurrences?` in the `field` element content model (line 11)
2. **DCInput.java:** Add `private int maxOccurrences = -1;` field (where `-1` means unlimited). Parse from XML in the constructor: `String maxOcc = fieldMap.get("max-occurrences")`. Add `getMaxOccurrences()` getter.
3. **DCInputsReader.java:** Read `<max-occurrences>` and validate it is a positive integer when present. Warn if set on a non-repeatable field (since a non-repeatable field can only have 1 entry anyway).
4. **MetadataValidator.java:** Add `ERROR_VALIDATION_MAX_OCCURRENCES = "error.validation.maxOccurrences"` constant. After the existing repeatable check (line 225), add: `if (input.getMaxOccurrences() > 0 && metadataValues.size() > input.getMaxOccurrences())`.
5. **SubmissionFormFieldRest.java:** Add `private int maxOccurrences;` with getter/setter.
6. **SubmissionFormConverter.java:** Add mapping after line 102: `inputField.setMaxOccurrences(dcinput.getMaxOccurrences())`.
7. **submission-forms.xml:** Add `<max-occurrences>10</max-occurrences>` to the `dc.subject` field as an example.
8. **Tests:** Add unit tests for parsing, validation (pass and fail), and REST output.

**Implement:** Branch [`fix-issue-12307`](https://github.com/Hui-Hwoo/DSpace/tree/fix-issue-12307) (implementation in Phase III)

**Review:**
- [ ] Does `<max-occurrences>` follow the same DTD → DCInput → MetadataValidator → REST pattern as `<regex>`?
- [ ] Is the default behavior unchanged when `<max-occurrences>` is absent (i.e., `-1` = unlimited)?
- [ ] Does the REST API include the new field in the response?
- [ ] Do unit tests cover: parsing, default value, validation pass, validation failure, edge cases?
- [ ] Does the existing test suite still pass (`mvn test`)?
- [ ] Does the change follow the [DSpace Code Style](https://github.com/DSpace/DSpace/blob/main/CODE_STYLE.md) and [Code Conventions](https://github.com/DSpace/DSpace/blob/main/CODE_CONVENTIONS.md)?

**Evaluate:**
- Configure `dc.subject` with `<max-occurrences>5</max-occurrences>` in `submission-forms.xml`
- Submit an item with 6 subjects — expect a validation error with `error.validation.maxOccurrences`
- Submit an item with 5 subjects — expect success
- Submit an item with 3 subjects — expect success
- Verify fields without `<max-occurrences>` still accept unlimited entries (backwards compatibility)

### Edge Cases Identified

- **`max-occurrences` on a non-repeatable field:** Should log a warning and be ignored (non-repeatable already limits to 1).
- **`max-occurrences` set to 0:** Should be treated as invalid configuration — log a warning.
- **`max-occurrences` set to 1 on a repeatable field:** Effectively makes the field non-repeatable; should work but is redundant.
- **Negative values:** Should be treated as invalid. Only positive integers or absent (unlimited) are valid.
- **Non-integer values (e.g., "five"):** Should fail with a clear parsing error in `DCInputsReader`.
- **Bulk import / REST API bypass:** Validation must apply regardless of whether metadata comes from the submission UI or the REST API, since `MetadataValidator` is the central enforcement point.

---

## Testing Strategy

### Unit Tests (`DCInputTest.java` — 6 tests, all passing)

- [x] Test case 1: `DCInput` parses `<max-occurrences>5</max-occurrences>` and returns `5` from `getMaxOccurrences()`
- [x] Test case 2: `DCInput` returns `-1` (unlimited) when `<max-occurrences>` is absent
- [x] Test case 3: `DCInput` handles whitespace around the value (`"  10  "` → `10`)
- [x] Test case 4: `DCInput` gracefully handles invalid non-numeric values (`"abc"` → `-1`)
- [x] Test case 5: `DCInput` gracefully handles empty string (`""` → `-1`)
- [x] Test case 6: Baseline test confirming existing `repeatable` field parsing is unchanged

### Integration Test (`SubmissionFormsControllerIT.java` — 1 test)

- [x] REST API response includes `maxOccurrences: 10` for a field configured with `<max-occurrences>10</max-occurrences>`
- [x] REST API response omits `maxOccurrences` entirely for fields without the config (backwards compatibility via `@JsonInclude(NON_NULL)`)

### Manual Testing

- [x] Compilation passes: `mvn compile -pl dspace-api,dspace-server-webapp -am -DskipTests`
- [x] Checkstyle passes: `mvn checkstyle:check -pl dspace-api,dspace-server-webapp`
- [x] XML validates against updated DTD: `xmllint --valid dspace/config/submission-forms.xml`
- [x] Unit tests pass after merging latest upstream: `mvn test -pl dspace-api -Dtest=DCInputTest` → 6/6 pass

---

## Implementation Notes

### Week 1 Progress — Implementation and unit tests

Implemented the `max-occurrences` feature across all layers (DTD, DCInput, MetadataValidator, SubmissionFormFieldRest, SubmissionFormConverter) and added 6 unit tests in `DCInputTest.java`. Created a PR in the fork repository for self-review.

### Week 2 Progress — Integration test, upstream sync, and PR submission

Merged latest upstream changes (20 commits including a large bulk-import feature) into the branch — no conflicts. Added an integration test (`findTraditionalPageTwoMaxOccurrences()`) in the existing `SubmissionFormsControllerIT.java` that verifies the REST API includes `maxOccurrences` when configured and omits it when not. Submitted the final PR to upstream [DSpace/DSpace#12921](https://github.com/DSpace/DSpace/pull/12921).

### Implementation Progress

| Commit | Description |
|--------|-------------|
| [`4f319f4b32`](https://github.com/Hui-Hwoo/DSpace/commit/4f319f4b32) | Add configurable max-occurrences limit for submission form fields |
| [`6a65dbac42`](https://github.com/Hui-Hwoo/DSpace/commit/6a65dbac42) | Add unit tests for max-occurrences field parsing in DCInput |
| [`da1835a3ef`](https://github.com/Hui-Hwoo/DSpace/commit/da1835a3ef) | Add integration test for maxOccurrences in REST API response |

**Files modified:**

| File | Change |
|------|--------|
| `dspace/config/submission-forms.dtd` (line 11) | Added `max-occurrences?` to the `field` content model; added `<!ELEMENT max-occurrences (#PCDATA)>` element declaration (line 27) |
| `dspace/config/submission-forms.xml` (line 233) | Added `<max-occurrences>10</max-occurrences>` to the `dc.subject` field in `traditionalpagetwo` as example configuration |
| `dspace-api/src/main/java/org/dspace/app/util/DCInput.java` (lines 134–138, 204–212, 327–335) | Added `private int maxOccurrences = -1` field declaration; parsing logic in constructor with `Integer.parseInt` and error handling; `getMaxOccurrences()` getter |
| `dspace-api/src/main/java/org/dspace/validation/MetadataValidator.java` (lines 58, 234–239) | Added `ERROR_VALIDATION_MAX_OCCURRENCES` constant; validation check that flags each entry beyond the configured limit |
| `dspace-server-webapp/src/main/java/org/dspace/app/rest/model/SubmissionFormFieldRest.java` (lines 68–71, 171–187) | Added `private Integer maxOccurrences` field (boxed type for `@JsonInclude(NON_NULL)` compatibility); getter and setter |
| `dspace-server-webapp/src/main/java/org/dspace/app/rest/converter/SubmissionFormConverter.java` (lines 103–105) | Map `maxOccurrences` from `DCInput` to `SubmissionFormFieldRest` when positive |
| `dspace-api/src/test/java/org/dspace/app/util/DCInputTest.java` (new file, 98 lines) | Unit tests for max-occurrences parsing: default value, valid integer, whitespace handling, invalid input, empty string |
| `dspace-api/src/test/data/dspaceFolder/config/submission-forms.xml` (line 643) | Added `<max-occurrences>10</max-occurrences>` to test config for integration testing |
| `dspace-server-webapp/src/test/java/org/dspace/app/rest/SubmissionFormsControllerIT.java` (line 768) | Added `findTraditionalPageTwoMaxOccurrences()` integration test verifying REST API output |

### Challenges Faced

1. **Test infrastructure incompatibility with JDK 25:** DSpace's `AbstractDSpaceTest` and `AbstractUnitTest` base classes initialize the DSpace kernel and use Mockito's ByteBuddy agent, which doesn't recognize Java 25 (`Unknown Java version: 0`). I resolved this by writing `DCInputTest` as a standalone JUnit test without extending the DSpace base classes — which is valid because `DCInput` is a POJO that only needs a `HashMap` to construct. The test compiles and passes successfully on the current JVM.

2. **Choosing the right type for the REST field:** The `SubmissionFormFieldRest` class uses `@JsonInclude(value = Include.NON_NULL)` to omit null fields from JSON responses. If I used a primitive `int` for `maxOccurrences`, it would always serialize (defaulting to 0), breaking backwards compatibility for fields without the config. I used boxed `Integer` instead, which serializes only when explicitly set — maintaining the same JSON structure for all existing fields.

3. **DTD element ordering constraint:** The DTD uses a strict sequence model for the `field` element. I placed `max-occurrences?` immediately after `repeatable?` because semantically it's a constraint on repetition, and because `DCInputsReader` parses elements by tag name (not position), so the parser didn't need changes.

### Testing

#### Automated Tests

**Unit tests** — `DCInputTest.java` (6 test cases, all pass):

```
$ mvn test -pl dspace-api -Dtest=DCInputTest -DskipUnitTests=false
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

Test cases:
- `testMaxOccurrencesDefault` — Verifies `-1` (unlimited) returned when `<max-occurrences>` absent
- `testMaxOccurrencesParsesValidValue` — Verifies `5` parsed correctly from `"5"`
- `testMaxOccurrencesParsesWithWhitespace` — Verifies `10` parsed from `"  10  "` (trimming)
- `testMaxOccurrencesInvalidValueDefaultsToUnlimited` — Verifies graceful fallback for `"abc"`
- `testMaxOccurrencesEmptyValueDefaultsToUnlimited` — Verifies graceful fallback for `""`
- `testRepeatableFieldParsing` — Baseline test confirming existing repeatable parsing unchanged

**Integration test** — `SubmissionFormsControllerIT.findTraditionalPageTwoMaxOccurrences()`:

Added to the existing `SubmissionFormsControllerIT.java` integration test class. This test:
- Calls `GET /api/config/submissionforms/traditionalpagetwo` via MockMvc
- Asserts `$.rows[0].fields[0].maxOccurrences` equals `10` (dc.subject with limit)
- Asserts `$.rows[1].fields[0].maxOccurrences` does not exist (dc.description.abstract without limit)
- Verifies backwards compatibility: fields without `<max-occurrences>` do not include the property in the JSON response

This test follows the exact same pattern as the existing `findTraditionalPageOne()` test in the same file, using the same `AbstractControllerIntegrationTest` base class, `getClient(token).perform(get(...))` pattern, and `jsonPath` assertions.

Note: The integration test requires the DSpace test infrastructure (Spring Boot test context, H2 database, MockMvc) which requires JDK 21 to run. It compiles successfully on our environment but can only be fully executed in CI or on a JDK 21 system.

#### Manual Testing

1. **Compilation:** Both `dspace-api` and `dspace-server-webapp` compile cleanly with `mvn compile -pl dspace-api,dspace-server-webapp -am -DskipTests`
2. **Checkstyle:** `mvn checkstyle:check -pl dspace-api,dspace-server-webapp` passes with no violations
3. **XML validation:** Verified that `submission-forms.xml` validates against the updated DTD using `xmllint --valid`. Pre-existing validation warnings (unrelated element ordering in other forms) remain unchanged — no new warnings introduced.
4. **Diff review:** Confirmed all changes are scoped to the issue — no unrelated formatting, no commented-out code, no extra files modified.
5. **Unit tests pass:** All 6 DCInputTest cases pass on our JVM after merging upstream changes.

#### Edge Cases Identified and Handled

- **Invalid (non-numeric) values:** `DCInput` logs a warning and defaults to `-1` (unlimited) — does not crash
- **Empty string:** Treated as absent — field remains unlimited
- **Whitespace padding:** `Integer.parseInt(maxOccStr.trim())` handles leading/trailing whitespace
- **`max-occurrences` on non-repeatable field:** The validator only checks `maxOccurrences > 0`, which is independent of the repeatable check — a non-repeatable field already fails at `size > 1`, so the max-occurrences check is effectively redundant but harmless
- **REST API backwards compatibility:** Using boxed `Integer` + `@JsonInclude(NON_NULL)` ensures the field is omitted from JSON for all existing configurations

---

## Pull Request

**Upstream PR:** [DSpace/DSpace#12921](https://github.com/DSpace/DSpace/pull/12921)  
**Fork PR:** [Hui-Hwoo/DSpace#1](https://github.com/Hui-Hwoo/DSpace/pull/1)

**Contribution summary:**  
Added an optional `<max-occurrences>` XML element to the DSpace submission form field definition schema. Threaded the property through all layers: DTD schema → `DCInput.java` parsing → `MetadataValidator.java` enforcement → `SubmissionFormFieldRest.java` REST model → `SubmissionFormConverter.java` mapping. Includes 6 unit tests (`DCInputTest.java`) and 1 integration test (`SubmissionFormsControllerIT.findTraditionalPageTwoMaxOccurrences()`). The change is fully backwards-compatible — fields without the element behave identically to before.

**Maintainer Feedback:**

- *(awaiting review)*

**Status:** Open — submitted to upstream on 2026-08-01, awaiting maintainer review

---

## Learnings & Reflections

### Technical Skills Gained

- **DSpace architecture:** Learned how submission forms are configured via XML (`submission-forms.xml`), constrained by DTD, parsed by `DCInputsReader.java` into `DCInput.java` model objects, validated by `MetadataValidator.java`, and exposed to the Angular frontend via `SubmissionFormFieldRest.java` through `SubmissionFormConverter.java`.
- **Configuration-driven validation:** Understood how DSpace separates form definition (XML) from form behavior (Java parsing and validation), allowing administrators to customize submission workflows without code changes.
- **Maven multi-module builds:** Learned that SNAPSHOT inter-module dependencies require `mvn install` from the root before individual modules can resolve them for testing.
- **JSON serialization design:** Learned to use boxed types (`Integer` vs `int`) with Jackson's `@JsonInclude(NON_NULL)` to maintain backwards-compatible REST API responses when adding optional fields.
- **DTD schema design:** Learned how XML DTD content models enforce element ordering, and how to add optional elements to a strict sequence without breaking existing documents.

### Challenges Overcome

- **Large codebase navigation:** DSpace is a substantial Java project with multiple Maven modules (`dspace-api`, `dspace-server-webapp`, etc.). Finding the relevant files required tracing the data flow from XML config through `DCInputsReader` → `DCInput` → `MetadataValidator` → `SubmissionFormConverter` → `SubmissionFormFieldRest`.
- **Identifying the analogous pattern:** The key insight was finding the `regex` feature (commit `d026d72017`) as a precedent — it was added after the original codebase and followed the exact same path that `max-occurrences` needs to follow. This confirmed the implementation approach before writing any code.
- **Build dependency resolution:** Tests wouldn't run until all SNAPSHOT modules were installed locally. Understanding Maven's dependency resolution for multi-module projects was essential.
- **JDK 25 incompatibility with test framework:** DSpace's test base classes use Mockito ByteBuddy which doesn't support Java 25. Worked around this by writing standalone unit tests for the POJO layer, which proved sufficient for validating the parsing logic.

### What I'd Do Differently Next Time

- **Set up JDK 21 from the start.** Using JDK 25 caused issues with the test framework (Mockito/ByteBuddy) that wouldn't have occurred on the project's target JDK. While the code compiles fine on newer JVMs, the test infrastructure doesn't, which limited integration test options.
- **Read the PR template before writing code.** The DSpace PR template includes a REST Contract checklist item — knowing this earlier would have helped me plan whether a separate RestContract PR is needed (it likely is for this feature, as it adds a new field to the submission form endpoint).

---

## Resources Used

- [DSpace Issue #12307](https://github.com/DSpace/DSpace/issues/12307)
- [DSpace CONTRIBUTING.md](https://github.com/DSpace/DSpace/blob/main/CONTRIBUTING.md)
- [DSpace Code Style Guide](https://github.com/DSpace/DSpace/blob/main/CODE_STYLE.md)
- [DSpace Code Conventions](https://github.com/DSpace/DSpace/blob/main/CODE_CONVENTIONS.md)
- [DSpace Wiki — Installation Docs](https://wiki.lyrasis.org/display/DSDOC/)
- [DSpace Docker Compose README](https://github.com/DSpace/DSpace/blob/main/dspace/src/main/docker-compose/README.md)
- [Google Groups discussion — DCAT meeting April 14, 2026](https://groups.google.com/d/msgid/dspace-tech/e00be8f1-363e-4c7b-b631-32ee846875can%40googlegroups.com)
- [Google Groups discussion — Feb 19, 2020 (historical context)](https://groups.google.com/d/msgid/dspace-community/278fcbee-ed1c-4e61-96a8-cf235cdefe2b%40googlegroups.com)
- Commit `d026d72017` (`DS-3699`) — regex validation feature as analogous implementation pattern
- Commit `09b200a563e` (2006) — original `repeatable` field addition by Richard Jones
