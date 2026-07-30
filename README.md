# dotfiles

macOS development environment, managed with [chezmoi](https://chezmoi.io).

Covers the customized surface: zsh profile and aliases, powerlevel10k prompt,
Ghostty, VS Code settings and extensions, Claude Code configuration, and the
handful of Homebrew formulae the shell config depends on.

Language runtimes are deliberately **not** installed. See [Runtimes](#runtimes).

## New machine

Install the Xcode Command Line Tools and Homebrew, then:

```bash
brew install chezmoi
chezmoi init --apply --source ~/projects/dotfiles https://github.com/BobbieBarker/dotfiles.git
```

Use the HTTPS URL, not SSH. SSH keys are set up *after* this step, so an SSH
clone here fails with `Permission denied (publickey)`. To switch the remote to
SSH once your key is on GitHub:

```bash
git -C ~/projects/dotfiles remote set-url origin git@github.com:BobbieBarker/dotfiles.git
```

You get three prompts: machine role, git name, git email. Everything else is
automatic. Open a new terminal when it finishes.

Answer `client` at the role prompt for contract machines. That drops the
forge-symphony aliases, the personal credentials, and the Claude Code
`bypassPermissions` default. See [Roles](#roles).

Afterwards, two things chezmoi deliberately does not do for you:

```bash
gh auth login                                    # git and GitHub credentials
ssh-keygen -t ed25519 -C "you@example.com"       # then add the key to GitHub
```

## Roles

The role is chosen once at `chezmoi init` and stored in
`~/.config/chezmoi/chezmoi.toml`.

| | `personal` | `client` |
|---|---|---|
| git identity | prompted | prompted |
| `sj`, `ccj` aliases | yes | no |
| `LINEAR_API_KEY`, `OPENROUTER_API_KEY` | from Keychain | not exported |
| Claude Code `defaultMode` | `bypassPermissions` | `default` |
| Claude Code forge-symphony/obsidian MCP allowlist | yes | no |
| VS Code `allowDangerouslySkipPermissions` | yes | no |
| Brewfile extras (flyctl, hcloud, ffmpeg, poppler, codex) | yes | no |
| `~/.tool-versions` | yes | no |
| zsh, prompt, Ghostty, VS Code, aliases | same | same |

To change role later, edit `role` in `~/.config/chezmoi/chezmoi.toml` and run
`chezmoi apply`.

## Secrets

Nothing sensitive is in this repo. Credentials are read from the macOS Keychain
at shell startup.

Every lookup is non-fatal. A missing entry produces an empty variable, not an
error, so a machine with an empty keychain still gets a working shell. **The
client role needs no keychain entries at all.**

On a personal machine, populate them once:

```bash
security add-generic-password -s github-pat      -a "$USER" -w
security add-generic-password -s npm-token       -a "$USER" -w
security add-generic-password -s linear-api      -a "$USER" -w
security add-generic-password -s openrouter-api  -a "$USER" -w
```

`run_onchange_after_40-keychain-check.sh` prints exactly which of these are
missing on every apply.

## Runtimes

`chezmoi apply` installs no language runtime and no database. The core Brewfile
is twelve formulae whose full transitive closure is seven small libraries
(`ca-certificates`, `libgit2`, `libssh2`, `llhttp`, `oniguruma`, `openssl@3`,
`pcre2`). No erlang, postgres, node, or python appears anywhere in it.

`brew bundle` also never uninstalls, so applying this on an existing machine can
only add.

Runtimes live in a separate file that no script touches:

```bash
brew bundle --file=~/.config/homebrew/Brewfile.runtimes

asdf plugin add erlang && asdf plugin add elixir
asdf plugin add nodejs && asdf plugin add rust && asdf plugin add rebar
asdf install    # reads .tool-versions in the current directory
```

On a client machine, prefer that project's `.tool-versions` over the personal
pins carried here.

## Day to day

```bash
chezmoi edit ~/.zshrc     # edit the source, not the target
chezmoi diff              # preview pending changes
chezmoi apply             # write them
chezmoi re-add            # pull target-side edits back into the source
chezmoi cd                # shell into the source directory
```

VS Code rewrites `settings.json` in place whenever you change a setting through
its UI, which shows up as chezmoi drift. `chezmoi re-add` is the fix, not a
problem to work around.

## Layout

| Path | Target |
|---|---|
| `dot_zshrc.tmpl` | `~/.zshrc` |
| `dot_zshenv.tmpl` | `~/.zshenv`, Keychain lookups |
| `dot_zprofile` | `~/.zprofile` |
| `dot_p10k.zsh` | `~/.p10k.zsh` |
| `dot_gitconfig.tmpl` | `~/.gitconfig` |
| `dot_config/ghostty/config` | `~/.config/ghostty/config` |
| `dot_config/homebrew/Brewfile.tmpl` | `~/.config/homebrew/Brewfile` |
| `dot_config/homebrew/Brewfile.runtimes` | opt-in, never auto-applied |
| `dot_claude/` | `~/.claude`, config and skills only |
| `Library/Application Support/Code/User/` | VS Code settings, keybindings, MCP |
| `.chezmoiscripts/` | bootstrap, ordered 10 to 40 |

## Vendored skills

`dot_claude/skills/rust-nif` and `dot_claude/skills/rust-programming` are
vendored copies rather than submodules, so a bootstrap reproduces exactly what
the source machine had rather than whatever upstream looks like that day.

| Skill | Upstream | Pinned at |
|---|---|---|
| `rust-nif` | `github.com/BadBeta/Skill_Elixir_Rust_NIF` | `bcbfb36` |
| `rust-programming` | `github.com/badbeta/Rust_programming_skill` | `5cdd175` |

## Design

`docs/design.md` records the inventory of the source machine, the decisions
behind this layout, and the twelve configuration defects corrected during the
port.
