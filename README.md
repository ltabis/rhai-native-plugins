# Rhai Dylib

This crate exposes a simple API to load `dylib` Rust crates in a [Rhai](https://rhai.rs/) engine using [Rhai modules](https://rhai.rs/book/rust/modules/index.html).

> 🚧 This is a work in progress, the API is subject to change. Please do make recommendations on what you want it to be via issues, discussions or pull requests !

## Loader

`PluginLoader` is a trait that is used to build objects that load plugins in memory. A [libloading](https://github.com/nagisa/rust_libloading) implementation is available, which enables you to load plugins via a `cdylib` or `dylib` rust crate.

Check the `simple` example for more details.

## Module Resolver

This crate also expose a [Rhai Module Resolver](https://rhai.rs/book/rust/modules/resolvers.html) that loads dynamic libraries at the given path.

```rust
use rhai_dylib::DylibModuleResolver;

let mut engine = rhai::Engine::new();

// use `rhai::module_resolvers::ModuleResolversCollection` if you need to resolve using
// other resolvers.
// Check out https://docs.rs/rhai/latest/rhai/module_resolvers/struct.ModuleResolversCollection.html
engine.set_module_resolver(DylibModuleResolver::new());

engine.run(r#"
// import your dynamic library.
import "path/to/my/dylib";

// ...
"#).expect("failed to run script");
```

Check the `module_resolver` example for more details.

## Pitfalls

There are multiple limitations with this implementation.

> TL;DR
> To use this crate, you need to:
> - Compile **EVERYTHING**, plugins and program that will load them, inside the **SAME** workspace or **WITHOUT** a workspace.
> - Export the `RHAI_AHASH_SEED` environment variable with the **SAME** four u64 array (i.e. `RHAI_AHASH_SEED="[1, 2, 3, 4]"`) when building your plugins and the program that will load them.

### TypeId

Rust [`TypeId`](https://doc.rust-lang.org/std/any/struct.TypeId.html) is an object used to compare types at compile time. Rhai uses those to check which type a [`Dynamic`](https://docs.rs/rhai/1.10.1/rhai/struct.Dynamic.html) object is. This is a problem for dynamic libraries because `TypeIds` sometime change between compilations.

That means that in certain situations, Rhai cannot compare two types, even tough they are the same, because the `TypeId` of said types is different between the plugin and the binary.

To fix this, you will need to compile your main binary **AND** plugins inside the **SAME** workspace, or compile everything **OUTSIDE** of a workspace. Compiling, for example, a binary in a workspace, and a plugin outside will probably result in `TypeIds` mismatch.

>  You can use
> ```rust
> println!("{:?}", std::any::TypeId::of::<rhai::Map>());
> ```
> In your binary & plugins to check the type id value.

If you have any idea of how the compiler generates those typeids between workspaces and single crates, please help us complete this readme !

### Hashing

Rhai uses the [`ahash`](https://github.com/tkaitchuck/ahash) crate under the hood to create identifiers for function calls. For each compilation of your code, a new seed is generated when hashing the types. Therefore, compiling your main program and your plugin different times will result in a hash mismatch, meaning that you won't be able to call the API of your plugin.

To bypass that, you need to inject the `RHAI_AHASH_SEED` environment variable with an array of four `u64`.

```sh
export RHAI_AHASH_SEED="[1, 2, 3, 4]" # The seed is now fixed and won't change between compilations.

# Compiling will create the same hashes for functions.
cargo build --manifest-path ./my_program/Cargo.toml
cargo build --manifest-path ./my_plugin/Cargo.toml
```

instead of exporting the variable like above, you can use a [cargo config](https://doc.rust-lang.org/cargo/reference/config.html) file.

```toml
# .cargo/config.toml
[env]
# Replace the seed to your own liking, it must be the same for every plugins.
RHAI_AHASH_SEED = "[1, 2, 3, 4]"
```

Beware: the code handling the `RHAI_AHASH_SEED` environment variable is not yet merged into the main branch of Rhai.
This crate uses a personal fork of [schungx](https://github.com/schungx/rhai) for the time being.

## Rust ABI

You also can implement a plugin using the Rust ABI, which is unstable and will change between compiler versions.

This means that all of the plugins that you will use in your main program need to be compiled with the **EXACT** same
compiler version.

## TODO

Here is a list of stuff that we could implement or think about. (to move in issues)

- [ ] How could we "restrain" the API access of the rhai engine ? Lock those behind features ? Using a new type that wraps the engine (Proxy) ?
- [ ] Lock plugin loaders behind features.
- [ ] Create macros that generate entry points.
- [ ] Update seeds for ahash.
- [ ] Add some unit & integration tests.
- [ ] the "internals" feature is necessary for the impl of the module resolver. We should directly merge that into Rhai itself.