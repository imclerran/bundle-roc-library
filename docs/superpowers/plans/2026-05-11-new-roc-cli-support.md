# Support new Roc compiler's `roc bundle` CLI — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the action transparently support both the legacy Roc compiler (`roc build --bundle`) and the new Roc compiler (`roc bundle`) by autodetecting which CLI is available, with no yaml change required for existing consumers.

**Architecture:** A single `detectCli()` helper probes `roc bundle --help` once at startup. The result routes to one of two `bundleLibrary*` functions (legacy vs new) and selects the bundle file extension (`.tar.br`/`.tar.gz`/`.tar` vs hardcoded `.tar.zst`) used by the directory-scan in `getBundlePath`. A new optional `compression` action input is plumbed to the new CLI's `--compression` flag. Mismatched-input combinations log a warning and proceed.

**Tech Stack:** TypeScript, `@actions/core`, `@vercel/ncc` for bundling to `dist/index.js`, GitHub Actions composite/JS-action runtime (Node 16).

**Spec:** `docs/superpowers/specs/2026-05-11-new-roc-cli-support-design.md`

---

## File Structure

| File                              | Action     | Responsibility                                                                                                                 |
| --------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `action.yaml`                     | modify     | Drop `required: true` from `bundle-type`; add new optional `compression` input.                                                |
| `src/index.ts`                    | modify     | Add `detectCli`; split `bundleLibrary` into legacy/new variants; route extension to `getBundlePath`; read `compression` input. |
| `dist/index.js`                   | regenerate | Compiled output that GitHub Actions actually executes. CI fails if it's stale.                                                 |
| `.github/workflows/run-self.yaml` | rewrite    | Matrix over legacy + new compilers; switch to `roc-lang/setup-roc@main`.                                                       |
| `README.md`                       | modify     | Document `compression`; note which inputs apply to which CLI.                                                                  |

---

## Task 1: Update `action.yaml` inputs

**Files:**

- Modify: `action.yaml`

- [ ] **Step 1: Edit `action.yaml`**

Replace the entire `inputs:` block so that:

- `bundle-type` becomes optional (drop `required: true`; keep default)
- a new `compression` input is added (optional, no default)

Final `inputs:` block:

```yaml
inputs:
  library:
    description: The path to the library's entrypoint file.
    required: true
  roc-path:
    description: The absolute path to Roc. If this variable is not specified the action will try to find Roc using the `PATH` environment variable.
    required: true
    default: roc
  bundle-type:
    description: The filetype of the bundled library, either `.tar`, `.tar.gz` or `.tar.br`. Only applies to the legacy Roc CLI; ignored (with a warning) on the new `roc bundle` CLI, which always produces `.tar.zst`. Defaults to `.tar.br`.
    default: .tar.br
  compression:
    description: zstd compression level (1–22) for the new `roc bundle` CLI. Ignored (with a warning) on the legacy CLI. If unset, the compiler's default is used.
  release:
    description: Whether or not the bundled library should be uploaded to the repository's releases. Defaults to only uploading when the action is triggered by a release event.
    required: true
    default: ${{ github.event_name == 'release' }}
  tag:
    description: The tag of the release to upload to. Defaults to the current event's name.
    default: ${{ github.ref }}
  token:
    description: A GitHub token.
    default: ${{ github.token }}
```

(Leave `name`, `description`, `outputs`, and `runs` blocks alone.)

- [ ] **Step 2: Verify the file parses as valid YAML**

Run: `python3 -c "import yaml; yaml.safe_load(open('action.yaml'))"`
Expected: no output, exit code 0.

- [ ] **Step 3: Commit**

```bash
git add action.yaml
git commit -m "Add compression input and make bundle-type optional"
```

---

## Task 2: Rewrite `src/index.ts` with CLI detection and dispatch

**Files:**

- Modify: `src/index.ts`

This task replaces the file's contents in one shot — the changes are interleaved across most functions, and showing partial states between sub-steps would be confusing.

- [ ] **Step 1: Replace the contents of `src/index.ts`**

Final file:

```typescript
import { execSync } from "child_process";
import * as core from "@actions/core";
import * as fs from "fs";
import * as gh from "@actions/github";
import * as path from "path";

type Octokit = ReturnType<typeof gh.getOctokit>;
type BundleType = ".tar" | ".tar.gz" | ".tar.br";
type CliVersion = "legacy" | "new";

const NEW_CLI_EXTENSION = ".tar.zst";
const DEFAULT_BUNDLE_TYPE: BundleType = ".tar.br";

const detectCli = (rocPath: string): CliVersion => {
  try {
    execSync(`${rocPath} bundle --help`, { stdio: "pipe" });
    return "new";
  } catch {
    return "legacy";
  }
};

const quoteIfSpaces = (x: string): string => (x.includes(" ") ? `"${x}"` : x);

const bundleLibraryLegacy = (
  rocPath: string,
  libraryEntrypointPath: string,
  bundleType: BundleType,
  compression: string,
) => {
  if (compression !== "") {
    core.warning("Ignoring 'compression' input on legacy Roc CLI.");
  }
  const bundleCommand = [
    rocPath,
    "build",
    "--bundle",
    bundleType,
    libraryEntrypointPath,
  ]
    .map(quoteIfSpaces)
    .join(" ");
  core.info(`Running bundle command '${bundleCommand}'.`);
  const stdOut = execSync(bundleCommand);
  core.info(stdOut.toString());
};

const bundleLibraryNew = (
  rocPath: string,
  libraryEntrypointPath: string,
  bundleType: BundleType,
  compression: string,
) => {
  if (bundleType !== DEFAULT_BUNDLE_TYPE) {
    core.warning(
      "Ignoring 'bundle-type' input on new Roc CLI; bundles are always .tar.zst.",
    );
  }
  const outputDir = path.dirname(libraryEntrypointPath);
  const args = [rocPath, "bundle", "--output-dir", outputDir];
  if (compression !== "") {
    args.push("--compression", compression);
  }
  args.push(libraryEntrypointPath);
  const bundleCommand = args.map(quoteIfSpaces).join(" ");
  core.info(`Running bundle command '${bundleCommand}'.`);
  const stdOut = execSync(bundleCommand);
  core.info(stdOut.toString());
};

const getBundlePath = async (
  libraryEntrypointPath: string,
  extension: string,
): Promise<string> => {
  const libraryFolder = path.dirname(libraryEntrypointPath);
  core.info(
    `Looking for bundled library in '${libraryFolder}' with extension '${extension}'.`,
  );
  const bundleFileName = fs
    .readdirSync(libraryFolder)
    .find((x) => x.endsWith(extension));
  if (bundleFileName === undefined) {
    throw new Error(
      `Couldn't find bundled library in '${libraryFolder}' with extension '${extension}'.`,
    );
  }
  const bundlePath = path.resolve(path.join(libraryFolder, bundleFileName));
  core.info(`Found bundled library at '${bundlePath}'.`);
  return bundlePath;
};

const publishBundledLibrary = async (
  releaseTag: string,
  bundlePath: string,
  octokitClient: Octokit,
) => {
  core.info(`Publishing to release associated with the tag '${releaseTag}'.`);
  const release = await octokitClient.rest.repos
    .getReleaseByTag({
      ...gh.context.repo,
      tag: releaseTag,
    })
    .catch((err: Error) => {
      const createReleaseUrl = `https://github.com/${gh.context.repo.owner}/${gh.context.repo.repo}/releases/new`;
      core.error(
        [
          `Failed to find release associated with the tag '${releaseTag}'.`,
          `You can go to '${createReleaseUrl}' to create a release.`,
        ].join(" "),
      );
      throw err;
    });
  core.info(`Found release '${release.data.name}' at '${release.url}'.`);
  octokitClient.rest.repos
    .uploadReleaseAsset({
      ...gh.context.repo,
      release_id: release.data.id,
      name: path.basename(bundlePath),
      data: fs.createReadStream(bundlePath) as unknown as string,
      headers: {
        "content-length": fs.statSync(bundlePath).size,
        "content-type": "application/octet-stream",
      },
    })
    .catch((err: Error) => {
      core.error(`Failed to upload bundle '${bundlePath}'.`);
      throw err;
    });
};

const main = async () => {
  try {
    // Get inputs
    const isRequired = { required: true };
    const token = core.getInput("token");
    const bundleType = core.getInput("bundle-type") as BundleType;
    const compression = core.getInput("compression");
    const libraryEntrypointPath = core.getInput("library", isRequired);
    const release = core.getBooleanInput("release", isRequired);
    const releaseTag = core
      .getInput("tag", { required: release })
      .replace(/^refs\/(?:tags|heads)\//, "");
    const rocPath = core.getInput("roc-path", isRequired);
    const octokitClient = gh.getOctokit(token);

    // Detect which Roc CLI we're talking to
    const cli = detectCli(rocPath);
    core.info(`Detected ${cli} Roc CLI.`);

    // Bundle the library
    if (cli === "new") {
      bundleLibraryNew(rocPath, libraryEntrypointPath, bundleType, compression);
    } else {
      bundleLibraryLegacy(
        rocPath,
        libraryEntrypointPath,
        bundleType,
        compression,
      );
    }

    const extension = cli === "new" ? NEW_CLI_EXTENSION : bundleType;
    const bundlePath = await getBundlePath(libraryEntrypointPath, extension);
    core.setOutput("bundle-path", bundlePath);

    // Publish the bundle
    if (release) {
      await publishBundledLibrary(releaseTag, bundlePath, octokitClient);
    } else {
      core.info(
        `The input 'publish' was set to false, so skipping publish step.`,
      );
    }
  } catch (err) {
    core.setFailed((err as Error).message);
  }
};

main();
```

- [ ] **Step 2: Install dependencies (if not already installed)**

Run: `npm ci`
Expected: completes without errors. (Skip if `node_modules/` already populated.)

- [ ] **Step 3: Lint**

Run: `npm run lint`
Expected: exit code 0, no errors.

If errors appear, fix them before proceeding. Common ones to expect: none if the file above is followed verbatim.

- [ ] **Step 4: Type-check via build**

Run: `npm run build`
Expected: produces `dist/index.js` (and `dist/licenses.txt`) without TypeScript errors.

This both type-checks and produces the artifact needed in Task 3. Don't commit yet — that's Task 3.

- [ ] **Step 5: Commit the source change**

```bash
git add src/index.ts
git commit -m "Add Roc CLI autodetection and new-CLI bundle dispatch"
```

(Leave `dist/` for the next task so the rebuild lives in its own commit.)

---

## Task 3: Rebuild and commit `dist/index.js`

**Files:**

- Modify: `dist/index.js` (regenerated)
- Modify: `dist/licenses.txt` (regenerated; only if dependencies changed — usually unchanged)

The CI workflow `.github/workflows/check-transpiled.yaml` fails if `dist/index.js` is out of sync with `src/`. The action runtime executes `dist/index.js`, not the TypeScript source.

- [ ] **Step 1: Rebuild**

Run: `npm run build`
Expected: produces `dist/index.js`. (Already run in Task 2 Step 4 — re-run here to be sure the artifact is fresh against the just-committed source.)

- [ ] **Step 2: Verify the rebuild produced changes**

Run: `git status --porcelain dist/`
Expected: at least `dist/index.js` shown as modified. If empty, the rebuild produced byte-identical output, which means the source changes didn't affect the bundle — investigate before continuing.

- [ ] **Step 3: Commit the rebuilt artifact**

```bash
git add dist/
git commit -m "Rebuild dist/ for new Roc CLI support"
```

---

## Task 4: Replace self-test workflow with a CLI matrix

**Files:**

- Modify: `.github/workflows/run-self.yaml`

- [ ] **Step 1: Replace the file's contents**

Final file:

```yaml
name: Run self

on:
  # Run on all PRs
  pull_request:
    paths-ignore:
      - "**.md"
  # Run when a PR is merged into main
  push:
    branches:
      - main
    paths-ignore:
      - "**.md"

jobs:
  run-self:
    name: Run self (${{ matrix.cli }})
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - cli: legacy
            roc-version: alpha4-rolling
            example-repo: Hasnep/roc-html
            library: roc-html/src/main.roc
            expected-extension: .tar.br
          - cli: new
            roc-version: nightly-new-compiler
            example-repo: imclerran/rtils
            library: rtils/package/main.roc
            expected-extension: .tar.zst
    steps:
      - name: Checkout the repo
        uses: actions/checkout@v3
      - name: Checkout an example repo
        uses: actions/checkout@v3
        with:
          repository: ${{ matrix.example-repo }}
          path: ${{ matrix.example-repo == 'Hasnep/roc-html' && 'roc-html' || 'rtils' }}
      - name: Install Roc
        uses: roc-lang/setup-roc@main
        with:
          version: ${{ matrix.roc-version }}
      - name: Run the action
        id: bundle-library
        uses: ./
        with:
          library: ${{ matrix.library }}
          token: ${{ secrets.GITHUB_TOKEN }}
      - name: Check bundled library exists
        run: |
          BUNDLE_PATH="${{ steps.bundle-library.outputs.bundle-path }}"
          test -f "$BUNDLE_PATH"
          case "$BUNDLE_PATH" in
            *${{ matrix.expected-extension }}) echo "Extension OK: $BUNDLE_PATH" ;;
            *) echo "::error::Expected extension ${{ matrix.expected-extension }}, got $BUNDLE_PATH"; exit 1 ;;
          esac
```

Notes for the engineer:

- The `path:` for the example checkout is conditionally `roc-html` or `rtils` to keep the entrypoint paths in `matrix.library` valid.
- `roc-lang/setup-roc@main` takes a `version:` input (not `roc-version:`); there's no `token:` input, unlike the previous `Hasnep/setup-roc@main`.
- `fail-fast: false` so a failure in one leg doesn't mask the other.

- [ ] **Step 2: Verify the file parses as valid YAML**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/run-self.yaml'))"`
Expected: no output, exit code 0.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/run-self.yaml
git commit -m "Self-test against legacy and new Roc compilers"
```

---

## Task 5: Update README

**Files:**

- Modify: `README.md`

- [ ] **Step 1: Replace the file's contents**

Final file:

````markdown
# bundle-roc-library

A GitHub Action to bundle and release a Roc library.

The action works with both the legacy Roc compiler (which exposes
`roc build --bundle`) and the new Roc compiler (which exposes `roc bundle`).
The CLI is autodetected at runtime by probing for the `roc bundle`
subcommand — no configuration is required.

## Inputs

| Input         | Required | Default                                | Notes                                                                                                                            |
| ------------- | -------- | -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `library`     | yes      | —                                      | Path to the library's entrypoint file.                                                                                           |
| `roc-path`    | no       | `roc`                                  | Path to the Roc executable.                                                                                                      |
| `bundle-type` | no       | `.tar.br`                              | Legacy CLI only. One of `.tar`, `.tar.gz`, `.tar.br`. Ignored with a warning on the new CLI, which always produces `.tar.zst`.   |
| `compression` | no       | _(unset)_                              | New CLI only. zstd compression level (1–22). Ignored with a warning on the legacy CLI. If unset, the compiler's default is used. |
| `release`     | no       | `true` on release events, else `false` | Whether to upload the bundle to the repository's releases.                                                                       |
| `tag`         | no       | `${{ github.ref }}`                    | Tag of the release to upload to.                                                                                                 |
| `token`       | no       | `${{ github.token }}`                  | GitHub token used to upload the release asset.                                                                                   |

## Usage

```yaml
name: Example workflow

on:
  # Run when a release is published
  release:
    types:
      - published

jobs:
  bundle-and-release:
    name: Bundle and release library
    runs-on: ubuntu-latest
    permissions:
      contents: write # Used to upload the bundled library
    steps:
      - name: Check out the repository
        uses: actions/checkout@v3
      - name: Install Roc
        uses: roc-lang/setup-roc@main
        with:
          version: nightly-new-compiler # or e.g. alpha4-rolling for the legacy compiler
      - name: Bundle and release the library
        uses: hasnep/bundle-roc-library@main
        with:
          library: path/to/main.roc # Path to the library's entrypoint
          # compression: 19          # optional, new CLI only
          token: ${{ github.token }}
```
````

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "Document new-CLI behaviour and compression input"
```

---

## Final verification

After all tasks complete:

- [ ] **Step 1: Confirm the working tree is clean**

Run: `git status`
Expected: `nothing to commit, working tree clean`

- [ ] **Step 2: Confirm the commit history**

Run: `git log --oneline -10`
Expected: five new commits matching the messages above (action.yaml, src/index.ts, dist/, workflow, README), on top of the previous head.

- [ ] **Step 3: Confirm CI will pass the transpilation check locally**

Run: `npm run build && git status --porcelain dist/`
Expected: empty output (the rebuilt `dist/` matches what's committed).

- [ ] **Step 4: Push and let CI run the matrix**

Push the branch and verify both `Run self (legacy)` and `Run self (new)` jobs pass on GitHub Actions. The autodetect probe and end-to-end bundling can only be verified against real Roc installs, which is what the matrix provides.
