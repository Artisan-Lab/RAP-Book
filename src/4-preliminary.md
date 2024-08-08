# Chatper 4. Preliminary: Compiler Internals

## Compiler Internal
We list the essential official documents to read for better understanding the internal data structures of Rust compiler.
 - [**HIR**](https://rustc-dev-guide.rust-lang.org/hir.html)
 - [**MIR**](https://rustc-dev-guide.rust-lang.org/mir/index.html)
 - [**TyCtxt**](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TyCtxt.html) is the central data structure of Rust compilers. We can obtain the hir or mir of a function based on the object.
```rust
let hir = tcx.hir();
let mir = optimized_mir(def_id); // def_id is of type DefId
```

## Command to Display HIR/MIR
Execute the following command to obtain the HIR/MIR of the source code.
```
cargo rustc -- -Z unpretty=hir-tree
cargo rustc -- -Zunpretty=mir
```
