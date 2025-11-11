# Wasmtime Code Relations and Architecture

This document explains how the code in the Wasmtime project is structured and how different components relate to each other.

## Overview

Wasmtime is a WebAssembly runtime built with a modular architecture. The code is organized into multiple crates that work together to provide:
1. **Compilation** - Converting WebAssembly to native machine code
2. **Runtime** - Executing compiled WebAssembly modules
3. **WASI** - Providing system interfaces to WebAssembly modules
4. **Tooling** - CLI and utilities for working with WebAssembly

## Core Architecture Flow

```
WebAssembly Binary
       ↓
[wasmparser] - Parsing and validation
       ↓
[wasmtime-environ] - IR generation and metadata
       ↓
[wasmtime-cranelift] - Compilation to machine code (or [wasmtime-winch] for baseline)
       ↓
[wasmtime] - Runtime execution
```

## Key Components and Their Relationships

### 1. Compilation Pipeline

#### wasmtime-environ (`crates/environ/`)
- **Role**: Defines the intermediate representation (IR) and compilation interface
- **Key Types**:
  - `Module` - Metadata about a WebAssembly module
  - `ModuleEnvironment` - Builds module metadata during parsing
  - `Compiler` trait - Interface that compilers must implement
  - `VMOffsets` - Memory layout calculations for runtime structures
- **Relations**:
  - Uses `wasmparser` to parse WebAssembly binaries
  - Defines structures that both compilers and runtime use
  - Acts as the bridge between parsing and compilation

#### wasmtime-cranelift (`crates/cranelift/`)
- **Role**: Optimizing compiler backend using Cranelift
- **Key Types**:
  - `Compiler` - Implements `wasmtime_environ::Compiler`
  - `FuncEnvironment` - Tracks state during function compilation
  - Trap handling and relocation generation
- **Relations**:
  - Implements the `Compiler` trait from `wasmtime-environ`
  - Uses Cranelift's IR and code generation
  - Produces machine code and metadata for the runtime
  - Depends on: `cranelift-codegen`, `cranelift-frontend`, `wasmtime-environ`

#### Cranelift (`cranelift/`)
- **Role**: Low-level code generator (separate project)
- **Key Crates**:
  - `cranelift-codegen` - Code generation and optimization
  - `cranelift-frontend` - High-level IR builder
  - `cranelift-entity` - Entity-based data structures
- **Relations**:
  - Independent code generator used by wasmtime-cranelift
  - Can be used standalone for non-WebAssembly code generation

### 2. Runtime System

#### wasmtime (`crates/wasmtime/`)
- **Role**: Main API crate and runtime execution engine
- **Key Types**:
  - `Engine` - Global compilation environment, thread-safe
  - `Store<T>` - Per-instance container for WebAssembly objects
  - `Module` - Compiled WebAssembly module
  - `Instance` - Instantiated module with memory/tables/etc
  - `Func`, `Memory`, `Table`, `Global` - WebAssembly objects
  - `Linker` - Connects imports to host functions
- **Code Organization**:
  - `runtime/` - Core execution engine
    - `vm/` - Low-level VM structures and runtime calls
    - `func.rs` - Function calling and trampolines
    - `instantiate.rs` - Module instantiation logic
    - `gc.rs` - Garbage collection for GC references
    - `linker.rs` - Import resolution
  - `compile/` - Integration with compilers
  - `engine/` - Engine configuration and management
- **Relations**:
  - Uses compiled code from `wasmtime-cranelift` or `wasmtime-winch`
  - Depends on `wasmtime-environ` for module metadata
  - Provides embedder API for host applications

### 3. WASI Implementation

#### wasmtime-wasi (`crates/wasi/`)
- **Role**: WASI Preview 2 (Component Model) implementation
- **Key Types**:
  - `WasiCtx` - Context holding WASI state (stdio, filesystem, etc)
  - `WasiView` trait - Access to WASI context from store
  - Generated bindings for WASI interfaces
- **Relations**:
  - Implements WASI interfaces using host OS capabilities
  - Uses `wasmtime::component::Linker` for component imports
  - Depends on `cap-std` for capability-based security

#### wasi-common (`crates/wasi-common/`)
- **Role**: WASI Preview 1 (legacy) implementation
- **Relations**:
  - Legacy WASI implementation for older modules
  - Being phased out in favor of wasmtime-wasi

#### Specialized WASI Crates
- `wasmtime-wasi-nn` - Neural network support
- `wasmtime-wasi-http` - HTTP client/server
- `wasmtime-wasi-threads` - Threading support
- `wasmtime-wasi-config` - Configuration access
- `wasmtime-wasi-keyvalue` - Key-value storage
- `wasmtime-wasi-tls` - TLS support

### 4. Component Model

#### component (`wasmtime/src/runtime/component/`)
- **Role**: WebAssembly Component Model implementation
- **Key Types**:
  - `Component` - Compiled component (like `Module` for core wasm)
  - `Instance` - Component instance
  - `Linker` - Component import resolution
  - `Func`, `Resource` - Component-level types
- **Relations**:
  - Parallel API to core WebAssembly in the `wasmtime` crate
  - Uses the same `Engine` and `Store` as core wasm
  - Depends on `wit-parser` and `wit-component` for interface definitions

### 5. Supporting Infrastructure

#### wasmtime-cli (`/` - root crate)
- **Role**: Command-line interface
- **Key Commands**:
  - `run` - Execute WebAssembly modules/components
  - `compile` - Ahead-of-time compilation
  - `serve` - HTTP server for components
  - `wast` - Run WebAssembly test scripts
- **Relations**:
  - Uses `wasmtime` crate for execution
  - Uses `wasmtime-wasi` for WASI support
  - Entry point: `src/bin/wasmtime.rs`

#### wasmtime-cache (`crates/cache/`)
- **Role**: Persistent module cache
- **Relations**:
  - Caches compiled modules to disk
  - Used by `Engine` when cache is enabled

#### Internal Utility Crates
- `wasmtime-fiber` - Fiber/stack switching for async
- `wasmtime-jit-debug` - Debug info integration
- `wasmtime-jit-icache-coherence` - Cache coherency on some platforms
- `wasmtime-slab` - Slab allocator for runtime objects
- `wasmtime-unwinder` - Stack unwinding for traps

## Key Code Relationships

### Compilation Flow

```
1. Host calls Module::new(&engine, wasm_bytes)
   ↓
2. wasmtime-environ parses with wasmparser
   ↓
3. ModuleEnvironment builds IR and metadata
   ↓
4. wasmtime-cranelift::Compiler::compile_function()
   ↓
5. Cranelift generates machine code
   ↓
6. Module stores compiled code + metadata
```

### Instantiation Flow

```
1. Host calls Linker::instantiate(&mut store, &module)
   ↓
2. Runtime allocates Instance structure
   ↓
3. Instance::initialize() sets up:
   - Linear memory (Memory)
   - Tables (Table)
   - Globals (Global)
   - Function pointers
   ↓
4. Runs start function if present
   ↓
5. Returns Instance handle
```

### Function Call Flow

```
1. Host calls func.call(&mut store, params)
   ↓
2. TypedFunc validates parameter types
   ↓
3. Trampoline handles calling convention conversion
   ↓
4. Compiled WebAssembly code executes
   ↓
5. May call back to host functions via Store
   ↓
6. Results returned through trampoline
```

### Store and Context Relationships

The `Store<T>` is central to the runtime:
- Owns all WebAssembly objects (`Func`, `Memory`, etc.)
- Provides access through `AsContext`/`AsContextMut` traits
- Types like `Caller<T>` borrow from `Store<T>`
- Host data `T` is accessible in host functions

```rust
Store<T>
  ├─ Engine (shared, read-only)
  ├─ StoreInner
  │   ├─ Instance data
  │   ├─ Memory data
  │   ├─ Table data
  │   └─ GC heap
  └─ T (user data)
```

### WASI Integration

```
1. Host creates WasiCtx with stdio/filesystem/etc
   ↓
2. Store created with WasiCtx as user data
   ↓
3. Linker populated with WASI functions
   ↓
4. Module instantiated with WASI imports
   ↓
5. WASI functions access WasiCtx via Caller<WasiCtx>
   ↓
6. WASI implements operations using cap-std
```

## Trait-based Abstractions

### Compiler Abstraction
```rust
// wasmtime-environ defines:
trait Compiler {
    fn compile_function(&self, ...) -> CompiledFunction;
}

// wasmtime-cranelift implements it
// wasmtime-winch implements it (baseline compiler)
```

### Context Abstraction
```rust
// wasmtime defines:
trait AsContext {
    type Data;
    fn as_context(&self) -> StoreContext<'_, Self::Data>;
}

// Implemented by: Store, Caller, StoreContext, etc.
```

### WASI View Abstraction
```rust
// wasmtime-wasi defines:
trait WasiView {
    fn ctx(&mut self) -> &mut WasiCtx;
    fn table(&mut self) -> &mut ResourceTable;
}

// Implemented by types in Store's user data
```

## Component Model Architecture

The Component Model adds a layer on top of core WebAssembly:

```
Component (wit interfaces)
    ↓
Lowered to core WebAssembly modules
    ↓
Adapter modules handle interface calls
    ↓
Core runtime executes
```

Key relations:
- `wit-parser` - Parses `.wit` interface files
- `wit-component` - Component encoding/decoding
- `wasmtime-component-macro` - Procedural macro for bindgen
- `wasmtime::component` - Runtime support

## Memory Safety Architecture

Wasmtime maintains safety through:

1. **Ownership**: `Store` owns all WebAssembly objects
2. **Handles**: Types like `Func` are lightweight handles
3. **Context passing**: Every operation requires `&mut Store`
4. **Bounds checking**: Memory accesses validated
5. **Signal handling**: Traps caught via OS signals

## Build and Testing Infrastructure

- `build.rs` - Build-time code generation
- `crates/test-programs/` - Test WebAssembly modules
- `crates/fuzzing/` - Fuzzing harness
- `tests/` - Integration tests
- `benches/` - Performance benchmarks

## Concrete Code Examples

### Basic Module Execution

Here's how the main types relate in a simple execution:

```rust
// 1. Create Engine - global compilation environment
let engine = Engine::default();

// 2. Compile Module - uses wasmtime-environ + wasmtime-cranelift
let module = Module::from_file(&engine, "example.wat")?;

// 3. Create Store - runtime container with user data
let mut store = Store::new(&engine, MyState { count: 0 });

// 4. Define host function - accessed via Caller
let host_func = Func::wrap(&mut store, |mut caller: Caller<'_, MyState>| {
    caller.data_mut().count += 1;  // Access user data
    println!("Called from wasm!");
});

// 5. Instantiate - creates Instance with Memory, Tables, etc.
let instance = Instance::new(&mut store, &module, &[host_func.into()])?;

// 6. Get exported function - type-safe handle
let run = instance.get_typed_func::<(i32, i32), i32>(&mut store, "add")?;

// 7. Call function - executes compiled code
let result = run.call(&mut store, (5, 7))?;
```

**Code Relations in this flow**:
- `Engine` holds compiled `Module` (wasmtime → wasmtime-environ)
- `Module::from_file` calls parser → environ → cranelift
- `Store` borrows `Engine` and owns instance data
- `Func::wrap` creates trampoline (in wasmtime/runtime/func.rs)
- `Instance::new` uses linker logic (wasmtime/runtime/instantiate.rs)
- `TypedFunc::call` goes through trampolines to compiled code

### Linker Pattern

For complex imports, use `Linker`:

```rust
let engine = Engine::default();
let mut linker = Linker::new(&engine);

// Define multiple host functions
linker.func_wrap("env", "print", |caller: Caller<'_, ()>| {
    println!("print called");
})?;

linker.func_wrap("env", "add", |_: Caller<'_, ()>, a: i32, b: i32| -> i32 {
    a + b
})?;

// Instantiate with all imports at once
let mut store = Store::new(&engine, ());
let module = Module::from_file(&engine, "example.wat")?;
let instance = linker.instantiate(&mut store, &module)?;
```

**Code Relations**:
- `Linker` (wasmtime/runtime/linker.rs) stores import definitions
- `Linker::instantiate` resolves imports and calls `Instance::new`
- Host functions registered once, reused for multiple instances

### WASI Example

WASI integration shows how host interfaces work:

```rust
use wasmtime_wasi::WasiCtxBuilder;

let engine = Engine::default();
let mut linker = Linker::new(&engine);

// Add all WASI functions to linker
wasmtime_wasi::add_to_linker_sync(&mut linker)?;

// Create WASI context with stdio, filesystem, etc.
let wasi = WasiCtxBuilder::new()
    .inherit_stdio()
    .inherit_args()?
    .build();

let mut store = Store::new(&engine, wasi);
let module = Module::from_file(&engine, "wasi_program.wasm")?;
let instance = linker.instantiate(&mut store, &module)?;

// Call WASI program's main
let start = instance.get_typed_func::<(), ()>(&mut store, "_start")?;
start.call(&mut store, ())?;
```

**Code Relations**:
- `wasmtime_wasi::add_to_linker_sync` registers WASI functions
- WASI functions (in wasmtime-wasi/src/) access `WasiCtx` via `Caller`
- `WasiCtx` uses `cap-std` for filesystem operations
- Store's user data type implements `WasiView` trait

### Component Model Example

Components have parallel structure to core modules:

```rust
use wasmtime::component::*;

let engine = Engine::default();
let mut linker = Linker::new(&engine);

// Add component imports
linker.root().func_wrap("greet", |_cx, (name,): (String,)| {
    Ok((format!("Hello, {}!", name),))
})?;

let component = Component::from_file(&engine, "component.wasm")?;
let mut store = Store::new(&engine, ());
let instance = linker.instantiate(&mut store, &component)?;

// Call component function
let func = instance.get_typed_func::<(String,), (String,)>(&mut store, "run")?;
let (result,) = func.call(&mut store, ("World".to_string(),))?;
```

**Code Relations**:
- `component::Component` parallels `Module`
- `component::Linker` handles interface resolution
- Component code in wasmtime/src/runtime/component/
- Uses same `Engine` and `Store` as core wasm

### Compiler Abstraction Example

Switching compilers (internal architecture):

```rust
// In wasmtime-cranelift/src/compiler.rs:
impl wasmtime_environ::Compiler for Compiler {
    fn compile_function(
        &self,
        translation: &ModuleTranslation,
        index: DefinedFuncIndex,
        data: FunctionBodyData,
        tunables: &Tunables,
    ) -> Result<CompiledFunction> {
        // Use Cranelift to compile function
        let func = translate_wasm_function(data, ...)?;
        let code = cranelift_codegen::compile(&func, ...)?;
        Ok(CompiledFunction { code, ... })
    }
}

// In wasmtime/src/compile.rs:
// Engine selects compiler based on Config
let compiler: Box<dyn Compiler> = if config.use_cranelift {
    Box::new(wasmtime_cranelift::builder())
} else {
    Box::new(wasmtime_winch::builder())
};
```

**Code Relations**:
- `Compiler` trait in wasmtime-environ defines interface
- wasmtime-cranelift and wasmtime-winch implement it
- `Engine` holds the compiler implementation
- `Module::new` calls `compiler.compile_function()` for each function

### Async Execution Example

Async shows Store's flexibility:

```rust
let mut config = Config::new();
config.async_support(true);
let engine = Engine::new(&config)?;

let module = Module::from_file(&engine, "async.wasm")?;
let mut store = Store::new(&engine, ());

// Host async function
let func = Func::wrap_async(
    &mut store,
    |_caller, param: i32| Box::new(async move {
        tokio::time::sleep(Duration::from_secs(1)).await;
        Ok(param * 2)
    })
);

let instance = Instance::new(&mut store, &module, &[func.into()])?;
let run = instance.get_typed_func::<(), ()>(&mut store, "run")?;

// Async call
run.call_async(&mut store, ()).await?;
```

**Code Relations**:
- Config enables async support (uses wasmtime-fiber for stack switching)
- `Func::wrap_async` creates async trampolines
- `call_async` returns Future that yields to executor
- Fiber implementation in wasmtime/crates/fiber/

## Summary

The key to understanding Wasmtime's code relations:

1. **Layered architecture**: Parsing → Compilation → Runtime
2. **Trait-based abstraction**: Compilers, contexts, WASI views
3. **Store-centric design**: All WebAssembly state in Store
4. **Component Model parallel**: Separate but similar API
5. **Safety through ownership**: Rust's type system enforces correctness

Each crate has a focused responsibility, with well-defined interfaces between them. This modularity allows:
- Swapping compilers (Cranelift vs Winch)
- Multiple WASI implementations
- Embedding in various contexts
- Testing components independently
