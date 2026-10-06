# Schedule performance and risk report

## Experiment setup

All tests use the local V2 CHASM Scheduler in namespace `fx-test`, one Go SDK
Worker, and task queue `fx-test-task-queue`.

The Workflow is `OneHourTimerWorkflow`. It takes no arguments and contains only
one one-hour timer:

```go
func oneHourTimerWorkflow(ctx workflow.Context) error {
    return workflow.Sleep(ctx, time.Hour)
}
```

The shared Schedule spec is:

| Setting | Value |
| --- | --- |
| Schedule interval | Every `1h`, phase `0` |
| Schedule action | Start `OneHourTimerWorkflow` |
| Task queue | `fx-test-task-queue` |
| Test span | `168h` (one week) |
| Expected actions | `24 * 7 = 168` |

The three tests differ only in overlap policy and how the one-week span is
processed:

| Test name | Make target | Schedule configuration | One-week operation |
| --- | --- | --- | --- |
| **Allow-All with Time Skipping** | `demo-week-test` | `AllowAll`; time skipping enabled | `--fast-forward 168h` |
| **Allow-All with Backfill (No Time Skipping)** | `demo-backfill-test` | `AllowAll`; time skipping disabled | Backfill the preceding `168h` |
| **Buffer-All with Time Skipping** | `demo-buffer-all-week-test` | `BufferAll`; time skipping enabled | `--fast-forward 168h` |

The backfill end is one second before an exact hourly boundary so the inclusive
backfill range produces exactly 168 hourly actions, not 169.

Run the tests from `~/projects/cli/vts-schedule-demo`:

```bash
make demo-start
make demo-week-test
make demo-backfill-test
make demo-buffer-all-week-test
make demo-status
make demo-clean
```

`make demo-start` builds and starts the local Temporal server, Web UI, and demo
Worker. Keep them running while executing the three test targets. Use
`make demo-status` and `make demo-logs` while inspecting a run.

## Metric definitions

- **Schedule actions** is the Schedule's reported action count after the test
  reaches its expected one-week boundary.
- **All Workflows started and visible** is wall-clock time until visibility
  contains one Workflow execution for every Schedule action. This answers how
  long it took to start all 168 Workflows as observed by the CLI. Because the
  measurement includes polling and visibility-indexing delay, it is an upper
  bound on pure server-side Workflow start time.
- **Completed / running** is the execution-status split when all expected
  Workflow starts have become visible. The test does not wait for running
  Workflows to finish unless `BufferAll` and virtual time naturally drain them.

## Observed results

The following results were recorded on 2026-10-06 from one local run of each
scenario:

| # | Test | Actions | All Workflows started and visible | Completed / running |
| ---: | --- | ---: | ---: | ---: |
| 1 | Allow-All with Time Skipping | 168 | 5s | 2 / 166 |
| 2 | Allow-All with Backfill (No Time Skipping) | 168 | 2s | 0 / 168 |
| 3 | Buffer-All with Time Skipping | 168 | 339s | 168 / 0 |

Tested revisions:

- CLI branch: `fx/demo-vts-scheduler`
- Server branch: `fx/vts-chasm-schedule` at `4a85881d7`
- Go SDK branch: `fx/vts-schedule` at `8fb357f4`

- **Allow-All with Time Skipping:** all starts were visible in 5s. Workflow
  timers extending beyond the bound remained running.
- **Allow-All with Backfill (No Time Skipping):** all starts were visible in 2s
  and their timers continued in real time.
- **Buffer-All with Time Skipping:** all 168 Workflows ran sequentially and
  were started and visible in 339s. That is an observed average of
  `339s / 168 = 2.02s` per Workflow.

Test 3 takes much longer because `BufferAll` permits only one Workflow at a
time. For each of the 168 actions, the server starts the Workflow, dispatches
its Workflow Task, advances its one-hour virtual timer, completes it, and then
drains the next buffered action. Time skipping removes the one-hour real-time
wait, but the orchestration work remains sequential. The 2.02s value is an
end-to-end average that also includes polling and visibility delay; it is not
the timer-skip operation alone.

## Performance and correctness risks

- These are single local development-server runs using in-memory persistence;
  they are not production capacity results or stable performance baselines.
- Visibility measurements include CLI polling, RPC latency, and visibility
  indexing. They are not server scheduling time alone.
- Timing has one-second resolution. Repeated runs and percentile statistics are
  needed before using these values as regression thresholds.
- AllowAll creates a burst of concurrent executions. CPU, memory, persistence,
  matching, and visibility limits can materially change the result in a remote
  cluster or at larger intervals.
- BufferAll performance depends on Workflow duration and Worker/server task
  throughput. The 339-second result is specific to this one-timer Workflow and
  the tested local revisions.
- The AllowAll tests intentionally pass when all starts are visible; they do not
  wait an hour for every Workflow to complete. BufferAll verifies all 168 starts
  and completions because virtual time drains them sequentially.
- The test accepts a minimum of 167 actions to tolerate a Schedule boundary
  difference, but every recorded run produced exactly 168. Unexpectedly lower
  counts should be investigated.
- A Schedule containing many simultaneous AllowAll actions may expose the known
  Web UI duplicate-`actualTime` rendering-key problem even when server data is
  valid.

For a meaningful comparison, run each target repeatedly on the same machine and
revisions, then record median and tail latency together with CPU, memory,
persistence, and Worker configuration.

## Test controls

Defaults can be overridden without editing the script:

```bash
TEMPORAL_WEEK_FAST_FORWARD=336h \
TEMPORAL_WEEK_TIMEOUT_SECONDS=900 \
TEMPORAL_WEEK_MIN_WORKFLOWS=335 \
make demo-week-test

TEMPORAL_BACKFILL_TIMEOUT_SECONDS=900 \
TEMPORAL_BACKFILL_MIN_WORKFLOWS=167 \
make demo-backfill-test

TEMPORAL_BUFFER_ALL_WEEK_FAST_FORWARD=336h \
TEMPORAL_BUFFER_ALL_WEEK_TIMEOUT_SECONDS=1200 \
TEMPORAL_BUFFER_ALL_WEEK_MIN_WORKFLOWS=335 \
make demo-buffer-all-week-test
```

Each run creates a unique Schedule ID. `make demo-status` prints every retained
test Schedule and its Web UI link. `make demo-clean` removes the Schedules,
processes, generated binaries, logs, and run metadata.
