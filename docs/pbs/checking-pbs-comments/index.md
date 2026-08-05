# FAQ: Why Did My Job Fail?

This FAQ covers the most common reasons PBS jobs fail or don't run on
Derecho and Casper, organized by what you're actually seeing happen —
not by PBS's internal terminology.

---

## Start Here: General Diagnostic Flow

Before searching below, walk through this checklist:

1. **Read standard error/output** (`job_name.e<id>` / `.o<id>`) for error messages.
   - System messages (segfaults, killed, out of memory)
   - Common script bugs (bad paths, missing modules, typos)
2. **Check application-specific log files** — many codes write their own logs
   that contain the *real* error, separate from stdout/stderr.
3. **Query PBS directly:**
   ```
   qstat -xf <job_number>
   ```
   - Read the `comment` field — see categorized examples below.
   - Read `Exit_status` — see the [Exit Status Reference](#exit-status-reference) below.
4. **Use debugging tools** if the above doesn't explain it (gdb4hpc, Totalview,
   or compiling with bounds/array checking enabled).

**Tip:** To find out *which* command in your script actually failed (since
`Exit_status` only reflects the *last* command run):
- Add `set -x` (bash) or `set echo` (csh) near the top of your script.
- Add `#PBS -j oe` (joins stdout+stderr into one file) and `#PBS -k oed`
  (writes that file live, directly, to a shared filesystem like `$SCRATCH`,
  so you can `tail -f` it while the job runs).

---

## Category 1: My Job Won't Start (Stuck Pending)

These are almost all resource-matching problems — either the scheduler can't
find what you asked for right now, or your request is malformed and can
*never* be satisfied. Check `qstat -xf <job_number>` and look at the `comment`
line to tell which.

### "Not Running" vs "Can Never Run" — what's the difference?
- **`Not Running`** = temporary. The resources you asked for exist on the
  system but aren't free *right now*. Your job will run once they free up.
  No action needed (unless it's taking unreasonably long — see below).
- **`Can Never Run`** = permanent. Your request can't be satisfied as
  written, no matter how long you wait. This means your script has a
  problem you need to fix.

### `comment = Not Running: Insufficient amount of resource: Qlist`
This usually means you submitted directly to an **execution queue** instead
of a **routing queue**. Submit to the appropriate routing queue and let PBS
route it — don't target the execution queue directly.

### `comment = Not Running: Insufficient amount of resource: ncpus`
Not enough free CPUs cluster-wide (or in the queue you targeted) at this
moment to satisfy your `select` statement. This is transient — the job will
start once cores free up. If it persists unusually long, double check you
haven't requested more cores than the queue's per-job maximum.

### `comment = Not Running: Insufficient amount of resource: mem`
Example:
```
Resource_List.select = 8:ncpus=128:mpiprocs=128:mem=240GB:ompthreads=1
```
Not enough free memory available right now to satisfy the request. Also
worth checking: **each Derecho CPU node maxes out at 235GB of usable RAM**.
There's no "largemem" node type — if you need more memory per core than the
default 235GB/128-core ratio provides, you must *under-subscribe* the node
(request fewer `ncpus` than physically exist per node) to raise your
effective memory-per-core.

### `comment = Not Running: Insufficient amount of resource: ngpus (R: 4 A: 3 T: 328)`
Read this literally: **R**equested / **A**vailable / **T**otal.
Example: `R: 4 A: 3 T: 328` means you asked for 4 GPUs, only 3 are free
cluster-wide right now, out of 328 total GPUs on the system.

Important nuance: if your `select` line requests all GPUs on a **single
node** (e.g., `nodect=1` with `ngpus=4`, exclusive host), scattered free
GPUs across *different* nodes don't help — you specifically need a whole
4-GPU node free at once. This is not a problem with your job; it's just
waiting for a full node to open up.

### `comment = Not Running: Insufficient amount of resource: vnode` (Casper)
Example:
```
Resource_List.select = 1:ncpus=1:mem=4GB:vnode=casper44:ompthreads=1
```
You requested a *specific named node* (`vnode=casper44`) and that particular
node isn't free. Unlike generic resource requests, pinning to one vnode means
you're waiting on that exact machine — no other node will satisfy it, even
if idle.

### `comment = Can Never Run: No Select`
Your `select` line has a syntax problem. Common cause: **uppercase resource
names are not allowed.** For example, this will never run:
```
Resource_List.select = 1:NCPUS=64:MPIPROCS=64:mem=235gb:ompthreads=1
```
Fix: use lowercase resource keywords (`ncpus`, `mpiprocs`, `mem`, `ompthreads`).

### `Not Running: Job is requesting an exclusive node and node is in use`
You asked for exclusive access to a node, but it's currently occupied by
another job. Transient — wait, or reconsider whether exclusive access is
necessary.

### `Not Running: Job would conflict with reservation or top job`
The scheduler has a standing reservation (or a high-priority "top job") that
your job's resource request would interfere with. Transient; will clear once
the reservation/top job passes.

### `Not Running: Not enough free nodes available`
Example:
```
Resource_List.select = 200:ncpus=128:mpiprocs=32:ompthreads=4
Resource_List.walltime = 12:00:00
```
Large node-count requests (here, 200 whole nodes) simply take longer to find
an opening for. This is normal queueing behavior for big jobs, not an error.

### `Not Running: User has reached queue jhublogin running job limit` (Casper)
You already have a JupyterHub session (or sessions) occupying your allowed
slot(s) on the `jhublogin` queue. Close/stop an existing session before
starting a new one.

---

## Category 2: My Job Starts, Then Dies Immediately

### `comment = job held, too many failed attempts to run`
Example:
```
run_count = 21
eligible_time = 149:44:45
Exit_status = -3
```
PBS actually tried to *launch* your job repeatedly (here, 21 times). Each
attempt failed right at the launch/execution step, so PBS auto-requeued it.
After enough consecutive failures, PBS stops trying on its own and puts the
job in a **held** state (`job_state = H`) so it isn't burning scheduler
cycles on something that clearly can't succeed unattended.

This points to a problem at job *startup* — not inside your application
logic. Common causes: a bad `#PBS` directive, a broken environment/module
load, permissions issues, or (frequently, with Dask-based jobs) a worker
launch script that fails before your actual code ever runs. Check the job's
stdout/stderr from the *first* attempt, and any Dask/scheduler logs, for the
real cause — then release the hold once fixed (`qrls`).

---

## Category 3: My Job Runs, Then Gets Killed Mid-Execution

### Exceeded walltime
Example:
```
comment = Job run at Wed Jul 22 at 07:48 on (dec2385:ncpus=64:mem=246415360kb:ngpus=0) and exceeded resource walltime
Exit_status = -29
```
Your job ran longer than the `walltime` you requested, and PBS killed it.
Fix: increase `Resource_List.walltime`, or optimize/checkpoint your job so
it completes (or can be restarted) within the requested window.

### Exceeded memory during execution
If your job dies partway through with an out-of-memory signal (rather than
being killed by PBS immediately), check whether your `mem` request in
`select` matches what your application actually needs at peak usage — not
just its average usage.

---

## Category 4: My Job Finished, But Something's Wrong

### Understanding `Exit_status` {#exit-status-reference}

| Range | Meaning |
|---|---|
| **0** | Success — but only for the *last* command PBS ran. An earlier command in your script may have failed silently. Use `set -x` / `set echo` (see top of this FAQ) to confirm which commands actually ran and succeeded. |
| **1–127** | An error from *your script or application*. This is a standard Unix exit code from your program. |
| **128+** | The process was killed by the operating system via a signal. Subtract 128 from the exit status to get the signal number (e.g., 137 → 128+9 → `SIGKILL`), then look up that signal number to identify the cause (often OOM-kill, segfault, or an external `kill`). |
| **Negative numbers** | Generated by the **PBS scheduler or MOM daemon** itself, not your application. These correspond to the `comment` field categories above (e.g., `-29` = walltime exceeded). Cross-reference with PBS Pro documentation for less common negative codes. |

### My job "succeeded" (`Exit_status = 0`) but the output is wrong
Almost always means a command *earlier* in a multi-step script failed, but
the final command in the pipeline still returned success. Add `set -x` (bash)
or `set echo` (csh) and re-run to see exactly which line executed what.

---

## Reference: Derecho Memory Notes

- All Derecho CPU nodes are identical — there's no separate "largemem" vs
  "smallmem" node class.
- Max usable RAM per CPU node: **235GB**.
- If you need more memory per core than the default (235GB ÷ 128 cores),
  you must under-subscribe the node — i.e., request fewer `ncpus` per node
  than physically exist, leaving some cores idle to raise memory-per-used-core.

---

## Debugging Tools

If the above doesn't reveal the cause:
- **gdb4hpc** — parallel debugger for HPC-scale runs
- **Totalview** — GUI-based parallel debugger
- **Recompile with checking enabled** — e.g., array-bounds checking,
  floating-point trap flags — to surface memory/logic errors that run
  silently otherwise

---

*Source material: Derecho/Casper PBS comment examples and
[NCAR HPC Docs: Moving from Cheyenne to Derecho](https://ncar-hpc-docs.readthedocs.io/en/latest/compute-systems/derecho/moving-from-cheyenne/)*
