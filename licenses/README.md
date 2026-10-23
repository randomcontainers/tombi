# Extra license files

`scripts/cargo-licenses.py` adds the files in `crates/` to those of the crates the tombi binary records, and the build copies `schemastore/` into the image. The build stops when the binary records a crate without any license file, and when a directory here or a line of `crates/native-covered` matches no crate it records. A directory named without a version, `crates/<crate>/`, is kept across versions; its `EXPRESSION` file holds the crate's license expression, and the build stops when the crate's expression changes, so the files are checked again.

| Files | Source |
|---|---|
| `crates/defmt-parser/LICENSE-MIT`, `crates/defmt-parser/LICENSE-APACHE` | `LICENSE-MIT` and `LICENSE-APACHE` of [knurling-rs/defmt](https://github.com/knurling-rs/defmt) at commit 4a8cdb44, from which defmt-parser 1.0.0 was published; the same texts as in the defmt crate |
| `crates/gloo-net/LICENSE-MIT`, `crates/gloo-net/LICENSE-APACHE`, `crates/gloo-utils/LICENSE-MIT`, `crates/gloo-utils/LICENSE-APACHE` | `LICENSE-MIT` and `LICENSE-APACHE` of [ranile/gloo](https://github.com/ranile/gloo) (formerly rustwasm/gloo) at commit 40c6daa6, from which gloo-net 0.6.0 was published; unchanged since 2019, so the same at f3bb1d1a, from which gloo-utils 0.2.0 was published |
| `crates/psl/MPL-2.0` | the text of the [Mozilla Public License 2.0](https://www.mozilla.org/media/MPL/2.0/index.txt) |
| `crates/psl/NOTICE` | written for this repository: psl compiles in the Public Suffix List, which is under MPL-2.0 |
| `crates/psl/DECLARED`, `crates/psl/EXPRESSION` | `DECLARED` is the expression psl declares, `MIT/Apache-2.0`, and the build stops when psl changes it; `EXPRESSION` adds MPL-2.0 for the list, so the crate's source goes into the image |
| `crates/tombi-glob/EXPRESSION` | tombi-glob is part of the Tombi repository, whose `LICENSE` is MIT, but its `Cargo.toml` has no `license` |
| `crates/native-covered` | written for this repository: the crates with a `links` key, and the license files deeper inside ring, for the code it took from fiat-crypto and once_cell |
| `schemastore/LICENSE`, `schemastore/NOTICE` | `LICENSE` and `NOTICE` of [SchemaStore/schemastore](https://github.com/SchemaStore/schemastore) at commit 57967c6f, unchanged since 2021; Tombi's built-in schemas for `Cargo.toml` and `pyproject.toml` are copies of SchemaStore's |
