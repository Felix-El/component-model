# Flat ABI for Fixed and Bounded Length Lists

## Motivation

The current flat ABI for fixed length lists expands all N element slots
into the core function's parameter/result list.
For large N this causes parameter overflow (exceeding
`MAX_FLAT_PARAMS` or `MAX_FLAT_RESULTS`), forcing an early fallback to
memory passing for the entire argument list when even one or two lists
are large.

If the current approach was naively adopted for bounded length lists
(which are still theory at this time), expanded form would even waste
flat-register space on uninitialized elements for partially-populated lists.

A second, independent motivation is that fixed and bounded length lists
are, unlike tuples, indexed at runtime and registers in core WebAssembly
cannot be indexed efficiently. Therefore, the expanded-element flat representation
forces register/memory (de)composition on either guest, adding to overhead.

This proposal replaces the expanded-element flat representation with a
pointer-based one for *direct* fixed and bounded length lists in parameters —
lists which are eligible for flat representation (not nested, immediately or
transitively, in another `list`).

## Flat representation

| Type                     | Flat representation                                                                            |
| ------------------------ | ---------------------------------------------------------------------------------------------- |
| `list<T>` (unbounded)    | `[ptr, len]` — pointer to `len` contiguous elements + actual length (unchanged, for reference) |
| `list<T, N>` (fixed)     | `[ptr]` — one pointer to `N` contiguous elements                                               |
| `list<T, ..N>` (bounded) | `[ptr, len]` — pointer to `len` contiguous elements + actual length                            |

## Memory representation

| Type                     | Memory representation                                                                          |
| ------------------------ | ---------------------------------------------------------------------------------------------- |
| `list<T>` (unbounded)    | `[ptr, len]` — pointer to `len` contiguous elements + actual length (unchanged, for reference) |
| `list<T, N>` (fixed)     | `[e0, e1, ...]` — `N` contiguous elements                                                      |
| `list<T, ..N>` (bounded) | `[len*, e0, e1, ...]` — `N` contiguous elements, uninitialized beyond actual length            |

bounded actual *len*: smallest integer that can represent values up to (including) N

## Lowering and Lifting

### Parameters

Below explanation is for bounded length lists which carry a runtime length.
The flow is similar for fixed length lists - length just would not be passed
as it is statically known.

If all core params fit within `MAX_FLAT_PARAMS`, they are passed flat, otherwise
we fall back to a in-memory tuple representation (*param area*). This is *not new*.

1. Caller
   - assume list `L` is already constructed in guest memory as `L_ptr` and `L_len` (from API use)
   - if passing flat: pass `L_ptr` and `L_len` directly, classic unbounded `list` ownership semantics
   - if passing in memory: move list elements inplace into param area

2. Host, before call forwarded
   - if passing flat:
     - allocate twin list `L2` in the exporting guest (optimization: bulk alloc, cache, etc.)
     - move elements between backing memories (import -> export)
     - pass `L2_ptr` and `L2_len = L_len`, classic unbounded `list` ownership semantics
   - if passing in memory:
     - move elements from inbound param area to outbound param area

3. Callee
   - if passing flat: take `L2_ptr` and `L2_len` from core params
   - if passing in memory: read `L2_len` from param area, let `L2_ptr` point to first inplace element in param area
   - either way: construct list from `L2_ptr`, `L2_len` for API use

### Results

Memory-passing is enforced when even a single *direct* fixed/bounded length list is in the result type -
that's a *new* second switchover reason beyond exceeding `MAX_FLAT_RESULTS`.
A return area (*ret area*) is used to forward all results as an in-memory tuple.

1. Caller
   - result area is preallocated with enough room to hold all fixed/bounded length lists inline
   - result area pointer is passed as out core param

2. Host, after call forwarded
   - copy inplace elements from returned export-side ret area into import-side ret area provided by out param

3. Callee
   - construct the list by moving elements from ret area for API use

### Fused allocation

When multiple direct lists exist among parameters and passing flat, the host might service
their backing (export) memory needs from a single allocation as an optimization.

