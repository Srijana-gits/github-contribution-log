# Contribution [1]: (codex assist) Review Hashtable error-map usage in AddEditDocument2Action #2215

**Contribution Number:** 1  
**Student:** Srijana  
**Issue:** https://github.com/carlos-emr/carlos/issues/2215
**Status:** Phase II Complete

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

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
