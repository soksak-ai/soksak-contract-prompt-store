# soksak-spec-plugin-prompt-store

A **prompt store**: a place to keep the text a model is given, addressed by what the text *is*
rather than by where it was put, and a way to assemble a finished prompt out of one such text plus
the values that vary.

This contract exists because a producer of prompts — a pipeline, a loop, an agent — must be able to
keep one copy of a template shared by a thousand nodes, and must be able to hand a model a prompt
that is *complete*. It pins this contract; the store declares it implements it. Neither side names
the other.

This is not a board, and a board is not this. A surface that shows issues as cards has no business
holding text a model will read; a store of addressed text has no business drawing anything. They are
separate contracts because they are separate jobs — a producer that needs both declares both, and an
implementer that offers both declares both. Folding one into the other would force every board ever
written to become a text store, which is a standard widened until it means nothing.

## Discovery

An implementer declares the contract in its manifest:

```json
{ "implements": ["soksak-spec-plugin-prompt-store"] }
```

A consumer discovers implementers by contract id alone:

```
sok plugin.implementers '{"contract":"soksak-spec-plugin-prompt-store"}'
```

and addresses whichever it finds as `plugin.<discovered id>.<command>`. A consumer that hard-codes an
implementer's id has not implemented this contract; it has merely used one store.

A consumer that also needs another contract from the *same* implementer — because the two are joined
by something it stores, such as a card that carries the address of its prompt — takes the
intersection of the two discoveries and refuses loudly when the intersection is empty. Picking an
implementer per contract independently would put the address in one place and the text it addresses
in another, and the deref would find nothing.

## The address

An address is derived from the content, never issued by a counter and never chosen by the consumer.

- **Same text, same address.** Storing a value that is already stored returns the same address and
  adds no second copy. This is what makes the store a store and not a log: a template shared by a
  thousand nodes exists once, and each node holds only its address.
- **The address is the store's.** A consumer must not construct or predict it, and must not assume
  an address minted by one store means anything in another. Ask the store; keep what it returns.
- **The address survives what the consumer forgets.** Two producers that store the same text
  independently converge on one entry — the store deduplicates by content, so neither has to know
  the other exists.

## Commands

An implementer exposes these. Names and shapes are the contract; how the text is kept, indexed, or
garbage-collected is the implementer's own business.

### `prompt.put`

```
prompt.put { value: string | object } → { hash: string }
```

Stores a value and returns its address. The value is text (a template, a directive) or a structure (a
schema the model must answer in). It is stored as what it is — a store that stringifies a structure
on the way in and hands back a string on the way out has lost the value and returned a rumour of it.

Storing is idempotent by content: `put(x)` twice yields one entry and one address.

### `prompt.get`

```
prompt.get { hash } → { value } | refusal
```

Reads a value back by address, in the shape it was stored in. An address the store does not have is a
refusal (`NOT_FOUND`), never a successful read of nothing: a caller handed `null` under an `ok` has
been told the lookup succeeded and the answer is empty, which is a different fact from "it is not
here" and leads to the wrong repair.

### `prompt.resolve`

```
prompt.resolve { hash, vars?: object, refs?: object } → { prompt: string } | refusal
```

Assembles the finished prompt: takes the template at `hash` and binds its `{{key}}` markers.

| binding | what it is for |
|---|---|
| `vars` | The small values that differ per prompt, given inline: `{ "item": "…" }`. |
| `refs` | The large values shared across prompts, given as **addresses**: `{ "directive": "<hash>" }`. The store dereferences each one and binds the text it finds. |

`refs` is why the store exists. A directive shared by every node in a run is stored once and named by
its address a thousand times; a consumer that inlined it into every node would have copied it a
thousand times and would have to rewrite all of them to change it once.

## Refusals

**A miss is a refusal, everywhere.** `get` and `resolve` on an address the store does not have refuse
with `NOT_FOUND` — and `resolve` refuses the same way on an address holding something that is not a
template. A store that answers a miss with a success and an empty value has reported that the lookup
worked, which sends the consumer looking for the wrong fault.

**A missing ref is a refusal, not a hole.** If any address in `refs` is not in the store, `resolve`
refuses with `NOT_FOUND` and names the key that failed. Binding the rest and shipping the remainder
would hand a model a prompt with a piece of its instructions missing — and the model would answer
anyway, plausibly and wrongly, which is the worst outcome available.

**An unbound marker is never quietly filled.** A `{{key}}` for which neither `vars` nor `refs` supplies
a binding is left standing, exactly as written. The store must not substitute the empty string for
it: an empty string is indistinguishable from a value the consumer meant to supply, whereas the
marker still says what is missing. A consumer that must not ship an unbound prompt can therefore see
that it is unbound.

## What an implementer must not do

- **Never mint an address that is not derived from the content.** A counter, a timestamp, or a random
  id would make `put` of the same text return two addresses and store two copies, which defeats the
  only reason to address by content.
- **Never mutate what an address points at.** An address names a value; if the value behind it can
  change, then a node that recorded the address recorded nothing. A new value is a new address.
- **Never require the consumer to know the store.** No implementer-specific field may be mandatory.

## How an implementer is judged

An implementer satisfies this contract when all of these hold:

1. `put` of the same value twice returns the same `hash`, and the store holds one copy.
2. `put` of a structure, then `get` of its address, returns the structure — not a string of it.
3. `resolve` binds `{{key}}` from `vars` inline.
4. `resolve` binds `{{key}}` from `refs` by dereferencing the address and substituting the text it
   finds — so that changing the shared value at its address changes every prompt that names it.
5. `get` of an unknown address refuses `NOT_FOUND` — it does not succeed with an empty value.
6. `resolve` on an unknown template address refuses `NOT_FOUND`.
7. `resolve` with a `refs` entry whose address is unknown refuses `NOT_FOUND` and names the key.
8. `resolve` leaves an unbound `{{key}}` standing, and never substitutes an empty string for it.
