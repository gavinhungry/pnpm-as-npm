# pnpm-as-npm

`pnpm-as-npm` lets package builds that call `npm` use
[pnpm](https://pnpm.io) instead, so their build scripts can run unchanged.
It supports common build commands, but isn't a drop-in replacement for
npm in general use.

## Installing

Install pnpm so that `pnpm --version` works in your shell. From a checkout
of this repository, install both wrappers in the same directory and put it
ahead of any existing `npm` and `npx` on `PATH`:

```sh
install -Dm755 -t "$HOME/.local/bin" npm npx
export PATH="$HOME/.local/bin:$PATH"
```

To keep that `PATH` order in new shells, add the `export` line to your
shell startup file.

### Arch Linux package

The package installs the wrappers in `/usr/lib/pnpm-as-npm/bin` and adds
that directory to `PATH` with `/etc/makepkg.conf.d/pnpm-as-npm.conf`, so
they're used only while makepkg builds a package. If an AUR helper doesn't
pick up the drop-in, load makepkg's configuration in a wrapper script
before running the helper:

```sh
source /usr/share/makepkg/util/config.sh && load_makepkg_config
```

## Configuration

| Variable           | Effect                                                  |
|--------------------|---------------------------------------------------------|
| `PNPM_AS_NPM_PNPM` | pnpm executable to run (default: `pnpm` from `PATH`)    |
| `pnpm_config_*`    | Overrides any of the defaults below                     |
| `npm_config_cache` | Same as `--cache`: puts pnpm's store and cache under it |

To behave like npm, pnpm is run with these settings unless overridden:

| Setting                                                     | Value     | Why                                                    |
|-------------------------------------------------------------|-----------|--------------------------------------------------------|
| `node_linker`                                               | `hoisted` | Flat `node_modules` like npm's                         |
| `pm_on_fail`                                                | `ignore`  | Don't reject projects with `"packageManager": "npm@…"` |
| `manage_package_manager_versions`, `package_manager_strict` | `false`   | The same for pnpm 10 and older                         |
| `update_notifier`                                           | `false`   | Keep build output quiet                                |

Each is set as both `pnpm_config_*` (pnpm 11+) and `npm_config_*` (pnpm
10 and older). For these defaults, a `pnpm_config_*` override takes
precedence over `npm_config_*`; either is applied to both versions.

## Supply-chain protections

pnpm's supply-chain settings are deliberately not touched, so your
`~/.config/pnpm/config.yaml` and pnpm's own defaults apply. On pnpm 11+
that means, unlike npm:

- Versions published less than a day ago are avoided
  (`minimumReleaseAge: 1440`). The built-in default is not strict: if no
  old enough version matches, pnpm installs a newer one. Set
  `minimumReleaseAge` explicitly in `config.yaml`, even to `1440`, to
  make it a hard limit.
- Dependencies with unapproved install scripts fail the install
  (`strictDepBuilds`). pnpm may also create or edit `pnpm-workspace.yaml`
  in the project to list them for approval.
- Git and tarball dependencies of dependencies are rejected
  (`blockExoticSubdeps`).

pnpm 10 is looser: there is no release delay, and unapproved install
scripts are skipped with a warning.

To relax a protection for one build, set it in the environment, e.g.:

```sh
pnpm_config_dangerously_allow_all_builds=true npm ci
pnpm_config_minimum_release_age=0 npm ci
pnpm_config_block_exotic_subdeps=false npm ci
```

## Supported commands

| npm                                                   | Runs                                                                                   |
|-------------------------------------------------------|----------------------------------------------------------------------------------------|
| `install`, `i` (no packages)                          | `pnpm import` if needed, then `pnpm install --no-frozen-lockfile`                      |
| `install <pkg>…`                                      | `pnpm add`, with `-D`, `-O`, `-E`, `--save-peer` mapped                                |
| `ci`                                                  | checks lockfiles, removes `node_modules`, re-imports if needed, then installs frozen    |
| `install-test`, `install-ci-test`                     | the above, then the `test` script                                                      |
| `uninstall`, `update`                                 | `pnpm remove`, `pnpm update`                                                           |
| `run <script> [-- args]`                              | `pnpm run <script> args`                                                               |
| `test`, `start`, `stop`, `restart`                    | the pnpm command of the same name                                                      |
| `exec`, `npx`                                         | a locally installed binary directly, otherwise `pnpm dlx`                              |
| `prefix`, `root`, `bin`                               | npm's local prefix, `pnpm root`, `pnpm bin`                                            |
| `cache clean`, `cache verify`                         | `pnpm store prune`, `pnpm store status`                                                |
| `-v`, `--version`                                     | `pnpm --version`                                                                       |
| `ls`, `pack`, `publish`, `audit`, `config`, `view`, … | passed through to pnpm unchanged                                                       |

`pnpm import` runs when the project has a `package-lock.json`,
`npm-shrinkwrap.json` or `yarn.lock` but no `pnpm-lock.yaml`. Import
uses the locked versions as preferences; it does not guarantee an
identical npm dependency tree. The wrapper leaves the generated
`pnpm-lock.yaml` and its checksum (`.pnpm-as-npm-lock.sha256`) in place.

`ci` re-imports a single source lockfile on every build, even if only a
transitive dependency changed. If both source and pnpm lockfiles exist,
`ci` refuses to overwrite a pnpm lockfile with a missing or mismatched
checksum. To regenerate it, first move the pnpm lockfile aside. With
only a pnpm lockfile, `ci` uses it directly.

Options are parsed anywhere on the command line up to `--`, as npm does.
Common npm options are translated (`--prefix`, `-w`/`--workspace`,
`--workspaces`, `--registry`, `--cache`, `--loglevel`, `-s`, `--omit`
and `--include` (for `dev` and `optional`),
`--production`, `--ignore-scripts`, `--if-present`, …), npm-only ones
such as `--no-audit` and `--legacy-peer-deps` are ignored, and other
`--name=value` options are exported as `npm_config_name` for scripts to
read. Unknown commands fail rather than guessing.

## Limitations

- Global mode (`npm install -g`) is unsupported. Local project selection
  with `--prefix` is supported.
- `npm version <bump>` is not supported.
- `--no-save` still saves to `package.json` (pnpm has no equivalent).
- npm workspaces are not detected automatically from a subdirectory;
  pass `-w` or run from the workspace root.
- Commands passed through to pnpm may take different options or print
  different output than npm's.
- Tools that inspect npm itself (e.g. `npm config get prefix`) see
  pnpm's answers.
