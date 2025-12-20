# Component spec — cprime-syntax

> **Spec set:** cprime
> **Status:** Approved (RFC 07 amendments promoted by the user on 2025-12-20, after the Phase 6 re-audit)
> **Last updated:** 2025-12-20

### SPEC: cprime-syntax
**File:** `crates/cprime-syntax/src/lib.rs` (modules `lexer_cprime.rs`, `lexer_opq.rs`, `parse_cprime.rs`, `parse_opq.rs`, `span.rs`, `limits.rs`) | **Package:** `cprime-syntax` | **Phase:** 1 | **Dependencies:** `cprime-core` (types only: `ast`, `diag`, `ids`); third-party: `chumsky`

> **Mode:** greenfield
> **derivedFromHld:** 0.12.0

#### Purpose

`cprime-syntax` turns source text into the untyped AST defined in `cprime_core::ast`. It owns the normative concrete syntax of both surfaces: `.cprime` (contracts a person approves, ADR-046, in the Go-like syntax of ADR-050) and `.opq` (agent implementations). It reports every parse error in one pass, with spans, so an agent can fix a submission in one round trip. It assigns the pre-order node ids that every later diagnostic refers to. It performs no name resolution, no type checking and no I/O: both entry functions are pure functions of `(file, text)`.

#### Approach

Each surface is parsed in two `chumsky` stages: a lexer producing `(Token, Span)` pairs, then a token parser building the AST. Contract expressions use a precedence-climbing parser with one level per row of the precedence table below. Recovery uses `chumsky`'s recovery combinators: skip-until-anchor for `.cprime` items and contract clause lists, nested-delimiter recovery for `.opq` forms. Node ids are assigned by a separate pre-order walk over the finished `.opq` AST, not during parsing, so recovery cannot leave gaps or duplicates. Rejected alternative: a scannerless single-stage parser — keyword boundaries and comment skipping would be repeated in every rule, and spans for multi-character operators become harder to compute. Rejected alternative: assigning node ids during parsing — a recovered subtree would consume ids that never appear in the final tree. The `.cprime` lexer inserts a `TERM` token at each newline that follows a qualifying token (Go's semicolon-insertion rule), so the parser sees terminated statements and never inspects indentation. `.cprime` recovery has two levels: at item level, skip to an item keyword at column 1; inside a body, skip to the next `TERM` or the closing `}` at the same brace depth, so each faulty line yields its own diagnostic. Rejected alternative: indentation-significant bodies — indentation would carry meaning and a misplaced line could join the wrong declaration (ADR-050).

#### Shared Context

Copied from `phase-2-architecture.md` §2C. A reader implements from this section alone.

```rust
// ---------- cprime_core::ids ----------
/// Pre-order index of a node in the submitted .opq syntax tree; 0 is the `impl` form (SA-32).
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, serde::Serialize)]
#[serde(transparent)]
pub struct NodeId(pub u32);

// ---------- cprime_core::diag ----------
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)]
pub enum DiagnosticKind {
    ParseError, UnknownName, ArityMismatch, TypeMismatch, UnauthorizedEffect,
    MissingDecreases, NonExhaustiveMatch, UnconstrainedOutput, InvalidParMap,
    ContractViolation, ReachableTrap,
    NonDecreasing, VerificationInconclusive,
    NonDeterministicProvider,
    InvalidServer,
    VacuousContract, VacuityUnknown,                    // (D37) never produced by this crate
}
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "lowercase")]
pub enum Tier { T0, T1, T2, T3 }
#[derive(Clone, Copy, PartialEq, Eq, serde::Serialize)] #[serde(rename_all = "lowercase")]
pub enum Severity { Error, Note }                       // (D37) this crate produces Error only

#[derive(Clone, PartialEq, Eq, serde::Serialize)]
pub struct Span { pub file: String, pub line: u32, pub column: u32 }
// file: project-relative path with '/' separators, or "<submission>" for the submitted source; line/column 1-based, column in Unicode scalar values.

#[derive(Clone, serde::Serialize)]
pub struct Diagnostic {
    pub kind: DiagnosticKind,
    pub tier: Tier,
    pub severity: Severity,
    pub node: NodeId,
    #[serde(skip_serializing_if = "Option::is_none")] pub span: Option<Span>,
    pub message: String,                                            // one English sentence, no trailing full stop
    #[serde(skip_serializing_if = "Option::is_none")] pub clause: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")] pub counterexample: Option<Counterexample>,
    #[serde(skip_serializing_if = "Option::is_none")] pub reason: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")] pub witness: Option<WitnessTrace>,   // (D22) defined in cprime_core::diag
    #[serde(skip_serializing_if = "Option::is_none")] pub concrete: Option<ConcreteCheck>, // (D43) defined in cprime_core::diag; always None here
}
// cprime-syntax never sets clause, counterexample, reason or witness; `witness` is always None for its diagnostics.

// ---------- cprime_core::db ----------
pub struct ParsedModule { pub module: Option<ast::Module>, pub diagnostics: Vec<Diagnostic> }

// ---------- cprime_core::ast ----------
#[derive(Clone, PartialEq, Eq)] pub struct Ident { pub name: String, pub span: Span }
#[derive(Clone, PartialEq, Eq)] pub struct Path { pub segments: Vec<Ident>, pub span: Span }

pub struct Module { pub file: String, pub path: Path, pub items: Vec<Item> }
pub enum Item { Type(TypeDef), Capability(CapabilityDef), Spec(SpecDef), Entry(EntryDef), Bind(BindDef), Server(ServerDef) }

pub struct TypeDef { pub name: Ident, pub generics: Vec<Ident>, pub body: TypeDefBody, pub span: Span }
pub enum TypeDefBody { Alias(TypeExpr), Record(Vec<FieldDecl>), Sum(Vec<VariantDecl>) }
pub struct FieldDecl { pub name: Ident, pub ty: TypeExpr }
pub struct VariantDecl { pub name: Ident, pub fields: Vec<FieldDecl>, pub span: Span }
pub struct TypeExpr { pub path: Path, pub args: Vec<TypeArg>, pub span: Span }
pub enum TypeArg { Type(TypeExpr), Nat(u32, Span) }

pub struct CapabilityDef { pub name: Ident, pub ops: Vec<CapOpDecl>, pub span: Span }
pub struct CapOpDecl { pub deterministic: bool, pub name: Ident, pub params: Vec<Param>, pub ret: TypeExpr, pub span: Span }
pub struct Param { pub name: Ident, pub ty: TypeExpr }

pub struct SpecDef {
    pub name: Ident, pub params: Vec<Param>, pub ret: TypeExpr,
    pub uses: Vec<Path>,
    pub assurance: Option<(AssuranceLit, Span)>,
    pub unconstrained: Vec<WaiverItem>,
    pub requires: Vec<ContractExpr>,
    pub ensures: Option<(Ident, Vec<ContractExpr>)>,
    pub span: Span,
}
pub enum AssuranceLit { Proved, Tested }
pub enum WaiverItem { Field(Ident), VariantField(Ident, Ident), VariantTest(Span) }
pub struct EntryDef { pub name: Ident, pub params: Vec<Param>, pub ret: TypeExpr, pub uses: Vec<Path>, pub span: Span }
pub struct BindDef { pub capability: Path, pub provider: Path, pub span: Span }
pub struct ServerDef { pub name: Ident, pub handler: Path, pub transport: Path, pub span: Span }   // ADR-051; handler: spec path, transport: provider path

pub struct ContractExpr { pub span: Span, pub text: String, pub kind: ContractExprKind }
pub enum ContractExprKind {
    Int(i128), Bool(bool), Bytes(Vec<u8>), Unit,
    Path(Path),
    Call { callee: Path, args: Vec<ContractExpr> },
    Field { value: Box<ContractExpr>, name: Ident },
    Is { value: Box<ContractExpr>, variant: Ident },
    Unary { op: UnaryOp, operand: Box<ContractExpr> },
    Binary { op: BinaryOp, lhs: Box<ContractExpr>, rhs: Box<ContractExpr> },
}
pub enum UnaryOp { Not, Neg }
pub enum BinaryOp { Implies, Or, And, Eq, Ne, Lt, Le, Gt, Ge, Concat, Add, Sub, Mul, Div, Rem }

pub struct Impl {
    pub file: String,
    pub header: Option<String>,
    pub spec: Path,
    pub params: Vec<(Ident, NodeId)>,
    pub decreases: Option<Decreases>,
    pub body: Expr,
    pub node_count: u32,
}
pub struct Decreases { pub node: NodeId, pub measure: Vec<Expr> }
pub struct Expr { pub node: NodeId, pub span: Span, pub kind: ExprKind }
pub enum ExprKind {
    Int(i128), Bool(bool), Bytes(Vec<u8>), Unit,
    Name(Path),
    Let { bindings: Vec<(Ident, NodeId, Expr)>, body: Box<Expr> },
    If { cond: Box<Expr>, then_: Box<Expr>, else_: Box<Expr> },
    Match { scrutinee: Box<Expr>, arms: Vec<Arm> },
    Perform { op: Path, args: Vec<Expr> },
    Apply { head: Path, args: Vec<Expr> },
    Field { value: Box<Expr>, name: Ident },
    Vec { items: Vec<Expr> },
    ParMap { spec: Path, xs: Box<Expr> },
}
pub struct Arm { pub node: NodeId, pub variant: Path, pub binders: Vec<(Ident, NodeId)>, pub body: Expr }
```

Module/file convention (SA-11): module `A::B` lives in `<root>/a/b.cprime`; each path segment is converted from UpperCamelCase to snake_case. Implementations of module `A::B` live in `<root>/a/b/<spec>.opq`.

Configuration keys read: none.

#### Public Interface

```rust
/// Parse one .cprime module. `file` is the project-relative path (e.g. "auth.cprime").
/// `module` is Some only when `diagnostics` is empty.
pub fn parse_cprime(file: &str, text: &str) -> ParsedModule;

/// Parse one .opq implementation. `file` is the project-relative path (e.g. "auth/generate_token.opq")
/// or "<submission>" for source received over RPC.
/// The Impl is Some only when the Vec<Diagnostic> is empty.
pub fn parse_opq(file: &str, text: &str) -> (Option<ast::Impl>, Vec<Diagnostic>);

/// Limits, public for tests and for callers that pre-check sizes.
pub const MAX_SOURCE_BYTES: usize = 1_048_576;   // 1 MiB
pub const MAX_NESTING: u32 = 256;
pub const MAX_DIAGNOSTICS: usize = 100;

```

Callers (from §2E): `cprime-daemon` (implements the `Db::parse_module` hook with `parse_cprime`; parses submissions and on-disk `.opq` files with `parse_opq`), `cprime-cli` (release re-check).

Segment-to-file conversion is `cprime_core::paths::snake_case` (D1), which this crate calls and does not redefine: an underscore precedes an upper-case letter that follows a lower-case letter or digit, or that ends an acronym run before a lower-case letter; then every character is lower-cased. `Auth` → `auth`, `SessionStore` → `session_store`, `HTTPServer` → `http_server`, `HttpV2` → `http_v2`.

##### Normative lexical grammar (both surfaces)

Source is UTF-8 text (`&str`). Columns count Unicode scalar values; a tab is one column.

```ebnf
(* shared *)
letter        = "A".."Z" | "a".."z" ;
digit         = "0".."9" ;
hexdigit      = digit | "a".."f" | "A".."F" ;
IDENT         = ( letter | "_" ) { letter | digit | "_" } ;      (* not a reserved word of the surface *)
UPPER_IDENT   = "A".."Z" { letter | digit } ;                    (* module path segments only *)
INT           = digit { digit } ;                                (* no separators, no leading "+" *)
whitespace    = " " | "\t" | "\r" | "\n" ;

(* .cprime only *)
CP_BYTES      = "x\"" { hexdigit hexdigit } "\"" ;              (* even count, zero allowed: x"" is empty Bytes *)
TERM          = ";" | newline after a qualifying token ;
cprime_comment = "//" { any character except "\n" } ;

(* .opq only *)
NEG_INT       = "-" INT ;                                        (* "-" immediately followed by a digit *)
BYTES         = "0x" { hexdigit hexdigit } ;                     (* even count, zero allowed: "0x" is empty Bytes *)
opq_comment   = ";" { any character except "\n" } ;
```

`TERM` insertion (`.cprime`, ADR-050): a newline produces a `TERM` token when the last token on its line, ignoring comments, is an `IDENT`, a path, an `INT`, a `CP_BYTES`, `true`, `false`, `proved`, `tested`, `variant`, `)`, `]` or `}`. A line that ends in any other token (an operator, `,`, `(`, `[`, `{`, `==>`, `.`) continues onto the next line. Blank lines and comment-only lines produce no `TERM`. A written `;` is a `TERM`. The rule applies inside parentheses too, so a call broken across lines ends the broken line with `,`, as in Go.

Literal ranges checked by the lexer: `INT` value ≤ 18,446,744,073,709,551,615 (`u64::MAX`); `NEG_INT` magnitude ≤ 9,223,372,036,854,775,808. A `BYTES` or `CP_BYTES` token with an odd number of hex digits is a `ParseError` (E-HEX). In `.cprime`, `0x` is E-BYTES-0X. A `0x` prefix is recognised only when the character before it is not an identifier character. Paths are written without whitespace around `::`.

Reserved words, `.cprime`: `module type struct enum capability deterministic spec uses assurance unconstrained requires ensures entry server bind is true false`. The words `proved`, `tested` and `variant` are contextual (keywords only after `assurance` and inside an `unconstrained` line respectively), and so are `handler` and `transport` (keywords only at the start of a line in a `server` body). `fn` is not reserved in `.cprime` (ADR-050); `fn` directly before an operation name in a capability body is E-FN. `result` is not reserved, but a parameter named `result` is E-RESULT-PARAM.

Reserved words, `.opq`: `impl decreases let if match perform vec par_map and or not true false` (`par_map`: D14, added by RFC 02).

Future-reserved words (SA-49), `.cprime`: `use priv interface impl func`; `.opq`: `fn loop invariant`. The lexer emits them as reserved tokens; any occurrence is E-RESERVED, recorded at the token's span, after which parsing continues as if the token were an identifier so that later errors are still reported. Reserved type names (SA-49): `U16 U32 U128 I8 I16 I32 I128 F32 F64 Str Char`. A `type`, `capability` or generic parameter declared with one of these names is E-RESERVED-TYPE; references to them are not checked here (they resolve to `UnknownName` in `cprime-types`).

A leading U+FEFF (byte-order mark) is a `ParseError` on both surfaces.

##### Normative grammar — `.cprime`

```ebnf
file        = { TERM } "module" module_path TERM { { TERM } item } { TERM } EOF ;
module_path = UPPER_IDENT { "::" UPPER_IDENT } ;
item        = alias | struct | enum | capability | spec | entry | server | bind ;   (* every item keyword starts its line: column 1 *)

alias       = "type" IDENT [ generics ] "=" type_expr TERM ;
struct      = "struct" IDENT [ generics ] fields TERM ;
enum        = "enum" IDENT [ generics ] "{" stmts(variant) "}" TERM ;   (* at least two variants *)
fields      = "{" stmts(field) "}" ;
field       = IDENT { "," IDENT } type_expr ;                            (* names sharing one type *)
variant     = IDENT [ fields ] ;
generics    = "[" IDENT { "," IDENT } "]" ;
type_expr   = path [ "[" type_arg { "," type_arg } "]" ] ;
type_arg    = type_expr | INT ;                                         (* INT → TypeArg::Nat, must fit u32 *)

capability  = "capability" IDENT "{" stmts(cap_op) "}" TERM ;
cap_op      = [ "deterministic" ] IDENT "(" [ params ] ")" [ type_expr ] ;

spec        = "spec" IDENT "(" [ params ] ")" [ type_expr ] "{" stmts(clause) "}" TERM ;
clause      = "uses" path { "," path }
            | "assurance" ( "proved" | "tested" )
            | "unconstrained" waiver { "," waiver }
            | "requires" cexpr
            | "ensures" cexpr ;
waiver      = "variant" | IDENT "." IDENT | IDENT ;                     (* VariantTest | VariantField | Field *)

entry       = "entry" IDENT "(" [ params ] ")" [ type_expr ] "{" stmts(uses_line) "}" TERM ;   (* at most one uses line *)
uses_line   = "uses" path { "," path } ;
bind        = "bind" path "=" path TERM ;
server      = "server" IDENT "{" stmts(server_line) "}" TERM ;   (* ADR-051: each line exactly once *)
server_line = "handler" path | "transport" path ;

params      = group { "," group } [ "," ] ;
group       = IDENT { "," IDENT } type_expr ;                            (* a, b U64 *)
path        = IDENT { "::" IDENT } ;
stmts(X)    = { TERM } [ X { TERM { TERM } X } { TERM } ] ;
```

Mapping to the AST (unchanged, ADR-050): `alias` → `TypeDefBody::Alias`, `struct` → `Record`, `enum` → `Sum`. A `field` or `group` with *n* names yields *n* `FieldDecl`/`Param` entries in source order, each with a clone of the shared `TypeExpr`. An absent return type is `TypeExpr { path: Unit, args: [], span: <span of ")"> }`. `requires` lines become `SpecDef.requires` in order. `ensures` lines become `SpecDef.ensures = Some((Ident { name: "result", span: <span of the first ensures keyword> }, [...]))`; with no `ensures` line, `ensures` is `None`. An absent `assurance` line is `None`, which canonical serialisation encodes as `proved` (`cprime-core`). An entry with no `uses` line has `uses: []`. A `server` becomes `Item::Server(ServerDef { name, handler, transport, span })`, `span` at the `server` keyword (D31).

Clause rules: the clause kinds appear in the order `uses, assurance, unconstrained, requires, ensures`; `uses`, `assurance` and `unconstrained` appear at most once; all `requires` lines are consecutive and all `ensures` lines are consecutive. A violation is E-CLAUSE-ORDER naming the clause. A spec or entry parameter named `result` is E-RESULT-PARAM. A spec or entry header not followed by `{` on the same line is E-BODY.

##### Contract expressions (`cexpr`) — precedence and associativity

| Level (low → high) | Operators / forms | Associativity | AST |
|---|---|---|---|
| 1 | `==>` | right | `Binary { op: Implies }` |
| 2 | `\|\|` | left | `Binary { op: Or }` |
| 3 | `&&` | left | `Binary { op: And }` |
| 4 | `==` `!=` `<` `<=` `>` `>=`, `e is V` | non-associative | `Binary { Eq … Ge }`, `Is` |
| 5 | `++` | left | `Binary { op: Concat }` |
| 6 | `+` `-` | left | `Binary { op: Add / Sub }` |
| 7 | `*` `/` `%` | left | `Binary { op: Mul / Div / Rem }` |
| 8 | prefix `!` `-` | right (prefix) | `Unary { op: Not / Neg }` |
| 9 | postfix `.IDENT` | left | `Field` |
| 10 | primary | — | literal, path, call, parenthesised |

```ebnf
cexpr       = implies ;
implies     = or_expr [ "==>" implies ] ;
or_expr     = and_expr { "||" and_expr } ;
and_expr    = cmp_expr { "&&" cmp_expr } ;
cmp_expr    = cat_expr [ cmp_op cat_expr | "is" IDENT ] ;       (* at most one comparison or test *)
cmp_op      = "==" | "!=" | "<" | "<=" | ">" | ">=" ;
cat_expr    = add_expr { "++" add_expr } ;
add_expr    = mul_expr { ( "+" | "-" ) mul_expr } ;
mul_expr    = unary { ( "*" | "/" | "%" ) unary } ;
unary       = ( "!" | "-" ) unary | postfix ;
postfix     = primary { "." IDENT } ;
primary     = INT | CP_BYTES | "true" | "false" | "(" ")"
            | path "(" [ cexpr { "," cexpr } [ "," ] ] ")"         (* Call *)
            | path                                                  (* Path *)
            | "(" cexpr ")" ;
```

A `TERM` ends a contract expression. In `.cprime` every integer literal is non-negative; `-5` is `Unary { Neg, Int(5) }`. `e.V.f` (variant-field access, SA-19) parses as two nested `Field` nodes; `cprime-types` decides which segment is a variant. `!result is Ok` parses as `(!result) is Ok`; the variant test is negated as `!(result is Ok)`. A chain such as `a < b < c` or `a is V == b` is a `ParseError`. Parentheses do not create an AST node; the inner node keeps its own span and the outer `text` covers the parentheses only for the enclosing node.

`ContractExpr.text` is the exact source slice from the first character of the node to its last character, with no normalisation; for a clause it excludes the clause keyword. `ContractExpr.span` is the position of the first character.

##### Normative grammar — `.opq`

```ebnf
file        = [ header_line ] impl EOF ;                (* exactly one impl; comments and whitespace around it *)
header_line = ";; cprime-accepted" { any character except "\n" } "\n" ;   (* only as the very first line *)
impl        = "(" "impl" path "[" { IDENT } "]" [ decreases ] expr ")" ;
decreases   = "(" "decreases" ( expr | "[" expr { expr } "]" ) ")" ;

expr        = INT | NEG_INT | BYTES | "true" | "false" | "(" ")"        (* Int, Int, Bytes, Bool, Bool, Unit *)
            | path                                                      (* Name *)
            | "(" "let" "[" binding { binding } "]" expr ")"
            | "(" "if" expr expr expr ")"
            | "(" "match" expr arm { arm } ")"
            | "(" "perform" path { expr } ")"
            | "(" "." expr IDENT ")"                                    (* Field *)
            | "(" "vec" { expr } ")"
            | "(" "par_map" path expr ")"                              (* ParMap, D14 *)
            | "(" op_head { expr } ")"                                  (* Apply, head is the operator *)
            | "(" path { expr } ")" ;                                   (* Apply: prelude fn, spec call, constructor *)
binding     = IDENT expr ;
arm         = "[" ( "(" path { IDENT } ")" | path ) expr "]" ;
op_head     = "+" | "-" | "*" | "/" | "%" | "==" | "!=" | "<" | "<=" | ">" | ">=" | "++"
            | "and" | "or" | "not" | "wrapping_add" | "wrapping_sub" | "wrapping_mul" ;
```

Operator heads become `Apply { head }` where `head` is a one-segment `Path` whose `name` is the operator text (`"+"`, `"and"`, `"wrapping_add"`). Arity is not checked here (`cprime-types` reports `ArityMismatch`/`TypeMismatch`). `(f)` is a zero-argument `Apply`; `()` is the `Unit` literal. `[(None) body]` and `[None body]` are the same arm with no binders. There is no wildcard arm. A `-` followed immediately by a digit is a `NEG_INT`; `(- a b)` is subtraction.

The `impl` path is the fully qualified spec path (`Auth::generate_token`). When `file` is not `"<submission>"`, the path must agree with the file: the snake_case conversion of every path segment except the last, joined by `/`, followed by `/<last segment>.opq`, must equal `file`.

##### Node id numbering (SA-32)

Ids are assigned by one pre-order walk after a successful parse, starting at 0:

1. the `impl` form;
2. each parameter binder, left to right;
3. the `decreases` form (when present), then each measure expression subtree, left to right;
4. the body expression subtree.

Within an expression subtree the expression itself is numbered first, then:

- `Let`: for each binding in order — the binder, then the value subtree; then the body subtree;
- `If`: condition, then-branch, else-branch subtrees;
- `Match`: scrutinee subtree; then for each arm — the arm, each arm binder in order, the arm body subtree;
- `Perform`, `Apply`, `Vec`: argument subtrees in order (the head path gets no id);
- `ParMap`: the `xs` subtree (the spec path gets no id);
- `Field`: the value subtree (the field name gets no id);
- literals and `Name`: no children.

`Impl.node_count` is the number of ids assigned. Paths, identifiers, the header line and comments receive no id.

##### Example — concept example

`auth.cprime`:

```text
module Auth

capability CryptoCap {
    deterministic hmac_sha256(key, msg Bytes) Hash
}

struct SessionToken {
    user_id    U64
    expires_at U64
    sig        Hash
}

spec generate_token(user_id, now U64, secret Bytes) SessionToken {
    uses CryptoCap
    requires user_id > 0
    requires now <= U64::MAX - 3600
    requires len(secret) >= 32
    ensures result.user_id == user_id
    ensures result.expires_at == now + 3600
    ensures result.sig == CryptoCap::hmac_sha256(secret, u64_be(user_id) ++ u64_be(result.expires_at))
}
```

`parse_cprime("auth.cprime", text)` returns `ParsedModule { diagnostics: [], module: Some(m) }` with (spans abbreviated as `L:C`):

```text
Module { file: "auth.cprime", path: [Auth]@1:8, items: [
  Capability { name: CryptoCap@3:12, ops: [
    CapOpDecl { deterministic: true, name: hmac_sha256, params: [key: Bytes, msg: Bytes], ret: Hash } ] },
  Type { name: SessionToken@7:8, generics: [], body: Record([user_id: U64, expires_at: U64, sig: Hash]) },
  Spec { name: generate_token@13:6,
    params: [user_id: U64, now: U64, secret: Bytes], ret: SessionToken,
    uses: [CryptoCap], assurance: None, unconstrained: [],
    requires: [
      Binary{Gt, Path(user_id), Int(0)}                       text "user_id > 0",
      Binary{Le, Path(now), Binary{Sub, Path(U64::MAX), Int(3600)}}   text "now <= U64::MAX - 3600",
      Binary{Ge, Call{len, [Path(secret)]}, Int(32)}           text "len(secret) >= 32" ],
    ensures: Some((result@18:5, [
      Binary{Eq, Field{Path(result), user_id}, Path(user_id)},
      Binary{Eq, Field{Path(result), expires_at}, Binary{Add, Path(now), Int(3600)}},
      Binary{Eq, Field{Path(result), sig},
             Call{CryptoCap::hmac_sha256, [Path(secret),
                  Binary{Concat, Call{u64_be,[Path(user_id)]}, Call{u64_be,[Field{Path(result),expires_at}]}}]}} ])) } ] }
```

The dump is the AST of the RFC 04 revision of this example (old syntax) with three differences: spans; `assurance: None` where the old example wrote `assurance proved` (canonical serialisation encodes both as `proved`, so the `DefId` is the same, `cprime-core` step 6); and the `ensures` binder `result`, synthesised at the first `ensures` keyword. The shared-type groups `key, msg Bytes` and `user_id, now U64` expand to one `Param` per name.

`auth/generate_token.opq`:

```lisp
;; implements Auth::generate_token
(impl Auth::generate_token [uid now secret]
  (let [exp (+ now 3600)
        msg (++ (u64_be uid) (u64_be exp))
        sig (perform CryptoCap::hmac_sha256 secret msg)]
    (SessionToken uid exp sig)))
```

`parse_opq("auth/generate_token.opq", text)` returns `(Some(impl), [])`; `header` is `None` (the first line is a comment but not an acceptance header). Node ids:

| Id | Node | Id | Node |
|---|---|---|---|
| 0 | `impl` form | 12 | `Name uid` |
| 1 | binder `uid` | 13 | `Apply u64_be` |
| 2 | binder `now` | 14 | `Name exp` |
| 3 | binder `secret` | 15 | binder `sig` |
| 4 | `Let` | 16 | `Perform CryptoCap::hmac_sha256` |
| 5 | binder `exp` | 17 | `Name secret` |
| 6 | `Apply +` | 18 | `Name msg` |
| 7 | `Name now` | 19 | `Apply SessionToken` |
| 8 | `Int 3600` | 20 | `Name uid` |
| 9 | binder `msg` | 21 | `Name exp` |
| 10 | `Apply ++` | 22 | `Name sig` |
| 11 | `Apply u64_be` | | |

`node_count` = 23. A verifier diagnostic about `(+ now 1000)` in the faulty variant of this file carries `"node": 6`.

##### Example — sum type, variant tests and `match`

`geo.cprime`:

```text
module Geo

enum Shape {
    Circle { r U64 }
    Square { side U64 }
}

spec area(s Shape) U64 {
    requires s is Circle ==> s.Circle.r <= 1000
    requires s is Square ==> s.Square.side <= 1000
    ensures s is Square ==> result == s.Square.side * s.Square.side
}
```

The first `requires` clause parses as `Binary{Implies, Is{Path(s), Circle}, Binary{Le, Field{Field{Path(s), Circle}, r}, Int(1000)}}`.

`geo/area.opq`:

```lisp
(impl Geo::area [s]
  (match s
    [(Circle r) (* 3 (* r r))]
    [(Square side) (* side side)]))
```

Node ids: 0 `impl`, 1 binder `s`, 2 `Match`, 3 `Name s`, 4 arm `Circle`, 5 binder `r`, 6 `Apply *`, 7 `Int 3`, 8 `Apply *`, 9 `Name r`, 10 `Name r`, 11 arm `Square`, 12 binder `side`, 13 `Apply *`, 14 `Name side`, 15 `Name side`; `node_count` = 16.

##### Example — server item (ADR-051)

`api.cprime`:

```text
module Api

struct Request {
    method  Bytes
    path    Bytes
    query   Bytes
    headers Vec[Bytes]
    body    Bytes
}

struct Response {
    status  U64
    headers Vec[Bytes]
    body    Bytes
}

spec handle(req Request) Response {
    uses Log
    ensures req.path == x"2f6865616c7468" ==> result.status == 200
}

server Api {
    handler Api::handle
    transport acme::HttpTransport
}

bind Log = host::Log
```

`parse_cprime("api.cprime", text)` returns, among the items, `Server(ServerDef { name: Api@24:8, handler: [Api, handle], transport: [acme, HttpTransport], span: 24:1 })`. The request and response field lists must equal the transport's records; that check belongs to `cprime-effects` (SA-65).

##### Example — parse error

Input to `parse_opq("<submission>", …)`:

```lisp
(impl Auth::generate_token [uid now secret]
  (let [exp (+ now 3600]
    exp))
```

Result: `(None, [d])` where `d` serialises as:

```json
{"kind":"ParseError","tier":"t0","severity":"error","node":0,
 "span":{"file":"<submission>","line":2,"column":24},
 "message":"expected `)` to close `(` opened at 2:13, found `]`"}
```

#### Internal Logic

`parse_cprime(file, text)`:

1. **Size check.** If `text.len() > MAX_SOURCE_BYTES`, return `ParsedModule { module: None, diagnostics: [E-SIZE] }` without lexing. No dependency. Logged: nothing (the crate does not log).
2. **BOM check.** If `text` starts with U+FEFF, record E-BOM at 1:1 and continue lexing from the next character so further errors are still reported.
3. **Lex.** Run the `.cprime` lexer (`chumsky` stage 1) producing `Vec<(Token, ByteRange)>`. Comments and whitespace are dropped; `TERM` tokens are inserted at newlines by the rule of the lexical grammar, and a written `;` is a `TERM`. Lexer errors: E-CHAR (unexpected character), E-INT (literal above `u64::MAX`), E-HEX (odd hex digit count in `x"…"`), E-BYTES-0X (`0x` literal). On a lexer error the lexer skips the offending character, or the whole `0x…` literal, and continues.
4. **Map byte ranges to spans.** Build a line-start index once; convert each byte offset to `(line, column)` by binary search on line starts and counting Unicode scalars from the line start.
5. **Parse header.** Expect `module module_path TERM`. Missing → E-MODULE-MISSING at the first token. A segment that is not `UPPER_IDENT` → E-MODULE-SEGMENT.
6. **Check module path against file.** Compute `expected = segments.map(cprime_core::paths::snake_case).join("/") + ".cprime"`. If `expected != file`, record E-MODULE-FILE. Parsing continues.
7. **Parse items.** Repeat until end of input: dispatch on the item keyword (`type struct enum capability spec entry server bind`). A declared type, capability or generic parameter name in the reserved type names → E-RESERVED-TYPE. A second `module` keyword → E-MODULE-DUP. In a capability body, `fn` followed by an `IDENT` and `(` → E-FN, then parse the rest of the line as the operation. After a spec or entry header, a token other than `{` → E-BODY. Expand each shared-type `group` and `field` into one entry per name. A parameter named `result` → E-RESULT-PARAM. Inside a spec body, track the clause kinds seen; a kind out of the order `uses, assurance, unconstrained, requires, ensures`, a second `uses`/`assurance`/`unconstrained` line, or a `requires`/`ensures` line separated from the previous one of its kind → E-CLAUSE-ORDER. Contract clauses are parsed with the precedence table; a second comparison at level 4 → E-NONASSOC. Synthesise the `Unit` return type and the `result` binder as described under the grammar. In a `server` body, a line that does not start with `handler` or `transport`, a repeated line, or a missing line → E-SERVER-LINE.
8. **Recovery (`.cprime`).** On a parse error at item level, skip tokens until a token that is one of `type struct enum capability spec entry server bind` **and** starts at column 1, then resume. Inside any body (`spec`, `entry`, `struct`, `enum`, `capability`), skip to the next `TERM` or the matching `}` at the same brace depth and resume with the next statement, so each faulty line yields its own diagnostic. Inside a `uses` or `unconstrained` list, skip to the next `,` or `TERM`.
9. **Depth guard.** The contract-expression and type-expression parsers carry a depth counter; exceeding `MAX_NESTING` → E-DEPTH at the token that crossed the limit, then recovery as in step 8.
10. **Cap diagnostics.** After `MAX_DIAGNOSTICS` diagnostics, stop parsing and append E-TOO-MANY.
11. **Build slices.** For each `ContractExpr`, set `text = text[start_byte..end_byte]` of the node; a clause's text starts after its `requires`/`ensures` keyword.
12. **Return.** `module = Some(..)` only if no diagnostic was recorded; diagnostics are sorted by `(line, column)` with insertion order breaking ties.

`parse_opq(file, text)`:

1. **Size check.** As `.cprime` step 1.
2. **BOM check.** As `.cprime` step 2.
3. **Header capture.** If the first line starts with `;; cprime-accepted`, store the whole first line (without `\n`) in `header`. No validation here; `cprime_core::accept::parse_header` validates it.
4. **Lex.** `.opq` lexer: delimiters `( ) [ ]`, operator tokens (longest match: `==`, `!=`, `<=`, `>=`, `++` before single characters), `NEG_INT` when `-` is directly followed by a digit, `BYTES`, `INT`, identifiers and paths (`IDENT { "::" IDENT }` as one token), reserved and future-reserved words. `;` comments and whitespace dropped. Lexer errors E-CHAR, E-INT, E-HEX as for `.cprime`.
5. **Delimiter pass.** Before parsing, match every `(`/`[` with its closer using a stack. An unexpected closer → E-CLOSE-UNEXPECTED (the token is dropped). A closer of the wrong kind → E-CLOSE-MISMATCH (expected closer named). Unclosed at end of input → E-UNCLOSED for each open delimiter, innermost first. These diagnostics are recorded and parsing proceeds on the repaired token stream (dropped closers removed, missing closers inserted at the error position).
6. **Parse.** Expect exactly one `impl` form. Nothing → E-NO-IMPL. Anything after the `impl` form other than comments → E-EXTRA-FORM. Within forms, the `nested_delimiters` recovery replaces a malformed form by an error placeholder spanning its delimiters, records the diagnostic and continues with the next sibling.
7. **Form-specific checks.** `let` with an empty binding list or an odd number of items in `[…]` → E-LET. `if` without exactly three subexpressions → E-IF. `match` without arms → E-MATCH. `decreases` with an empty tuple → E-DECREASES. `(. e f)` where `f` is not an `IDENT` → E-FIELD. `par_map` not followed by exactly one path and one expression → E-PARMAP.
8. **Path/file agreement.** If `file != "<submission>"`, compute the expected file from the `impl` path (see grammar) and compare; mismatch → E-IMPL-FILE.
9. **Depth guard.** As `.cprime` step 9 over form nesting.
10. **Cap diagnostics.** As `.cprime` step 10.
11. **Number nodes.** If no diagnostic was recorded, run the pre-order numbering walk and set `node_count`.
12. **Return.** `(Some(impl), [])` or `(None, diagnostics)` sorted as in `.cprime` step 12.

All diagnostics from this crate have `kind: ParseError`, `tier: T0`, `severity: Error`, `node: NodeId(0)`, `span: Some(..)`, and `clause`, `counterexample`, `reason` unset. Message format for a generic unexpected token: ``expected <list>, found <found>`` where `<list>` is the expected tokens, each in backticks, sorted, at most six, joined by `, ` with the last pair joined by ` or `; when more than six are possible the list is the first six followed by ` or another token`; `<found>` is the token in backticks or `end of input`. A `TERM` inserted at a newline is named `end of line`; a written `;` is named `` `;` ``.

#### Data Model

N/A — no persistent state. The crate is stateless; its only data structures are the token vector, the line-start index and the AST types owned by `cprime-core`, all dropped on return.

#### Error Table

All rows: Status = `rejected` (returned diagnostic; the parse result is `None`), Code = `ParseError`. Response Body is the `message` field; every diagnostic also carries `"kind":"ParseError","tier":"t0","severity":"error","node":0` and the `span`.

| Id | Condition | Status | Code | Response Body |
|----|-----------|--------|------|---------------|
| E-SIZE | Source longer than 1,048,576 bytes | rejected | ParseError | `source is 1048577 bytes; the limit is 1048576` (span 1:1) |
| E-BOM | Source starts with U+FEFF | rejected | ParseError | `byte-order mark is not allowed` |
| E-CHAR | Character not part of any token | rejected | ParseError | ``unexpected character `@` `` |
| E-INT | Integer literal above `u64::MAX` (or `NEG_INT` magnitude above 2^63) | rejected | ParseError | ``integer literal `18446744073709551616` is out of range`` |
| E-HEX | `x"…"` (`.cprime`) or `0x` (`.opq`) literal with an odd number of hex digits | rejected | ParseError | ``byte literal `x"abc"` has an odd number of hex digits`` |
| E-TOKEN | Token does not fit the grammar | rejected | ParseError | ``expected `,` or `}`, found `user_id` `` |
| E-RESERVED | A future-reserved word (SA-49) appears in either surface | rejected | ParseError | `` `loop` is reserved for a future version of C′ and cannot be used as a name `` |
| E-RESERVED-TYPE | A type, capability or generic parameter is declared with a reserved type name (SA-49) | rejected | ParseError | `` `Str` is reserved for a future built-in type and cannot be declared `` |
| E-MODULE-MISSING | `.cprime` does not start with `module` | rejected | ParseError | ``expected `module`, found `type` `` |
| E-MODULE-DUP | Second `module` declaration | rejected | ParseError | `a file declares exactly one module` |
| E-MODULE-SEGMENT | Module segment not UpperCamelCase | rejected | ParseError | ``module path segment `auth` must start with an upper-case letter and contain only letters and digits`` |
| E-MODULE-FILE | Module path disagrees with file path (SA-11) | rejected | ParseError | ``module `Auth::Session` must live in `auth/session.cprime`, not `auth/sessions.cprime` `` |
| E-CLAUSE-ORDER | Clause kinds out of order, `uses`/`assurance`/`unconstrained` repeated, or `requires`/`ensures` lines not consecutive | rejected | ParseError | ``clause `requires` must come after `unconstrained` and before `ensures`; `uses`, `assurance` and `unconstrained` appear at most once`` |
| E-RESULT-PARAM | A spec or entry parameter is named `result` (ADR-050) | rejected | ParseError | `` `result` names the spec's result in `ensures` and cannot be a parameter name `` |
| E-BYTES-0X | `0x…` literal in `.cprime` | rejected | ParseError | ``byte literals in .cprime are written x"…"; found `0xab` `` |
| E-FN | `fn` before an operation name in a capability body | rejected | ParseError | ``capability operations are written without `fn`: `deterministic hmac_sha256(…) Hash` `` |
| E-BODY | A spec or entry header not followed by `{` on the same line | rejected | ParseError | ``expected `{` to open the body of `spec generate_token`, found end of line`` |
| E-SERVER-LINE | A `server` body line other than `handler`/`transport`, a repeated line, or a missing line (ADR-051) | rejected | ParseError | ``unexpected `uses` in server `Api`; a server body has one `handler` line and one `transport` line`` or ``server `Api` has no `transport` line`` |
| E-NONASSOC | Two comparisons/variant tests chained at level 4 | rejected | ParseError | ``comparison operators do not chain; add parentheses or `&&` `` |
| E-DEPTH | Nesting deeper than 256 | rejected | ParseError | `nesting is deeper than 256 levels` |
| E-CLOSE-UNEXPECTED | `)` or `]` with no open delimiter | rejected | ParseError | ``unexpected `)` with no matching `(` `` |
| E-CLOSE-MISMATCH | Closer of the wrong kind | rejected | ParseError | ``expected `)` to close `(` opened at 2:13, found `]` `` |
| E-UNCLOSED | End of input with open delimiter | rejected | ParseError | ``unclosed `(` opened at 1:1`` |
| E-NO-IMPL | `.opq` contains no `impl` form | rejected | ParseError | ``expected `(impl …)`, found end of input`` |
| E-EXTRA-FORM | Anything after the `impl` form | rejected | ParseError | `a .opq file contains exactly one impl form` |
| E-LET | `let` bindings empty or unpaired | rejected | ParseError | `let bindings must be non-empty name/value pairs` |
| E-IF | `if` without exactly three operands | rejected | ParseError | `if takes a condition, a then-branch and an else-branch` |
| E-MATCH | `match` without arms | rejected | ParseError | `match needs at least one arm` |
| E-DECREASES | `(decreases [])` | rejected | ParseError | `decreases needs at least one measure` |
| E-FIELD | `(. e x)` where `x` is not an identifier | rejected | ParseError | ``expected a field name after the value in `(. …)`, found `3` `` |
| E-PARMAP | `par_map` without exactly a path and one expression | rejected | ParseError | ``par_map takes a spec name and a vector, as (par_map Spec xs)`` |
| E-IMPL-FILE | `impl` path disagrees with on-disk file | rejected | ParseError | ``implementation of `Auth::generate_token` must live in `auth/generate_token.opq`, not `auth/gen.opq` `` |
| E-TOO-MANY | More than 100 diagnostics | rejected | ParseError | `too many parse errors; stopped after 100` |

#### Acceptance Criteria (Gherkin)

```gherkin
Feature: cprime-syntax

  Scenario: Happy path — concept example parses on both surfaces
    Given the file "auth.cprime" with the concept module text
    And the file "auth/generate_token.opq" with the concept implementation text
    When parse_cprime and parse_opq are called with those paths
    Then both return no diagnostics
    And the spec "generate_token" has 3 requires clauses and 3 ensures clauses
    And the second requires clause has text "now <= U64::MAX - 3600"
    And the parameters are user_id: U64, now: U64, secret: Bytes in that order
    And the implementation has node_count 23 and node 6 is the Apply with head "+"

  Scenario: Happy path — sum type, variant test and match
    Given the files "geo.cprime" and "geo/area.opq" from the sum-type example
    When both are parsed
    Then the first requires clause is Implies(Is(s, Circle), Le(Field(Field(s, Circle), r), 1000))
    And the match has two arms with binders ["r"] and ["side"]

  Scenario: Edge case — acceptance header is captured, not validated
    Given an .opq whose first line is ";; cprime-accepted v1 context=zz evidence=proved assumed=-"
    When parse_opq is called
    Then the result is Some with header equal to that line without the newline
    And no diagnostic is reported

  Scenario: Edge case — lexicographic decreases tuple and negative literal
    Given the source "(impl Math::ack [m n] (decreases [m n]) (- n -1))"
    When parse_opq is called with file "<submission>"
    Then decreases has 2 measure expressions
    And the body is Apply "-" with arguments Name n and Int -1

  Scenario: Edge case — empty byte literal and unit
    Given the source "(impl M::f [] (++ 0x ()))"
    When parse_opq is called with file "<submission>"
    Then the Apply "++" has arguments Bytes [] and Unit

  Scenario: Error path — every faulty requires line reported in one pass
    Given a spec body with the lines "requires x >" and "requires y <"
    When parse_cprime is called
    Then 2 ParseError diagnostics are returned, one per line
    And module is None

  Scenario: Error path — mismatched delimiter
    Given the submission "(impl Auth::generate_token [uid now secret] (let [exp (+ now 3600] exp))"
    When parse_opq is called with file "<submission>"
    Then the result is None
    And a ParseError at line 1 says "expected `)` to close `(` opened at" the position of "(+"

  Scenario: Error path — module path does not match file
    Given the file "auth/sessions.cprime" declaring "module Auth::Session"
    When parse_cprime is called
    Then a ParseError with message "module `Auth::Session` must live in `auth/session.cprime`, not `auth/sessions.cprime`" is returned

  Scenario: Error path — chained comparison
    Given a requires clause "a < b < c"
    When parse_cprime is called
    Then a ParseError "comparison operators do not chain; add parentheses or `&&`" is returned

  Scenario: Error path — oversized and over-nested input
    Given a source of 1048577 bytes
    When parse_opq is called
    Then exactly one ParseError "source is 1048577 bytes; the limit is 1048576" is returned
    Given a source with 300 nested "(not " forms
    When parse_opq is called
    Then a ParseError "nesting is deeper than 256 levels" is returned and no stack overflow occurs

  Scenario: Error path — future-reserved word used as a name
    Given an .opq implementation that binds (let [loop 1] loop)
    When parse_opq is called
    Then the diagnostics contain one ParseError with message "`loop` is reserved for a future version of C′ and cannot be used as a name"
    And its span covers the first occurrence of "loop"

  Scenario: Error path — reserved type name declared
    Given a .cprime module that declares "type Str = Bytes"
    When parse_cprime is called
    Then the diagnostics contain one ParseError with message "`Str` is reserved for a future built-in type and cannot be declared"

  Scenario: Edge case — no return type means Unit and no ensures line means no binder
    Given a spec "spec log_line(msg Bytes) {\n    uses Log\n    requires msg != x\"\"\n}"
    When parse_cprime is called
    Then the spec's ret is the path Unit
    And ensures is None
    And the requires clause is Binary{Ne, Path(msg), Bytes []}

  Scenario: Edge case — a call broken after a comma continues the line
    Given an ensures line ending in "hmac_sha256(secret," followed by the line "u64_be(user_id))"
    When parse_cprime is called
    Then no diagnostic is reported and the clause is one Call with two arguments

  Scenario: Error path — parameter named result
    Given "spec f(result U64) U64 {}"
    When parse_cprime is called
    Then a ParseError "`result` names the spec's result in `ensures` and cannot be a parameter name" is returned

  Scenario: Error path — requires lines not consecutive
    Given a spec body with the lines "requires a > 0", "ensures result > 0" and "requires b > 0"
    When parse_cprime is called
    Then a ParseError E-CLAUSE-ORDER is returned at the second requires

  Scenario: Error path — 0x literal and odd x"…" literal in a contract
    Given a requires line "requires k != 0xab"
    When parse_cprime is called
    Then a ParseError "byte literals in .cprime are written x\"…\"; found `0xab`" is returned
    Given a requires line "requires k != x\"abc\""
    When parse_cprime is called
    Then a ParseError "byte literal `x\"abc\"` has an odd number of hex digits" is returned

  Scenario: Error path — fn in a capability and a missing body brace
    Given "capability C {\n    fn op() U64\n}"
    When parse_cprime is called
    Then a ParseError "capability operations are written without `fn`: `deterministic hmac_sha256(…) Hash`" is returned
    Given "spec f() U64\n    requires true"
    When parse_cprime is called
    Then a ParseError "expected `{` to open the body of `spec f`, found end of line" is returned

  Scenario: Happy path — parallel map parses
    Given an .opq implementation whose body is (par_map Auth::verify_one tokens)
    When parse_opq is called
    Then there are no diagnostics
    And the body is ParMap with spec "Auth::verify_one" and xs Name("tokens")

  Scenario: Error path — malformed parallel map
    Given an .opq implementation whose body is (par_map tokens)
    When parse_opq is called
    Then a ParseError "par_map takes a spec name and a vector, as (par_map Spec xs)" is returned

  Scenario: Happy path — server item parses
    Given the file "api.cprime" from the server example
    When parse_cprime is called
    Then there are no diagnostics
    And the module has one Server item with handler "Api::handle" and transport "acme::HttpTransport"

  Scenario: Error path — malformed server body
    Given "server Api {\n    handler Api::handle\n    uses Log\n}"
    When parse_cprime is called
    Then a ParseError "unexpected `uses` in server `Api`; a server body has one `handler` line and one `transport` line" is returned
    Given "server Api {\n    handler Api::handle\n}"
    When parse_cprime is called
    Then a ParseError "server `Api` has no `transport` line" is returned
```

#### Performance, Security, Observability

- **Performance targets:** `parse_opq` ≤ 1 ms p95 for an implementation of ≤ 200 IR instructions (≤ 8 KiB source) — 20 % of the T0 budget of 5 ms. `parse_cprime` ≤ 10 ms p95 for a 2,000-line module (on-disk modules are parsed once per change and cached by Salsa via the daemon's hook, so this is outside the per-request T0 budget). Memory: AST ≤ 64 bytes per source byte. Linear time in input length; recovery never re-lexes.
- **Security:** Input is untrusted agent text. Bounded by `MAX_SOURCE_BYTES` (1 MiB), `MAX_NESTING` (256, prevents stack exhaustion in recursive combinators) and `MAX_DIAGNOSTICS` (100, bounds reply size). No I/O, no allocation proportional to anything but input size, no `unsafe`. Diagnostic messages quote at most one token (truncated to 40 characters with `…`), never the whole source.
- **Observability:** The crate does not log. Callers measure its time inside `timings_us.t0` and log at their own levels. Every diagnostic carries a span for user-facing location.

#### Gaps

None.
