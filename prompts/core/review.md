# Task: decide which of these mutants are worth reporting

Another agent planted mutants in a Bitcoin Core file and claims each one is a
plausible bug that the test suite would miss. Their manifest is in
`$IN_MUTANTS` and their patches are in `$PATCH_DIR`. You did not generate them
and you owe their author nothing. Your only job is to decide, mutant by mutant,
whether it is worth telling the author of the code about it.

A surviving mutant is only worth reporting when it points at a test the author
should write: a bug a developer could really make, with a consequence someone
would notice, that a test could reliably catch. A mutant no test catches is not
enough on its own. One that only resizes a cache, rewords a log line, or opens a
race no test can hit wastes the author's time just as surely as one the first
unit test kills.

Your inputs:

- `$IN_MUTANTS` - the mutants to judge.
- `$PATCH_DIR` - the patch for each, named `<id>.patch`.
- `$BITCOIN_SRC` - the Core tree the patches were cut against, clean.
- `$BIPS_SRC` - a full clone of `bitcoin/bips`, for when a claimed spec
  violation turns on what a BIP actually says.

You are in a disposable container with full permissions. Read anything, run
anything. Do not build the node; static reading is enough and a build would burn
your whole budget. You may apply a patch to look at it in context, but revert it
with `git -C $BITCOIN_SRC checkout -- <file>` before moving on, and leave the
tree clean when you finish.

## Method

For each mutant, in order:

1. **Read the patch itself.** Not the manifest's description of it. `original`
   and `mutated` in the manifest are the author's account of their own change
   and may not match the diff. Open `$PATCH_DIR/<id>.patch`, and read the
   surrounding code in `$BITCOIN_SRC` with enough context to know what the
   function does: the enclosing scope, the callers, what the early returns skip.

2. **Decide whether it is worth reporting.** The harness compiles and runs the
   test suite on every mutant itself, so do not judge whether it compiles or
   whether a test kills it. Ask three questions instead, and answer each from
   the code, not from the manifest's `severity_if_undetected`:
   - **Consequence:** construct, concretely, an input that reaches the mutated
     line and diverges, then say what a user or caller would notice. A wrong
     RPC result or error, a crash or assert, corrupt or stale on-disk data, a
     consensus or policy change, a privacy leak, memory beyond a configured
     limit, or a regression on the default configuration all count.
     Performance under a non-default flag, cache sizing within the limits, and
     log or help text do not. If no input diverges, it has no consequence.
   - **Plausibility:** would a developer write this? A copy-paste from a nearby
     line, `break` for `continue`, a flipped comparison, or a dropped call are
     plausible. An edit that needs an odd constant or would make a reviewer stop
     is not.
   - **Testability:** could a deterministic unit or functional test catch it?
     Sketch the test in one sentence. A race that needs precise thread timing,
     a value no code path exposes, or a state that only mainnet reaches and no
     test can build, fails this.

3. **Judge.** Assign exactly one verdict:
   - `good` - passes all three questions in step 2. Worth reporting.
   - `noise` - fails at least one of them. Say which.
   - `invalid` - the patch does not match what the manifest claims, or the diff
     is empty or malformed.

Being wrong in either direction costs the same. Do not call a mutant good to
be agreeable, and do not dismiss one as noise because tracing its consequence
is tedious. Either way you must name the file and lines that settled it.

## Rules

- Never add a mutant. Anything you think would have been a better mutation is
  out of scope.
- Never drop a mutant, including ones you call invalid. Every input id must
  appear in the output exactly once, with all of its original fields unchanged.
- Read every line you cite. Cite nothing from memory, and do not lean on what
  you recall about Core - the files in front of you are the only authority.
- A mutant the harness already marked `"apply_ok": false` or
  `"compiles_ok": false` is `invalid`; say so in `verdict_reason` and move on
  without spending budget on it.
- Ignore `kill_prediction` and `compile_confidence`; the harness settles both
  by running the mutant.
- If a mutant is marked `duplicate_of`, judge it on its merits but say in
  `verdict_reason` that it duplicates that id.

## Output

Write `$OUT_REVIEWED`: the object from `$IN_MUTANTS`, unchanged except that
every entry in `mutants` gains exactly these fields:

```json
{
  "verdict": "good | noise | invalid",
  "suggested_test": "one sentence describing a test that would catch it, or null when the verdict is not good",
  "verdict_reason": "2-4 sentences: the input you constructed, what diverges, what a user would notice, and why it is or is not plausible and testable. Name the files and lines you read, especially when they are not the ones the mutant cited."
}
```

Nothing else in that file: no markdown fences, no prose, no new top-level keys.
Validate with `jq empty $OUT_REVIEWED` before you finish, confirm the mutant
count matches the input, and confirm `git -C $BITCOIN_SRC status --porcelain` is
empty. Your final chat message should be one line per mutant: id, verdict, and a
few words of why.
