# FarmTracker — Code Audit

**Auditor:** Farbod Naji  
**Date:** 2026-05-15  
**Reviewed before any code changes.**

---

## Issues Found

### Backend Bugs Not Yet Frontend Exposed

**Bug 1: Paddock `animal_count` does not update correctly on reassignment** (`routes/animals.js`, `PUT /:id`)  
When an animal is moved to a new paddock, the new paddock count increases, but the old paddock count is not reduced. No edit UI exists yet, but any future reassignment feature or external API client would receive inaccurate paddock counts.

**Bug 2: `DELETE` can push `animal_count` below zero** (`routes/animals.js`)  
The delete route subtracts 1 from `animal_count` without checking whether the count is already zero or incorrect. This can create negative paddock counts if the stored value has already drifted.

---

### Frontend and API Bugs

**Bug 3: Pagination offset is incorrect** (`routes/animals.js`)  
`page` is used directly as the SQL `OFFSET` instead of being converted into a page offset. For example, page 1 skips 1 record instead of 10 when the limit is 10. This affects users browsing paginated animal lists.

**Bug 4: `POST /animals` returns 200 instead of 201** (`routes/animals.js`)  
Other create endpoints return `201 Created`, but this endpoint returns `200 OK`. This makes the API response inconsistent.

**Bug 5: Animal detail page shows raw `paddock_id` instead of the paddock name** (`frontend/animal-detail.html`)  
The animal list page shows readable paddock names, but the detail page only shows the numeric ID. This is immediately visible to users opening an animal record.

**Bug 6: Missing `animalId` is not handled on the detail page** (`frontend/animal-detail.html`)  
If the page is opened without an `?id=` query parameter, the page requests `/animals/null` and stays stuck on "Loading…". This should be handled with an early error message.

---

### UX Issues

**Bug 7: Event type dropdown has no placeholder** (`frontend/animal-detail.html`)  
"Vaccination" is selected by default, so the `required` attribute does not force the user to make a deliberate choice.

**Bug 8: Table text appears editable** (`frontend/styles.css`)  
Table rows allow text selection and can show a text cursor. `cursor: default` and `user-select: none` would make the table feel less misleading.

---

### Design Concern

The main structural concerns are the manually maintained `animal_count` value and the mixing of business logic, request handling, and raw SQL inside route files. These are discussed further in `ARCH_PROPOSAL.md`.

---

## Prioritisation

**Fix first:** Bugs 3 and 5, because they are visible in the current frontend and affect normal user flows.

**Fix next:** Bug 1, because the backend reassignment logic is already incorrect and would affect any future edit feature.

**Fix alongside:** Bugs 4 and 6, because both are small correctness fixes with clear solutions.

**Fix last:** Bug 2 and UX Bugs 7 and 8, because they are lower immediate risk.

**Out of scope:** Authentication, input sanitisation, and rate limiting are not covered in this pass.

**Deferred:** Broader architecture changes are covered in `ARCH_PROPOSAL.md`.