# Analysis: Shell Keywords Missing from unproxyableReason

## What is unproxyableReason?

`unproxyableReason` in `internal/cli/cli.go` is a function that rejects shell
builtins that **must run in the parent shell** to have any effect. Running them
in a subprocess (which is what snip does) is a silent no-op or produces a
syntax error.

Currently it covers:
- Directory changes: `cd`, `chdir`, `pushd`, `popd`
- Environment: `export`, `unset`, `alias`, `unalias`, `readonly`, `declare`, `typeset`, `local`
- Shell options: `set`, `shopt`, `setopt`, `unsetopt`, `emulate`
- Execution control: `eval`, `exec`, `exit`, `logout`, `return`, `break`, `continue`
- Job control: `wait`, `bg`, `fg`, `disown`, `jobs`, `suspend`
- Shell configuration: `bindkey`, `bind`, `complete`, `trap`, `umask`, etc.

## What is Missing

Shell **keywords** (not builtins) that form compound command structures:

| Keyword   | Used in          | Example                                          |
|-----------|------------------|--------------------------------------------------|
| `for`     | Loops            | `for f in *.txt; do wc -l "$f"; done`              |
| `while`   | Loops            | `while true; do curl localhost; sleep 1; done`     |
| `until`   | Loops            | `until ping -c1 host; do sleep 1; done`            |
| `select`  | Menus            | `select opt in a b c; do echo $opt; done`          |
| `do`      | Loop body marker | (part of for/while/until)                          |
| `done`    | Loop end marker  | (closes for/while/until)                           |
| `if`      | Conditionals     | `if [ -f /tmp/lock ]; then rm /tmp/lock; fi`       |
| `then`    | Conditional body | (part of if)                                        |
| `else`    | Conditional else | (part of if)                                        |
| `elif`    | Conditional else-if | (part of if)                                      |
| `fi`      | If end marker    | (closes if)                                         |
| `case`    | Pattern matching | `case $1 in start) echo running;; esac`             |
| `esac`    | Case end marker  | (closes case)                                        |
| `time`    | Timing prefix    | `time curl example.com`                             |
| `[[`      | Conditional expr | `[[ -f "$file" ]] && echo exists`                    |

## How the Gap Manifests

### Via opencode-snip plugin

The plugin splits multi-command strings on `;`, `&&`, `||` and prefixes each
segment with `snip run --`. A for loop like:

```bash
for f in a b c; do echo $f; done
```

becomes:

```
snip run -- for f in a b c; snip run -- do echo $f; snip run -- done
```

Each segment is executed independently. snip's `unproxyableReason` does not
recognize `for`, `do`, or `done`, so snip attempts to exec them as system
commands. Result:

```
snip: passthrough: exec: "for": executable file not found in $PATH
```

The loop never iterates. The only output is error noise.

### Via direct snip usage

A user or agent wrapping a loop manually:

```bash
snip for i in 1 2 3; do echo $i; done
```

gives the same error.

## Why It Matters

1. **Silent failure** — "executable file not found" looks like a PATH problem,
   not a shell keyword issue. Users waste time debugging.
2. **Agent confusion** — AI coding agents don't know why the loop failed and
   may retry with variations, burning tokens.
3. **Inconsistent with existing coverage** — snip already handles `break`,
   `continue`, `return`, and `exit` (shell keywords used inside loops), but
   not the loop constructs themselves.

## Why It Only Matters When the Plugin Is Active

When commands run directly in a shell (without snip), the shell interprets
`for`, `do`, `done` as keywords correctly. The issue only occurs when snip
intercepts the command and tries to proxy each `;`-separated segment as an
independent command.
