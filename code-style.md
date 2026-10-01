# Code Style: Write Code That Is a Pleasure to Read

Write code for the next person who reads it. That person must understand what the code does,
why it does it, and where its limits are, without running it or asking anyone.

These rules apply to all code you write or change, in any language and any kind of file.
The examples are Python, but they only illustrate the rule. Apply the idea, not the syntax.
When a rule conflicts with the existing style of a file or project, follow the project and mention the conflict.

---

## 1. Principles

1. **Clarity over cleverness.** Prefer the obvious version, even if it is a few lines longer.
2. **Explain the why, not the what.** The code says what happens. Documentation and comments say why,
   what the contract is, and what can go wrong.
3. **Be honest about limits.** If the code has a known weakness, a surprising edge case, or a
   trade-off, write it down next to the code. Do not hide it.
4. **Keep the shape flat.** Validate early, exit early, and keep the main path at the lowest
   indentation level.
5. **One name per concept.** Use the same name for the same thing everywhere in the codebase.
6. **Make the right use easy and the wrong use hard.** Design interfaces so a caller cannot
   easily misuse them, and validate at the boundary so inner code can trust its inputs.

---

## 2. Naming

- A name says what the thing **is** or what it **does**, at the level the reader needs.
- Operations (functions, methods, commands, jobs, endpoints) start with a verb that says what they
  return or do: `paginate_query`, `make_like_arg`, `get_config`, `init_debugging`.
- A name that describes a sequence tells the order: `_commit_fleet_then_main`.
- Values that are steps of one process carry their role in the name, so the steps line up:
  `count_query` → `count`, `slice_query` → `slice_result`, `port_str` → `port`.
- Booleans and flags read as options or conditions: `scalars`, `strip`, `is_locked`, `has_errors`.
- Replace magic numbers and magic strings with named constants. Put a constant in the narrowest
  scope that owns it (module, class, config).
- Mark internal parts as internal with the language's convention (for example a leading `_` in Python).
- Do not abbreviate unless the abbreviation is standard in the domain (`db`, `id`, `url`, `http`).
- Use the domain's words. If the business says "logger" and "shipment", the code says the same.
- Avoid unclear naming and unnecessary shortcuts. Prefer longer, but precise names. The goal is that person reading the code block can have no wide context and still understand it with it's naming.

---

## 3. Interfaces

An interface is anything other code or people depend on: function signatures, class APIs, HTTP endpoints,
CLI arguments, config keys, database schemas, message formats, file formats, environment variables.

- Required inputs come first. Options come after, with sensible defaults, and are named explicitly
  (keyword-only arguments, named flags, named fields), so a call site reads without a lookup.
- Declare types wherever the language or format allows it (annotations, schemas, column types).
- Put validation in the declaration when the tool supports it (field constraints, regex patterns,
  database constraints), so every reader of the value can trust it.
- Keep an interface small. Add a parameter, field, or flag only when a real caller needs it.
- Do not break an existing interface silently. If a change is breaking, say so in the documentation
  and in your summary.

```python
async def paginate_query(
    session: AsyncSession,
    query: Select,
    page: int,
    show: int,
    *,
    max_page_size: int = MAX_PAGE_SIZE,
    scalars: bool = True,
) -> PageDict:
```

---

## 4. Documentation

### 4.1 What to document

Document **every unit a reader can meet without context**. Examples:

- modules, packages, scripts, and files
- functions, methods, classes, and types
- constants, config values, environment variables, and feature flags
- database models, columns, migrations, and indexes
- endpoints, CLI commands, jobs, events, and messages
- infrastructure resources and deployment settings

Skip documentation only when the name and the signature already say everything
(for example a trivial getter or an obvious test helper).

Use the documentation format the language or tool expects (docstrings, JSDoc, XML doc comments,
schema descriptions, `description` fields in Terraform or OpenAPI, and so on).

### 4.2 Length matches weight

- A small, obvious unit gets one sentence:
  `"""Return the session `DualDBSessionMiddleware` stashed on the request state."""`
- A unit with a contract, edge cases, or a history gets as much text as the reader needs, and no more.

### 4.3 Order of content

1. **What it does or returns**, in one sentence, in the imperative.
   "Return a page out of a query's results."
2. **The contract:** what callers can rely on, what the output is compatible with.
   "The returned dict is compatible with the `Page[Foo]` pydantic model if ..."
3. **Special behaviour and edge cases**, in plain words.
   "... if it starts with a `"` then it's a prefix search ..."
4. **Known limits and surprises**, marked with `NOTE:` when the reader could get it wrong.
   "NOTE: This will let % and _ characters through which may be surprising to a user ..."
5. **What it does not guarantee.**
   "We can still end up with one of the transactions committed and the other rolled back since this
   isn't a proper distributed transaction setup."
6. **The history of a non-obvious decision**, if one exists. First person is fine.
   "I have not been able to get remote debugging to work in multiple processes ... So instead it is
   called in ``main.py`` ..."

### 4.4 Inputs and settings

Describe each input by **what the code does with it**, not by its type (the type is already declared).

```
:param session: Execute queries in this session
:param page: Return the page at this index (starting from 1)
:param max_page_size: Raise a 400 error if page size is larger than this value
```

When an option changes behaviour, say what each value does and why you would choose it.

### 4.5 Values and defaults

For any value with a default or a limit (config, constant, timeout, batch size, retry count):

- Say what the value controls.
- Say **why the default is this value** (measurement, experiment, external limit).
- Link the source of hard external limits.
- Say when to change it.
- Write the value in the form that shows its meaning (`6 * 1024`, `timedelta(minutes=5)`, not `6144` or `300`).

```python
bulk_action_chunk_size: int = Field(default=6 * 1024, le=32 * 1024)
"""
Size of chunks of loggers in bulk actions. The 6K value has worked well in testing. Making it larger,
we get warning about exceeding memory constraints for range optimization and execution slows down.

32K is a limit of PostgreSQL. See Bind section in https://www.postgresql.org/docs/11/protocol-message-formats.html
"""
```

---

## 5. Comments

- **Label the steps** of a longer block with a short comment: `# Load a slice`.
- **Group related declarations** under a short section comment: `# Subsystem Configurations`.
- **State invariants you rely on**, and where they come from:
  `# The config field regex guarantees that we can split and convert to int`
- **Explain cleanup, failure, and concurrency paths** in full sentences. Readers get these parts wrong most often.
  ```python
  # Closing rolls back any still-open transaction. If we committed above this is a no-op;
  # on a failed request it performs the rollback. Always close both — even if the first
  # raises — and let the original exception (if any) propagate.
  ```
- **TODOs have a trigger and a goal**, not only a wish: when to do it, and what we gain.
  ```python
  # TODO once we remove support for sqlalchemy 1, examine if we would get better typing with separate functions for
  #      the scalar and non-scalar cases. If the type variable in the Select can be propagated to the PageDict then
  #      we have gained quite a lot.
  ```
- **Disabled code** stays only when it records something you tried, with the result:
  `# patch_multiprocessing=True,  # This did not help with the multi-processing issues`
- Do not write comments that repeat the code.
- Keep comments true. When you change code, update or delete the comments about it.

---

## 6. Control flow and structure

- **Validate first.** Put guard clauses at the top and fail fast with a clear message.
- **Exit early** for the "nothing to do" case.
- **Name intermediate values** instead of nesting expressions.
- Declare the type of an intermediate value when it is not obvious from the expression.
- Prefer forms that read like the rule they express (for example `1 <= show <= max_page_size`).
- Extract a helper when a block has its own name and purpose, not only to make a unit shorter.
- Keep one level of abstraction per unit. Do not mix high-level steps with low-level details.
- Load optional or development-only dependencies only where they are used, so production does not need them.

```python
if page < 1:
    raise HTTPException(status_code=HTTP_400_BAD_REQUEST, detail="Page numbers start from 1")
if not 1 <= show <= max_page_size:
    raise HTTPException(status_code=HTTP_400_BAD_REQUEST, detail=f"Page size must be between 1 and {max_page_size}")
```

---

## 7. Errors, logging, and resources

- **User-facing errors state the rule**, not only the failure: "Page numbers start from 1".
- **Developer-facing messages suggest the probable cause and fix**:
  "Could not find `_db_session` in the request state. Did you make sure to add `DualDBSessionMiddleware`
  to your FastAPI application?"
- Include the values that help diagnose the problem (the key, the limit, the address), never secrets.
- Catch the specific error you expect. Do not catch everything.
- When you catch an error only to add context, log it and re-raise the original.
- Release resources in a guaranteed cleanup path (`finally`, `defer`, `using`, context managers).
  Make cleanup of several resources independent, so one failure does not skip the others.

---

## 8. Formatting

- Follow the project's formatter and linter. Do not fight their output.
- Maximum line length: 120 characters, unless the project sets another value.
  Wrap documentation and comments at the same limit.
- Use the modern syntax of the language version the project targets.

---

## 9. Checklist before you finish

1. Does every unit a reader can meet without context have documentation that matches its weight?
2. Does the documentation start with what the unit does, and state its contract and its limits?
3. Does every value with a default or a limit say why it has that value?
4. Does every comment explain why or state an invariant, and is every comment still true?
5. Are names consistent, in the domain's words, and free of unclear abbreviations?
