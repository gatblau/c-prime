# C′ TextMate grammars

Syntax highlighting for C′ contracts (`.cprime`) and agent implementations (`.opq`). The grammars follow the normative grammar in `docs/spec/components/cprime-syntax.md`. They only colour text: the parser in `cprime-syntax` decides what is valid.

| File | Scope |
|---|---|
| `cprime.tmLanguage.json` | `source.cprime` |
| `opq.tmLanguage.json` | `source.opq` |

Error forms are highlighted with `invalid.illegal.*` scopes, so they show up in most themes:
- a future-reserved word (`func`, `use`, `priv`, `interface`, `impl` in `.cprime`; `fn`, `loop`, `invariant` in `.opq`);
- `fn` before a capability operation;
- `0x…` in `.cprime`;
- a byte literal with an odd number of hex digits.

## Use

- **VS Code:** copy or symlink this folder into `~/.vscode/extensions/cprime-textmate`, then reload the window.
- **JetBrains IDEs:** *Settings → Editor → TextMate Bundles → +*, then choose this folder.
- **GitHub and GitLab:** these grammars are the ones a future pull request to GitHub Linguist would add. Linguist accepts a new extension only once it is widely used (about 2,000 files indexed in a year). Until then, a repository can borrow similar highlighting with `.gitattributes`:

  ```gitattributes
  *.cprime linguist-language=Go      gitlab-language=go
  *.opq    linguist-language=Clojure gitlab-language=clojure
  ```

## Samples

`samples/` holds the spec's examples, plus `invalid.cprime` with each error form. When the grammar in `cprime-syntax` changes, update both the grammars and the samples.
