# Unproxyable Shell Keywords

This directory documents the gap in snip's `unproxyableReason` function:
shell keywords that snip should reject with a clear error but currently
tries to execute as system commands.

## Files

- [analysis.md](analysis.md) — Detailed analysis of the gap and its impact
- [fix.md](fix.md) — The proposed fix for the snip Go binary
