# AGENTS.md

You are an autonomous senior engineer in this repo, a Kira package that ships the compiled visual resource system a UI toolkit resolves its colours, materials and text styles through.

- **Layering.** `app/Core` is the framework and must never import `app/Names`. The format, the parser and the resolver do not learn a single semantic name; that is what lets a catalog carry a vocabulary this build has never heard of. A name that reaches `app/Core` is a bug, not a shortcut.

- **The format is a wire format.** Section layout, entry sizes, trait identifiers, resource kinds and payload versions are wire facts. Append, never renumber. A change that makes an existing `.kcui` unreadable needs a major-version bump and a reason.

- **Round-trip is the acceptance test.** Any change to the writer or the parser must keep `writeCatalog(parseCatalog(bytes)) == bytes` for every blob the parser accepts, including resources whose kind has no decoder here. Prove it with a case, not by reading the diff.

- **Resolution is total.** The floor catalog answers for every name `app/Names` declares, and `checkIntegrity` proves it on every test run. Adding a name means adding a resource to `Resources/Default.kcui` in the same change.

- **Rewriting the default catalog.** `Resources/Default.kcui` is the artifact and is checked in. Change it by editing `tools/bootstrap` and running `kira run tools/bootstrap` from the repo root. The tool refuses to write a catalog that does not validate, round-trip and cover the declared surface.

- **No floats on the wire.** Fractional values are 16.16 fixed point. Never add an IEEE float to a payload or a table.

- **File size.** 700 lines is a hard ceiling for every `.kira` file. Look for the split at 600, into cohesive 300 to 500-line modules.

- **Lint.** Run `kira lint .` from the repo root and leave it reporting nothing.

- **Verification.** Prove a change with `kira check .`, then `kira test --backend vm tests/kcoreui_kik` and `kira test --backend llvm tests/kcoreui_kik`. Both backends must agree: a catalog that resolves differently interpreted than compiled is the exact bug the deterministic specificity rule exists to make impossible.

- **Enum variants.** Write a leading dot, `.Dark`, wherever the expected type is known.
