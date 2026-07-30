# Portable dev environment: design

Date: 2026-07-30
Status: approved design, not yet implemented

## Problem

Replicate the customized parts of a macOS development environment on a second
machine used for contract work. The source machine runs Windsurf; the target
runs VS Code. The two machines must not share credentials or personal project
configuration.

Scope is the *custom* surface: shell profile, aliases, prompt, terminal theme,
editor settings, and Claude Code configuration. Language runtimes are
explicitly out of the automatic path (see "Runtimes" below).

## Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Manager | chezmoi | Templating handles personal/client divergence from one branch. Writes real files, not symlinks, so abandoning it costs nothing. Single bootstrap command. |
| Secrets | macOS Keychain, resolved at runtime | Extends the pattern already present at `.zshrc:184`. Nothing sensitive on disk or in git. |
| Scoping | One repo, machine role chosen at init | Client machine gets the tooling and look without personal credentials or forge-symphony wiring. |
| Editor config | In the repo, not Settings Sync | Reviewable in git, no account sign-in on a client machine, keeps client and personal config from co-mingling. |
| Repo location | `~/projects/dotfiles` | chezmoi source dir set via `--source`. Keeps this out of the forge-symphony checkout. |

## Bootstrap contract

On a fresh machine, after Xcode Command Line Tools and Homebrew:

```bash
brew install chezmoi
chezmoi init --apply --source ~/projects/dotfiles git@github.com:BobbieBarker/dotfiles.git
```

chezmoi prompts once for role, git name, and git email; writes all managed
files; then runs the ordered scripts. Re-running `chezmoi apply` is idempotent.

## Repository layout

```
dotfiles/
├── .chezmoi.toml.tmpl                       prompts: role, gitName, gitEmail
├── .chezmoiignore                           role-conditional exclusions
├── .chezmoiscripts/
│   ├── run_onchange_after_10-homebrew.sh.tmpl
│   ├── run_onchange_after_20-oh-my-zsh.sh.tmpl
│   ├── run_onchange_after_30-vscode-extensions.sh.tmpl
│   └── run_onchange_after_40-keychain-check.sh.tmpl
├── dot_zshenv.tmpl
├── dot_zprofile
├── dot_zshrc.tmpl
├── dot_p10k.zsh
├── dot_npmrc
├── dot_gitconfig.tmpl
├── dot_tool-versions                        personal role only
├── dot_config/
│   ├── ghostty/config
│   ├── bat/config
│   ├── git/ignore
│   └── homebrew/
│       ├── Brewfile.tmpl                    config dependencies only
│       └── Brewfile.runtimes                opt-in, never auto-applied
├── dot_claude/
│   ├── CLAUDE.md
│   ├── settings.json.tmpl
│   ├── themes/kanagawa-wave.json
│   ├── hooks/executable_cbm-code-discovery-gate
│   ├── hooks/executable_cbm-session-reminder
│   └── skills/                              6 skills, verbatim
├── Library/Application Support/Code/User/
│   ├── settings.json
│   ├── keybindings.json
│   └── mcp.json.tmpl
└── docs/design.md                           this file
```

All scripts are `after` so they run once managed files exist. `run_onchange_`
means each re-fires only when its own content hash changes.

## Machine role

`.chezmoi.toml.tmpl` prompts once at init and persists the answers to
`~/.config/chezmoi/chezmoi.toml`:

```
{{- $role := promptStringOnce . "role" "Machine role (personal|client)" "personal" -}}
{{- $gitName := promptStringOnce . "gitName" "Git user.name" -}}
{{- $gitEmail := promptStringOnce . "gitEmail" "Git user.email" -}}

sourceDir = {{ (printf "%s/projects/dotfiles" .chezmoi.homeDir) | quote }}

[data]
role     = {{ $role | quote }}
gitName  = {{ $gitName | quote }}
gitEmail = {{ $gitEmail | quote }}
```

`sourceDir` must be set here. Without it, `--source` applies only to the `init`
invocation and every later bare `chezmoi apply` would look in the default
`~/.local/share/chezmoi` and find nothing.

What the role switches:

| Surface | `personal` | `client` |
|---|---|---|
| git `user.name` / `user.email` | BobbieBarker / GitHub noreply | prompted at init |
| `sj`, `ccj` aliases | present | absent |
| `LINEAR_API_KEY`, `OPENROUTER_API_KEY` | resolved | not exported |
| `~/.claude/settings.json` `defaultMode` | `bypassPermissions` | `default` |
| `skipDangerousModePermissionPrompt` | `true` | omitted |
| forge-symphony MCP allowlist entries | present | absent |
| `~/.tool-versions` | managed | ignored |
| Everything else | identical | identical |

The `defaultMode` split is deliberate. `bypassPermissions` plus
`skipDangerousModePermissionPrompt: true` is not a default to inherit silently
onto a client's codebase.

## Secrets

Generated `~/.zshenv`:

```zsh
if [[ -z "${DOTFILES_SECRETS_LOADED:-}" ]]; then
  export GITHUB_PERSONAL_ACCESS_TOKEN="$(security find-generic-password -s github-pat -w 2>/dev/null)"
  export NPM_TOKEN="$(security find-generic-password -s npm-token -w 2>/dev/null)"
{{- if eq .role "personal" }}
  export LINEAR_API_KEY="$(security find-generic-password -s linear-api -w 2>/dev/null)"
  export OPENROUTER_API_KEY="$(security find-generic-password -s openrouter-api -w 2>/dev/null)"
{{- end }}
  export DOTFILES_SECRETS_LOADED=1
fi
```

The `DOTFILES_SECRETS_LOADED` guard is load-bearing. `.zshenv` is sourced by
every zsh invocation including non-interactive `zsh -c`, and forge-symphony
spawns many of those. Exporting the sentinel alongside the values means child
shells inherit both and skip the lookups, so the keychain cost is paid once per
terminal rather than once per subprocess.

chezmoi's `keyring` template function is deliberately **not** used: it resolves
at apply time and writes plaintext into `~/.zshenv` on disk. That keeps secrets
out of git but not off disk. Runtime lookup keeps them out of both.

`~/.npmrc` is committable because npm expands environment variables natively:

```
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
```

`~/.config/gh/` is not managed. Run `gh auth login` on the new machine.
`~/.ssh` is not managed. Generate a fresh key and add it to GitHub.

`run_onchange_after_40-keychain-check.sh` stores nothing. It probes each
required keychain service and, for any that are missing, prints the exact
`security add-generic-password -s <service> -a <account> -w` command to run.

Every lookup is non-fatal by construction. `security find-generic-password`
returns empty and exits non-zero when the entry is absent, and `2>/dev/null`
plus the surrounding `export` swallow it. A machine with an empty keychain gets
empty variables and a working shell, not an error.

That makes the **client role's keychain setup empty by default**. It exports no
Linear or OpenRouter key at all, and `run_onchange_after_40-keychain-check.sh`
reports nothing to do. `gh auth login` covers git and GitHub credentials on its
own, so `GITHUB_PERSONAL_ACCESS_TOKEN` only needs a keychain entry if some tool
reads it directly.

### Credential rotation: declined

`LINEAR_API_KEY` and `OPENROUTER_API_KEY` were plaintext in `~/.zshenv`, and an
npm publish token was plaintext in `~/.npmrc`. All three were read into an
assistant session transcript during inventory. Rotation was considered and
declined; the keys stay as they are.

This does not change the design. The keys cannot live in the repo regardless of
their rotation state, because the repo is published to GitHub. Keychain lookup
is what lets a committed `.zshenv` produce a working personal machine. Setup is
a one-time `security add-generic-password` per key on the personal machine, and
nothing at all on the client machine.

## Homebrew

Two files, both written under `~/.config/homebrew/`. Only the first is applied
automatically.

`run_onchange_after_10-homebrew.sh` runs
`brew bundle --file="$HOME/.config/homebrew/Brewfile"` against the already-written
file, and carries a trigger comment so it re-fires when the Brewfile changes:

```bash
# Brewfile hash: {{ include "dot_config/homebrew/Brewfile.tmpl" | sha256sum }}
```

**`Brewfile.tmpl`** is the set the shell configuration *depends on*. Without
these, `ls`, `cat`, `cd`, `grep`, `find`, and `diff` all fail, because the
zshrc aliases them to replacements.

```
tap "romkatv/powerlevel10k"

brew "eza"                    # alias ls, ll, tree
brew "bat"                    # alias cat, and ~/.config/bat/config
brew "fd"                     # alias find, and FZF_DEFAULT_COMMAND
brew "ripgrep"                # alias grep
brew "git-delta"              # alias diff, GIT_PAGER, gitconfig core.pager
brew "zoxide"                 # alias cd
brew "fzf"
brew "jq"
brew "gh"
brew "lazygit"
brew "zsh-syntax-highlighting"
brew "powerlevel10k"

cask "ghostty"
cask "visual-studio-code"
cask "font-jetbrains-mono-nerd-font"
cask "font-fira-code"
{{ if eq .role "personal" }}
cask "codex"
brew "flyctl"
brew "hcloud"
brew "ffmpeg"
brew "poppler"
{{ end }}
```

The full transitive dependency closure of that list, verified on the source
machine, is seven small libraries:

```
ca-certificates  libgit2  libssh2  llhttp  oniguruma  openssl@3  pcre2
```

No erlang, postgres, node, wxwidgets, or python appears anywhere in it. Applying
this Brewfile cannot install a language runtime or a database. `brew bundle`
also never uninstalls: removing an entry has no effect on an already-installed
formula, and pruning requires an explicit `brew bundle cleanup --force` that no
script in this repo runs.

`brew leaves` is **not** a valid basis for this list. `ripgrep` is installed on
the source machine as a transitive dependency and does not appear in `brew
leaves`, so a leaves-derived Brewfile would silently break `alias grep='rg'`.
The list is curated by walking every alias and export in `.zshrc` back to the
binary it needs.

`visual-studio-code` is added as a cask; on the source machine it is a manual
`/Applications` install and appears in no brew list.

**`Brewfile.runtimes`** holds `asdf`, `cmake`, `postgresql@14`, `libpqxx`,
`sqlite`, and the Erlang build dependencies (`wxwidgets`, `fop`, `libxslt`,
`tcl-tk`, `zlib`). No script references it. Install runtimes with an explicit
`brew bundle --file=~/.config/homebrew/Brewfile.runtimes` when the contract work
needs them.

## Runtimes

Out of the automatic path by decision. The client machine's Elixir and Erlang
versions should follow that project's `.tool-versions`, not this one.

The shell configuration remains asdf-aware regardless: the `asdf` oh-my-zsh
plugin no-ops harmlessly when asdf is absent, and the explicit shims PATH entry
(see fix 3) is a no-op when the directory does not exist. Installing asdf later
therefore needs no config change.

`~/.tool-versions` is carried on the personal role only, as a record of the
current pins:

```
elixir 1.20.1-otp-28
erlang 28.2
nodejs 22.8.0
rust 1.94.1
rebar 3.24.0
```

## Defects fixed in transit

These are current faults on the source machine. Carrying them forward verbatim
would reproduce them, so each is corrected as part of the port.

| # | Current state | Correction |
|---|---|---|
| 1 | `.zshrc:145` sources `/opt/homebrew/Cellar/powerlevel10k/1.20.0/share/...`, a version-pinned path that dies on the next `brew upgrade` and will not exist on a new machine | `/opt/homebrew/opt/powerlevel10k/share/powerlevel10k/powerlevel10k.zsh-theme` (verified present) |
| 2 | `.zshrc:149` sources `/Users/chad/zsh-syntax-highlighting/`, a stray manual clone in `$HOME` hardcoded to the username, installed by nothing | brew formula `zsh-syntax-highlighting` (0.8.0), sourced from `$(brew --prefix)/share/...` |
| 3 | asdf shims reach PATH through three implicit hops: the omz `asdf` plugin locates `$(brew --prefix asdf)/libexec/asdf.sh` and sources it. Nothing in any rc file mentions asdf except the plugin name on line 90. The compat shim is dated Jun 2025 while asdf is on 0.18 | explicit, guarded `export PATH="${ASDF_DATA_DIR:-$HOME/.asdf}/shims:$PATH"` |
| 4 | Both editors set `terminal.integrated.fontFamily` to `'MesloLGM Nerd Font'`, which is **not installed**. It has been silently falling back | `'JetBrainsMono Nerd Font'`, confirmed installed and matching the Ghostty config |
| 5 | `$HOME/.local/bin` prepended to PATH three times (lines 160, 195, 199) | once |
| 6 | `nvm` loaded in the omz plugin list while asdf owns node | removed |
| 7 | `.zshrc:157` prepends Windsurf's bin directory to PATH | removed |
| 8 | `.zprofile` prepends `/Library/Frameworks/Python.framework/Versions/3.10/bin` unconditionally; pyenv is installed with zero versions and a `.zprofile.pysave` exists | guarded on directory existence |
| 9 | VS Code settings carry dead keys: `eslint.autoFixOnSave`, `terminal.integrated.shell.linux`, `bracketPairColorizer.depreciation-notice`, `sync.gist`, `npm.keybindingsChangedWarningShown` | stripped |
| 10 | `~/.claude/settings.json` hardcodes `/Users/chad` in `additionalDirectories` and the hook paths | `{{ .chezmoi.homeDir }}` |
| 11 | `.zshrc:190` comments that zoxide must be last, but lines 194-199 come after it. zoxide's doctor warning fires on every shell start | PATH exports moved above the zoxide init |
| 12 | Extension `ms-ossdata.vscode-postgresql` (Windsurf) was renamed upstream | `ms-ossdata.vscode-pgsql` |

## zshrc structure

The current file scatters PATH exports across lines 143, 160, 163, 195, and
198-199, and aliases across 124-140 and 166-175. The rewrite consolidates these
into ordered sections.

Relative source order is preserved exactly where it is load-bearing, because
the current order works:

1. powerlevel10k instant prompt (must stay at the top)
2. oh-my-zsh: `ZSH`, `ZSH_THEME`, plugin list, then `source $ZSH/oh-my-zsh.sh`
3. PATH, single consolidated block
4. powerlevel10k theme, then `~/.p10k.zsh`
5. zsh-syntax-highlighting
6. aliases, single consolidated block
7. exports: `KERL_CONFIGURE_OPTIONS`, `FZF_*`, `GIT_PAGER`
8. zoxide init, last, followed by `alias cd='z'`

## Editor

The base is **Windsurf's** `settings.json` (modified 2026-04-22), not VS Code's
(2026-03-02). The two are approximately 95 percent identical; Windsurf's is
newer and retains `workbench.colorTheme: "Shades of Purple (Super Dark)"` where
VS Code's has reverted to plain `"Shades of Purple"`.

VS Code-only keys retained from the VS Code copy: `claudeCode.preferredLocation`
(set to `sidebar`), `githubPullRequests.pullBranch`, `chat.mcp.gallery.enabled`.

`claudeCode.allowDangerouslySkipPermissions`,
`claudeCode.initialPermissionMode`, and
`github.copilot.chat.claudeAgent.allowDangerouslySkipPermissions` are gated to
the personal role, for the same reason as the `~/.claude` `defaultMode` split.

Extensions are installed by `run_onchange_after_30-vscode-extensions.sh`, which
iterates a list and calls `code --install-extension --force`. The full source
list is 60 extensions and is carried for both roles, since the contract work is
also Elixir.

`Library/Application Support/Code/User/mcp.json.tmpl` templates the
`codebase-memory-mcp` absolute path off `{{ .chezmoi.homeDir }}`.

Note: VS Code rewrites `settings.json` in place when settings are changed
through the UI, which produces chezmoi drift. `chezmoi re-add` is the intended
response, not a problem to design around.

## oh-my-zsh

`run_onchange_after_20-oh-my-zsh.sh` installs oh-my-zsh unattended if absent,
then clones the three custom plugins to `$ZSH_CUSTOM/plugins/` if absent.
Upstreams confirmed from the source machine's git remotes:

| Plugin | Upstream |
|---|---|
| `autoupdate` | `https://github.com/TamCore/autoupdate-oh-my-zsh-plugins` |
| `zsh-autosuggestions` | `https://github.com/zsh-users/zsh-autosuggestions` |
| `zsh-claudecode-completion` | `https://github.com/wbingli/zsh-claudecode-completion` |

The stock `example` plugin and `example.zsh-theme` are omz-provided and are not
managed.

## Terminal

`~/.config/ghostty/config` is carried verbatim (15 lines). The `Kanagawa Wave`
theme is built into Ghostty 1.3.1 and requires no theme file; the source
machine's `~/.config/ghostty/themes/` directory is empty and is not carried.

Font `JetBrainsMono Nerd Font` comes from the `font-jetbrains-mono-nerd-font`
cask in `Brewfile.tmpl`.

## Claude Code

Managed: `CLAUDE.md` (174 lines), `settings.json` (templated per role),
`themes/kanagawa-wave.json`, both hooks, and all six skills
(`codebase-memory`, `erlang-nif`, `quiche-http3`, `rust-nif`, `rust-programming`,
`tauri`).

Not managed, via `.chezmoiignore`:

| Path | Reason |
|---|---|
| `.claude/projects` | 2.0 GB of session transcripts |
| `.claude/file-history` | 56 MB |
| `.claude/shell-snapshots` | 5.7 MB, regenerated |
| `.claude/backups`, `cache`, `debug`, `paste-cache` | regenerated |
| `.claude/history.jsonl`, `todos`, `sessions`, `tasks`, `teams`, `plans` | machine-local state |
| `.claude/plugins` | refetched from the configured marketplace |
| `~/.claude.json` | 756 KB of OAuth credentials and per-project state |

`enabledPlugins` and `extraKnownMarketplaces` are carried in
`settings.json.tmpl`, so plugins reinstall from the marketplace on first run.

## Explicitly excluded

Windsurf entirely. pyenv (installed, zero versions). `~/.zsh_history` (11,947
lines). `~/.ssh`. `~/.config/gh`. `~/.aws`, `~/.fly`, `~/.cloudflare`,
`~/.mcp-auth`, `~/.docker`. The six stale `.zcompdump-*` files. The brew `zsh`
formula, since the login shell is `/bin/zsh` and the brew build is vestigial.

## Verification

The design is complete when, on a fresh machine:

1. `chezmoi init --apply` completes without manual intervention beyond the
   three init prompts and the keychain entries the check script names.
2. A new Ghostty window opens with the powerlevel10k prompt, Kanagawa Wave, and
   JetBrainsMono Nerd Font, with no zoxide doctor warning and no
   `command not found`.
3. `ls`, `ll`, `tree`, `cat`, `cd`, `grep`, `find`, `diff` all resolve to their
   intended replacements.
4. `git config user.email` returns the role-appropriate address.
5. `env | grep -c LINEAR_API_KEY` returns 0 on the client role and 1 on
   personal.
6. On the client role, `brew list --formula | grep -cE 'erlang|postgresql'`
   returns 0 after a full `chezmoi apply` on a fresh machine.
7. `grep -rE 'lin_api_|sk-or-v1-|npm_[A-Za-z0-9]{36}' ~/projects/dotfiles`
   returns nothing.
8. VS Code opens with Shades of Purple (Super Dark), the material icon theme,
   and an integrated terminal rendering Nerd Font glyphs.

## Open items

- Whether to carry `~/.zsh_history` to the personal machine only.
- Whether to prune the 60-extension list. Candidates with no evident current
  use: `meera.ppdm-spec`, `maty.vscode-mocha-sidebar`, `ms-vsliveshare.vsliveshare`,
  `mjmcloug.vscode-elixir` (superseded by `jakebecker.elixir-ls`), and the three
  Lua extensions. Deferred; the list ships whole and is trivial to trim later.
