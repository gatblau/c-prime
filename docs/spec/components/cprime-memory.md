# Component spec — cprime-memory

> **Spec set:** cprime
> **Status:** Approved (RFC 07 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit)
> **Last updated:** 2025-12-20

### SPEC: cprime-memory
**File:** `crates/cprime-memory/src/lib.rs` (modules `borrow.rs`, `liveness.rs`, `rc.rs`, `edges.rs`, `reuse.rs`) | **Package:** `cprime-memory` | **Phase:** 2 | **Dependencies:** `cprime-core`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose

`cprime-memory` makes C′-IR memory-safe without programmer annotations. It infers, per function parameter, whether the callee borrows or owns the argument (ADR-022), then inserts the reference-counting instructions `RcDup`, `RcDrop` and `DropReuse` into lowered IR (Perceus, ADR-002) so that every heap object is freed exactly once, after its last use, and memory of a dropped value is reused in place for a same-layout construction when its count is 1 (FBIP). Its output is the only IR the back ends (`cprime-codegen`) accept. The Phase 2 exit criterion — zero leaked objects across the test corpus — is a property of this crate's output.

#### Approach

The pass is a classic Perceus insertion over SSA with block arguments: backward liveness per block, then a forward walk that counts the references each instruction needs, then explicit edge code for values that die on one control-flow edge but not another, then a peephole FBIP pass that pairs a drop with a later same-layout construction in the same block. Borrow inference follows the Perceus borrowing extension: a parameter is borrowed unless some path consumes it, with one reuse-oriented exception (rule B3 below) so that FBIP can fire on recursive traversals. Alternatives rejected: (a) every parameter owned — simplest, kept as the fallback for unimplemented callees, but emits a dup/drop pair at most call sites; (b) dropping at the end of the defining block instead of after last use — frees memory later and defeats FBIP, which needs the drop before the construction.

#### Shared Context

Copied from `phase-2-architecture.md` §2C and §2E. Only the items this crate uses.

```rust
// cprime_core::ids
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct DefId(pub [u8; 32]);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct CapId(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)] pub struct Symbol(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash)] pub struct ValueId(pub u32);
#[derive(Clone, Copy, PartialEq, Eq, Hash)] pub struct BlockId(pub u32);

// cprime_core::ir
#[derive(Clone, PartialEq, Eq, Hash)]
pub enum Type {
    Unit, Bool, U8, U64, I64, Bytes, FixedBytes(u32), Vec(Box<Type>),
    Struct(DefId), Sum(DefId),
    Fn { params: Vec<Type>, ret: Box<Type>, effects: BTreeSet<CapId> },
    Token,
}
#[derive(Clone, PartialEq, Eq, Hash)]
pub enum Const { Unit, Bool(bool), U8(u8), U64(u64), I64(i64), Bytes(Vec<u8>) }
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub enum BinOp { Add, Sub, Mul, Div, Rem, WrappingAdd, WrappingSub, WrappingMul,
                 Eq, Ne, Lt, Le, Gt, Ge, And, Or, BytesConcat }
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub enum IntrinsicOp { U64Be, Push, Slice, ToBytes }
#[derive(Clone)]
pub enum Inst {
    Const(Const),
    Bin { op: BinOp, lhs: ValueId, rhs: ValueId },
    Not(ValueId),
    Call { func: DefId, args: Vec<ValueId> },
    Perform { cap: CapId, op: Symbol, args: Vec<ValueId> },
    Intrinsic { op: IntrinsicOp, args: Vec<ValueId> },
    MakeStruct { ty: DefId, fields: Vec<ValueId>, reuse: Option<ValueId> },
    GetField { value: ValueId, index: u32 },
    MakeVariant { ty: DefId, tag: u32, fields: Vec<ValueId>, reuse: Option<ValueId> },
    GetVariantField { value: ValueId, tag: u32, index: u32 },
    MakeVec { elem: Type, items: Vec<ValueId> },
    Len(ValueId),
    Index { vec: ValueId, index: ValueId },
    ParMap { func: DefId, xs: ValueId },
    RcDup(ValueId), RcDrop(ValueId), DropReuse(ValueId),
}
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub enum TrapKind { Overflow, DivisionByZero, IndexOutOfBounds, Unreachable }
#[derive(Clone)]
pub enum Terminator {
    Return(ValueId),
    Jump { target: BlockId, args: Vec<ValueId> },
    Branch { cond: ValueId, then_target: BlockId, then_args: Vec<ValueId>,
             else_target: BlockId, else_args: Vec<ValueId> },
    Switch { scrutinee: ValueId, cases: Vec<(BlockId, Vec<ValueId>)> },
    Trap(TrapKind),
}
#[derive(Clone)]
pub struct Block { pub params: Vec<(ValueId, Type)>, pub insts: Vec<(ValueId, Inst)>, pub term: Terminator }
#[derive(Clone)]
pub struct Function { pub def: DefId, pub ty: Type, pub entry: BlockId, pub blocks: Vec<Block>, pub value_count: u32,
                     pub node_map: BTreeMap<ValueId, NodeId> }  // (D3, D23) source node of every parameter, non-constant instruction, join parameter and Return value; filled by lowering; preserved by this crate
pub struct Layouts { pub structs: BTreeMap<DefId, Vec<Type>>, pub sums: BTreeMap<DefId, Vec<Vec<Type>>> }

// cprime_core::program
#[derive(Clone, PartialEq, Eq)] pub struct BorrowSig { pub borrowed: Vec<bool> }

// cprime_core::ir_verify (owned by cprime-core; signature from §2E)
pub fn verify_function(f: &Function, layouts: &Layouts, stage: IrStage) -> Result<(), IrVerifyError>;
pub enum IrStage { Lowered, RcInserted }
pub struct IrVerifyError { pub function: DefId, pub block: u32, pub inst: Option<u32>, pub rule: &'static str, pub message: String }   // D2
```

Runtime protocols binding on this crate (§2E, verbatim in substance):

- **Protocol 5 — ownership at boundaries.** Arguments to `Perform`, `Intrinsic`, `Len`, `Index`, `GetField`, `GetVariantField`, `Bin`, the vector `xs` of `ParMap` (D18) and runtime functions are borrowed. Fields of `MakeStruct`, `MakeVariant`, `MakeVec`, block arguments and the `Return` value are consumed (owned). `Call` arguments follow the callee's `BorrowSig`. Results of `Call`, `Perform`, `Intrinsic`, `Make*`, `ParMap` are owned; no reuse token is threaded through a `ParMap`. A heap-typed result of `GetField`, `GetVariantField` or `Index` is borrowed from its parent; `cprime-memory` makes it owned by inserting `RcDup` immediately after it.
- **Protocol 6 — reuse.** `DropReuse(x)` calls `cprime_rt_drop_reuse`: when `rc == 1` it drops `x`'s heap children and returns `x`'s address as the token; otherwise it decrements `rc` and returns 0. A `Make*` with `reuse: Some(t)` reuses the cell at `t` when `t != 0`, else allocates. The count-is-1 test runs inside `cprime_rt_drop_reuse`, at run time.
- **Protocol 2 — trap sites.** `Bin` `Add`/`Sub`/`Mul`/`Div`/`Rem`, `Index` and `Intrinsic Slice` are implicit trap sites; `Terminator::Trap(Unreachable)` is the only explicit trap.

Definitions used throughout this spec:

- **Heap-typed**: a value whose type is `Bytes`, `FixedBytes(_)`, `Vec(_)`, `Struct(_)` or `Sum(_)` (SA-25: every struct and sum is boxed). `Unit`, `Bool`, `U8`, `U64`, `I64` and `Token` are never counted. `Fn` never appears as a value type (SA-12).
- **Owned value**: a heap-typed value for which the current function holds one reference: every heap-typed instruction result except borrowed projections (which become owned through the inserted `RcDup`), every heap-typed non-entry block parameter, and every heap-typed entry parameter whose `BorrowSig` entry is `false`.
- **Borrowed value**: a heap-typed entry parameter whose `BorrowSig` entry is `true`. The function never drops it.
- **Layout of a value** (for FBIP): `(kind, field_count)` where kind is `struct` or `variant` and `field_count` is the number of payload words. Only struct values and variant values whose tag is statically known (rule R10) have a layout. `Bytes`, `FixedBytes` and `Vec` values have no layout for reuse (their payload size is dynamic or not constructed by `Make*`).

#### Public Interface

```rust
// crates/cprime-memory/src/lib.rs

/// Infers, for every function in `functions`, which parameters the callee borrows.
/// `fixed` gives the signatures of callees outside `functions` (accepted callees already
/// compiled, and unimplemented callees, which the caller supplies as all-owned).
/// Calls to a DefId found in neither map are treated as all-owned.
/// Deterministic: the result depends only on the inputs.
pub fn infer_borrows(
    functions: &BTreeMap<DefId, ir::Function>,
    fixed: &BTreeMap<DefId, BorrowSig>,
) -> BTreeMap<DefId, BorrowSig>;

/// Returns a copy of `f` with RcDup / RcDrop / DropReuse inserted and edges split where required.
/// `f` must be lowered IR (verifies at IrStage::Lowered) containing no RC instruction.
/// `borrows` must contain an entry for `f.def` and for every DefId that `f` calls.
/// The result verifies at IrStage::RcInserted; any failure is returned as IrVerifyError.
pub fn insert_rc(
    f: &ir::Function,
    layouts: &Layouts,
    borrows: &BTreeMap<DefId, BorrowSig>,
) -> Result<ir::Function, IrVerifyError>;

/// (D45, RFC 07) Options for differential testing of RC insertion. `insert_rc(f, l, b)` is exactly
/// `insert_rc_opts(f, l, b, RcOptions { reuse: true })`.
#[derive(Clone, Copy, PartialEq, Eq, Debug)]
pub struct RcOptions { pub reuse: bool }

/// As `insert_rc`; with `opts.reuse == false` Part 5 steps 18–20 are skipped, so the result contains no
/// DropReuse and every construction keeps `reuse: None`. Called only by the cprime-codegen test suite.
pub fn insert_rc_opts(
    f: &ir::Function,
    layouts: &Layouts,
    borrows: &BTreeMap<DefId, BorrowSig>,
    opts: RcOptions,
) -> Result<ir::Function, IrVerifyError>;
```

`BorrowSig.borrowed.len()` equals the function's parameter count. Entries for scalar parameters are `true` and carry no meaning (no RC applies to them).

Crate-internal items (`pub(crate)`) are not part of the contract: `Liveness`, `RcPlan`, `EdgeOps`, `ReusePairing`.

`IrVerifyError.rule` values produced by this crate itself (in addition to those produced by `verify_function`) — each is a `&'static str`; `function` = `f.def`; `block` and `inst` locate the offending instruction, or are `0` and `None` when the fault is not tied to one (missing or mis-sized `BorrowSig` for `f.def`):

| `rule` | Meaning |
|---|---|
| `"memory/input-has-rc"` | `f` already contains `RcDup`, `RcDrop`, `DropReuse`, a `reuse: Some(_)` or a `Token`-typed value |
| `"memory/missing-borrow-sig"` | `borrows` lacks `f.def` or a called DefId |
| `"memory/borrow-arity"` | a `BorrowSig` length differs from the parameter or argument count |
| `"memory/input-invalid"` | `verify_function(f, layouts, IrStage::Lowered)` failed; `message` carries the verifier's message |

##### Example

Input (lowered IR of the concept example `Auth::generate_token`, values `v0`–`v9`):

```text
fn Auth::generate_token(v0: U64, v1: U64, v2: Bytes) -> Struct(SessionToken)
b0(v0: U64, v1: U64, v2: Bytes):
  v3 = Const U64(3600)
  v4 = Bin Add v1 v3                       ; trap site (Overflow)
  v5 = Intrinsic U64Be [v0]
  v6 = Intrinsic U64Be [v4]
  v7 = Bin BytesConcat v5 v6
  v8 = Perform CryptoCap::hmac_sha256 [v2, v7]
  v9 = MakeStruct SessionToken [v0, v4, v8] reuse None
  Return v9
```

`infer_borrows({generate_token}, {})` → `{generate_token: BorrowSig { borrowed: [true, true, true] }}` (`v2` is only read by `Perform`, a borrowed position).

`insert_rc(...)` output:

```text
b0(v0: U64, v1: U64, v2: Bytes):
  v3  = Const U64(3600)
  v4  = Bin Add v1 v3
  v5  = Intrinsic U64Be [v0]
  v6  = Intrinsic U64Be [v4]
  v7  = Bin BytesConcat v5 v6
  v10 = RcDrop v5                          ; last use of v5 was v7
  v11 = RcDrop v6                          ; last use of v6 was v7
  v8  = Perform CryptoCap::hmac_sha256 [v2, v7]
  v12 = RcDrop v7                          ; last use of v7 was v8 (borrowed)
  v9  = MakeStruct SessionToken [v0, v4, v8] reuse None   ; v8 consumed, no RC
  Return v9                                ; v9 consumed
; v2 is borrowed: never dropped. value_count = 13.
```

Reuse example (sum-type list increment):

```text
; .cprime: enum List { Cons { head U64; tail List }; Nil }
;         spec inc(l List) List { ... }
b0(v0: Sum(List)):
  Switch v0 [b1(), b2()]
b1():
  v1 = GetVariantField v0 tag 0 index 0    ; U64
  v2 = GetVariantField v0 tag 0 index 1    ; List (borrowed projection)
  v3 = Const U64(1)
  v4 = Bin Add v1 v3
  v5 = Call inc [v2]
  v6 = MakeVariant List tag 0 [v4, v5] reuse None
  Return v6
b2():
  v7 = MakeVariant List tag 1 [] reuse None
  Return v7
```

`infer_borrows` → `inc: [false]` (owned, by rule B3: `v0` is a `Switch` scrutinee and arm `b1` constructs a `(variant, 2)` value, the layout of tag 0). `insert_rc` output:

```text
b0(v0: Sum(List)):
  Switch v0 [b1(), b2()]
b1():
  v1  = GetVariantField v0 tag 0 index 0
  v2  = GetVariantField v0 tag 0 index 1
  v8  = RcDup v2                           ; protocol 5: projection made owned
  v9  = DropReuse v0                       ; was RcDrop v0 (last use of v0); same layout as v6
  v3  = Const U64(1)
  v4  = Bin Add v1 v3
  v5  = Call inc [v2]                      ; owned argument: v2 transferred
  v6  = MakeVariant List tag 0 [v4, v5] reuse Some(v9)
  Return v6
b2():
  v10 = DropReuse v0                       ; edge drop at head of b2 (single predecessor), then paired
  v7  = MakeVariant List tag 1 [] reuse Some(v10)
  Return v7
```

At run time, when the incoming list cell has `rc == 1`, `cprime_rt_drop_reuse` decrements the tail's count (2 → 1, undoing the `RcDup`) and the new `Cons` is written into the old cell: the traversal allocates nothing.

#### Internal Logic

**Part 1 — `infer_borrows`**

1. **Initialise.** For every function `f` in `functions` (iterated in ascending `DefId` order), set every parameter to borrowed (`true`). Scalar parameters stay `true` permanently. No dependency called; no error possible; logs nothing.
2. **Classify parameter uses** for each heap-typed entry parameter `p` of `f` (rules B1–B3). `p` becomes owned (`false`) when any of these holds:
   - **B1 (consuming position).** `p` appears as a field of `MakeStruct`/`MakeVariant`/`MakeVec`, as a block argument of `Jump`/`Branch`/`Switch`, as the operand of `Return`, or as a `Call` argument at position *i* where the callee's current signature has `borrowed[i] == false`. The callee's current signature is read from the working map when the callee is in `functions`, else from `fixed`, else all-owned.
   - **B2 (flow into a block parameter).** Covered by B1: any block argument is consuming.
   - **B3 (reuse candidate).** `p` is the scrutinee of a `Switch` in `f`, and some case block *k* of that `Switch` contains a `MakeStruct`/`MakeVariant` whose layout equals the layout of variant *k* of `p`'s sum type (`(variant, field_count of tag k)`); or `p` has a struct type and `f` contains a `MakeStruct` with the same field count in a block where `p` is used. B3 exists so that FBIP (Part 4) can reuse `p`'s cell; a borrowed parameter can never be reused.
3. **Iterate to a fixed point.** Repeat step 2 over all functions in ascending `DefId` order until one full sweep changes no entry. Changes only go from `true` to `false`, so the loop ends after at most (total heap-typed parameter count + 1) sweeps. Recursive components (self- and mutual recursion) converge because a call to a function in the set reads its signature from the working map.
4. **Return** the map with one entry per function in `functions`. Entries of `fixed` are not copied into the result. Logs one `debug` event `memory.borrows` per function with fields `def` (64 hex), `borrowed` (array of booleans), `sweeps` (u32).

**Part 2 — `insert_rc`: preconditions and liveness**

5. **Check preconditions.** Call `cprime_core::ir_verify::verify_function(f, layouts, IrStage::Lowered)`; on failure return `IrVerifyError { rule: "memory/input-invalid", .. }`. Scan `f` for `RcDup`, `RcDrop`, `DropReuse`, `reuse: Some(_)` or any `Token`-typed block parameter; if found return rule `"memory/input-has-rc"`. Check `borrows` contains `f.def` with length equal to the entry block's parameter count, and contains every `Call.func` with length equal to that call's argument count; else return `"memory/missing-borrow-sig"` or `"memory/borrow-arity"`. Errors are logged at `error` level as event `memory.precondition` with fields `def`, `rule`.
6. **Compute value types.** Build `ty: Vec<Option<Type>>` indexed by `ValueId` from block parameters and instruction result types (result types follow the IR typing rules of cprime-core's verifier; RC instructions are absent at this point). Mark each heap-typed value as owned or borrowed per the Shared Context definitions.
7. **Compute liveness** of heap-typed values by backward data flow over blocks until stable: `live_out(B) = ⋃ live_in(S)` over successors `S` of `B`, excluding `S`'s block parameters; `live_in(B) = uses(B) ∪ (live_out(B) − defs(B))`, where uses include terminator operands (`Return` value, `Jump`/`Branch`/`Switch` arguments, the `Switch` scrutinee). Iterate blocks in reverse post-order of the reversed control-flow graph; represent sets as bit vectors of length `value_count`. A `Trap` terminator has no successors and no uses.

**Part 3 — `insert_rc`: reference counting per instruction**

8. **Walk each block forward** (blocks in index order, instructions in order). Maintain the per-block set `alive` of owned values the function currently holds a reference to (initialised from `live_in(B)` ∩ owned, plus the block's owned parameters; for the entry block, owned entry parameters).
9. **Count uses per instruction (rule R1).** For instruction `i` and each heap-typed operand `v`, let `o` = number of occurrences of `v` in consuming positions of `i` (protocol 5; `Call` positions via `borrows[func]`), `b` = number of occurrences in borrowed positions, and `after` = `v` is used by a later instruction of the block, by the terminator, or is in `live_out(B)`.
   - If `v` is **owned**: `hold = 1 if (b > 0 or after) else 0`; `need = o + hold`. Insert `need − 1` instructions `RcDup(v)` immediately before `i` when `need ≥ 2`. If `hold == 1` and not `after`, insert `RcDrop(v)` immediately after `i` (the reference kept alive across `i` for its borrowed use is released). If `o == 0`, `b > 0` and not `after`, this same rule drops `v` after `i`.
   - If `v` is **borrowed**: insert `o` instructions `RcDup(v)` before `i` (each consuming use needs a reference the function does not hold); never insert `RcDrop(v)`.
10. **Projections (rule R2, protocol 5).** For `GetField`, `GetVariantField` and `Index` whose result `r` is heap-typed, insert `RcDup(r)` immediately after the instruction; from then on `r` is owned. Rule R1 applies to the parent operand as a borrowed use, so the parent stays alive across the projection and its `RcDup`. Ordering of insertions after one instruction: the projection's `RcDup(r)` first, then any `RcDrop` of the parent produced by R1.
11. **Unused definitions (rule R3).** An owned value defined by an instruction or as an owned block parameter, with no later use and not in `live_out(B)`, is dropped immediately after its definition (for block parameters and owned entry parameters: at the head of the block, before the first instruction).
12. **Terminator uses (rule R4).** `Return(v)`: one consuming use; R1 applies with `after = false`. `Jump`: each argument is a consuming use on the single edge; R1 applies, with `after` true when `v` is also in `live_in(target)`. `Branch` and `Switch`: reference counts are placed per edge in Part 4 (step 14); the `Branch` condition is scalar; the `Switch` scrutinee is a borrowed use.
13. **Trap terminator and trap sites (rule R5).** A block ending in `Trap(_)` receives no drops for values alive at its end: the trap aborts the program (release) or the caller discards the run (interpreter, JIT, with `AllocTracker::free_all`). Trap-site instructions get no special cleanup: drops are placed after them as for any instruction, and they never execute when the trap fires.

**Part 4 — `insert_rc`: edges**

14. **Per-edge reference requirement (rule R6).** For a `Branch` or `Switch` in block `B` and each outgoing edge `e = (B → S, args)`, and each owned value `v` alive at the terminator: `req(v, e) = (occurrences of v in args) + (1 if v ∈ live_in(S) else 0)`. The function holds one reference. Edge operations for `e`: `req − 1` × `RcDup(v)` when `req ≥ 2`; one `RcDrop(v)` when `req == 0`. Borrowed values: `req` counts only argument occurrences, and that many `RcDup(v)` are emitted; no drop.
15. **Place edge operations (rule R7, edge-split strategy).**
   - If `S` has exactly one predecessor edge in the whole function (one predecessor block, and `B` names `S` once), insert the edge operations at the head of `S`, before its first instruction. Duplicating at the head of `S` is sound because nothing between the terminator and the head of `S` can free `v`.
   - Otherwise split the edge: append a new block `E` whose parameters are fresh values with the same types as `S`'s parameters, whose instructions are the edge operations, and whose terminator is `Jump { target: S, args: E's parameters }`; retarget edge `e` in `B`'s terminator to `E` with the original arguments. New blocks receive the next free `BlockId`s in the order the edges are visited (blocks ascending, edges in terminator order: `then`, `else`, then `Switch` cases by tag).
   - A `Jump` has one edge; its operations are emitted in `B` before the terminator (step 12), never split.
   - Fresh `ValueId`s for inserted instructions and new block parameters are allocated from `f.value_count` upwards in insertion order; the result's `value_count` is updated.
   - `node_map` (D3): every entry of the input is copied unchanged (no existing `ValueId` is renumbered); values created by this crate (`RcDup`, `RcDrop`, `DropReuse` results, split-edge block parameters) get no entry.
16. **Switch arms that bind payloads (rule R8).** The arm block reads payload fields with `GetVariantField` on the scrutinee (borrowed parent, step 10 duplicates heap fields). The scrutinee is `live_in` to every arm that reads it, so R1 drops it after its last use inside the arm; in arms that do not read it, R6/R7 drop it on the edge (head of the arm when it has a single predecessor, which the lowering of `match` guarantees for arm blocks).
17. **Verify.** Call `verify_function(&g, layouts, IrStage::RcInserted)` on the result of Parts 3–4. On failure return the error unchanged (it is a compiler bug; A-17 routes it to a daemon failure) and log `error` event `memory.verify_failed` with `def`, `rule`, `pass = "rc"`.

**Part 5 — `insert_rc`: FBIP reuse** (skipped, from step 18 to step 20, when called through `insert_rc_opts` with `reuse: false`; steps 21 and 22 still run, D45)

18. **Known tags (rule R10).** In a block that is a `Switch` case target for tag *k* with a single predecessor, the scrutinee's tag is statically *k* for that block. A sum value has a layout only inside such a block. Struct values always have a layout.
19. **Pair drops with constructions (rule R9).** For each block, scan instructions in order. For each `RcDrop(x)` at position *p* where `x` has a layout `L`, find the first `MakeStruct` (when `L.kind = struct`) or `MakeVariant` (when `L.kind = variant`) at position *q > p* in the same block whose `reuse` is `None`, whose field count equals `L.field_count`, and that is not already paired. If found, replace `RcDrop(x)` with `t = DropReuse(x)` (result type `Token`, fresh `ValueId`) and set that construction's `reuse` to `Some(t)`. Each drop pairs with at most one construction and each construction with at most one drop; pairing is greedy in drop order. Drops inserted on split edge blocks (step 15) are never paired, because those blocks contain no construction.
20. **Token rules.** A `Token` value is used exactly once, by the `reuse` field of the paired construction; it is never counted, never passed to a call, never a block argument. Field types of the reused cell may differ from the new value's (a different struct of equal field count); the runtime rewrites `len`, `tag`, `kind` and `ptr_mask` (protocol 6).
21. **Verify again.** `verify_function(&g, layouts, IrStage::RcInserted)`; on failure, act as in step 17 with `pass = "reuse"`. Return `Ok(g)`.
22. **Log** one `debug` event `memory.rc_inserted` with `def`, `dups` (u32), `drops` (u32), `reuses` (u32), `split_edges` (u32), `elapsed_us` (u64).

#### Data Model

N/A — no persistent storage. In-memory working structures, all dropped when `insert_rc` returns:

| Structure | Shape | Lifetime |
|---|---|---|
| `Liveness` | `live_in`, `live_out`: one bit vector of length `value_count` per block | one `insert_rc` call |
| `RcPlan` | per block: `Vec<(position, Inst)>` insertions before/after each instruction | one call |
| `EdgeOps` | `BTreeMap<(BlockId, edge index), Vec<Inst>>` | one call |
| `ReusePairing` | `Vec<(drop position, make position)>` per block | one call |
| Working borrow map | `BTreeMap<DefId, Vec<bool>>` | one `infer_borrows` call |

#### Error Table

| Condition | Status | Code | Response Body |
|---|---|---|---|
| Input IR fails the `Lowered` verifier | `Err(IrVerifyError)` | `memory/input-invalid` | `IrVerifyError { function, block, inst, rule: "memory/input-invalid", message: "<verifier message>" }` |
| Input already contains RC instructions, a reuse token or a `Token` value | `Err(IrVerifyError)` | `memory/input-has-rc` | `{ rule: "memory/input-has-rc", message: "RcDup found in block 3 before RC insertion" }` |
| `borrows` lacks `f.def` or a called DefId | `Err(IrVerifyError)` | `memory/missing-borrow-sig` | `{ rule: "memory/missing-borrow-sig", message: "no BorrowSig for callee 9f3c…" }` |
| A `BorrowSig` length differs from the parameter or argument count | `Err(IrVerifyError)` | `memory/borrow-arity` | `{ rule: "memory/borrow-arity", message: "BorrowSig has 2 entries, call passes 3 arguments" }` |
| Output fails the `RcInserted` verifier after Parts 3–4 or after Part 5 | `Err(IrVerifyError)` | rule from `verify_function` | verifier error unchanged; daemon reports JSON-RPC daemon failure (A-17) |
| `infer_borrows` meets a call to a DefId in neither map | not an error | — | callee treated as all-owned; logged at `debug` as `memory.borrows_unknown_callee` with `callee` |

#### Acceptance Criteria (Gherkin)

```gherkin
Feature: cprime-memory

  Scenario: Happy path — generate_token gets drops after last use and no RC on its borrowed parameter
    Given the lowered IR of Auth::generate_token from the Example
    And infer_borrows returned [true, true, true] for it
    When insert_rc is called
    Then the result contains exactly RcDrop v5 and RcDrop v6 after the BytesConcat, and RcDrop v7 after the Perform
    And the result contains no RcDup and no RcDrop of v2
    And verify_function at IrStage::RcInserted returns Ok

  Scenario: Happy path — FBIP reuse in a recursive list traversal
    Given the lowered IR of List inc from the Example
    When infer_borrows is called with inc alone
    Then inc's BorrowSig is [false]
    When insert_rc is called
    Then block b1 contains DropReuse v0 before MakeVariant tag 0 and that MakeVariant has reuse Some(token)
    And running the result through the interpreter on a 1000-element list with rc 1 performs 0 allocations for Cons cells

  Scenario: Edge — value consumed twice gets one RcDup
    Given a function whose owned value v is the field of two MakeStruct instructions
    When insert_rc is called
    Then exactly one RcDup v is inserted before the first MakeStruct
    And no RcDrop v is inserted

  Scenario: Edge — owned and borrowed use of the same value in one call
    Given a call f(v, v) where f's BorrowSig is [true, false] and v has no later use
    When insert_rc is called
    Then one RcDup v precedes the call and one RcDrop v follows it

  Scenario: Edge — unused owned parameter is dropped at entry
    Given a function with an owned Bytes parameter that is never used
    When insert_rc is called
    Then the first instruction of the entry block is RcDrop of that parameter

  Scenario: Edge — value dead on one branch is dropped at the head of that branch
    Given a Branch whose then-target has a single predecessor and does not use owned value v, and whose else-target uses v
    When insert_rc is called
    Then RcDrop v is the first instruction of the then-target
    And no RcDrop v is placed in the else-target before its last use

  Scenario: Edge — join block with two predecessors forces an edge split
    Given a Branch whose two targets are the same block J with two predecessors, and v is not live into J
    When insert_rc is called
    Then a new block E is appended containing RcDrop v and Jump to J with E's parameters
    And the Branch edge that needed the drop targets E

  Scenario: Edge — trap terminator receives no cleanup
    Given a block ending in Trap(Unreachable) while owned value v is alive
    When insert_rc is called
    Then that block contains no RcDrop v

  Scenario: Edge — mutual recursion converges deterministically
    Given functions even and odd that call each other and each stores its parameter in a MakeStruct on one path
    When infer_borrows is called twice with the same inputs
    Then both calls return identical maps in which both parameters are owned

  Scenario: Error path — input already contains RC instructions
    Given IR containing RcDup in block 0
    When insert_rc is called
    Then it returns Err with rule "memory/input-has-rc"

  Scenario: Error path — missing borrow signature for a callee
    Given IR that calls a DefId absent from borrows
    When insert_rc is called
    Then it returns Err with rule "memory/missing-borrow-sig"

  Scenario: Leak criterion — zero live objects across the corpus
    Given every corpus function after insert_rc and the interpreter with an AllocTracker
    When each function runs on its corpus inputs and the result is dropped with cprime_rt_drop
    Then AllocTracker.live_objects is 0 after every run that did not trap

  Scenario: Edge case — parallel map borrows its vector
    Given a function that computes (par_map Auth::half xs) and then returns len(xs) plus the first result
    When insert_rc runs
    Then no RcDup of xs is inserted for the ParMap
    And the ParMap result is dropped exactly once on every path

  Scenario: Edge case — node_map entries added by D23 survive RC insertion
    Given the lowered concept example whose node_map is { v0: 1, v1: 2, v2: 3, v4: 6, v5: 11, v6: 13, v7: 10, v8: 16, v9: 19 }
    When insert_rc runs
    Then the output node_map contains each of those nine entries unchanged
    And no RcDup, RcDrop or DropReuse value created by insert_rc has an entry
    And ir_verify::verify_function(.., IrStage::RcInserted) returns Ok

  Scenario: Edge case — reuse can be turned off for differential tests (D45)
    Given the list-map example of Part 5 whose RC-inserted output pairs two constructions with DropReuse tokens
    When insert_rc_opts runs with RcOptions { reuse: false }
    Then the output contains no DropReuse and every MakeVariant has reuse None
    And ir_verify::verify_function(.., IrStage::RcInserted) returns Ok
    And insert_rc_opts with RcOptions { reuse: true } returns the same function as insert_rc
```

#### Performance, Security, Observability

- **Performance targets:** `infer_borrows` + `insert_rc` together ≤ 3 ms p95 for one function of ≤ 200 IR instructions (part of the T1 ≤ 20 ms budget, HLD G5), measured by a criterion-style benchmark over the corpus. Liveness is O(blocks × value_count / 64) word operations per iteration. Memory ceiling: ≤ 2 MiB of working structures per call at 200 instructions.
- **Security:** no input from outside the compiler; inputs are compiler-produced IR. A defect here can cause use-after-free or leaks in compiled programs, so this crate is in the trusted computing base. Mitigations: verifier after every sub-pass (steps 17, 21), the leak harness (Phase 2 exit), interpreter/JIT/AOT differential tests that compare live-object counts (see `cprime-codegen`). No unsafe code in this crate.
- **Observability:** `debug` events `memory.borrows` (`def`, `borrowed`, `sweeps`), `memory.rc_inserted` (`def`, `dups`, `drops`, `reuses`, `split_edges`, `elapsed_us`), `memory.borrows_unknown_callee` (`callee`); `error` events `memory.precondition` (`def`, `rule`) and `memory.verify_failed` (`def`, `rule`, `pass`). No metrics; elapsed time contributes to the daemon's `timings_us.t1`.

#### Gaps

None.
