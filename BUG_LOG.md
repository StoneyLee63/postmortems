# Bug Log & Postmortems

**Author:** Ronnie Mitchell · **Compiled:** July 2026

This is a running log of real bugs hit while building real things not staged, not textbook. Every entry here happened during actual work: a Bash maintenance tool pushed to production on GitHub, or a multi-stage security-ops agent pipeline built stage by stage. Nothing in here was written to look good. It's written because the habit of naming a bug precisely what broke, why, how it was found, how it was fixed is the same habit that makes someone useful in a SOC and dangerous in a red team engagement. Silent failures are the ones that matter most in both jobs. Most of what's below is a silent failure, not a crash.

Each entry follows the same shape: what happened, what it actually was underneath, how it got caught, how it got fixed, and what it teaches that generalizes past this one bug.

---

## Project: Groundwork (Bash, el-tools suite)
Repo: `github.com/StoneyLee63/Bash_Tools`

### Bug 1 — Missing closing brace silently broke the script
**Symptom:** Running the script threw `unexpected end of file`. No line number pointing at the actual problem, no obvious clue in the last thing typed.

**Root cause:** The `header()` function was missing its closing `}`. Bash doesn't fail at the point the brace should have been — it fails at true end-of-file, because from the shell's perspective the function body just kept going, silently swallowing every function defined after it into one unclosed block.

**Diagnosis:** Ran `cat -n filename` to get line numbers, then read the file structurally instead of guessing — checked every function's open/close brace pairing top to bottom until the mismatch surfaced.

**Fix:** Added the missing `}`.

**What it teaches:** Bash's error location is often the *symptom* location, not the *cause* location. A missing terminator anywhere upstream reports as failure at the very end. The fix isn't reading the error message harder — it's structurally verifying pairs (braces, quotes, heredoc delimiters) instead of trusting where the interpreter says the problem is.

---

### Bug 2 — The working directory was never a git repository
**Symptom:** `git push` and related commands failed with `fatal: not a git repository`.

**Root cause:** The local `~/projects/el-tools` directory had been built and edited for weeks but was never actually initialized as, or cloned from, a real git repo. All the work existed only as local files with no version control wired to it.

**Diagnosis:** Ran `find ~ -maxdepth 4 -iname ".git" -type d` across the home directory to check what, if anything, was actually a git repo — rather than assuming the working directory was correctly set up because the code in it worked.

**Fix:** Cloned `Bash_Tools` fresh into `~/projects/Bash_Tools`, then copied the finished script in, rather than trying to retrofit git onto the existing directory.

**What it teaches:** Environment assumptions ("this folder is obviously a repo, I've been committing to it") need to be checked, not trusted, before you build a process on top of them. This is the same discipline that matters in incident response — verify the actual state of a system before acting on what you assume it is.

---

### Bug 3 — GitHub rejected password authentication
**Symptom:** `git push` failed with `remote: Invalid username or token. Password authentication is not supported for Git operations.`

**Root cause:** GitHub deprecated password auth for git operations years ago in favor of tokens — using an account password where a Personal Access Token was expected.

**Diagnosis:** Read the rejection message literally instead of retrying the same credentials — it stated the actual constraint directly.

**Fix:** Generated a GitHub Personal Access Token (classic, `repo` scope) and used it in place of the password, then set `git config --global credential.helper store` to cache it for future pushes (caching takes effect starting from the next successful push after that config command, not retroactively — a detail worth being precise about, since assuming it applied immediately caused a moment of confusion mid-fix).

**What it teaches:** Auth failures are usually policy changes, not user error — read the actual rejection text before troubleshooting the wrong layer. Also a small but real lesson in precision: "the credential is now cached" and "the credential will be cached starting next push" are different claims, and conflating them costs trust in the moment it gets caught.

---

## Project: OHE — Security-Ops Agent Pipeline
Anthropic Course work / CCA-F exam prep vehicle. 13-stage pipeline: raw API → MCP → Agent SDK → Claude Code, findings to client debrief.

### Bug 4 — A loop-scoping indentation drift silently dropped a pipeline phase (Stage 8)
**Symptom:** The stage appeared to run successfully — no error, no crash — but output was missing data that should have been there.

**Root cause:** An indentation error caused part of a loop body to execute outside the loop's actual scope, so a phase that should have processed every item in a batch only processed a subset without any signal that anything was wrong.

**Diagnosis:** Caught by comparing output count against input count — not by an error message, because there wasn't one. This is the pattern worth naming: a script that "ran clean" told a false story; only checking the shape of the output against the shape of the input exposed the gap.

**Fix:** Corrected the loop scoping so the full batch processed inside the intended block.

**What it teaches:** The most dangerous bugs don't throw. Verifying quantity (input count vs. output count) is a cheap, general-purpose sanity check that catches an entire class of silent data loss that no amount of "did it crash?" testing would ever surface.

---

### Bug 5 — MCP server launched under the wrong Python interpreter (Stage 9)
**Symptom:** The stage completed with zero findings. No crash, no error — just an empty, plausible-looking result.

**Root cause:** The MCP server was launched using the system's default `python3`, which didn't have the `mcp` package installed. The correct interpreter (with the right environment) existed but wasn't the one actually invoked.

**Diagnosis:** Traced through three separate hops of the pipeline before the actual failure point surfaced — the symptom (zero findings) was far downstream of the cause (wrong interpreter at launch).

**Fix:** Pinned the MCP server launch to the correct interpreter/environment explicitly instead of relying on whatever `python3` resolved to in the shell's PATH.

**What it teaches:** "Zero results" and "no errors" is not the same as "it worked" — it can just as easily mean the tool never actually ran. Environment ambiguity (which interpreter, which PATH, which venv) is a recurring root cause across totally different projects — this is the same category of problem as Bug 2, just in Python instead of git.

---

### Bug 6 — A tool-permission callback never fired, once, in the entire codebase (Stage 9)
**Symptom:** Tool-use permission logic that should have gated certain actions never actually triggered — not once, across the whole run.

**Root cause:** `prompt_stream()` was closing the stream early, which meant the `can_use_tool` callback never had the opportunity to run at all.

**Diagnosis:** Traced execution flow to confirm the callback function itself was correctly written and correctly registered — the bug wasn't in the logic inside the callback, it was upstream, in whether the callback was ever reachable.

**Fix:** Set `permission_mode="dontAsk"` to correct the stream-closing behavior that was preventing the callback from firing.

**What it teaches:** A permission gate that silently never engages is functionally the same as having no permission gate at all — worse, actually, because it *looks* like a safety control exists. This is directly relevant to red-teaming: assuming a control works because the code for it exists is a mistake attackers exploit and defenders make. The control has to be observed firing, not just present in the source.

---

### Bug 7 — A pipeline stage inherited the operator's entire global config (Stage 9)
**Symptom:** Stage 4 behaved differently than expected, pulling in settings that had nothing to do with the stage's own scoped configuration.

**Root cause:** The stage wasn't running in an isolated configuration context — it inherited SoulRa's entire global Claude Code settings by default, rather than only the settings explicitly scoped to that stage.

**Diagnosis:** Compared expected scoped behavior against actual behavior and traced the discrepancy back to configuration inheritance rather than logic inside the stage itself.

**Fix:** Set `strict_mcp_config=True` and `setting_sources=[]` to force the stage to run with only its own explicit configuration, with no implicit inheritance from the broader environment.

**What it teaches:** Config leakage is a real attack surface, not just a tidiness issue. A component that silently inherits more context/permissions/settings than it needs is exactly the shape of a privilege-escalation or data-leakage bug in a security context — least-privilege scoping has to be enforced explicitly, not assumed by default.

---

### Bug 8 — A drift check silently passed on an empty run (Stage 9)
**Symptom:** A verification check meant to catch pipeline drift reported "pass" even when the run had produced no actual data to check.

**Root cause:** The check's logic compared new output against expected output, but had no floor condition for "zero items processed" — an empty run trivially satisfied the comparison because there was nothing to disagree about.

**Diagnosis:** Noticed the check couldn't logically distinguish "everything matched" from "nothing ran," and traced that back to a missing zero-floor case in the verification logic itself.

**Fix:** Added an explicit zero-floor check so an empty run fails the verification instead of passing it by default.

**What it teaches:** A verification step needs its own failure modes tested, not just the thing it's verifying. "The check passed" is meaningless if the check can pass vacuously. This is the same failure category as Bug 5 (zero results looking like success) but one layer up — the safety net itself had the blind spot this time.

---

## Patterns Across All Eight

Read together, these aren't eight unrelated mistakes — they cluster into a small number of recurring failure categories that show up again whether the language is Bash or Python, whether the project is a CLI tool or an agent pipeline:

**Silent failures over loud crashes.** Bugs 4, 5, and 8 all produced clean-looking output with nothing actually wrong on the surface. None of them would have been caught by "did it run without an error." Each needed an independent check — input/output count, expected-vs-actual, a zero floor — that didn't rely on the tool honestly reporting its own failure.

**Environment and scope assumptions left unverified.** Bugs 2, 5, and 7 all trace back to trusting an assumed environment (this is a git repo, this is the right interpreter, this stage only sees its own config) instead of confirming it. Same root shape, three different surfaces.

**Controls that exist in code but don't actually engage.** Bug 6 is the clearest version of this — a permission gate that was correctly written but never reachable. Worth flagging on its own because it's the failure mode most likely to matter directly in security work: a control that looks present in a code review but never actually fires in production.

**Precision under pressure.** Bug 3's fix was correct, but the explanation of *when* it took effect wasn't precise the first time, and that gap surfaced immediately. Technical accuracy extends to describing the fix correctly, not just applying it correctly.

These patterns are the actual point of keeping this log. The bugs themselves get fixed and forgotten. The categories don't — they're the same shapes to watch for in the next system, and the same shapes worth probing for when the goal shifts from building the system to breaking one.

---

*This log grows as new bugs get caught. Nothing here is retroactively cleaned up to look better than it was — the value is in the accuracy.*
