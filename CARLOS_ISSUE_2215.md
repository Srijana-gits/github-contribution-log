# Contribution [1]: (codex assist) Review Hashtable error-map usage in AddEditDocument2Action #2215

**Contribution Number:** 1  
**Student:** Srijana  
**Issue:** https://github.com/carlos-emr/carlos/issues/2215
**Status:** Phase IV Complete

---

## Why I Chose This Issue

I selected this issue because it is an active, unassigned task designated as a "good first issue" with highly specific, well-defined acceptance criteria. It provides a contained, practical opportunity for me to work on legacy Java architecture, understand request-scoping vs. thread synchronization, and practice safe refactoring without overcomplicating the project's core business logic. 

Furthermore, working on an Electronic Medical Record (EMR) system aligns with my interest in seeing how software architecture handles crucial data validation workflows in real-world environments. This issue allows me to build confidence in navigating a large codebase, running targeted Maven tests, and safely updating boundaries between controllers (Actions) and views (JSPs).

---

## Understanding the Issue

### Problem Description

This is not a functional bug — the add/edit document workflow works correctly today. It is a tech-debt / modernization task. ``AddEditDocument2Action`` builds its validation error maps (exposed to the views as the docerrors request attribute) using the `legacy java.util.Hashtable`. The issue asks whether these maps can become `HashMap / Map<String,String>` without changing any behavior. Hashtable dates back to `Java 1.0` and synchronizes every get/put; that locking is wasted effort for request-scoped data that only a single request thread ever touches. `PR #2092` modernized the surrounding upload-validation code but deliberately left the error-map plumbing alone to keep that PR focused. `Issue #2215` is the follow-up to finish that cleanup.

### Expected Behavior

- Error maps are constructed with `HashMap` and passed around as `Map<String,String>` at boundaries.
- The docerrors request-attribute name and all existing error keys (descmissing, typemissing,  filenameinvalid, uploaderror) stay byte-for-byte identical.
- Add- and edit-document validation errors render in the JSPs exactly as before — zero visible change for the user.

### Current Behavior

`docerrors` is a `synchronized Hashtable`, and the concrete type leaks into the view layer: `editDocument.jsp` hard-casts the request attribute to Hashtable and iterates it with the `Hashtable-only .keys() / Enumeration API`. It is functionally correct but legacy, and tightly coupled to the concrete implementation type rather than the Map interface.

### Affected Components

- `AddEditDocument2Action.java` - `Hashtable errors = new Hashtable()` (~`lines` `321`, `371`, `542`) and `request.setAttribute("docerrors", errors)` (~`lines` `520`, `525`, `672`, `676`, `680`).
- `editDocument.jsp` - the local declaration (~`line 89`), the (`Hashtable`) cast (~`line 91`), and the `Enumeration/.keys()` loop (~`line 270`). This is the file that actually couples to the concrete type.
- `addDocument.jsp` — reads `docerrors` purely through `JSTL/EL (${docerrors[...]}, <c:forEach>)`, which works on any `Map`, so it needs no change.
- `AddEditDocument2ActionUnitTest` — any assertion that casts to `Hashtable`.
- Out of scope (same pattern, tracked separately): `AddEditHtml2Action` + `addedithtmldocument.jsp` use an identical
`linkhtmlerrors` `Hashtable`.


---

## Reproduction Process

### Environment Setup

Development runs entirely inside the project's devcontainer (isolated Docker, synthetic data — no real PHI). There can be a cache issue with Docker where build steps — particularly large ones like Playwright browser installation — appear to hang indefinitely after downloading. This happens because Docker's internal WSL virtual disk fills up silently, leaving no space to extract downloaded files, causing the process to freeze without any error message. The fix is to prune unused Docker resources to free up space:

```bash
docker system prune -a --volumes -f
```

This removes unused images, containers, build cache, and volumes, freeing space in Docker's underlying virtual disk. After pruning, retry the devcontainer build and it should complete successfully.

Toolchain:
Maven 3.8.7, Java 21 (Temurin), Tomcat 11. I authenticated gh via the GitHub device-code flow so I could read the issue thread and the maintainer's pointers directly. Building with the jspc Maven profile precompiles every JSP at build time instead of lazily on first request, which is the key to surfacing JSP-layer type errors before runtime.

▎ Note: because this is a refactor rather than a defect, there is no pre-existing user-facing bug to reproduce. "Reproduction" here means 

▎ (a) establishing a known-good baseline and 

▎(b) demonstrating the latent coupling that a naive refactor would break — so I know exactly what the change has to keep in sync.

### Steps to Reproduce

1. Baseline build on a clean tree: `mvn -B -Pjspc package -DskipTests` → `BUILD SUCCESS` (~3.5 min). Confirms all JSPs compile
and the toolchain is healthy.
2. Simulate a partial/naive refactor: in `editDocument.jsp`, change `Hashtable` docerrors to `Map/HashMap` and the cast to
(Map), but leave the `.keys()` loop unchanged. Re-run the same jspc build.
3. Observed result: `BUILD FAILURE` — The method keys() is undefined for the type Map in editDocument.jsp. This is the
build-time failure mode jspc is meant to catch.
4. Second (runtime-only) failure mode: if instead only the Action is switched to HashMap while editDocument.jsp still
casts to (Hashtable), the project compiles fine but throws ClassCastException: HashMap cannot be cast to Hashtable at
runtime — specifically when the edit form re-renders after a validation error (edit a document, clear the Description,
click Update → the action returns the failEdit result and forwards back to editDocument.jsp).


### Reproduction Evidence

- Commit showing reproduction: N/A — this is a refactoring task, not a defect; there is no pre-existing failing state to commit. See the jspc failure log below, which demonstrates the latent coupling.
- Screenshots/logs: jspc failure output -
```An error occurred at line: [272] in the jsp file: [/WEB-INF/jsp/documentManager/editDocument.jsp]
The method keys() is undefined for the type Map
[INFO] BUILD FAILURE
[ERROR] ...jspc-maven-plugin:5.0.0:compile (jspc) on project carlos: Failure processing jsps
```
- My findings: The refactor is only safe as a single coordinated change across the Action + editDocument.jsp (declaration,
cast, and .keys() loop) + the unit test. addDocument.jsp is already safe because it uses EL. The JSP-side breakage is
caught at build time by jspc; the cast breakage is invisible until the exact failEdit re-render path is exercised, which
is why it does not show up in casual UI clicking.

---

## Solution Approach

### Analysis

The root cause is two-fold: (1) use of the legacy synchronized Hashtable for request-scoped data that never needs
synchronization, and (2) leakage of Hashtable's concrete API into the view. The two coupling points are both in
editDocument.jsp — the (Hashtable) cast and the .keys()/Enumeration iteration. addDocument.jsp avoids the problem entirely
by reading the map through EL. Therefore the swap is safe only if every coupling point is moved onto the Map interface in
the same change.

### Proposed Solution

- Action: construct error maps with new HashMap<>() and type fields/locals/parameters as Map<String,String>. Keep the
docerrors attribute name and all error keys unchanged.
- editDocument.jsp: declare Map docerrors, cast to (Map), and replace the Enumeration/.keys() loop with a keySet() (or
entrySet()) iteration. containsKey(...) already works on Map, so those lines are untouched.
- Test: update any Hashtable cast in AddEditDocument2ActionUnitTest to Map.
- addDocument.jsp: leave untouched.

### Implementation Plan

**Understand**: Swap the request-scoped Hashtable error maps to HashMap/Map in AddEditDocument2Action without changing the
docerrors contract (attribute name, keys) or the rendered validation-error output.

**Match**: addDocument.jsp's EL-based read is the target pattern — depend on the Map interface, not the concrete type.
Map<String,String> is already the idiom across the codebase, and keySet()/entrySet() iteration is the standard replacement
for the legacy Enumeration/.keys() style.

**Plan**:
1. In AddEditDocument2Action.java, change the three Hashtable constructions to HashMap, type all boundaries as
Map<String,String>, and fix the import.
2. In editDocument.jsp, change the declaration and cast to Map and convert the .keys() loop to a keySet()/entrySet() loop;
add java.util.Map/HashMap imports if needed.
3. Update AddEditDocument2ActionUnitTest to remove the Hashtable cast.

**Implement**: Branch refactor/replace-hashtable-docerrors-hashmap. [commit links — TODO as I work]

**Review**: Self-review checklist — docerrors attribute name and error keys unchanged; no behavioral/UI change; OWASP encoding
in the JSP left intact; no new SpringUtils shims introduced; coupled edits (Action + JSP decl + JSP cast + JSP loop +
test) landed together; encoder null-safety lint still passes.

**Evaluate**: Verified via the targeted unit test and the jspc precompile build (details in the Testing Strategy section).

---

## Testing Strategy

### Unit Tests

  All unit tests live in AddEditDocument2ActionUnitTest and were run with mvn -B -Dtest=AddEditDocument2ActionUnitTest
  test. Final run: 29 tests, 0 failures, 0 errors.

  - [x] Test case 1 (new): shouldReturnFailAdd_whenDescriptionMissing — blank description in add mode returns failAdd and
  the docerrors map contains descmissing → dms.error.descriptionInvalid. Covers a validation path that previously had no
  test.
  - [x] Test case 2 (new): shouldReturnFailAdd_whenDocumentTypeMissing — blank document type in add mode returns failAdd
  and docerrors contains typemissing → dms.error.typeMissing. Also previously untested.
  - [x] Test case 3 (adjusted): existing error-path tests widened to the Map interface — the four assertions that
  previously cast the docerrors attribute to Hashtable<String,String> (uploaderror missing-upload, uploaderror
  empty-replacement on failEdit, and two filenameinvalid cases) were changed to Map<String,String>. These are the runtime
  guard: left as Hashtable, they would throw ClassCastException once the action emits a HashMap. AssertJ's containsEntry
  works on any Map, so behavior assertions are unchanged.

  Tests were run after each step (not just at the end): Step A (widen casts + add tests) → 29 pass against the old
  Hashtable; Step B (action → HashMap/Map) → 29 pass, proving behavior is unchanged.


### Integration Tests

  - [x] JSP precompilation (jspc) — mvn -B -Pjspc package -DskipTests → BUILD SUCCESS (~3:35). This compiles every JSP,
  including editDocument.jsp, and is the build-time check that catches the JSP-side coupling. As documented in
  Reproduction, a partial refactor makes this exact build fail with The method keys() is undefined for the type Map.
  - N/A — No new database/Spring integration tests were needed. docerrors is request-scoped, in-memory data with no
  persistence layer, so there is nothing to exercise at the DB/Hibernate level.

### Manual Testing

  Because the regression is a compile-time / cast-type issue rather than a behavioral one, verification was done
  deterministically at build time (jspc + unit tests) rather than by clicking through the UI. The end-to-end sequence run
  this session:

  1. Baseline jspc on the clean tree → BUILD SUCCESS (toolchain healthy, nothing mismatched).
  2. Synthetic partial refactor (type → Map, leave .keys()) → BUILD FAILURE (reproduces the latent coupling).
  3. Coordinated fix applied → unit tests 29/0/0 and jspc → BUILD SUCCESS.

  The UI path a reviewer can use to confirm the runtime behavior manually: eDoc → open a document's Edit form → clear the 
  Description (or Type) → Update, which forces the failEdit re-render of editDocument.jsp (the path that would throw
  ClassCastException if the JSP cast were left as Hashtable). Browser-based manual testing was not performed for this
  change; the jspc precompile and unit tests cover the regression.


---

## Implementation Notes

  - Confirmed project conventions in CONTRIBUTING.md and .github/PULL_REQUEST_TEMPLATE.md before writing code (Conventional
  Commits, mandatory DCO sign-off via git commit -s, target develop, focused PR, add tests).
  - Implemented the refactor in three coordinated steps, running tests after each (Step A: tests; Step B: action; Step C:
  JSP + jspc).
  - Set a repo-local git identity (Srijana-gits) using the GitHub privacy noreply email so no personal email is exposed —
  confirmed compliant with CONTRIBUTING.md (no email/privacy rule) and DCO (sign-off email matches author email).
  - Committed locally with DCO sign-off (f162720183).

### Code Changes
    
- **Files modified:** 
    - src/main/java/io/github/carlos_emr/carlos/documentManager/actions/AddEditDocument2Action.java — 3× Hashtable →
  Map<String,String> / new HashMap<>(); swapped import java.util.Hashtable for HashMap + Map; updated two stale "hashtable"
  comments.
    - src/main/webapp/WEB-INF/jsp/documentManager/editDocument.jsp — declaration and cast → Map; converted the
  Enumeration/.keys() loop to a keySet() loop (Map already importable via the existing java.util.* page import).
    - src/test/java/io/github/carlos_emr/carlos/documentManager/actions/AddEditDocument2ActionUnitTest.java — 4 casts
  widened to Map<String,String>; swapped Hashtable import for Map; added the two new validation tests.
    - Diff summary: 3 files changed, 50 insertions(+), 15 deletions(-).

- **Key commits:** f162720183 — refactor: replace Hashtable error maps with HashMap in AddEditDocument2Action (committed
  locally with DCO sign-off and  pushed).

- **Approach decisions:** - Coordinated single change. The Hashtable→Map swap is only safe if the action, the JSP (decl + cast + .keys() loop),
  and the test casts move together — done in one logical commit.
    - Map at boundaries, HashMap at construction (per the maintainer's guidance), so the view depends on the interface, not
  the concrete type.
    - addDocument.jsp left untouched — it reads docerrors via JSTL/EL, which is already Map-agnostic.
    - Kept the contract identical — docerrors attribute name and all error keys unchanged; no behavioral/UI change.
    - Scope discipline — left a pre-existing unused FileValidationException import and a pre-existing sanitizeFileName
  deprecation warning alone (both predate this change; confirmed against HEAD).



---

## Pull Request

  **PR Link:** https://github.com/carlos-emr/carlos/pull/2976

  **PR Description:** Modernizes the request-scoped `docerrors` validation error maps in
  `AddEditDocument2Action` from the legacy synchronized `Hashtable` to `HashMap`/`Map<String,String>`,
  with coordinated updates to `editDocument.jsp` (decl/cast → `Map`, `.keys()` → `keySet()`) and the
  unit-test casts. The `docerrors` attribute name and all error keys are unchanged; `addDocument.jsp`
  needed no change (JSTL/EL is already `Map`-agnostic). Verified with the targeted unit test
  (29 passing, incl. two new `descmissing`/`typemissing` tests) and the `jspc` precompile build.
  Closes #2215. (Full template body: Description / Related Issues / How Was This Tested / Checklist.)



  **Maintainer Feedback:**
  - 2026-06-21: Opened PR and pinged @Ben-Heerema for review.
  - 2026-06-22: @Ben-Heerema reviewed and asked me to apply the generics the CI bots
      (Gemini Code Assist, Sourcery) flagged — raw `Map`/`HashMap` in the `editDocument.jsp`
      scriptlet (lines 91, 271). Pushed a follow-up commit using `Map<String,String>`/`HashMap<>`
      for the declaration and cast, and a typed `String` key in the `keySet()` loop (dropping the
      redundant `(String)` cast). Re-verified: unit test 29 passing + `jspc` build green.

    **Status:** Generics feedback addressed and pushed; awaiting bot re-scan and human maintainer review.

  ---

  ## Learnings & Reflections

  ### Technical Skills Gained

  - **Collections internals:** the practical difference between `Hashtable` (legacy, synchronizes
    every operation) and `HashMap`, and why synchronization is pointless for request-scoped data
    touched by a single thread. Learned to program to the `Map` interface at boundaries rather than
    a concrete type.
  - **JSP mechanics:** that a `.jsp` is translated to a Java servlet and compiled, so scriptlet code is real type-checked Java. Used the `jspc` Maven profile
    to precompile every JSP at build time and surface type errors early. Also saw that JSTL/EL
    (`${docerrors[...]}`) is `Map`-agnostic, which is why `addDocument.jsp` needed no change.
  - **Struts2 flow:** how `failAdd`/`failEdit` results forward back to a JSP and how request
    attributes carry the error map to the view.
  - **Tooling & workflow:** targeted Maven test runs (`-Dtest=...`), the `jspc` profile, Conventional
    Commits, DCO sign-off (`git commit -s`), using a GitHub privacy noreply email, and the
    fork → feature-branch → PR workflow.

  -  **Generics vs. raw types in JSP scriptlets:** scriptlet code is real Java, so raw `Map`
      triggers unchecked-warning/raw-type lint just like in `.java` files. Parameterizing the
      declaration *and* the cast (`(Map<String,String>)`) lets you drop manual casts in the loop —
      type-safe and lint-clean.

  ### Challenges Overcome

  - **"Why doesn't it break in the UI?"** Initially confusing that the app looked fine. Learned the
    two distinct failure modes: a build-time `jspc` failure (`.keys()` undefined on `Map`) and a
    runtime `ClassCastException` on the `failEdit` re-render — neither visible from casual clicking.
  - **Coordinated refactor:** realized the swap is only safe if the action, the JSP (decl + cast +
    `.keys()` loop), and the test casts all move together; ran the unit tests after each step to keep
    the tree green throughout.
  - **Devcontainer disk-full hang:** a Docker/WSL virtual disk filling silently froze a build; fixed
    with `docker system prune -a --volumes -f`.
  - **Push authentication:** the first push was denied because the dev-container credential helper
    authenticated as a different GitHub account; resolved by pushing with the `gh`-authenticated
    credentials.

  ### What I'd Do Differently Next Time

  - Run the `jspc` precompile as a routine verification step earlier, not just at the end.
  - Apply generics to scriptlet locals from the start, not raw types — the bots flag raw `Map`
      the same as in Java code, which cost a review round.

  ---

  ## Resources Used

  - Project docs: `CONTRIBUTING.md` and `CLAUDE.md` (commit format, DCO, security/encoding standards, test conventions).
  - Issue #2215 thread and the maintainer's implementation pointers from @Ben-Heerema.
  - Java documentation for `java.util.Hashtable`, `HashMap`, and `Map`.
  - [Conventional Commits](https://www.conventionalcommits.org/) and the [Developer Certificate of 
  Origin](https://developercertificate.org/).
  - `jspc-maven-plugin` (used via the project's `jspc` Maven profile) for JSP precompilation.


---

