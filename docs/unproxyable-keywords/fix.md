# Fix: Add Shell Keywords to unproxyableReason

## Change Location

`internal/cli/cli.go` — the `unproxyableReason` function

## Current State

The function uses a switch statement. Adding a new case for shell keywords:

```go
func unproxyableReason(command string) string {
    switch command {
    // ... existing cases ...
    case "for", "while", "until", "select", "do", "done":
        return "it is a shell loop keyword (use snip on individual commands inside the loop)"
    case "if", "then", "else", "elif", "fi":
        return "it is a shell conditional keyword (use snip on individual commands inside the branch)"
    case "case", "esac":
        return "it is a shell case keyword (use snip on individual commands inside the case)"
    case "time":
        return "it is a shell timing keyword (wrap the timed command directly)"
    case "[[", "]]":
        return "it is a shell conditional expression keyword"
    }
    return ""
}
```

## Behavior After Fix

```bash
$ snip for f in a b c; do echo $f; done
snip: for cannot be proxied (it is a shell loop keyword — use snip on individual commands inside the loop)

$ snip if [ -f /tmp/lock ]; then rm /tmp/lock; fi
snip: if cannot be proxied (it is a shell conditional keyword — use snip on individual commands inside the branch)
```

The error goes to stderr and snip exits with code 1. The user/agent sees a
clear message explaining why and how to fix it.

## What This Does NOT Fix

This change only makes snip reject these keywords with a clear error instead
of a confusing "executable not found" message. It does **not** make snip
understand loop/conditional syntax.

The opencode-snip plugin will still split loops on `;` and prefix each
segment. Each `for`/`do`/`done` segment will still be rejected — but now
with a clear message instead of "executable not found".

## Alternative Approaches Not Taken

1. **Make snip understand compound commands** — parsing `for...do...done`
   syntax in snip would add significant complexity. Not worth it.
2. **Make the plugin skip shell keywords** — the plugin would need to know
   about shell syntax. The snip binary is the right place for this.
3. **Keep current behavior** — the confusing error persists. Users hit this
   and waste time debugging PATH issues.

## Acceptance Criteria

- [ ] `snip for ...` exits with code 1 and a clear error message
- [ ] `snip while ...` exits with code 1 and a clear error message
- [ ] `snip if ...` exits with code 1 and a clear error message
- [ ] `snip case ...` exits with code 1 and a clear error message
- [ ] Existing behavior for `cd`, `source`, `export` etc. is unchanged
- [ ] All existing tests pass
- [ ] New tests cover each added keyword
