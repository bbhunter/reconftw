---
phase: 01-resilient-resume-timeout-safety
reviewed: 2026-05-13T11:45:00Z
depth: standard
files_reviewed: 6
files_reviewed_list:
  - lib/parallel.sh
  - modules/core.sh
  - modules/modes.sh
  - modules/utils.sh
  - reconftw.cfg
  - tests/unit/test_verbosity.bats
findings:
  critical: 4
  warning: 7
  info: 4
  total: 15
status: issues_found
---

# Phase 01: Code Review Report

**Reviewed:** 2026-05-13T11:45:00Z
**Depth:** standard
**Files Reviewed:** 6
**Status:** issues_found

## Summary

Phase 01 adds three intertwined resilience features: a `.inprogress_<fn>` sentinel lifecycle (01-01),
mid-run disk-full guards (01-02), and `PARALLEL_JOB_TIMEOUT_SECONDS` enforcement in the parallel
heartbeat (01-03). The implementation is mostly clean, but three signal/trap correctness defects
together break the headline guarantee — "if a function is killed mid-run, the next invocation re-runs
it":

1. The EXIT trap fires even on SIGINT/SIGTERM (via `cleanup_on_exit -> exit`), wiping every
   `.inprogress_<fn>` sentinel that a Ctrl+C was supposed to leave behind. For the most common
   way users abort a recon, the resume mechanism silently produces zero sentinels and the
   resume banner never fires.
2. `PARALLEL_JOB_TIMEOUT_SECONDS` enforcement is gated on `OUTPUT_VERBOSITY >= 1`, so `--quiet`
   mode (OUTPUT_VERBOSITY=0) silently disables every timeout — the configured cap is ignored.
3. `_timeout_kill_job` sends SIGTERM/SIGKILL to the parallel subshell PID, not to the actual
   external tool process spawned inside it. Long-running children (dnsx, puredns, ffuf, etc.)
   are orphaned and continue consuming CPU/network well past the configured timeout.

Plus several quality/regression issues around an unguarded `touch` to `$called_fn_dir` (pre-existing,
but the new guard pattern at line 1480 should have been applied to line 1483 in the same commit),
serial blocking inside the heartbeat loop, and a documentation drift in `start_func`'s own comment
versus actual behavior.

## Critical Issues

### CR-01: EXIT trap defeats `.inprogress_<fn>` resume on SIGINT/SIGTERM

**File:** `modules/modes.sh:121-122`, `modules/utils.sh:116-152`
**Issue:**
The phase 01-01 design relies on `.inprogress_<fn>` sentinels surviving abnormal termination so
the next run can detect "this function was killed mid-execution and must re-run". The
implementation installs two traps in `start()`:

```bash
trap 'cleanup_on_exit' INT TERM
trap '_cleanup_inprogress' EXIT  # silent EXIT-only sentinel sweep (D-02)
```

But `cleanup_on_exit` ends with `exit "$exit_code"` (modules/utils.sh:145). Per the Bash manual,
any explicit `exit` triggers the EXIT trap. So the call chain on Ctrl+C is:

1. User presses Ctrl+C → SIGINT delivered to the running shell
2. `cleanup_on_exit` runs (INT trap) — does its cleanup, then `exit 130`
3. EXIT trap fires `_cleanup_inprogress`, which runs `rm -f "$called_fn_dir"/.inprogress_*`
4. Every sentinel for every in-flight function is deleted

Net effect: the next run finds NO sentinels (the resume banner at modules/modes.sh:166-181 never
fires) and the existing `.<fn>` checkpoints are unchanged, so any function that completed
end_func is correctly skipped but any function that was running but had not yet completed is
**also skipped** because there's no `.inprogress_<fn>` to flag it. Resume is silently broken
for the most common interruption path.

The start_func comment at modules/core.sh:1431-1434 itself says:

```
# end_func removes it before touching .<fn>; EXIT trap clears it on
# graceful exit. A leftover on the next run signals the function was killed
# mid-execution and must re-run.
```

"Graceful exit" is the misleading framing — SIGINT/SIGTERM is not graceful but it still hits
the EXIT trap. The resume feature only works when the EXIT trap does NOT fire, i.e. SIGKILL,
hardware crash, OOM kill, or bash itself dying. None of those are how users normally interrupt.

**Fix:**
Either remove the EXIT trap entirely (and accept stale sentinels from normal exits — they're
already idempotent because `.inprogress_<fn>` only takes effect when paired with a missing
`.<fn>`), or gate `_cleanup_inprogress` on "did we exit cleanly?":

```bash
# Track clean-exit state at the very end of run() / end()
_RECONFTW_CLEAN_EXIT=true

trap 'rc=$?; if [[ "${_RECONFTW_CLEAN_EXIT:-false}" == "true" ]]; then
    _cleanup_inprogress
fi' EXIT
```

Or — simpler — drop the EXIT-trap sweep entirely. The `start()` flow already removes orphans
under `FORCE_RESCAN` (modules/modes.sh:82), and a stale `.inprogress_<fn>` paired with a present
`.<fn>` is harmless: the resume banner detection at modules/modes.sh:166-181 fires the warning
but the checkpoint guard still skips the function. The only real harm of leftover sentinels is
a noisy banner on the next run, which is a small price for actually working resume.

Either way, document the actual behavior accurately in start_func's comment.

---

### CR-02: `PARALLEL_JOB_TIMEOUT_SECONDS` is silently disabled in `--quiet` mode

**File:** `lib/parallel.sh:469`, `lib/parallel.sh:579`
**Issue:**
The two heartbeat loops that enforce `PARALLEL_JOB_TIMEOUT_SECONDS` are gated on
`OUTPUT_VERBOSITY >= 1`:

```bash
local hb="${PARALLEL_HEARTBEAT_SECONDS:-20}"
if [[ "${PARALLEL_MODE:-true}" == "true" ]] && [[ "${OUTPUT_VERBOSITY:-1}" -ge 1 ]] && \
   [[ "$hb" =~ ^[0-9]+$ ]] && ((hb > 0)); then
    # ... heartbeat loop with _timeout_kill_job enforcement ...
fi
```

The heartbeat was originally a UI feature (showing live progress), and gating it on verbosity
made sense when it was UI-only. But phase 01-03 piggybacked the timeout enforcement onto the
same loop without separating the concerns. So under `--quiet` (OUTPUT_VERBOSITY=0, intended
for CI runs which are the EXACT use case the cfg comment cites: "600 for CI smoke runs"), the
loop is skipped, no `_timeout_kill_job` call is ever made, and any wedged job runs forever (or
until the user gives up and kills the whole script).

This contradicts the cfg comment at reconftw.cfg:313-318 which advertises `PARALLEL_JOB_TIMEOUT_SECONDS=600`
as a recommended CI setting. CI is precisely where users will run `--quiet`, so the feature is
disabled exactly where it's most needed.

**Fix:**
Split the gate. Run the heartbeat loop whenever timeout is enabled OR verbosity warrants UI
updates; suppress the UI updates separately:

```bash
local hb="${PARALLEL_HEARTBEAT_SECONDS:-20}"
local _to="${PARALLEL_JOB_TIMEOUT_SECONDS:-0}"
local _timeout_active=false
[[ "$_to" =~ ^[0-9]+$ ]] && (( _to > 0 )) && _timeout_active=true

local _verbose_progress=false
[[ "${PARALLEL_MODE:-true}" == "true" ]] && [[ "${OUTPUT_VERBOSITY:-1}" -ge 1 ]] && _verbose_progress=true

if { [[ "$_verbose_progress" == "true" ]] || [[ "$_timeout_active" == "true" ]]; } \
    && [[ "$hb" =~ ^[0-9]+$ ]] && ((hb > 0)); then
    # heartbeat loop; conditionally call _parallel_snapshot only if _verbose_progress
    # but always check timeout
fi
```

Mirror the change at lib/parallel.sh:579 (the same gate is duplicated for the "remaining jobs"
flush path).

---

### CR-03: `_timeout_kill_job` only kills the parallel wrapper subshell, not the running tool

**File:** `lib/parallel.sh:53-77`, `lib/parallel.sh:432-438`
**Issue:**
Each parallel job runs inside a wrapper subshell:

```bash
(
    local _rc=0
    "$func" || _rc=$?
    date +%s >"${log_file%.log}.endts"
    exit "$_rc"
) >"$log_file" 2>&1 &
batch_pids+=("$!")
```

`$!` is the PID of the subshell, not the PID of any tool the function spawns (httpx, dnsx,
puredns, ffuf, etc.). When `_timeout_kill_job` runs:

```bash
kill -TERM "$pid" 2>/dev/null || true
# ... wait grace ...
kill -KILL "$pid" 2>/dev/null || true
```

`pid` is the subshell PID. SIGTERM to a bash subshell does NOT automatically propagate to its
children. Result: the subshell dies, `wait` returns, the parallel framework records FAIL, but
the actual long-running tool (e.g. a wedged `puredns bruteforce` doing tens of thousands of DNS
lookups) is now orphaned, re-parented to PID 1, and continues consuming CPU and network until
it completes naturally or is manually killed. On Axiom, this means orphaned remote workers.

Also: `.endts` is never written (line 436 never reached), so `_parallel_emit_job_output` falls
through to `date +%s` (lib/parallel.sh:522) for the end timestamp. That's technically correct
but it means the "duration" recorded is wall-clock-at-kill-time, which is fine.

The deeper problem: timeout enforcement doesn't actually stop the work, only the bookkeeping
about it.

**Fix:**
Two options:

1. **Process-group kill.** Launch the wrapper subshell with its own process group (`setsid`
   or `set -m` inside the subshell) and send signals to `-PID` (the negative form,
   targeting the whole process group):

   ```bash
   # In _timeout_kill_job:
   kill -TERM -- "-${pid}" 2>/dev/null || kill -TERM "$pid" 2>/dev/null || true
   ```

   But `set +m` is intentionally set in start() (modules/modes.sh:16), so process groups
   are inherited from the parent shell. `setsid` is the cleanest approach but requires
   the binary on the host.

2. **Recursive child kill.** Walk the process tree of the subshell PID and kill all
   descendants explicitly. portable across macOS/Linux:

   ```bash
   _kill_tree() {
       local parent="$1" sig="$2"
       local child
       for child in $(pgrep -P "$parent" 2>/dev/null); do
           _kill_tree "$child" "$sig"
       done
       kill "-$sig" "$parent" 2>/dev/null || true
   }
   ```

   Then in `_timeout_kill_job`:

   ```bash
   _kill_tree "$pid" TERM
   for ((i=0; i<grace; i++)); do
       kill -0 "$pid" 2>/dev/null || break
       sleep 1
   done
   _kill_tree "$pid" KILL
   ```

   `pgrep -P` is widely available on both BSD and GNU procps and is already an implicit
   dependency in many reconftw flows. Falls back gracefully if a child has already exited.

---

### CR-04: `touch "$called_fn_dir/.${fn}"` is unguarded — writes to root filesystem when called_fn_dir is empty

**File:** `modules/core.sh:1483`
**Issue:**
Phase 01-01 wrapped the new `.inprogress_<fn>` writes in a guard:

```bash
if [[ -n "${called_fn_dir:-}" ]]; then
    rm -f "$called_fn_dir/.inprogress_${fn}" 2>/dev/null || true
fi
touch "$called_fn_dir/.${fn}"   # <-- unguarded
```

If `called_fn_dir` is unset or empty (e.g. a custom workflow or a test runner that sources
core.sh without going through `start()`), the touch expands to `touch /.${fn}`. As root this
will silently create files at the filesystem root; as a non-root user it will produce a
permission-denied error to stderr that pollutes any normal-mode output.

The phase added the proper guard for the new sentinel write but did not apply it to the
existing checkpoint write. The two are adjacent and conceptually the same operation. The
test harness at tests/unit/test_verbosity.bats:32-33 happens to set `called_fn_dir` explicitly,
so the tests pass — but any future test that omits this setup will trip the bug.

**Fix:**

```bash
if [[ -n "${called_fn_dir:-}" ]]; then
    rm -f "$called_fn_dir/.inprogress_${fn}" 2>/dev/null || true
    touch "$called_fn_dir/.${fn}" 2>/dev/null || true
fi
```

Same fix applies to `end_subfunc` at modules/core.sh:1570 (`touch "$called_fn_dir/.${2}"` is
also unguarded).

---

## Warnings

### WR-01: `_timeout_kill_job` blocks the heartbeat loop serially for `grace` seconds

**File:** `lib/parallel.sh:60-65`
**Issue:**
The grace-period loop inside `_timeout_kill_job` runs in the foreground:

```bash
for ((i=0; i<grace; i++)); do
    kill -0 "$pid" 2>/dev/null || break
    sleep 1
done
```

When called from inside the heartbeat loop (lib/parallel.sh:484-486), this blocks the next
heartbeat iteration for up to `PARALLEL_KILL_GRACE_SECONDS` (default 10s). If multiple parallel
jobs hit timeout in the same batch, each gets the full grace serially — for 4 jobs at 10s grace,
that's 40s of blocked progress. During this stall, other still-alive jobs aren't being checked
for their own timeout, and no heartbeat progress update is emitted.

**Fix:**
Either reduce the polling interval (poll every 200ms instead of 1s, so cap is similar but more
responsive), or restructure `_timeout_kill_job` as a "fire and forget" — send TERM, record a
"deadline_at" timestamp per PID in a map, and let the next heartbeat iteration KILL if still
alive past the deadline. The current code is correct but not concurrent-friendly.

---

### WR-02: `_check_disk_mid_run || _abort_disk_full` aborts during parallel jobs causing aggregator skew

**File:** `modules/core.sh:1422`, `modules/core.sh:1551`, `modules/utils.sh:445-455`
**Issue:**
`start_func` and `end_func` both call the disk check, which on failure calls `_abort_disk_full`
which calls `exit 1`. When this fires inside a parallel job (i.e. inside the wrapper subshell at
lib/parallel.sh:433-438), only the subshell exits with rc=1. The parent shell continues with
the other parallel jobs; the aggregator records this job as FAIL and moves on. The user sees
ONE FAIL badge but the actual cause (disk exhaustion) is logged only via `_print_error` which
goes to stderr — and in the heartbeat-disabled `--quiet` path that stderr never reaches the
batch summary.

So a disk-full event during parallel scanning produces:
- One FAIL badge for whichever function happened to call start_func/end_func first
- All sibling parallel jobs continue and may also FAIL with cascading disk errors
- The actual "we ran out of disk" message is buried in stderr
- The parent shell does NOT abort, so subsequent stages still try to run

The EXIT trap also doesn't fire in the subshell's `exit 1` for the parent shell, so
`_cleanup_inprogress` won't be called for those parallel sentinels until the parent exits.

**Fix:**
`_abort_disk_full` should detect whether it's running in a parallel subshell and propagate
back to the parent. One pattern: write a marker file (e.g. `${called_fn_dir}/.abort_disk_full`)
before `exit 1`, and have `parallel_funcs` check for that marker after the wait loop and
re-call `_abort_disk_full` from the parent. Or simpler: drop the disk check from `start_func`
when running under a parallel wrapper (set a env var like `_PARALLEL_CHILD=1` in the wrapper
subshell, and `_check_disk_mid_run` becomes a no-op when that's set).

The current behavior — "abort the child, leave the parent and its siblings running" — is
strictly worse than either "abort the whole run" or "warn but continue".

---

### WR-03: Repeated `_timeout_kill_job` calls would re-fire if signal delivery races

**File:** `lib/parallel.sh:484-486`, `lib/parallel.sh:594-596`
**Issue:**
The current code triggers `_timeout_kill_job` once per heartbeat iteration if `kill -0 "$pid"`
succeeds AND `job_dur > _to`. After the first call:

- `kill -TERM` is sent
- grace-loop waits up to `grace` seconds, polling `kill -0`
- If still alive after grace, `kill -KILL` is sent

If the timeout triggers but the subshell happens to be paused inside a `wait` for its own
external tool (which is the typical case), SIGTERM may not be processed immediately by bash.
The 10s grace then ends, SIGKILL goes out, the subshell dies. So far OK.

But: between iterations of the OUTER heartbeat loop, `_timeout_kill_job` returns. The OUTER
loop continues. Next iteration: `kill -0` on a freshly killed PID returns false 99% of the
time (because SIGKILL was just sent), so the function isn't re-invoked. Good.

But there's a narrow race: on a heavily-loaded system, `kill -0` immediately after SIGKILL can
still succeed briefly (the kernel hasn't reaped the zombie). If that happens, the next
heartbeat iteration sees the PID is alive AND `job_dur > _to`, so it calls `_timeout_kill_job`
AGAIN. The second call will sleep up to 10 more seconds polling `kill -0` (the process IS dead,
just not reaped, so kill -0 returns 0 until parent waits), then send another SIGKILL (no-op,
already dead). Then it overwrites `.status_<fn>` and `.status_reason_<fn>` again — no real
harm, but wasteful and could double-log to `log_json`.

**Fix:**
Track which PIDs have already been signaled in a local associative array:

```bash
declare -A _timed_out_pids=()
# In heartbeat loop:
if [[ -z "${_timed_out_pids[${batch_pids[$idx]}]:-}" ]] && (( job_dur > _to )); then
    _timeout_kill_job ...
    _timed_out_pids[${batch_pids[$idx]}]=1
fi
```

---

### WR-04: Resume banner is shown only for first start() call; subsequent modes don't re-detect

**File:** `modules/modes.sh:166-181`
**Issue:**
The resume banner detection lives inside `start()`, which is called by `passive()`, `all()`,
`subs_menu()`, `zen_menu()`, and `monitor_mode()`. But the EXIT trap that wipes sentinels is
also installed inside `start()` (line 122). On a multi-stage workflow like `all()`, the second
call to `start()` (from within `recon()` -> via `subs_menu` etc) re-installs both traps over
the existing ones — but by that point, any `.inprogress_<fn>` from a prior aborted run in the
same target has already been wiped by the FIRST start() call's EXIT trap when the script
exited... wait, no — if we got here, the script DIDN'T exit. So the FIRST start() ran, installed
the EXIT trap, but we're still in the same shell so it hasn't fired yet. The banner from the
first start() call already reported the leftovers. By the second start() call, those sentinels
have presumably been re-created by the running functions.

So this is actually fine in the same-shell-multi-start case, but the documentation isn't clear
about it. A reader of the code at line 162-181 would reasonably expect the banner to fire on
every `start()` call. It only meaningfully fires once per shell invocation (the first one).

**Fix:**
Add a comment clarifying that the banner reflects pre-run-time state and that subsequent
`start()` calls in the same shell would see only sentinels from in-flight functions. Or
extract the banner check into a separate function `_resume_banner_emit_once` guarded by a
"shown" flag:

```bash
[[ "${_RESUME_BANNER_SHOWN:-false}" == "true" ]] && return 0
_RESUME_BANNER_SHOWN=true
```

This will reduce noise if any code path ever calls start() twice with sentinels present.

---

### WR-05: Heartbeat sleep 1 + timeout granularity skews on long thresholds

**File:** `lib/parallel.sh:498-504`, `reconftw.cfg:317`
**Issue:**
The cfg comment at line 317 says:

> Actual kill latency is the threshold + ~1s (heartbeat poll cadence) + PARALLEL_KILL_GRACE_SECONDS before SIGKILL.

But the heartbeat loop only fires the snapshot every `PARALLEL_HEARTBEAT_SECONDS` (default 20).
The TIMEOUT check, however, runs on every iteration (every 1s) because it's inside the inner
for-loop over `${!batch_pids[@]}` which runs every iteration of the outer `while :` loop with
`sleep 1`. So the timeout granularity IS ~1s — the comment is accurate.

But: if `PARALLEL_HEARTBEAT_SECONDS` is misconfigured to a non-positive value (e.g. user sets
`PARALLEL_HEARTBEAT_SECONDS=0` to silence heartbeat noise), the entire `while :` loop is
skipped (the `((hb > 0))` guard at line 469 prevents entry). When that happens, timeout
enforcement is also disabled — no wait loop runs at all between the time jobs are launched
and the time `wait "${batch_pids[$idx]}"` blocks at line 511. The user gets neither heartbeat
output nor timeout enforcement.

**Fix:**
Either document that `PARALLEL_HEARTBEAT_SECONDS=0` disables timeout enforcement (currently
undocumented), or detangle: run the loop unconditionally for timeout if `PARALLEL_JOB_TIMEOUT_SECONDS > 0`,
and emit heartbeat snapshots only if `PARALLEL_HEARTBEAT_SECONDS > 0`. Pair with CR-02 fix.

---

### WR-06: `DNS_BRUTE_TIMEOUT=6h` / `DNS_RESOLVE_TIMEOUT=4h` defaults are aggressive for axiom workflows

**File:** `reconftw.cfg:394-395`
**Issue:**
The defaults were flipped from `0` (disabled) to `6h`/`4h`. On a large target with axiom-distributed
puredns runs, 4-6h is well within normal completion times — but if the wall-clock hits exactly
4h+1s on a job that has 30s more work to do, the `timeout -k 10s 4h puredns ...` will SIGTERM
then SIGKILL with no graceful drain. This is a meaningful behavior change for users who
previously relied on "no timeout means it runs as long as it needs to".

Also: these timeouts go via `_dns_timeout_enabled` -> `timeout -k 10s "$timeout_value" "$@"`
in modules/utils.sh:1463. The `-k 10s` means SIGKILL 10s after SIGTERM. So the absolute hard
limit is 4h10s / 6h10s.

For an unattended VPS run with axiom and many targets, this could orphan partial puredns
outputs or leave wildcard-filter state in an inconsistent state if the kill hits during
puredns's resolver-trust round.

**Fix:**
This is a deliberate config default change and is probably acceptable. But the comment at
line 394-395 should be more explicit:

```cfg
# DNS bruteforce hard timeout (timeout/gtimeout -k 10s). Sends SIGTERM then SIGKILL 10s later.
# Defaults to 6h to protect against hung resolvers; set 0 to disable for unattended axiom runs
# where individual puredns/dnsx jobs may legitimately take longer than this on huge targets.
DNS_BRUTE_TIMEOUT=6h
```

Also worth noting in the phase 01-03 SUMMARY: the changeover may surface in long-running
axiom CI jobs that previously completed at 5h59m. Recommend a release note flag.

---

### WR-07: `start_subfunc` does not write `.inprogress_<fn>` — resume mechanism is silently disabled for sub-functions

**File:** `modules/core.sh:1556-1567`
**Issue:**
The resume sentinel pattern is only applied to `start_func`/`end_func`, not to
`start_subfunc`/`end_subfunc`. So `sub_passive`, `sub_crt`, `sub_active`, `sub_tls`, `sub_noerror`,
`sub_brute`, `sub_dns`, `sub_scraping`, `sub_analytics`, etc. (which all use `start_subfunc`) are
NOT protected by the resume mechanism. If any of these are killed mid-execution, the next run
has no sentinel to fall back on.

This might be deliberate (the parent `subdomains_full` function IS protected), but it's an
inconsistency that future contributors will miss. Sub-functions also touch their own `.<fn>`
checkpoint files (line 1570) and `.status_<fn>` files (line 1585), so they participate in the
checkpoint guard system — but without an inprogress marker, a kill mid-sub-function means the
next run sees no `.<fn>` (correct) and reruns it, BUT the parent's `.<fn>` might already exist
(making the parent skip the entire sub-function block).

**Fix:**
Either:
- Mirror the inprogress pattern in `start_subfunc`/`end_subfunc` for consistency.
- Document explicitly in start_subfunc's header comment that sub-functions are NOT covered
  by the resume sentinel system and rely solely on `.<fn>` / `.status_<fn>` checkpoints.

---

## Info

### IN-01: Resume banner uses `ls -1` instead of glob — empty case relies on stderr suppression

**File:** `modules/modes.sh:168`
**Issue:**
```bash
mapfile -t _leftover < <(ls -1 "${called_fn_dir}"/.inprogress_* 2>/dev/null)
```

`ls -1` with no matching files prints to stderr ("No such file or directory") which is
silenced by `2>/dev/null`. A pure bash glob is faster and more idiomatic:

```bash
shopt -s nullglob
local -a _leftover=("${called_fn_dir}"/.inprogress_*)
shopt -u nullglob
```

This avoids forking `ls` per `start()` call. Not a bug — just style and consistency with how
the rest of reconFTW handles file enumeration.

---

### IN-02: `local _to` re-declared inside hot loop

**File:** `lib/parallel.sh:483`, `lib/parallel.sh:593`
**Issue:**
`local _to="${PARALLEL_JOB_TIMEOUT_SECONDS:-0}"` is declared inside the for-loop over
`${!batch_pids[@]}`. Per parallel batch, this happens once per heartbeat iteration per job —
typically a few hundred times per scan. Bash's `local` does have overhead.

**Fix:**
Hoist `_to` to the outer `local` declaration block at line 470:

```bash
local last_hb now alive hb_active_list job_dur dur_fmt queue_count batch_elapsed hb_done_count _to
last_hb=$(date +%s)
_to="${PARALLEL_JOB_TIMEOUT_SECONDS:-0}"
[[ "$_to" =~ ^[0-9]+$ ]] || _to=0
while :; do
    ...
    if (( _to > 0 )) && (( job_dur > _to )); then ...
```

Trivial micro-optimization, but it also makes the code clearer about scope.

---

### IN-03: Disk-full status message goes to stderr via `_print_error` — invisible in JSONL mode

**File:** `modules/utils.sh:452`
**Issue:**
```bash
function _abort_disk_full() {
    _print_error "disk_full: aborting (${DISK_SPACE_INFO:-disk space exhausted})"
    log_json "ERROR" "${FUNCNAME[1]:-main}" "Disk space exhausted" "reason=disk_full" "info=${DISK_SPACE_INFO:-unknown}"
    exit 1
}
```

`_print_error` writes to stderr in human format. In `--log-format jsonl-strict` mode, the
operator will see ONLY the `log_json` line (which is structured), but the human stderr
message will still be emitted unsuppressed, breaking the JSONL-only contract.

**Fix:**
Use the standard ui-aware print helper that checks for JSONL mode:

```bash
function _abort_disk_full() {
    if declare -F ui_human_output_enabled >/dev/null 2>&1 && ui_human_output_enabled; then
        _print_error "disk_full: aborting (${DISK_SPACE_INFO:-disk space exhausted})"
    fi
    log_json "ERROR" "${FUNCNAME[1]:-main}" "Disk space exhausted" "reason=disk_full" "info=${DISK_SPACE_INFO:-unknown}"
    exit 1
}
```

Same pattern used at lib/parallel.sh:264-269 in `_parallel_emit_job_output`.

---

### IN-04: Bats tests do not exercise the new resume sentinel lifecycle

**File:** `tests/unit/test_verbosity.bats:36-41`
**Issue:**
The test stubs `_check_disk_mid_run` and `_abort_disk_full` to no-ops, which is fine. But the
tests don't verify the new behavior:

- That `start_func` creates `.inprogress_<fn>`
- That `end_func` removes `.inprogress_<fn>` and then touches `.<fn>`
- That the resume banner detection in `start()` finds leftover sentinels

The phase plan 01-01 likely has its own test suite, but this file (test_verbosity.bats) only
covers verbosity gating. If start_func's resume-write breaks, the existing tests still pass
because `called_fn_dir` is set in setup() and the touch-without-check succeeds.

**Fix:**
Add at minimum a smoke test:

```bash
@test "start_func creates .inprogress_<fn> sentinel" {
    OUTPUT_VERBOSITY=0
    run start_func "test_fn" "Testing function"
    [ "$status" -eq 0 ]
    [ -f "$called_fn_dir/.inprogress_test_fn" ]
}

@test "end_func removes .inprogress_<fn> before touching checkpoint" {
    OUTPUT_VERBOSITY=0
    start=1
    touch "$called_fn_dir/.inprogress_test_fn"
    run end_func "Results" "test_fn"
    [ "$status" -eq 0 ]
    [ ! -f "$called_fn_dir/.inprogress_test_fn" ]
    [ -f "$called_fn_dir/.test_fn" ]
}
```

This is consistent with the existing test pattern using sed-extracted functions.

---

_Reviewed: 2026-05-13T11:45:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
