# Hax Internal Architecture - Exhaustive Technical Documentation

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Compilation Pipeline](#compilation-pipeline)
3. [Frontend Architecture](#frontend-architecture)
4. [AST Representations](#ast-representations)
5. [Engine Architecture](#engine-architecture)
6. [Backend Code Generators](#backend-code-generators)
7. [Type System Implementation](#type-system-implementation)
8. [Memory and Performance](#memory-and-performance)
9. [Error Handling and Diagnostics](#error-handling-and-diagnostics)
10. [Interprocess Communication](#interprocess-communication)
11. [Plugin System](#plugin-system)
12. [Testing Infrastructure](#testing-infrastructure)

---

## Architecture Overview

### System Components

The hax system consists of five major architectural layers:

```
┌──────────────────────────────────────────────────────────────────┐
│                         User Code Layer                          │
│  Rust source code with hax-lib annotations and specifications    │
└────────────────────┬─────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Frontend Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐      │
│  │ Rust Driver  │  │ THIR Extract │  │ Attribute Parser   │      │
│  │ (callbacks)  │  │ (MIR→THIR)   │  │ (macro expansion)  │      │
│  └──────────────┘  └──────────────┘  └────────────────────┘      │
└────────────────────┬─────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────────┐
│                   Serialization Layer                            │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐      │
│  │ JSON Export  │  │ CBOR Export  │  │ Schema Validation  │      │
│  │ (human-read) │  │ (efficient)  │  │ (type checking)    │      │
│  └──────────────┘  └──────────────┘  └────────────────────┘      │
└────────────────────┬─────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Engine Layer                                │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐      │
│  │ AST Import   │  │ Simplifier   │  │ Name Resolution    │      │
│  │              │  │              │  │                    │      │
│  └──────────────┘  └──────────────┘  └────────────────────┘      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐      │
│  │ Type Checker │  │ Elaborator   │  │ Phase Splitter     │      │
│  │              │  │              │  │                    │      │
│  └──────────────┘  └──────────────┘  └────────────────────┘      │
└────────────────────┬─────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Backend Layer                                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐     │
│  │   F*     │ │   Lean   │ │   Coq    │ │     ProVerif     │     │
│  │ Backend  │ │ Backend  │ │ Backend  │ │     Backend      │     │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

### Data Flow Architecture

```
Source Files → Tokenization → Parsing → Name Resolution → Type Checking
    ↓              ↓            ↓            ↓                ↓
Attributes → Macro Expansion → HIR → MIR → THIR → Hax Extraction
    ↓              ↓            ↓      ↓      ↓         ↓
Contracts → Elaboration → Simplification → Backend Translation
    ↓              ↓            ↓              ↓
Specifications → Verification Conditions → Target Language
```

---

## Compilation Pipeline

### Stage 1: Rust Compilation Integration

#### 1.1 Driver Initialization

```rust
// cli/driver/src/callbacks_wrapper.rs
pub struct CallbacksWrapper<'a> {
    pub sub: &'a mut (dyn Callbacks + Send + 'a),
    pub options: ExporterOptions,
}

impl<'a> CallbacksWrapper<'a> {
    fn initialize_compilation(&mut self) {
        // 1. Set up rustc session
        // 2. Configure feature flags
        // 3. Initialize diagnostic handlers
        // 4. Set up path remapping
    }
}
```

The driver hooks into rustc's compilation pipeline through the `Callbacks` trait:

1. **after_crate_root_parsing**: Captures crate metadata
2. **after_expansion**: Processes macro expansions and attributes
3. **after_analysis**: Main extraction point for THIR

#### 1.2 Feature Detection

```rust
// cli/driver/src/features.rs
pub struct Features {
    pub adt_const_params: bool,      // Generic const parameters in ADTs
    pub generic_const_exprs: bool,   // Const expressions in generics
    pub register_tool: bool,          // Tool attributes registration
    pub auto_traits: bool,            // Auto trait implementations
    pub negative_impls: bool,         // Negative trait implementations
    pub registered_tools: HashSet<String>,
}
```

### Stage 2: AST Extraction

#### 2.1 THIR Extraction

The Typed High-level Intermediate Representation (THIR) extraction happens in several phases:

```rust
// frontend/exporter/src/thir_export.rs
pub fn extract_thir(tcx: TyCtxt<'_>) -> ThirBundle {
    ThirBundle {
        items: extract_items(tcx),
        types: extract_types(tcx),
        traits: extract_traits(tcx),
        impls: extract_implementations(tcx),
        metadata: extract_metadata(tcx),
    }
}
```

#### 2.2 Item Extraction Pipeline

```
Item Discovery → Dependency Analysis → Topological Sort → Extraction
      ↓                  ↓                    ↓              ↓
  Find Items      Build Dep Graph      Order Items    Extract Each
      ↓                  ↓                    ↓              ↓
  Filter by       Track Imports        Handle Cycles   Generate JSON
  Attributes      & References          with SCCs
```

### Stage 3: Intermediate Processing

#### 3.1 JSON/CBOR Serialization

```rust
// hax-types/src/driver_api.rs
#[derive(Serialize, Deserialize)]
pub enum HaxMessage {
    Diagnostic {
        diagnostic: Diagnostic,
        working_dir: Option<PathBuf>,
    },
    Item(ExtractedItem),
    TypeInfo(TypeInformation),
    ImplInfo(Implementation),
    Complete(ExtractionMetadata),
}
```

#### 3.2 Schema Validation

Each message is validated against a schema:

```rust
pub struct SchemaValidator {
    item_schema: Schema,
    type_schema: Schema,
    impl_schema: Schema,

    pub fn validate(&self, msg: &HaxMessage) -> Result<(), ValidationError> {
        match msg {
            HaxMessage::Item(item) => self.validate_item(item),
            HaxMessage::TypeInfo(ty) => self.validate_type(ty),
            // ...
        }
    }
}
```

---

## Frontend Architecture

### Rust Compiler Integration

#### Compiler Hooks

```rust
// frontend/exporter/src/lib.rs
pub struct HaxFrontend {
    tcx: TyCtxt<'tcx>,
    options: FrontendOptions,
    id_table: IdTable,
    name_resolver: NameResolver,
}

impl HaxFrontend {
    pub fn new(tcx: TyCtxt<'tcx>, options: FrontendOptions) -> Self {
        Self {
            tcx,
            options,
            id_table: IdTable::new(),
            name_resolver: NameResolver::new(tcx),
        }
    }

    pub fn extract_crate(&mut self) -> CrateData {
        let items = self.extract_all_items();
        let types = self.extract_type_definitions();
        let traits = self.extract_trait_definitions();
        let impls = self.extract_implementations();

        CrateData {
            items,
            types,
            traits,
            impls,
            metadata: self.extract_metadata(),
        }
    }
}
```

#### Attribute Processing

```rust
// frontend/exporter/src/attributes.rs
pub enum HaxAttribute {
    Requires(Expression),
    Ensures(Box<dyn Fn(Variable) -> Expression>),
    Decreases(Expression),
    LoopInvariant(Expression),
    Include,
    Exclude,
    Opaque,
    OpaqueType,
    VerificationStatus(Status),
    FStarOptions(Vec<String>),
    LeanOptions(Vec<String>),
}

pub struct AttributeProcessor {
    pub fn process_attributes(&self, attrs: &[Attribute]) -> Vec<HaxAttribute> {
        attrs.iter()
            .filter_map(|attr| self.parse_hax_attribute(attr))
            .collect()
    }

    fn parse_hax_attribute(&self, attr: &Attribute) -> Option<HaxAttribute> {
        let path = attr.path();
        match path.segments.last()?.ident.as_str() {
            "requires" => Some(self.parse_requires(attr)),
            "ensures" => Some(self.parse_ensures(attr)),
            "decreases" => Some(self.parse_decreases(attr)),
            // ...
        }
    }
}
```

### Type System Bridge

#### Type Translation

```rust
// frontend/exporter/src/types.rs
pub struct TypeTranslator<'tcx> {
    tcx: TyCtxt<'tcx>,
    cache: FxHashMap<Ty<'tcx>, TranslatedType>,
}

impl<'tcx> TypeTranslator<'tcx> {
    pub fn translate_ty(&mut self, ty: Ty<'tcx>) -> TranslatedType {
        if let Some(cached) = self.cache.get(&ty) {
            return cached.clone();
        }

        let translated = match ty.kind() {
            TyKind::Bool => TranslatedType::Bool,
            TyKind::Int(int_ty) => self.translate_int_ty(int_ty),
            TyKind::Uint(uint_ty) => self.translate_uint_ty(uint_ty),
            TyKind::Float(float_ty) => self.translate_float_ty(float_ty),
            TyKind::Adt(def, substs) => self.translate_adt(def, substs),
            TyKind::Array(elem_ty, len) => self.translate_array(elem_ty, len),
            TyKind::Slice(elem_ty) => self.translate_slice(elem_ty),
            TyKind::Ref(region, ty, mutability) => {
                self.translate_reference(region, ty, mutability)
            }
            TyKind::FnDef(def_id, substs) => self.translate_fn_def(def_id, substs),
            TyKind::Closure(def_id, substs) => self.translate_closure(def_id, substs),
            // ... handle all type kinds
        };

        self.cache.insert(ty, translated.clone());
        translated
    }
}
```

---

## AST Representations

### THIR Structure

```rust
// frontend/exporter/src/thir_ast.rs
pub struct ThirAst {
    pub crate_name: String,
    pub items: Vec<Item>,
    pub foreign_items: Vec<ForeignItem>,
    pub trait_items: Vec<TraitItem>,
    pub impl_items: Vec<ImplItem>,
    pub bodies: FxHashMap<BodyId, Body>,
    pub types: FxHashMap<TypeId, TypeDef>,
}

pub struct Body {
    pub params: Vec<Param>,
    pub value: Expr,
    pub locals: Vec<Local>,
    pub source: BodySource,
}

pub enum Expr {
    Use { source: Place },
    Call { fun: Box<Expr>, args: Vec<Expr>, from_hir_call: bool },
    Binary { op: BinOp, lhs: Box<Expr>, rhs: Box<Expr> },
    Unary { op: UnOp, arg: Box<Expr> },
    Cast { value: Box<Expr>, ty: Ty },
    Let { pattern: Pattern, init: Option<Box<Expr>>, else_block: Option<Block> },
    Block { block: Block },
    If { cond: Box<Expr>, then: Box<Expr>, else_opt: Option<Box<Expr>> },
    Match { scrutinee: Box<Expr>, arms: Vec<Arm> },
    Loop { body: Block, source: LoopSource },
    Break { label: Option<Label>, value: Option<Box<Expr>> },
    Continue { label: Option<Label> },
    Return { value: Option<Box<Expr>> },
    ConstBlock { body: Box<Expr> },
    Repeat { value: Box<Expr>, count: Const },
    Array { elements: Vec<Expr> },
    Tuple { fields: Vec<Expr> },
    Adt { adt_def: AdtDef, variant: VariantIdx, fields: Vec<FieldExpr> },
    PlaceTypeAscription { source: Place, ty: Ty },
    ValueTypeAscription { source: Box<Expr>, ty: Ty },
    Closure { def_id: DefId, substs: SubstsRef, upvars: Vec<Expr> },
    Literal { lit: Literal, neg: bool },
    NonHirLiteral { lit: ScalarInt, ty: Ty },
    ZstLiteral { ty: Ty },
    NamedConst { def_id: DefId, substs: SubstsRef },
    ConstParam { param: ParamConst },
    StaticRef { def_id: DefId },
    InlineAsm { template: Vec<InlineAsmTemplatePiece>, operands: Vec<InlineAsmOperand> },
    ThreadLocalRef { def_id: DefId },
    Yield { value: Box<Expr> },
}
```

### Simplified AST

After extraction, the AST is simplified:

```rust
// engine/lib/ast.ml
type expr =
  | Var of var
  | Const of constant
  | App of expr * expr list
  | Lambda of pattern * expr
  | Let of pattern * expr * expr
  | Match of expr * (pattern * expr) list
  | If of expr * expr * expr
  | Record of (field_name * expr) list
  | Field of expr * field_name
  | Construct of constructor * expr list
  | Array of expr list
  | ArrayIndex of expr * expr
  | Cast of expr * ty
  | Ascription of expr * ty

type pattern =
  | PVar of var
  | PWild
  | PConst of constant
  | PConstruct of constructor * pattern list
  | PRecord of (field_name * pattern) list
  | PArray of pattern list
  | POr of pattern list
  | PGuard of pattern * expr
```

---

## Engine Architecture

### OCaml Engine

#### Module Structure

```
engine/
├── lib/
│   ├── ast.ml           # AST definitions
│   ├── ast_utils.ml     # AST manipulation utilities
│   ├── phases/
│   │   ├── simplify.ml  # AST simplification
│   │   ├── elaborate.ml # Type elaboration
│   │   ├── specialize.ml# Monomorphization
│   │   └── inline.ml    # Function inlining
│   ├── import/
│   │   ├── json.ml      # JSON deserialization
│   │   └── thir.ml      # THIR import
│   └── backend/
│       ├── fstar.ml     # F* code generation
│       ├── lean.ml      # Lean code generation
│       └── coq.ml       # Coq code generation
```

#### Phase System

```ocaml
(* engine/lib/phases.ml *)
module type PHASE = sig
  type input
  type output
  val name : string
  val run : Context.t -> input -> output
end

module Simplify : PHASE with
  type input = Ast.expr and
  type output = Ast.expr

module Elaborate : PHASE with
  type input = Ast.expr and
  type output = ElaboratedAst.expr

module Monomorphize : PHASE with
  type input = ElaboratedAst.expr and
  type output = MonoAst.expr
```

#### Simplification Phase

```ocaml
(* engine/lib/phases/simplify.ml *)
let rec simplify_expr ctx = function
  | E_app (E_lambda (pat, body), arg) ->
      (* Beta reduction *)
      let subst = match_pattern pat arg in
      substitute subst body |> simplify_expr ctx

  | E_match (E_construct (c, args), branches) ->
      (* Match reduction *)
      begin match find_matching_branch c branches with
      | Some (pat, body) ->
          let subst = match_pattern pat (E_construct (c, args)) in
          substitute subst body |> simplify_expr ctx
      | None -> E_match (E_construct (c, args), branches)
      end

  | E_let (pat, value, body) when is_simple_pattern pat ->
      (* Let inlining *)
      let subst = match_pattern pat value in
      substitute subst body |> simplify_expr ctx

  | expr -> map_expr (simplify_expr ctx) expr
```

### Rust Engine (Experimental)

#### Architecture

```rust
// rust-engine/src/lib.rs
pub struct RustEngine {
    phases: Vec<Box<dyn Phase>>,
    context: Context,
}

pub trait Phase {
    fn name(&self) -> &str;
    fn run(&mut self, ast: AST, ctx: &mut Context) -> Result<AST, Error>;
}

impl RustEngine {
    pub fn new() -> Self {
        Self {
            phases: vec![
                Box::new(SimplifyPhase::new()),
                Box::new(ElaboratePhase::new()),
                Box::new(MonomorphizePhase::new()),
                Box::new(OptimizePhase::new()),
            ],
            context: Context::new(),
        }
    }

    pub fn process(&mut self, ast: AST) -> Result<AST, Error> {
        self.phases.iter_mut().fold(
            Ok(ast),
            |ast, phase| ast.and_then(|a| phase.run(a, &mut self.context))
        )
    }
}
```

---

## Backend Code Generators

### F* Backend

#### Code Generation Pipeline

```fsharp
(* engine/backends/fstar.ml *)
module FStarBackend = struct
  type options = {
    z3rlimit: int;
    fuel: int;
    ifuel: int;
    include_dirs: string list;
  }

  let generate_module ctx ast =
    let open FStar.Syntax in
    let decls = List.map (translate_item ctx) ast.items in
    let imports = generate_imports ast.dependencies in
    Module {
      name = ast.name;
      imports = imports;
      declarations = decls;
      attributes = generate_attributes ctx;
    }

  let translate_expr ctx = function
    | Ast.Const (Int n) -> FStar.Const (FStar.Int n)
    | Ast.App (f, args) -> FStar.App (translate_expr ctx f,
                                       List.map (translate_expr ctx) args)
    | Ast.Lambda (pat, body) ->
        let binders = pattern_to_binders pat in
        FStar.Abs (binders, translate_expr ctx body)
    | Ast.Match (scrut, branches) ->
        FStar.Match (translate_expr ctx scrut,
                     List.map (translate_branch ctx) branches)
    (* ... comprehensive translation ... *)

  let translate_type ctx = function
    | Ty.Int -> FStar.Type.Int
    | Ty.Bool -> FStar.Type.Bool
    | Ty.Array (elem, size) ->
        FStar.Type.App (FStar.Type.Array,
                        [translate_type ctx elem;
                         FStar.Type.Const size])
    (* ... all type translations ... *)
```

#### Specification Translation

```fsharp
let translate_spec ctx spec =
  match spec with
  | Requires pred ->
      FStar.Spec.Requires (translate_predicate ctx pred)
  | Ensures (var, pred) ->
      FStar.Spec.Ensures (var, translate_predicate ctx pred)
  | Decreases expr ->
      FStar.Spec.Decreases (translate_expr ctx expr)
  | LoopInvariant pred ->
      FStar.Spec.Assert (translate_predicate ctx pred)
```

### Lean Backend

#### Lean 4 Code Generation

```rust
// engine/backends/lean.rs
pub struct LeanBackend {
    options: LeanOptions,
    context: LeanContext,
}

impl LeanBackend {
    pub fn translate_item(&mut self, item: &Item) -> LeanDecl {
        match item {
            Item::Function(f) => self.translate_function(f),
            Item::Type(t) => self.translate_type_def(t),
            Item::Const(c) => self.translate_constant(c),
            Item::Theorem(t) => self.translate_theorem(t),
        }
    }

    fn translate_function(&mut self, func: &Function) -> LeanDecl {
        let params = self.translate_params(&func.params);
        let ret_type = self.translate_type(&func.return_type);
        let body = self.translate_expr(&func.body);

        // Generate panic-freedom proof if needed
        let panic_proof = if self.options.generate_panic_proofs {
            Some(self.generate_panic_freedom_proof(func))
        } else {
            None
        };

        LeanDecl::Function {
            name: func.name.clone(),
            params,
            ret_type,
            body,
            decreases: func.decreases.as_ref().map(|d| self.translate_expr(d)),
            spec: self.translate_spec(&func.spec),
            panic_proof,
        }
    }

    fn generate_panic_freedom_proof(&mut self, func: &Function) -> LeanProof {
        // Analyze function for potential panics
        let panic_points = self.find_panic_points(&func.body);

        // Generate proof obligations
        let obligations = panic_points.iter().map(|point| {
            match point {
                PanicPoint::Division(dividend, divisor) => {
                    LeanObligation::NonZero(divisor.clone())
                }
                PanicPoint::ArrayAccess(array, index) => {
                    LeanObligation::InBounds(array.clone(), index.clone())
                }
                PanicPoint::Overflow(op, lhs, rhs) => {
                    LeanObligation::NoOverflow(op.clone(), lhs.clone(), rhs.clone())
                }
            }
        }).collect();

        LeanProof::PanicFreedom {
            function: func.name.clone(),
            obligations,
        }
    }
}
```

### Coq Backend

#### Coq Translation

```fsharp
(* engine/backends/coq.ml *)
module CoqBackend = struct
  let translate_inductive ctx ind =
    let constructors = List.map (translate_constructor ctx) ind.constructors in
    Coq.Inductive {
      name = ind.name;
      params = translate_params ctx ind.params;
      indices = translate_indices ctx ind.indices;
      sort = translate_sort ctx ind.sort;
      constructors = constructors;
    }

  let translate_fixpoint ctx fix =
    let bodies = List.map (translate_fix_body ctx) fix.bodies in
    Coq.Fixpoint {
      struct_arg = fix.struct_arg;
      bodies = bodies;
      decreases = Option.map (translate_expr ctx) fix.decreases;
    }

  let generate_proof_obligation ctx obl =
    match obl with
    | Termination (f, arg) ->
        Coq.Lemma {
          name = Printf.sprintf "%s_terminates" f.name;
          statement = generate_termination_statement f arg;
          proof = Coq.ProofScript ["intros"; "induction " ^ arg; "auto"];
        }
    | Invariant (loop, inv) ->
        Coq.Lemma {
          name = Printf.sprintf "%s_invariant" loop.name;
          statement = generate_invariant_statement loop inv;
          proof = Coq.ProofScript ["intros"; "induction"; "auto"];
        }
```

---

## Type System Implementation

### Type Inference

```rust
// engine/src/type_system/inference.rs
pub struct TypeInferencer {
    unification_table: UnificationTable<TypeVar>,
    type_constraints: Vec<Constraint>,
    context: TypeContext,
}

impl TypeInferencer {
    pub fn infer_expr(&mut self, expr: &Expr) -> Result<Type, TypeError> {
        match expr {
            Expr::Var(v) => self.lookup_var(v),
            Expr::App(f, args) => {
                let f_ty = self.infer_expr(f)?;
                let (param_tys, ret_ty) = self.expect_function_type(f_ty)?;

                if args.len() != param_tys.len() {
                    return Err(TypeError::ArityMismatch);
                }

                for (arg, param_ty) in args.iter().zip(param_tys.iter()) {
                    let arg_ty = self.infer_expr(arg)?;
                    self.unify(arg_ty, param_ty.clone())?;
                }

                Ok(ret_ty)
            }
            Expr::Lambda(params, body) => {
                let param_tys: Vec<Type> = params.iter()
                    .map(|_| self.fresh_type_var())
                    .collect();

                self.push_scope();
                for (param, ty) in params.iter().zip(param_tys.iter()) {
                    self.bind_var(param, ty.clone());
                }

                let body_ty = self.infer_expr(body)?;
                self.pop_scope();

                Ok(Type::Function(param_tys, Box::new(body_ty)))
            }
            // ... handle all expression kinds
        }
    }

    pub fn unify(&mut self, ty1: Type, ty2: Type) -> Result<(), TypeError> {
        match (ty1, ty2) {
            (Type::Var(v1), Type::Var(v2)) => {
                self.unification_table.unify(v1, v2)
            }
            (Type::Var(v), ty) | (ty, Type::Var(v)) => {
                if ty.occurs_check(v) {
                    return Err(TypeError::InfiniteType);
                }
                self.unification_table.set(v, ty);
                Ok(())
            }
            (Type::Function(p1, r1), Type::Function(p2, r2)) => {
                if p1.len() != p2.len() {
                    return Err(TypeError::FunctionArityMismatch);
                }
                for (param1, param2) in p1.into_iter().zip(p2.into_iter()) {
                    self.unify(param1, param2)?;
                }
                self.unify(*r1, *r2)
            }
            (Type::Tuple(t1), Type::Tuple(t2)) => {
                if t1.len() != t2.len() {
                    return Err(TypeError::TupleSizeMismatch);
                }
                for (ty1, ty2) in t1.into_iter().zip(t2.into_iter()) {
                    self.unify(ty1, ty2)?;
                }
                Ok(())
            }
            // ... handle all type combinations
        }
    }
}
```

### Refinement Types

```rust
// engine/src/type_system/refinement.rs
pub struct RefinementType {
    base: Type,
    predicate: Predicate,
}

pub struct RefinementChecker {
    smt_solver: SmtSolver,
    context: RefinementContext,
}

impl RefinementChecker {
    pub fn check_refinement(&mut self, value: &Expr, ty: &RefinementType)
        -> Result<(), RefinementError> {
        // First check base type
        let value_ty = self.infer_base_type(value)?;
        if !self.subtype(&value_ty, &ty.base)? {
            return Err(RefinementError::BaseTypeMismatch);
        }

        // Then check refinement predicate
        let formula = self.generate_verification_condition(value, &ty.predicate);

        match self.smt_solver.check_sat(formula) {
            SatResult::Sat => Ok(()),
            SatResult::Unsat => Err(RefinementError::PredicateFailed),
            SatResult::Unknown => Err(RefinementError::SolverTimeout),
        }
    }

    fn generate_verification_condition(&self, value: &Expr, pred: &Predicate)
        -> Formula {
        let var = self.fresh_var();
        let binding = Formula::Eq(var.clone(), self.expr_to_smt(value));
        let predicate = self.predicate_to_smt(pred, &var);
        Formula::Implies(Box::new(binding), Box::new(predicate))
    }
}
