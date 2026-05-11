# Support the new Roc compiler's `roc bundle` CLI

## Background

The action currently invokes `roc build --bundle <type> <entry>` (the legacy
compiler's bundling syntax) and locates the resulting bundle by scanning the
entrypoint's directory for a file with the requested extension.

The new Roc compiler (the Zig rewrite, distributed as `nightly-new-compiler`
via `roc-lang/setup-roc`) replaces this with a dedicated `roc bundle`
subcommand:

```
roc bundle [--output-dir <PATH>] [--compression <N>] [ROC_FILES]...
```

Differences from the legacy CLI:

- Subcommand moved from `roc build --bundle` to `roc bundle`.
- Output format is fixed at `.tar.zst` (zstd). The legacy `.tar`, `.tar.gz`,
  `.tar.br` choices no longer apply.
- Compression is a numeric zstd level (`--compression 1..22`, default `3`)
  rather than a format selector.
- `--output-dir` lets the caller pick the destination directory; default is
  the current working directory (legacy dropped the bundle next to the
  entrypoint).

The action must work against either compiler so consumers don't have to fork
their workflows during the transition.

## Goals

- Make the action work with both the legacy and new Roc CLIs with **no yaml
  change required** for an existing consumer.
- Expose the new compiler's `--compression` knob.
- Keep the action's `bundle-path` output behaviour stable: bundle ends up
  next to the entrypoint, regardless of CLI.
- Cover both CLIs in the self-test workflow.

## Non-goals

- Exposing `--output-dir` as an action input. Hardcoded to the entrypoint's
  directory to preserve legacy behaviour. Can be added later if demand
  surfaces.
- Supporting the new `roc unbundle` subcommand. Out of scope.
- Adding a manual override (`cli: legacy|new|auto`) for the autodetect.
  Autodetect via subcommand probe is unambiguous; an override would be
  speculative.

## Design

### CLI detection

A single helper at the top of `main` decides which CLI the configured
`roc-path` exposes:

```ts
function detectCli(rocPath: string): "legacy" | "new" {
  try {
    execSync(`${rocPath} bundle --help`, { stdio: "pipe" });
    return "new";
  } catch {
    return "legacy";
  }
}
```

`stdio: "pipe"` keeps the help text out of the action log. The detected
result is logged once via `core.info`.

The probe is the canonical signal: the new compiler exposes a `bundle`
subcommand, the legacy compiler does not. No version-string parsing.

### Inputs

Updates to `action.yaml`:

| Input                     | Required | Default   | Applies to  | Change                                                                                                                      |
| ------------------------- | -------- | --------- | ----------- | --------------------------------------------------------------------------------------------------------------------------- |
| `library`                 | yes      | —         | both        | unchanged                                                                                                                   |
| `roc-path`                | yes      | `roc`     | both        | unchanged                                                                                                                   |
| `bundle-type`             | no       | `.tar.br` | legacy only | drop `required: true`; warn-and-ignore on new CLI                                                                           |
| `compression`             | no       | _(unset)_ | new only    | **new input**; numeric 1–22; if unset, omit `--compression` and let `roc bundle` use its default; warn-and-ignore on legacy |
| `release`, `tag`, `token` | —        | —         | both        | unchanged                                                                                                                   |

Output `bundle-path` unchanged.

### Mismatched-input handling

The two CLIs accept disjoint configuration knobs. When the user has set an
input that doesn't apply to the detected CLI:

- **`compression` set but legacy CLI detected:** `core.warning("Ignoring
'compression' input on legacy Roc CLI.")` and proceed.
- **`bundle-type` set to a non-default value but new CLI detected:**
  `core.warning("Ignoring 'bundle-type' input on new Roc CLI; bundles are
always .tar.zst.")` and proceed.

Comparing `bundle-type` against its default (`.tar.br`) before warning means
existing workflows that simply rely on the default don't get a noisy warning
when they upgrade their compiler.

### Dispatch

`bundleLibrary` becomes a switch on the detection result.

**Legacy branch (unchanged):**

```
roc build --bundle <bundle-type> <entry>
```

Plus the warn-on-`compression` check above.

**New branch:**

```
roc bundle --output-dir <dirname(entry)> [--compression <N>] <entry>
```

- `--output-dir` is always passed and pinned to the entrypoint's directory,
  so the bundle ends up where legacy users expect it and `getBundlePath` can
  keep scanning a single known directory.
- `--compression` is only included when the user set the input.
- Plus the warn-on-`bundle-type` check above.

### Output discovery

`getBundlePath` already takes an extension argument and scans the entry's
directory. The only change is routing the extension by CLI:

- Legacy: extension = the resolved `bundle-type` input.
- New: extension = `.tar.zst` (hardcoded; the new CLI offers no other format).

Because we pin `--output-dir` to the entrypoint's directory on the new CLI,
the existing scan logic continues to work without other changes.

### README

Update the example in `README.md` to:

- Add a short note that `bundle-type` is a legacy-only input and
  `compression` is a new-CLI-only input.
- Show the new `compression` input in the example block (commented or
  alongside).
- No change required for the basic example — the action works against
  either CLI with no extra yaml.

### Self-test workflow

Replace `.github/workflows/run-self.yaml` with a matrix job. Each leg
installs the right compiler via `roc-lang/setup-roc`, runs this action
against an example library known to compile on that CLI, and asserts the
bundle exists with the expected extension.

```yaml
strategy:
  fail-fast: false
  matrix:
    include:
      - roc-version: alpha4-rolling
        cli: legacy
        example-repo: Hasnep/roc-html
        library: roc-html/src/main.roc
        expected-extension: .tar.br
      - roc-version: nightly-new-compiler
        cli: new
        example-repo: imclerran/rtils
        library: rtils/package/main.roc
        expected-extension: .tar.zst
```

Each leg:

1. `actions/checkout@v3` for this repo.
2. `actions/checkout@v3` for `${{ matrix.example-repo }}`.
3. `roc-lang/setup-roc@main` with `version: ${{ matrix.roc-version }}`
   (replaces `Hasnep/setup-roc@main`; the `token` input is gone).
4. Run this action with `library: ${{ matrix.library }}`.
5. Assert: `test -f "${{ steps.bundle-library.outputs.bundle-path }}"` and
   that the path ends with `${{ matrix.expected-extension }}`.

## Testing

The repo has no unit tests for `src/index.ts` today; verification is done
end-to-end by the self-test workflow. This design preserves that pattern —
the matrix above exercises both CLIs against real Roc installs, which is the
only level at which the autodetect probe is meaningful.

No unit tests will be added in this change.

## Risks and open questions

- **`Hasnep/roc-html` drift:** if the legacy example repo stops compiling on
  `alpha4-rolling`, the legacy leg breaks. Out of scope to mitigate here;
  swap fixture if it becomes a problem.
- **`imclerran/rtils` drift:** same risk on the new-compiler leg. Owner
  (the user of this action) controls that repo.
- **Compiled output (`dist/index.js`):** the action ships pre-compiled JS.
  Implementation must rebuild and commit `dist/` so the action picks up the
  new code.
