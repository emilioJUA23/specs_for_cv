## nvim-pytest-runner — Neovim Plugin

**Target roles:** AI Engineering | Data Engineering | MLOps

**One-liner:** Neovim plugin that discovers and runs all pytest tests associated with a selected function.

**Core idea:**
- User places cursor on a function (or selects it) in the editor
- Plugin identifies the function name
- Runs all tests that reference or test that function
- Displays results inline or in a split

**Context:** TBD — flesh out during spec session.

**What you did:** TBD

**Impact:** TBD

**Keywords:** TBD

**CV bullet(s):**
- TBD

---

## Open Questions (to answer before implementation.md)

- How to identify which tests are associated with a function? (AST parsing, naming conventions, pytest markers, coverage data?)
- Language server / Treesitter for function detection?
- How to surface results? (quickfix list, float window, inline virtual text?)
- Scope: Python-only or multi-language?
- Plugin written in Lua or leverages existing test runners (neotest)?
