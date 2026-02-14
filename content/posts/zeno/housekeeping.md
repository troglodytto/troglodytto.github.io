+++
date = '2026-02-07T00:16:14+05:30'
draft = false
title = 'A scalable structure for our code'
slug = 'housekeeping'
toc = false
+++

📝 This post is tangential to the original series. If you'd like to continue reading, you can find the next post [**here**](../serial-output).

---

So far we've been writing our code in a single place because we were just experimenting with things.

But now, before we continue building features, we need to pause and set up a scalable foundation. Kernel projects grow quickly and without structure, they become painful to maintain. In this post, we'll perform a few housekeeping tasks to keep our code organized, robust, and future-proof.

Nothing new conceptually today. We're just organizing.

We'll break this down into three sections.

- [Restructuring the Kernel Crate](#restructuring-the-kernel-crate)
- [Enforcing Code Quality with Clippy](#enforcing-code-quality-with-clippy)
- [Developer Experience Improvements](#developer-experience-improvements)

## Restructuring the Kernel Crate

We'll keep our module structure fairly simple.

We'll have a central `lib.rs` which exports all its submodules. Those submodules will then be used by the `main.rs` file under a consolidated `kernel` package.

That way, sharing code among different modules would be easier.

The `main.rs` will only act as a driver and the entry point to our kernel, and the rest of the functionality will be delegated to the `lib.rs` module.

To do this, we first create the `kernel/src/lib.rs` file, and import it in the `main.rs` file as a module. We'll also add a stub init function which will grow as we add more functionality.

```rs {title="kernel/src/lib.rs"}
#![no_std] // 👈 remember to add the #![no_std] attribute in your lib.rs file as well

//! Zeno - Minimal x86-64 Operating System Kernel

use bootloader_api::BootInfo;

/// Initialize the Zeno OS Kernel by implementing the following:
/// -
pub fn init(_: &'static mut BootInfo) {
    // we'll initialize kernel sub-systems here
}
```

```diff {title="kernel/src/main.rs"}
...
- fn launch(_: &'static mut BootInfo) -> ! {
+ fn launch(boot_info: &'static mut BootInfo) -> ! {
+   kernel::init(boot_info);
    main();
...
```

Now that we've done that, we'll write most of our code in separate modules, initialize them in the `init` function, and then use the `main` and `launch` functions to drive them.

## Enforcing Code Quality with Clippy

To set our project up in such a way that it pushes us towards writing more robust, correct and idiomatic code, we'll add some lint rules. These lint rules will allow us to catch bugs early that we'd otherwise ignore as well as allow us to discover more idiomatic Rust code. Clippy has a bunch of lint rules that focus on correctness, style, complexity, and performance.

Initially, we'll create a very strict set of lint rules that disallow a bunch of things. We'll later disable the rules that start to become annoyances rather than guardrails.

Add these lints at the top of your `lib.rs` (and additionally, your `main.rs` file)

Don't worry about understanding each lint individually, you can copy this block as-is. We'll encounter them naturally as we build features.

```rs {title="kernel/src/lib.rs (and kernel/src/main.rs)"}
#![deny(
    clippy::pedantic,
    clippy::all,
    clippy::nursery,
    clippy::await_holding_lock,
    clippy::char_lit_as_u8,
    clippy::checked_conversions,
    clippy::dbg_macro,
    clippy::debug_assert_with_mut_call,
    clippy::doc_markdown,
    clippy::empty_enums,
    clippy::enum_glob_use,
    clippy::exit,
    clippy::expl_impl_clone_on_copy,
    clippy::explicit_deref_methods,
    clippy::explicit_into_iter_loop,
    clippy::fallible_impl_from,
    clippy::filter_map_next,
    clippy::flat_map_option,
    clippy::float_cmp_const,
    clippy::fn_params_excessive_bools,
    clippy::from_iter_instead_of_collect,
    clippy::if_let_mutex,
    clippy::implicit_clone,
    clippy::imprecise_flops,
    clippy::inefficient_to_string,
    clippy::invalid_upcast_comparisons,
    clippy::large_digit_groups,
    clippy::large_stack_arrays,
    clippy::large_types_passed_by_value,
    clippy::let_unit_value,
    clippy::linkedlist,
    clippy::lossy_float_literal,
    clippy::macro_use_imports,
    clippy::manual_ok_or,
    clippy::map_err_ignore,
    clippy::map_flatten,
    clippy::map_unwrap_or,
    clippy::match_same_arms,
    clippy::match_wild_err_arm,
    clippy::match_wildcard_for_single_variants,
    clippy::mem_forget,
    clippy::missing_enforced_import_renames,
    clippy::mut_mut,
    clippy::mutex_integer,
    clippy::needless_borrow,
    clippy::needless_continue,
    clippy::needless_for_each,
    clippy::option_option,
    clippy::path_buf_push_overwrite,
    clippy::ptr_as_ptr,
    clippy::rc_mutex,
    clippy::ref_option_ref,
    clippy::rest_pat_in_fully_bound_structs,
    clippy::same_functions_in_if_condition,
    clippy::semicolon_if_nothing_returned,
    clippy::single_match_else,
    clippy::string_add_assign,
    clippy::string_add,
    clippy::string_lit_as_bytes,
    clippy::todo,
    clippy::trait_duplication_in_bounds,
    clippy::unimplemented,
    clippy::unnested_or_patterns,
    clippy::unused_self,
    clippy::useless_transmute,
    clippy::verbose_file_reads,
    clippy::zero_sized_map_values,
    clippy::redundant_pattern,
    clippy::missing_panics_doc,
    clippy::missing_errors_doc,
    clippy::empty_docs,
    clippy::missing_safety_doc,
    future_incompatible,
    nonstandard_style,
    rust_2018_idioms,
    unused
)]
```

Phew! That many huh?

Well, we do want to be as strict as possible. And in any case, if we feel that a lint becomes too strict for a specific piece of functionality i.e., it becomes a hindrance rather than a guardrail, we can disable it selectively.

Additionally, instead of `warn`, we've marked all the lints as `deny`, meaning violation of any of these lints will be considered a hard error.

It might feel too strict at first, but future-you will thank you.

As soon as you add these lints, and run the `cargo clippy` command, you'll see a bunch of things popping up. Don't worry, it's just a bunch of warnings that we can fix by adding `const` in front of the `init` and `panic_handler`

> 💡 **Making sure our editor is also on the same page (regarding these lints)**
>
> Some LSP enabled editors only run the `cargo check` command when checking for any issues. Unfortunately for us, that means we won't know we've broken any of our lint rules unless we run the `cargo clippy` command ourselves.
>
> To prevent this, we should configure our editor such that it runs the `cargo clippy` command instead, so that whenever we are in violation of any of these lints, the editor will highlight the error and prompt us to fix it immediately, instead of us having to run the command ourselves and then fixing them later.
>
> - [Configuring Zed](https://github.com/zed-industries/zed/issues/8557#issuecomment-2842201234)
> - [Configuring Visual Studio Code](https://users.rust-lang.org/t/how-to-use-clippy-in-vs-code-with-rust-analyzer/41881)
>
> These are the editors that I personally use, so if you're using something else, please refer to the respective documentation for your editor.

## Developer Experience Improvements

To make our lives easier down the line, we'll also install the `x86_64` crate since it provides abstractions for a lot of x86-64 architecture primitives (such as register read/writes, flag set/unset, etc) so it'll be helpful for us later down the line.

Writing inline assembly is a bit tedious and error prone. Additionally, it requires a lot of `unsafe` blocks in our code, which will make audits more difficult. The `x86_64` crate will provide safe abstractions for these operations, making our code more readable and maintainable.

```fish
# in the kernel directory
cargo add x86_64
```

While we're at it, we should also add the `hlt` instruction in our launch function, to prevent the CPU from spinning indefinitely.

```diff {title="kernel/src/main.rs"}
fn launch(boot_info: &'static mut BootInfo) -> ! {
    kernel::init(boot_info);
    main();

    loop {
+       x86_64::instructions::hlt();
    }
}
```

---

With that, we're all set to continue working on our OS.

We now have:

- A modular kernel architecture
- Strict lint enforcement (via Clippy)
- Architecture abstractions (via the `x86_64` crate)

Let's continue where we left off and implement [Serial Output and Debugging](../serial-output) via Port I/O.
