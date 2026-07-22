# Neon for Zed

Registers `.neon` files as a language and starts [`neon-lsp`](../../lsp) against them.

**There is no syntax highlighting.** That is not an oversight, and as of now it is a
packaging problem rather than a missing grammar; see
[Why there is no highlighting](#why-there-is-no-highlighting) below. Everything else the
server offers does work, because Zed reads capabilities from the `initialize` response
and nothing in this extension has to declare them.

| Feature | Source |
| --- | --- |
| `.neon` recognised as the "Neon" language | `languages/neon/config.toml` |
| Inline diagnostics | `neon-lsp` (`publishDiagnostics`) |
| Format buffer / format on save | `textDocument/formatting` |
| Hover — type or signature, plus the `///` doc comment | `hoverProvider` |
| Go to definition, including into stdlib files | `definitionProvider` |
| Find all references (shadowing-correct) | `referencesProvider` |
| Rename symbol | `renameProvider` |
| Completion (locals and visible fns, with signatures and docs) | `completionProvider`, trigger `:` |
| Signature help | `signatureHelpProvider`, triggers `(` and `,` |
| Outline / symbol search, nested under `mod` and `impl` | `documentSymbolProvider` |
| Inlay hints — types on un-annotated `let`s | `inlayHintProvider` |
| `//` and `///` comment toggling and continuation | `languages/neon/config.toml` |
| `/* */` block comments, bracket matching, 4-space indent | `languages/neon/config.toml` |
| Structural selection, syntax folding, semantic highlighting | **absent** — needs the grammar |

The authoritative capability list is the `ServerCapabilities` literal near the top of
`lsp/src/main.rs`. It is not duplicated in `extension.toml`, deliberately: the copy that
used to live there said "diagnostics and formatting" and stayed wrong through eight
capabilities being added.

Rename declines a symbol whose definition lives in another file and returns an LSP error
rather than a partial edit. That is intended — a rename that silently missed the
definition would be worse than one that refused.

Inlay hints are controlled by Zed's own `inlay_hints` setting, per language:

```json
{
  "languages": {
    "Neon": { "inlay_hints": { "enabled": true } }
  }
}
```

## Install as a dev extension

The extension is a Rust crate that Zed compiles to WebAssembly (`wasm32-wasip2`); its own
repository, with its own `Cargo.toml`.

1. Install the wasm target once:

   ```sh
   rustup target add wasm32-wasip2
   ```

2. Build `neon-lsp` and put it somewhere on your `$PATH`:

   ```sh
   cargo build --release -p neon-lsp
   # then, e.g.
   ln -sf "$PWD/target/release/neon-lsp" ~/.local/bin/neon-lsp
   ```

3. In Zed, open the command palette and run **`zed: install dev extension`**, then select
   this directory (this repo). Zed compiles the crate itself; you do not need to run
   `cargo build` here.

After editing anything in this directory, run **`zed: reload extensions`**.

## Pointing it at the toolchain

Two things must be found: the server binary, and the sysroot.

### The binary

Resolved in this order:

1. `lsp.neon-lsp.binary.path` in your Zed settings.
2. `neon-lsp` on `$PATH`.

If neither resolves, the extension fails with a message naming both options rather than
launching something that does not exist.

### `NEON_SYSROOT` — read this one

`neon-lsp` loads the standard library from `$NEON_SYSROOT/stdlib`. If that variable is
unset or wrong, **the server still starts and still reports diagnostics** — but
`load_stdlib` returns nothing, the type checker is skipped entirely, and you silently get
lexer and parser errors only. Nothing in the editor indicates this. If Neon files show
syntax errors but never type errors, this is why.

The extension resolves it in this order:

1. `lsp.neon-lsp.binary.env.NEON_SYSROOT` in your Zed settings.
2. `NEON_SYSROOT` inherited from your shell environment.
3. Auto-detected: if the worktree root contains `stdlib/prelude.neon`, the worktree root is
   used. Opening this repository therefore works with no configuration at all.

If none apply, the variable is left unset rather than guessed at.

To set it explicitly, in Zed's `settings.json`:

```json
{
  "lsp": {
    "neon-lsp": {
      "binary": {
        "path": "/absolute/path/to/neon-lsp",
        "env": { "NEON_SYSROOT": "/absolute/path/to/neon/checkout" }
      }
    }
  }
}
```

To format on save, add:

```json
{
  "languages": {
    "Neon": { "format_on_save": "on" }
  }
}
```

A file that does not parse is left untouched by the formatter — it reprints from the AST,
so a half-written line produces no edits rather than an error popup.

## Highlighting

Wired. The grammar is [`tree-sitter-neon`](https://github.com/neon-tube/tree-sitter-neon),
its own repository whose root *is* the grammar — exactly the layout Zed's `[grammars.neon]`
fetch (by `repository` + `rev`) needs. `extension.toml` pins it by commit sha, and
`languages/neon/config.toml` sets `grammar = "neon"`.

The highlight queries Zed uses are in `languages/neon/highlights.scm` — a copy of the
grammar's `queries/highlights.scm` with **coarse capture duplicates** added. The grammar is
written in Neovim's fine-grained vocabulary (`@keyword.conditional`, `@variable.member`,
`@type.definition`, `@number.float`, …), and Zed maps a capture it does not recognise to
*nothing*, so each fine capture carries its coarse root (`@keyword`, `@variable`, `@type`,
`@number`, …) beside it. Predicate lines (`#any-of?`) are left untouched. Only
`highlights.scm` is brought across so far; Zed-flavoured `indents.scm`/`outline.scm` are a
later refinement — bracket and indent behaviour come from `config.toml` in the meantime.

The grammar's external scanner is mandatory, not optional: Neon's block comments nest, no
regular expression can count, and the depth is tracked in `src/scanner.c`, which Zed
compiles automatically because it is present in `src/`.

**A warning about older grammars.** An early Neon grammar circulated with different node
names (`binary_expr` vs `binary_expression`, `int_literal` vs `integer`, `enum_declaration`,
`if_let_expr`, …). Queries written for the current grammar do not degrade against it, they
fail to compile (`Invalid node type "doc_comment"`). If you pin a `rev`, pin one from the
current [tree-sitter-neon](https://github.com/neon-tube/tree-sitter-neon) history.

## What has and has not been verified

Verified on this machine:

- `cargo build --release --target wasm32-wasip2` succeeds and produces a wasm component;
  `cargo clippy` is clean. Resolved `zed_extension_api` 0.7.0, the current published
  release and the same version other installed extensions use.
- Both TOML files parse, and every key used was checked field-by-field against Zed's
  `ExtensionManifest`, `GrammarManifestEntry` and `LanguageConfig` structs in Zed's source
  rather than against the documentation. (The docs claim `grammar` is required in a
  language config; the source says otherwise, and the source is what runs.)

Not verified:

- **The extension has not been loaded into a running Zed.** No `zed` binary was reachable
  from this shell, so "installs and attaches successfully" is reasoned from Zed's source,
  not observed. Confirm with `zed: install dev extension` and check the language server
  logs (`zed: open log`).
- **The feature table above is a capability list, not an observation.** Every row was read
  off the `ServerCapabilities` literal in `lsp/src/main.rs`, and the ten capabilities were
  confirmed to arrive in an `initialize` response — but by a Neovim client, not by Zed.
  Zed consuming each of them is expected rather than seen.
- **Whether a newer Zed offers a local-path grammar option.** The claim that
  `GrammarManifestEntry` requires `repository` + `rev` is inherited from the earlier
  check against Zed's source and could not be re-checked here: no `zed` binary, no network,
  and `zed_extension_api` 0.7.0 carries nothing about grammars (they are resolved entirely
  on Zed's host side, not through the guest API). If you can reach Zed's source, re-check
  it before trusting that paragraph — it is the one claim here with no live evidence behind
  it.
