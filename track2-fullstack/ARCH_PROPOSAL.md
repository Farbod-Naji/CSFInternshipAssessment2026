# Architectural Proposal — Service + Repository Layer

Author: Farbod Naji
Date: 2026-05-16

---

## Problem

Route handlers in this codebase mix three distinct responsibilities: parsing HTTP
requests, enforcing business rules, and executing raw SQL — all in the same
function. This is not just an organisation problem; it has caused real bugs and
leaves the door open for more.

The clearest example is animal reassignment. When an animal moves to a new
paddock, three things must happen atomically:

1. Decrement the old paddock's `animal_count`
2. Increment the new paddock's `animal_count`
3. Update the animal's `paddock_id`

Currently these three steps are written inline inside `PUT /:id` with no
transaction wrapping them. If step 3 fails after steps 1 and 2 have already run,
the paddock counts are permanently corrupted with no rollback. This is the root
cause of the `animal_count` bugs identified in the audit — not just scattered
SQL, but unguarded business orchestration.

The same pattern appears in `POST /` (increments count with no transaction) and
`DELETE /:id` (decrements count with no transaction). Any route that touches
both animals and paddocks in the same operation is a latent data integrity risk.

A repository-only refactor would improve SQL organisation but would not solve
the core issue: multi-step business workflows would still be implemented in route
handlers without transactional guarantees.

---

## Proposed Structure

Introduce a minimal 3-layer architecture scoped to `animals` and `paddocks`:

```
backend/
  routes/
    animals.js          — HTTP only: parse request, call service, return response
    paddocks.js         — HTTP only
  services/
    animalService.js    — business rules, orchestration, transactions
    paddockService.js   — paddock-specific workflows (future scope); current
                          paddock orchestration is handled through animalService
  repositories/
    animalRepository.js — SQL only: animals, health events, weights
    paddockRepository.js — SQL only: paddocks
  db.js                 — unchanged
  server.js             — unchanged
```

### Layer responsibilities

**Routes** — read `req.params` and `req.body`, call one service function, map
the result or error to an HTTP response. No SQL, no business rules. All
endpoints that modify animal or paddock state (POST, PUT, DELETE) must call
service functions — direct repository access from routes is disallowed for
write operations.

**Services** — own all business logic. Validate inputs, decide when to update
paddock counts, and wrap multi-step operations in transactions. The only layer
allowed to call multiple repositories in one operation.

**Repositories** — execute SQL and return plain objects. No business decisions.
No knowledge of HTTP or services.

---

## The Money Function

The centerpiece of this refactor is `animalService.updateAnimal`. This is where
the `animal_count` orchestration problem is permanently solved:

```js
// services/animalService.js
function updateAnimal(id, updates) {
  const animal = animalRepo.findById(id);
  if (!animal) throw new NotFoundError('Animal not found');

  const next = {
    name:          updates.name          ?? animal.name,
    tag_number:    updates.tag_number    ?? animal.tag_number,
    breed:         updates.breed         ?? animal.breed,
    date_of_birth: updates.date_of_birth ?? animal.date_of_birth,
    paddock_id:    'paddock_id' in updates ? updates.paddock_id : animal.paddock_id,
  };

  // Wrap all three steps in a transaction — all succeed or all fail
  db.transaction(() => {
    if (next.paddock_id !== animal.paddock_id) {
      if (animal.paddock_id) paddockRepo.decrementCount(animal.paddock_id);
      if (next.paddock_id)   paddockRepo.incrementCount(next.paddock_id);
    }
    animalRepo.update(id, next);
  })();

  return animalRepo.findById(id);
}
```

The same transaction pattern applies to `createAnimal` (increment new paddock)
and `deleteAnimal` (decrement old paddock). Once this service exists, it is
structurally impossible to move an animal without both paddock counts updating
correctly — the logic has one owner and one place to break.

---

## What the Route Handler Becomes

```js
// routes/animals.js
router.put('/:id', (req, res) => {
  try {
    const updated = animalService.updateAnimal(req.params.id, req.body);
    res.json(updated);
  } catch (err) {
    if (err instanceof NotFoundError)   return res.status(404).json({ error: err.message });
    if (err instanceof ValidationError) return res.status(422).json({ error: err.message });
    throw err;
  }
});
```

The route has no knowledge of paddocks, counts, or SQL. It handles HTTP in and
HTTP out. The same error-mapping pattern applies to all other handlers.

---

## What This Fixes

- **Data integrity** — multi-step operations that touch both animals and paddocks
  are now wrapped in transactions. Partial failures no longer corrupt data.
- **`animal_count` correctness** — all paddock count logic lives in one service
  function. It cannot be partially implemented or forgotten by a future developer.
- **Testability** — service functions and repository functions can both be tested
  directly without an HTTP server.
- **`paddock_name` in responses** — `animalRepository.findById` uses a
  `LEFT JOIN` to return `paddock_name` alongside the animal, so no second API
  call is needed from the frontend.

---

## What This Does Not Change

- The SQLite schema is unchanged.
- All existing API endpoints, status codes, and response shapes are unchanged,
  with one intentional addition: the animal detail response gains a `paddock_name`
  field from the `LEFT JOIN` in `animalRepository.findById`.
- Health events and weight endpoints follow the same pattern but are not shown
  in full — the structure is identical.

---

## Migration Path

Each step is independently testable. The existing test suite acts as a safety net
throughout.

1. Create `repositories/animalRepository.js` and `repositories/paddockRepository.js`
   with all SQL extracted. Update routes to call repositories directly instead of
   `db`. Behavior remains unchanged at this point.
2. Create `services/animalService.js` and `services/paddockService.js` importing
   from repositories. Still no route changes beyond step 1.
3. Rewrite `routes/animals.js` to call services. Run `npm test`.
4. Rewrite `routes/paddocks.js`. Run `npm test`.
5. Delete any remaining direct `db` imports from route files.

---

## Effort Estimate

- Create repositories: ~1.5 hours
- Create services with transaction logic: ~1.5 hours
- Rewrite route handlers: ~45 minutes
- Update tests to cover service and repository functions directly: ~45 minutes

**Total:** approximately 4.5 hours for a developer familiar with the codebase.
