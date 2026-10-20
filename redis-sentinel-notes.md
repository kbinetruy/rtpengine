# rtpengine: Redis failover fixes, Sentinel support and TLS — notes

Background for the three pull requests to sipwise/rtpengine on branches `redis-failover-fixes`,
`redis-sentinel` and `redis-tls` of this fork: the full lab results, method, failure modes, locking
notes, per-option detail and test detail that the pull request descriptions leave out. The code,
`docs/rtpengine.md` and the commit messages are the reference; this file only adds the measurements
and the reasoning behind them.

Contents:

1. [Fixes (branch `redis-failover-fixes`)](#1-fixes-branch-redis-failover-fixes)
2. [Sentinel, repair and resync (branch `redis-sentinel`)](#2-sentinel-repair-and-resync-branch-redis-sentinel)
3. [TLS (branch `redis-tls`)](#3-tls-branch-redis-tls)

The lab is one 4 vCPU host with Docker: Redis 7.2.16 with one master, two replicas and three
Sentinels (quorum 2, `down-after-milliseconds 2000`, `failover-timeout 4000`,
`min-replicas-to-write 1`), rtpengine in userspace, and a load of about 21 Redis writes a second
over ng (SDES calls). "Today" means unpatched upstream master. Every other figure was measured on the
code of these branches.

## 1. Fixes (branch `redis-failover-fixes`)

No sleep and no attempt under the lock on reconnect, refused writes counted, no write-back of deleted calls, CLI arguments parsed as `str`.

Six commits to the Redis connection code and the CLI, found while measuring a Redis master failover. The first four commits of this PR were merged as `20bcdce7`, `e5ff3ea8`, `1c447b50` and `6b3bf829`. Two new options, `--redis-reconnect-pause` (1000 ms by default) and `--redis-reconnect-slow` (100 ms by default), set the pacing of reconnect attempts. Left unset, they give the behaviour described here.

**Behaviour change on reconnect (#799, #1865).** On a reconnect, rtpengine no longer sleeps 2 s under `r->lock` waiting for the master role, and no longer makes the attempt under that lock. One thread makes one attempt, without the lock. With a floating IP that moves within those 2 s, the sleep used to let a pending write through. Now, after a wrong role or a slow failure (one that took `--redis-reconnect-slow` or more), Redis operations on that connection are skipped for `--redis-reconnect-pause`. During an attempt, the other Redis operations on that connection wait for it for up to `--redis-reconnect-slow`, then are skipped too. A quicker reconnect lets them through, as before. Because `--redis-allowed-errors` counts these paced attempts, N allowed errors now take at least N pauses to use up (N seconds with the defaults; documented under that option).

**What a reconnect still costs signalling.** An operation waits at most `--redis-reconnect-slow` for a reconnect, however long the attempt takes, with two exceptions:

- the request whose own thread makes the attempt waits for it, up to `--redis-connect-timeout` plus `--redis-cmd-timeout` for each of `AUTH`, `SELECT` and `INFO`, once per pause;
- at the start of an outage, the first command on the established connection still holds `r->lock` until it is answered or `--redis-cmd-timeout` expires.

In the lab, with rtpengine cut from the master (packets dropped, figures below), ng p99 during the cut went from 1609–1635 ms with the attempt under the lock to 733–933 ms, and p90 from 619–641 ms to 103–108 ms.

**What a skipped write costs, and what upstream does.** Upstream also drops the update whose reconnect fails: `redis_update_onekey()` returns without writing, and nothing queues or retries it. Its attempt holds `r->lock`, so every other Redis user, and so every ng request that writes a call, waits behind it: about 2 s per attempt when the node has the wrong role, 1 s against a dead or cut node. With this PR, the dropped writes are the update of the failed attempt, plus the writes that arrive during an attempt longer than `--redis-reconnect-slow` or during the pause. Without the follow-up PR, a dropped `SET` is lost until the call's next signalling (a never-stored SRTP call is also rewritten by the call timer once a second until it is stored). A dropped `DEL` is lost until the key expires (`--redis-expires`): a takeover restores a dead call. When a deletion had to wait for a reconnect and a new call has taken its ID meanwhile, its `DEL` is not sent: the key belongs to the new call.

Measured with plain `--redis` and `--redis-cmd-timeout=1000`, in three scenarios (a failover with rtpengine pinned to the demoted master, a frozen master, rtpengine cut from the master), 452 ng requests per run:

- upstream (8 runs): 213–298 requests per run got no answer within the load tool's 10 s timeout, and the ng median during the event was 10 s;
- this PR (16 runs): no request reached the 10 s timeout; the slowest single request took about 2 s (the one whose thread makes the attempt);
- `--redis-reconnect-pause=0` against the default: about the same number of writes acknowledged but not stored (289–304 against 292–304 in the failover and the freeze, 232 against 237 in the cut), but up to about 40 reconnects a second against a node that answers with the wrong role, and an ng median of about 100 ms during a freeze or a cut (every write waits `--redis-reconnect-slow` for the attempt in flight).

Not measured, from the code: after a single failed attempt against an otherwise healthy server, the pause can drop up to one pause of updates more than pause 0 if the failure was slow or found the wrong role, and nothing more after a quick refusal.

**Deleting a call that was never written now takes `r->lock`,** as deleting a stored call always did. This closes a race: the "kill calls" timer rewrites SRTP calls that were never written, and can hold one while it waits for `r->lock`. If the call was deleted meanwhile, the deletion had decided without the lock that nothing was stored, and the timer wrote the call back. On a healthy Redis the extra lock is uncontended, and only calls that were never written take it. Behind a hung master, such a deletion waits like every write already does.

**CLI (TCP CLI and ng `cli`).** The handlers now parse their argument as a `str`. As with the stricter `str_to_X` helpers (`bc08b059`), a numeric `set` value followed by other characters (`10x`) is refused with "no digits found", where `strtol()` took the leading digits. The seven handlers that had no digit check (`loglevel`, `redisallowederrors`, `redisdisable`, `redisdisabletime`, `redisconnecttimeout`, `rediscmdtimeout`, `controltos`) refuse a non-number instead of reading it as 0, which for `rediscmdtimeout` disabled the timeout. `set loglevel 7x` and `set loglevel core x` are refused and leave the level unchanged; master's `atoi()` read them as 7 and 0. `set maxbw -1` is now refused with "no digits found"; master accepted it and set 2^64−1.

Audit of the numeric parses reachable from `cli_handle()`: every handler that the series changes checks for its default before using the value. Two sites keep master's lenient default and are not changed, as they are pre-existing and outside this series: `call … tag … delay` (`5x` sets a delay of 1) and `call … tag … detdtmf` (`0x` enables detection). `terminate` matches `all`, `own` and `foreign` exactly: a prefix such as `terminate a` used to terminate all calls.

Apart from the changes above, the two options and a new statistic, Redis behaviour changes only during a reconnect and after a write error.

25 files, +1583 −804; tests (`t/`, `perl/`) +1231 −675, most of the removed lines from five older tests moved to the shared fake Redis module.

### Problem (measured)

The lab: Redis 7.2.16 with one master, two replicas, three Sentinels and `min-replicas-to-write 1`. rtpengine points at the master by IP. The load is about 21 writes a second. The tests were `SENTINEL FAILOVER` and a 30 s freeze of the master, with 20 runs on unpatched master in all (per-scenario counts below).

- **`--redis` + `--redis-write`.** After the demotion, every write gets `READONLY` from the same IP (392–404 per run), and nothing is logged. 114–149 of 152–200 acknowledged creates were never stored (3 runs).
- **`--redis` alone.** The role check sleeps 2 s per update under `r->lock`. All 188 ng requests after the demotion timed out (3 runs).
- **Frozen master.** No ng reply for 34.3–35.5 s (5 runs), even with `--redis-cmd-timeout=1000`.
- **A deleted call written back** (the race above). It is rare in the lab: one deleted call was left in Redis in 2 of 25 frozen-master runs without the fix (with the follow-up PR's options on; the writer was the timer). With the fix, none came back in 9 frozen-master runs with those options. The proof is the deterministic test below.
- **ng `cli`.** The body of the request is a slice of the request buffer, followed by `7:call-id…`: `set rediscmdtimeout 0` set 7 ms, `set maxsessions 1000` set 10007, and an error message quoted the rest of the request.

### Commits

1. `redis: count and log writes refused by the server`: every reply to a pipelined `SET` or `DEL` is checked. An error reply is counted in `rediswriteerrors` (`redis_write_errors_total`) and logged at most once a second, and the connection is kept. An I/O error or a timeout drops it at once. Adds the test module `perl/NGCP/Rtpengine/FakeRedis.pm`.
2. `redis: do not sleep under the lock when reconnecting`: one attempt. After a wrong role, or an attempt that took `--redis-reconnect-slow` (100 ms) or more, the connection is skipped for `--redis-reconnect-pause` (1000 ms), because a failover takes longer than that. A refused connection is retried at once, as before. Adds the two options.
3. `cli: parse arguments without reading past the str`: the `set` handlers, `ksadd`, `ksrm` and `terminate` parse with `str_to_l`, `str_to_ul`, `str_to_ull`, `str_to_i` and a new `str_to_d` (for `maxcpu` and `maxload`, in the style of `str_to_x`, range errors by `errno == ERANGE`, no `<math.h>`), print the argument with `STR_FORMAT`, and `terminate` compares its keyword with `str_cmp()`. The messages and range checks stay.
4. `redis: do not write back a call that has been destroyed`: under `r->lock`, a call is written only while it is still in the call table, and the deletion decides "never stored" under the same lock.
5. `redis: reconnect without holding the lock`: the thread that reconnects sets `r->reconnecting`, releases `r->lock` for the attempt, and owns the connection until it clears the flag under the lock again. Other threads wait for it on a condition for up to `--redis-reconnect-slow`, then skip. After a release of the lock, a write checks again that the call is live, and a deletion leaves the key to a new call with the same ID. The CLI's reconnect refuses while an attempt runs, `set rediscmdtimeout` is now applied under the lock, and the notification thread takes the lock around its connection checks. A command on an established connection still holds the lock.
6. `redis: use FakeRedis.pm in the older Redis tests`: `redis`, `redis-json` and `redis-subscribe` use the module instead of comparing raw bytes, and the forked server that `redis-srtp-index` and `rollback-redis` each carried moves to the module as `redis_serve()`. Test-only, and last, so it can be reviewed on its own.

`--redis-delete-async` is compiled out (`#if 0`), so it is left alone.

### Testing

- New: `t/auto-daemon-tests-redis-errors.pl` (commits 1, 2, 3), `t/test-str.c` (commit 3), `t/auto-daemon-tests-redis-destroyed.pl` (commit 4) and `t/auto-daemon-tests-redis-reconnect.pl` (commit 5). Extended: `t/test-stats.c` (commit 1). Converted to `FakeRedis.pm` (commit 6): `t/auto-daemon-tests-redis.pl`, `-redis-json.pl`, `-redis-subscribe.pl`, `-redis-srtp-index.pl` and `t/auto-daemon-tests-rollback-redis.pl`.
- `redis-errors` starts two more daemons with non-default values of both options, and with a pause of 0. Mutants, each red: the pause value ignored, the slow threshold ignored, a pause of 0 treated as 1000 ms.
- `redis-errors` also sends, over the ng `cli` command with the body last in the request: `set maxsessions 1000`, `set maxcpu 99.5`, `set maxbw abc`, `terminate no-such-call`, `terminate a`, `set rediscmdtimeout 100x`, `set loglevel 7x` and `set loglevel core x` (refused, level unchanged, then `set loglevel 7`), then `set rediscmdtimeout 0`, which must reconnect both connections. `test-str` checks `str_to_d`: values, that the bytes after the `str` are not read, non-numbers, out of range (overflow and underflow) and the length limit.
- `redis-destroyed` has two cases, one for each half of commit 4. In the first, the call is deleted while the timer waits for the lock, and no write may follow. In the second, the deletion comes during the timer's write, and a `DEL` must follow it. Without the live check, the first case fails. If "never stored" is decided before the lock, the second fails (mutants: `not ok 16` and `not ok 21`).
- `redis-reconnect` holds a reconnect at its `AUTH` reply. Another offer returns after the 300 ms wait instead of the 2 s command timeout. An `AUTH` answered sooner wakes the waiting write at once. A call deleted during a reconnect, its own or one waiting for it, is not written. A deletion and a new call with the same ID that both wait for the attempt give a `SET` and no `DEL`. Mutants, each red: no wake-up broadcast (`not ok 33`), no live re-check after the release (`not ok 42`, `66`), no re-check for a waiting thread (`not ok 42`, `56`), no same-ID check (`not ok 56`).
- The converted tests keep their checks, names and counts (redis 125, redis-json 84, redis-subscribe 65, redis-srtp-index 132, rollback-redis 284), and fail on the same five `redis.c` mutants with the same failures as before (record expiry, `tos`, a failed restore deleting a foreign key, the SRTP index not restored, a checkpoint's pending state not written).
- Each commit's tests fail on its parent: commit 1: 11 failures; 2: 2, and its daemons with the new options do not start; 3: the six new ng `cli` checks (replies 10007, 99.6, the rest of the request quoted twice, "All calls terminated by operator.", "Success setting redis-cmd-timeout to 100"), then the timeout is set to 7 ms with no reconnect; 4: 2; 5: the first check (the held offer waits for the command timeout). Commit 6 changes only tests.
- Each commit builds without warnings and passes its own tests. Wall time: redis-errors 6.6 s, redis-destroyed 4.6 s, redis-reconnect 2.8 s.
- `make asan-check` on these commits plus the seven of the two follow-up PRs, on master `3865f896`, in Debian trixie and Ubuntu noble: exit 0, 14/14 unit tests, 49 daemon test plans, 9503 `ok`, 0 `not ok`, no ASan or LSan report. UBSan reports only at `call.c:5654` and `media_socket.c:666`, which the series does not touch.
- The patches apply with `git am` on master `3865f896`.

### Before / after (same lab and load)

"Before" is unpatched master. "After" is this branch.

| | Before | After |
|---|---|---|
| Refused writes seen by rtpengine | none logged | `rediswriteerrors` = server count in 6 of 6 failover runs |
| `--redis` alone, failover: failed ng requests | 188 | 0 (3 runs) |
| `--redis` alone, failover: longest time without an ng reply | 18.8 s | 0.21–0.22 s (3 runs) |
| Frozen master, 1 s command timeout: longest time without an ng reply | 35.5 s | 1.30–1.33 s (6 runs) |

**rtpengine cut from the master** (both directions dropped with `iptables` in rtpengine's network namespace for 22.6 s; the master keeps its replicas and the Sentinels, so there is no failover). Measured with the follow-up PR's Sentinel mode, repair and resync on. ng latency during the cut:

| | p50 | p90 | p99 | max | over 500 ms | Longest gap | Lost |
|---|---|---|---|---|---|---|---|
| Attempt under the lock (commits 1–4), 3 runs | 3.1–3.4 ms | 619–641 ms | 1609–1635 ms | 2009–2035 ms | 63–69 | 2.20–2.23 s | 0 |
| Attempt without the lock (commits 1–5), 5 runs | 3.1–7.6 ms | 103–108 ms | 733–933 ms | 1033–2037 ms | 17–21 | 1.30–1.33 s | 0 |

Each attempt fails after the 1 s connect timeout. The p90 is the 100 ms wait plus one command. The remaining tail is the request whose thread makes the attempt, and the first command at the start of the cut (the 1 s command timeout).

### Not fixed here

The loss stays, because rtpengine still writes to the old address. With `--redis` alone, 114–115 of 152 creates are acknowledged but not stored (3 runs). Before, those requests failed instead. Skipped writes are not written again either. An opt-in follow-up PR (branch `redis-sentinel`, which depends on this one) follows the master through Sentinel, and repairs refused and skipped writes.

### Open points

- The statistic can be split into its own commit, and the series can be split or reshaped if you prefer.
- The two options are not settable over the CLI, unlike `--redis-connect-timeout`; that can be added.

### The reconnect pause against upstream (plain `--redis`, `--redis-cmd-timeout=1000`)

Three scenarios, 452 ng requests per run: e1 `SENTINEL FAILOVER` with rtpengine pinned to the
demoted master (3 runs per configuration), e2 master frozen 30 s (3 runs), p1 rtpengine cut from the
master for 22.6 s (2 runs). "Acked nowhere" means ng replied, but the write was on no data node right
after the reply. "Attempts + skipped (log)" is what rtpengine logs, and is not comparable across
versions (it includes the call timer's once-a-second rewrite of never-stored SRTP calls).

#### Results

| Scenario, config | ng failed / 452 | acked nowhere | attempts + skipped (log) | creates lost at the end | DELs lost (zombies) | window (s) | failed attempts in window, per s | SYNs | ng p50 / p99 / max, window (ms) | > 500 ms in window |
|---|---|---|---|---|---|---|---|---|---|---|
| e1 upstream | 213-214 | 102 | 28-45 | 43-44 | 3 | 51.1-51.3 | 20, 0.39 | 29-46 | 10 008 / 10 011-10 014 / 10 023-10 026 | 404 |
| e1 PR 1 pause 1000 | 0 | 292 | 1378-1398 | 115 | 3 | 31.1 | 21, 0.67-0.68 | 24-25 | 1.2-1.4 / 11.5-13.7 / 19.7-27.7 | 0 |
| e1 PR 1 pause 0 | 0 | 289-292 | 1456-1466 | 114 | 3 | 31.1-31.3 | 1237-1247, **39.5-40.1** | **1456-1466** | 1.8-2.2 / 9.2-20 / 15.3-48.7 | 0 |
| e2 upstream | 286-287 | 46-47 | 33 | 20-21 | 1 | 34.0-34.2 | 33, 0.97 | 34 | 10 008 / 10 011-10 012 / 10 023-10 028 | 580-582 |
| e2 PR 1 pause 1000 | 0 | 304 | 1723-1724 | 115 | 2 | 33.7-34.0 | 17, 0.50 | 18 | 4.6-5.3 / 810-837 / 2030-2047 | 20 |
| e2 PR 1 pause 0 | 0-1 (no timeout; see below) | 303-304 | 663-751 | 115 | 2 | 33.6-33.9 | 33, 0.97-0.98 | 34-35 | 101-102 / 1066-1093 / 2047-2053 | 24-36 |
| p1 upstream | 298 | 14-15 | 35 | 0 | 1 | 35.6-35.7 | 35, 0.98 | 36 | 10 008 / 10 011 / 10 021-10 027 | 594-596 |
| p1 PR 1 pause 1000 | 0 | 237 (6 after the heal) | 966 | 0 | 3 | 22.6-22.7 | 11, 0.48-0.49 | 12 | 3.1-5.3 / 922-940 / 1036-2038 | 20-21 |
| p1 PR 1 pause 0 | 0 | 232 (2-4 after the heal) | 542-549 | 0 | 3 | 22.6-22.7 | 22, 0.97-0.98 | 23 | 101-102 / 1028 / 1492-2006 | 28-30 |

Whole-run ng p50 / p99 (ms):

- Upstream: 6 to 10 004 / 10 011 in every scenario.
- PR 1 with pause 1000: 1.3 to 2.9 / 12 to 825.
- PR 1 with pause 0: 1.8 to 2.3 / 10 to 17 in e1, and 101 / 835 to 1066 in e2 and p1.

Per-run figures are in `raw/rtpe-redis-pause/summary.md`.


- Upstream drops the update whose own attempt fails, as this branch does, and nothing queues or
  retries it. Its attempt holds `r->lock`, so the other requests wait behind it: 213–298 of 452
  requests per run got no answer within the load tool's 10 s timeout.
- This branch: no request reached the 10 s timeout in its 16 runs; the slowest single request took
  about 2 s (the one whose thread makes the attempt: PING timeout, then connect timeout). In two
  pause-0 e2 runs, that 2 s offer was overtaken by the load tool's own delete of the call, and the
  tool counted one failure: a load-tool race, not a timeout or a dropped write.
- Pause 1000 and pause 0 drop about the same number of writes. Pause 0 reconnects up to about 40
  times a second against a node that answers with the wrong role, and puts the ng median at about
  100 ms during a freeze or a cut. Its gain is a shorter tail after recovery (p1: 2–4 instead of 6
  writes dropped after the heal).
- For a single failed attempt against an otherwise healthy server (not measured), the pause can drop
  up to one pause of updates more than pause 0 if the failure was slow or found the wrong role, and
  nothing more after a quick refusal.
- A never-stored SRTP call is rewritten by the call timer once a second until it is stored, in both
  versions; updates of stored calls, deletions and plain RTP calls are not.

## 2. Sentinel, repair and resync (branch `redis-sentinel`)

### Summary

This series makes rtpengine's Redis persistence follow a Redis master failover, and write back what the failover lost (in the lab: every acknowledged create and deletion). Everything here is opt-in. Without the new options, behaviour is the same as with the fixes PR (see "What does not change").

Built on the fixes PR (branch `redis-failover-fixes`, six commits); the five commits of this part:

1. `MT#55283 redis: add --redis-require-master`
2. `MT#55283 redis: find the master through Redis Sentinel`
3. `MT#55283 redis: keyspace subscription follows the Sentinel master`
4. `MT#55283 redis: add --redis-repair-writes`
5. `MT#55283 redis: add --redis-resync-on-master-change`

The five commits on their own: https://github.com/kbinetruy/rtpengine/compare/redis-failover-fixes...redis-sentinel (21 files, +2898 −74; 1611 of the added lines are tests).

Each commit has its own tests, and can be reviewed, reshaped or dropped separately, apart from what it builds on. Sentinel builds on `--redis-require-master`, and the resync builds on the repair. The order lets you stop after commit 3: the repair and the resync are the last two.

#### Results

Lab: Redis 7.2.16 with one master and two replicas, three Sentinels (quorum 2), and `min-replicas-to-write 1`. The load is about 21 Redis writes a second. The same load and scripts were used in every column. "Today" is unpatched master. The other columns are the code of these branches.

| | Today | With the fixes PR | + Sentinel | + repair and resync |
|---|---|---|---|---|
| Clean failover: acknowledged creates never stored | 114–149 of 152–200 ¹ | 114–115 of 152 | 21–24 of 152 | **0 of 152** |
| Clean failover: deleted calls that come back | 3 | 3 | 3 | **0** |
| Clean failover: failed ng requests | 188 (`--redis` alone) | 0 | 0 | 0 |
| Master frozen 30 s: longest time without an ng reply | 34.3–35.5 s | 1.30–1.33 s | 1.31–1.33 s | 1.29–1.33 s |
| Master frozen 30 s: acknowledged creates never stored | 75–131 | 115 of 152 | 0 of 152 | 0 of 152 |
| Master frozen 30 s: deleted calls that come back | 0–1 | 2 | 2 | **0** |
| Long calls restored with stale state after a takeover | 4 of 4 | 4 of 4 | 0 of 4 | 0 of 4 |
| Takeover after a clean failover: restored / missing | misses every lost create | 41–42 / 114–115 | 132–135 / 21–24 | **156 / 0** |
| Refused writes seen by rtpengine | none logged | counter = server count | counter = server count | counter = server count |
| Runs per scenario | 3–5 | 3 | 3 | 7 |

¹ With `--redis` + `--redis-write` (role "any"). With `--redis` alone, 43–56 were lost, and the 188 requests after the demotion failed.

What is left with Sentinel alone: the new master, which has no replica yet, refuses writes with `NOREPLICAS` until Sentinel attaches one (from 1.44–1.84 s until 6.04–6.48 s after the failover was triggered). Those writes are counted, and the repair writes them again.

A resync of about 10 000 calls after a failover took 8.56–8.91 s (4 runs), about 5 s of it the new master refusing writes, then about 3420–4035 calls a second. During the resync, ng p99 was 5.9–14.5 ms and the maximum 9.3–34.0 ms, against 11.4 and 30.9 ms over the same window in a run without a failover, with no failed request and no gap. Details are under "Lab method and results".

The same scenarios on Valkey 8.1.10 gave the same results. Real partitions, `min-replicas-to-write 0` and the steady-state latency with and without Sentinel were also measured (see "Lab method and results").

### What does not change

- **No new option set: no new thread, connection or I/O.** On the Redis paths, the only new code is NULL-pointer tests of the Sentinel handle and, for the repair, a few option tests and one atomic read.
- **Checked by tests.**
  - The existing Redis daemon tests pass unchanged at the tip: redis 125, redis-json 84, redis-subscribe 65, redis-srtp-index 132 and rollback-redis 284 checks (moved to `FakeRedis.pm` by the fixes PR, commit 6), and the fixes PR's redis-errors (179) and redis-reconnect (73).
  - redis-errors scripts every command on its connection, so any extra command would fail it.
  - The `--redis-require-master` test checks that nothing of Sentinel appears in the log without the Sentinel options.
- **Floating IP (#799).** A VIP setup keeps working as with the fixes PR. `--redis-require-master` and `--redis-resync-on-master-change` can also be used without Sentinel, for example behind a VIP. That combination was not measured in the lab.

### How it works

#### `--redis-require-master` (commit 1)

- Without the option, a `--redis-write` connection accepts any role (as documented). After a failover it reconnects to the demoted node, and writes to a replica for good.
- With the option:
  - every connection that writes call data requires the master role at connect: `--redis-write`, and `--redis` when it is also the write connection (as it already does);
  - on those connections, a write refused with `READONLY` or `MASTERDOWN` drops the connection;
  - the next operation reconnects once and checks the role. If the node is still a replica, the attempt fails and is paced by the fixes PR: one attempt per `--redis-reconnect-pause`, made without the lock.
- `NOREPLICAS`, `LOADING` and other refusals keep the connection, because a reconnect would get the same answer.
- The drop happens only when the master role is required. With role "any", each write would reconnect to the same replica.

#### Sentinel discovery (commit 2)

**Configuration.**

- `--redis-sentinel=[PW@]IP:PORT[,IP:PORT...]` lists the Sentinels. `--redis-sentinel-master=NAME` names the monitored master.
- The keyword `sentinel` replaces `IP:PORT` in `--redis`, `--redis-write` or `--redis-subscribe`, for example `--redis-write=PW@sentinel/5`.
- Roles given as `IP:PORT` are unchanged, so a deployment can move one role at a time.

**The "redis sentinel" thread** does all Sentinel I/O, name resolution and role checks.

- It runs a round every `--redis-sentinel-interval` (default 1000 ms), and as soon as a connection finds that its server is not a master:
  - a write refused with `READONLY` or `MASTERDOWN`;
  - a reconnect that finds a replica;
  - an operation while no master is known.
- `NOREPLICAS` also wakes it, at most once per interval, because a master answers that for as long as it lacks replicas. A wake-up comes at least 250 ms after the last round.
- Each round asks every Sentinel, one after another, on a short-lived connection, in one pipeline:
  - `SENTINEL MASTER <name>`, for the **config epoch** only;
  - then `SENTINEL GET-MASTER-ADDR-BY-NAME <name>`, for the **address**.
- The `ip` field of `SENTINEL MASTER` is not used. The Sentinel that runs the failover raises the epoch at the promotion. It switches that field only after reconfiguring the other replicas, and until then it reports the old master at the new epoch. `GET-MASTER-ADDR-BY-NAME` names the new master from the start.
- In the lab, that window lasted 4.01–4.02 s on the leader, in 3 of 3 clean failovers. On the other two Sentinels, both sources switched together.
- The epoch is asked first, so a failover between the two replies can only pair an older epoch with the newer address.

**Choosing the master.**

- The answer with the **highest epoch** wins. At the same epoch, the master in use is kept, so there is no flapping while the Sentinels disagree.
- **Never go back to an older epoch.** A Sentinel that missed the failover still names the old master, and that node answers `role:master` until it is demoted. Its answer is ignored.
- **If every Sentinel that answers names an older epoch**, nothing is followed. An error is logged once, and the master in use is kept for as long as it answers as a master (its connection checks the role). This happens when the Sentinels were redeployed without their state, so their epochs start again. The error says that rtpengine must then be restarted.
- The candidate is checked with `ROLE`, using the data password of the first of `--redis-write`, `--redis`, `--redis-subscribe` that uses `sentinel`. A candidate that is not a master is not used, and the Sentinels are asked again 250 ms later (or after the interval, if it is shorter).
- The thread then publishes `{endpoint, epoch}` under a small mutex, and increments an atomic **generation number**.

**On the Redis side.** `redis_check_conn()` already runs under `r->lock` before every operation.

- It now reads the generation: one atomic read.
- When the generation has changed, it copies the address and the generation together under the small mutex, and drops the connection. It then reconnects with the existing `redis_connect()`, including the `INFO` role check, as any reconnect: without the lock (fixes PR).
- While another thread is reconnecting that connection, the move is not applied; the next check applies it.
- The next write therefore goes to the new master without first meeting a refusal.
- `redis_dup()` carries the Sentinel pointer, so the restore threads follow too. So does the CLI reconnect.

**Sentinel mode and the other options.**

- The roles that use Sentinel (`--redis-subscribe` included) require the master role, and drop the connection on `READONLY` or `MASTERDOWN`, as with `--redis-require-master`.
- These roles never re-resolve a name. `--redis-resolve-on-reconnect` applies to the Sentinels' host names, in the thread.
- **No option value is changed.** Without `--redis-cmd-timeout`, a warning is logged at startup. A frozen master then holds Redis operations until it answers again, not just until the switch. The same holds after the CLI's `set rediscmdtimeout 0`. The docs recommend 1000 ms. The thread's own Sentinel connections use 1 s when no command timeout is set, so discovery cannot hang.

**Startup.**

- One discovery round runs before the connections are made.
- If no Sentinel answers, rtpengine refuses to start, as it does with an unreachable data node. With `--no-redis-required`, it starts without a master, skips the restore, and connects at the first operation after a master is known.
- A candidate that answers as a replica is asked again once a second, 11 times in all, as the initial data connection is.
- These are refused at startup:
  - the Sentinel options without a `sentinel` endpoint, and a `sentinel` endpoint without `--redis-sentinel` or `--redis-sentinel-master`;
  - an empty Sentinel password;
  - an interval that is not positive.

#### Keyspace subscription follows the master (commit 3)

- **Today.** A subscriber subscribes again only when its connection drops. In the lab, a subscriber on the old master's address learned 0 of 24 calls written on the new master before the demotion. It learned 0 of 161 while the old master was frozen, and missed 27 more during the old master's full resync.
- **With `sentinel`.**
  - A one-shot, one-second libevent timer on the notification loop compares the generation with the one recorded when the subscription was made.
  - When it differs, the timer closes the subscription. `event_base_dispatch()` then has nothing left to wait for and returns, and `redis_notify_loop()` subscribes again on the new master.
  - The timer is re-armed only while nothing has changed, so it never keeps the loop running by itself.
- **Connect callback.** The subscription's context gets one. hiredis frees a context that fails to connect without calling the disconnect callback, which must still forget the context and stop the timer.
- **Locking.** The notification thread takes `r->lock` around its connection checks (fixes PR, commit 5).
- **With `--no-redis-required`**, it subscribes once a master is known.
- **A window of up to one second.** On a master change, the notification thread's own data connection follows at once, and the subscription within a second. A notification that arrives in between can read from the new master a record that the old master had not replicated. That read is stale, and is retried at the next notification of the call.
- **Not done.** Missed notifications are not replayed. Keyspace notifications are a contributed code path (#919), so this commit only makes the subscription follow the master.

#### `--redis-repair-writes` (commit 4)

- **Today.** A refused write is never made again. Unless the call is updated later, a call created during the refusal is missing, and a call deleted during it comes back at the next restore.
- **What is queued.** An own call whose `SET` or `DEL` was not stored:
  - an error reply (`NOREPLICAS`, `READONLY`, `MASTERDOWN`, `LOADING`…);
  - a connection lost before the reply;
  - a write skipped while the connection was down, paused, or being reconnected by another thread.

  A later stored write of the call clears it. So does deleting a call that was never written, so that no stray `DEL` follows. At most 100000 calls are queued (overflow is logged).
- **The "redis repair" thread** runs at normal scheduling. It is woken when calls are queued, and takes them one at a time.
  - A call still in the call table, and still own, is written again whole: its current state, not the refused one. The check is the one that the fixes PR (commit 4) adds to every write, under `r->lock`, made again after a wait for a reconnect (commit 5).
  - A call that is gone gets a `DEL` of its key, unless a new call with that ID exists by then. That is checked before the connection check and again after it, if the lock was released for a reconnect meanwhile.
  - So a newer write of the same call-id always wins, whether the lost write was a `SET` or a `DEL`.
  - Writes go through `redis_write_call()`, which `redis_update_onekey()` uses too.
- **Pacing.** Each repair write takes `r->lock` alone. Signalling writes count themselves around their `LOCK` while the option is on (`redis_lock_waiters_add()`, a no-op without it). After taking the lock, the repair waits on a condition (`cond_timedwait` on `r->lock`) while signalling writes are waiting, signalled each time one of them gets the lock, for 100 ms at most. So signalling goes first, and under continuous contention the repair still makes about ten writes a second: in a harness with a 5 ms fake server and four ng clients sending back to back, 5–10 repair writes a second, while signalling kept its rate (removing the give-way tripled the signalling p99; removing the 100 ms bound stopped the repair). There is no fixed rate: in the lab, a resync wrote about 3420–4035 calls a second once the master accepted writes.
- **Back-off.** After a write that fails again, the thread waits 1 s. A master that keeps refusing gets one attempt a second.
- **Reconnects.** The repair thread goes through the same connection check as signalling, so it makes reconnect attempts like any other thread, and holds no lock while it does. Behind an unreachable master, it is often the thread that makes the attempt once the pause ends; the requests that arrive meanwhile wait at most `--redis-reconnect-slow`.
- **Foreign calls** are never touched.
- **Why opt-in.** It adds a thread, and writes that answer no request.
- **Logs and statistic.** The thread logs the repairs at most once a second, and the number left at shutdown. Each stored repair write (`SET` or `DEL`) is counted in `redisrepairedwrites`.

#### `--redis-resync-on-master-change` (commit 5)

- **Why.** Replication is asynchronous. When a master is replaced, the writes it acknowledged but had not yet replicated are lost with it. In the lab with `min-replicas-to-write 0`, 9 operations per failover (4 creates, 4 re-offers, 1 delete) were acknowledged by the old master only, before rtpengine moved.
- **Trigger.** `redis_connect()` keeps `master_replid` from the `INFO` that it already sends.
  - When the write connection reconnects to a master with another ID, a resync is queued. This covers a Sentinel switch, a floating address that moved, or a fixed address that now leads elsewhere.
  - The first connection, or a reconnect to the same master, does nothing.
  - On Redis 7.2.16, a restarted master always got a new ID: from its RDB, without saving, and after SIGKILL. **So a Redis restart on the same address also rewrites every own call.** This is intended, since the restart may have lost writes too. In the lab, a master restarted empty on the same address got all 10 000 calls back in 2.32 s (1 run), with 0 missing and no failed request.
  - The option is refused at startup if the write connection does not require the master role, because on a replica `INFO` gives the master's ID.
- **What is queued** into the repair queue (this option enables it):
  - every own call stored at least once (calls that asked for `no-redis-update` stay out);
  - the `DEL`s of own calls of the last `--redis-resync-delete-window` (30 s by default), at most `--redis-resync-delete-max` (4096 by default), kept in a queue that is allocated only with the option. With the defaults, beyond about 135 deletions a second, the oldest are dropped, and their number is logged.
- **No `SCAN`.** Only keys that this instance wrote or deleted itself are touched. A replayed `DEL` assumes one writer per database, or call-ids that are not reused.
- **Writes** are retried once a second while the new master refuses them (`NOREPLICAS`). The resync and its duration are logged.
- **Statistics.** Each resync is counted in `redisresyncs`, and the calls and `DEL`s it queues in `redisresynccalls` and `redisresyncdeletes`. These two count what is queued; the writes actually stored are in `redisrepairedwrites`, so the numbers do not add up.

### Options and defaults

| Option | Default | Notes |
|---|---|---|
| `--redis-require-master` | off | Writing connections require the master role. `READONLY`/`MASTERDOWN` drops the connection. |
| `--redis-sentinel=[PW@]IP:PORT[,IP:PORT...]` | unset | The password can also come from `RTPENGINE_REDIS_SENTINEL_AUTH_PW`. A host name is resolved at startup, and before each query with `--redis-resolve-on-reconnect`. |
| `--redis-sentinel-master=NAME` | unset | Required with `--redis-sentinel`. |
| `--redis-sentinel-interval=MS` | 1000 | Must be positive. Refused without a `sentinel` endpoint. |
| `sentinel` in `--redis`, `--redis-write`, `--redis-subscribe` | — | That role follows the master, and behaves as with `--redis-require-master`. Set `--redis-cmd-timeout` too (a warning is logged without it). |
| `--redis-repair-writes` | off | One write at a time, giving way to signalling. 1 s back-off. At most 100000 queued. |
| `--redis-resync-on-master-change` | off | Enables the repair. Needs a write connection that requires the master role. |
| `--redis-resync-delete-window=MS` | 30000 | How long own `DEL`s are remembered for the resync. Must be positive. Refused without `--redis-resync-on-master-change`. |
| `--redis-resync-delete-max=INT` | 4096 | How many own `DEL`s are remembered at most. Must be positive. Refused without `--redis-resync-on-master-change`. |

All of them are in `docs/rtpengine.md` and `etc/rtpengine.conf`.

New statistics, in the ng `statistics` command, `rtpengine-ctl list totals` and Prometheus (as `rediswriteerrors`):

| Statistic | Prometheus | Counts |
|---|---|---|
| `redisrepairedwrites` | `redis_repaired_writes_total` | repair writes stored (`SET` or `DEL`) |
| `redisresyncs` | `redis_resyncs_total` | resyncs after a master change |
| `redisresynccalls` | `redis_resync_calls_total` | own calls queued by resyncs |
| `redisresyncdeletes` | `redis_resync_deletes_total` | own `DEL`s queued by resyncs |

### Configuration example

```
[rtpengine]
redis = password@sentinel/5
redis-sentinel = sentinelpw@10.0.0.11:26379,10.0.0.12:26379,10.0.0.13:26379
redis-sentinel-master = mymaster
redis-cmd-timeout = 1000
redis-repair-writes = true
redis-resync-on-master-change = true
```

This is the configuration of the lab runs, apart from the addresses and names.

### Failure modes

| Event | Behaviour | How we know |
|---|---|---|
| Clean failover (`SENTINEL FAILOVER`) | No ng request failed. Longest gap 0.20–0.22 s. The new master is used 1.26–1.66 s after the trigger. 0 lost with the repair. | Lab, 7 runs |
| Master frozen 30 s | Longest gap 1.29–1.33 s. 0 lost. | Lab, 7 runs |
| rtpengine cut from the master (packets dropped, no failover, 22.6 s) | The Sentinels keep naming the master, so rtpengine stays on it. Each attempt fails after the connect timeout, once per pause. ng p99 in the cut 733–933 ms, longest gap 1.30–1.33 s, no failure. The skipped writes are repaired after the cut. 0 lost. | Lab, 5 runs |
| Master isolated from every other node (34 s) | Sentinel switches, rtpengine follows, the resync rewrites the own calls. ng p99 in the cut 811–832 ms, longest gap 1.30–1.33 s. 0 lost. | Lab, 3 runs |
| `min-replicas-to-write 0` | The old master accepts writes until rtpengine moves (9 operations per failover run). The resync rewrites them. 0 lost. | Lab, 3 failover and 3 frozen runs |
| The Sentinel running the failover is the only one at the new epoch | Followed at once. | Test |
| A stale Sentinel (older epoch) next to up-to-date ones | Not followed. | Test (fake Sentinels) |
| Every answering Sentinel names an older epoch (stale, or a Sentinel set redeployed without its state) | Not followed. An error is logged once, and the master in use is kept while it is a master. A restart is needed after such a redeploy. | Test |
| Candidate that is a replica | Not used. Asked again within 250 ms. | Test |
| No Sentinel answers | Last master kept, logged once. At startup: refuse, unless `--no-redis-required`. | Test |
| Subscription closed by the server, master unchanged | Subscribed again. | Test |
| Subscription connect fails inside the event loop | Context forgotten, timer deleted, loop ends, subscribes again. | Code reading only (hiredis 0.14.1 and 1.2.0). Not reachable in the test harness. |
| New master refuses writes (`NOREPLICAS`, no replica yet) | Counted. Repaired once accepted, with `--redis-repair-writes`. | Lab, test |
| Writes acknowledged by the old master, not replicated | Rewritten, with `--redis-resync-on-master-change`. | Lab, test |
| Redis restarted on the same address | Every own call rewritten, with `--redis-resync-on-master-change`. | Test (new ID); lab, 1 run with 10 000 calls |
| A call deleted while a writer waits for `r->lock` or for a reconnect | Not written back (fixes PR, commits 4 and 5). | Tests, which fail without the checks |
| A new call takes the ID of a deleted call while the repair's `DEL` waits for a reconnect | No `DEL`; the new call is written. | Test |
| A never-written call deleted while its repair is pending | Repair mark cleared, no `DEL`. | Test |

Not handled or not measured:

- **Missed keyspace notifications.** Not replayed.
- **`--redis-delete-async`.** Its connection does not follow the master. The option is compiled out.
- **Epochs that go back.** rtpengine must be restarted.
- **Blackholed Sentinels.** Sentinels are queried one after another, so a Sentinel that drops packets costs one connect timeout per round, in the thread (inferred from the code).

### Locking and hot-path guarantees

- **No Sentinel I/O, name resolution or sleep under `r->lock`.** The thread never holds a lock across I/O.
- **Reconnects** (fixes PR) are made without `r->lock` by one thread, which owns the connection while `r->reconnecting` is set. Other threads wait on a condition for up to `--redis-reconnect-slow`, then skip. A Sentinel move found while the flag is set is applied at the next check, so the context is never freed under the connecting thread.
- **Re-checks after a release.** When the connection check released the lock, `redis_write_call()` checks again that the call is live and own, and the repair's `DEL` checks again that no new call has taken the ID.
- **The hot-path cost** is one atomic read per operation in Sentinel mode, and a NULL test otherwise. The small mutex is taken only to copy the address after a change.
- **The repair thread.**
  - One write per `r->lock` acquisition.
  - While signalling writes wait for `r->lock`, the next repair write waits on a condition until none is left waiting, for 100 ms at most.
  - Normal priority, so that a preempted repair thread never sits on the lock that signalling needs.
  - Its reconnect attempts follow the same rule as any other thread's.
- **Notification thread.** It takes `r->lock` around its connection checks.
- **Lock order** is always `r->lock`, then the Sentinel mutex, then the resync mutex, then the repair mutex, then the call table's read lock.
- **What tests cannot force.** The order of two threads waiting on one mutex cannot be forced from the harness. The reconnect cases are tested by holding the fake server's `AUTH` reply.

### Test coverage

The new tests follow `docs/tests.md` (a fake Redis in Perl, byte sequences). Timing checks are only generous "within N s" or "nothing for N s" checks, never latency bounds. The fake Redis and fake Sentinel helpers are in one module, `perl/NGCP/Rtpengine/FakeRedis.pm`, which the fixes PR adds and this PR extends. The fake Sentinels run in a child process, so that they answer while the test scripts the data nodes.

| File | Wall time | What it proves |
|---|---|---|
| `t/auto-daemon-tests-redis-require-master.pl` | 2.0 s | `NOREPLICAS` keeps the connection. `READONLY` drops it, the reconnect finds a replica and gives up after one `INFO`, there is no attempt within the pause, then the master is found. `MASTERDOWN` drops too. Nothing of Sentinel without the Sentinel options. |
| `t/auto-daemon-tests-redis-sentinel.pl` | 11.6 s | Startup discovery with both passwords, and `ROLE` checked before use. A failover seen first from the Sentinel running it alone: moved once, at once, nothing written to the old node. A stale Sentinel is not followed. Every Sentinel older (stale alone, then a redeployed set): not followed, error logged once, master kept. A replica candidate is refused, then asked again before the next poll. No Sentinel answering. A second failover gives no flip back. Uses `--redis-sentinel-interval=500`. |
| `t/auto-daemon-tests-redis-sentinel-start.pl` | 3.7 s | Start with no Sentinel (`--no-redis-required`) and a 60 s interval, so every query is prompted. Wake-ups by a skipped write, by `NOREPLICAS` (at most once per interval), by `READONLY`, and by a reconnect that finds a replica. The password comes from the environment. |
| `t/auto-daemon-tests-redis-sentinel-subscribe.pl` | 3.7 s | A subscription closed by the server is made again. The subscription and its connection follow the master there and back. |
| `t/auto-daemon-tests-redis-repair.pl` | 15.4 s | A refused write is rewritten once. A signalling write that waits for the lock goes before the next repair write. One attempt a second while refused. A newer stored write cancels the repair. A refused `DEL` is retried. A reused ID gets a `SET`, not a `DEL`. Skipped writes are repaired. A never-stored call deleted while the repair reconnects gets neither `SET` nor `DEL`. A new call that takes the ID while the repair's `DEL` waits for a reconnect: no `DEL`, then the new call is repaired. A foreign call is not repaired. A never-written call deleted while its write is pending gets no `DEL`. `redisrepairedwrites` counts the stored repair writes only. |
| `t/auto-daemon-tests-redis-resync.pl` | 8.8 s | Refused at startup without a master-role write connection. Same replication ID: nothing. New ID: every stored own call and the recent `DEL`, in any order. A `no-redis-update` call and foreign calls (after CLI `standby`) are left out. A second daemon with a 2 s window and at most 2 `DEL`s checks the window, both where it is applied (when a `DEL` is recorded, and at the resync), and the size, with its log line. The three counters equal the log after each resync. |

The new files of this PR take about 45 s in all (Ubuntu noble, non-ASan). The longest is `redis-repair`, 15.4 s.

<details>
<summary>Red before, green after, and mutation testing</summary>

- **Red on the parent commit:** the tests of commits 1, 2, 4 and 5 fail on their parent, because rtpengine refuses the unknown option. The subscriber test fails on the core Sentinel commit (`not ok 37`): the subscription stays on the old node.
- **Mutants**, each red:
  - the repair's `DEL` without its re-check after a release (redis-repair);
  - the repair's give-way removed (the signalling-first case of redis-repair; without the give-way the order depends on the scheduler, so the case is red in most runs, not all);
  - the write path without its live re-check after a release (redis-repair and the fixes PR's redis-reconnect);
  - the resync window ignored at the resync (`not ok 185`) or when a `DEL` is recorded (`not ok 127`), and the size ignored (`not ok 172`, `173`);
  - each of the four counters removed, and a no-op repair counted as a stored write.

</details>

**Not tested deterministically, and why:**

- **The locks.** The order of two threads waiting on one mutex cannot be forced from the harness.
- **The same-epoch rule** (keep the master in use).
- **A Sentinel move published during a reconnect.** It needs the fake Sentinels driven in step with a held data node. The guard is one precondition, left to code reading.
- **A subscription connect that fails inside the event loop.** The harness maps fake addresses to Unix sockets, so a refused connect fails at once inside `redisAsyncConnect()`.
- **The subscriber with no master at startup.** There is no subscriber in the `--no-redis-required` test.
- **The 100000-entry cap.** It would need that many writes. The `DEL` cap is tested with a size of 2.

**Build and `make asan-check`.**

- Each of the 11 commits builds without warnings and passes its own tests.
- `make asan-check` on the tip of the TLS follow-up PR (these 11 commits and its two), on master `3865f896`: exit 0 in Debian trixie and Ubuntu noble containers. 14/14 unit tests, 49 daemon test plans, 9503 `ok`, 0 `not ok`, no ASan or LSan report. UBSan reports only at `call.c:5654` and `media_socket.c:666`, which this series does not touch.
- The 11 patches apply with `git am` on master `3865f896`.

### Lab method and results

All the numbers below were measured on the code of these branches, apart from the "Today" figures (unpatched master) and the partition row marked "attempt under the lock (commits 1–4)", which is the explicit before/after of the fixes PR's commit 5 (reconnect without the lock).

<details>
<summary>Setup and method</summary>

- **Host.** One 4 vCPU VPS, Docker. Every run waited for a quiet host before starting.
- **Redis 7.2.16.** Three data nodes and three Sentinels, quorum 2, `down-after-milliseconds 2000`, `failover-timeout 4000`, `min-replicas-to-write 1`, `min-replicas-max-lag 10`, `parallel-syncs 1`.
- **rtpengine.** Userspace, built on Debian bookworm with hiredis 0.14.1. The CI runs above use hiredis 1.2.0. The Sentinel runs pass `--redis-cmd-timeout=1000`.
- **Load.** Over ng, at 5 ticks a second, for 40 s. Each tick creates one SDES call (offer and answer) and re-offers one of 4 long-lived calls with fresh keys. Every fourth tick deletes the call created 10 ticks earlier. That is about 21 Redis writes a second.
  - Each SDP carries a marker in its session name, so the stored value shows which operation it reflects.
  - After each reply, the tool reads the key on all three data nodes.
- **Observer.** Every 0.2 s it records Sentinel's master, each node's role, the client connections, `INFO errorstats` and `DBSIZE`.
- **Trigger** at t = 10 s: `SENTINEL FAILOVER`, `docker pause` of the master for 30 s, or `iptables` `DROP` rules (below).
- **After the load**, rtpengine is killed and a takeover rtpengine starts from the master that Sentinel names. Its call list is the takeover result.
- **Definitions.**
  - "Lost" means acknowledged over ng, but absent from the final master.
  - "Stale" means a long call whose stored state is older than its last acknowledged update.
  - "Deleted back" means a call deleted over ng whose key is still on the final master.
  - "Server count" means the `READONLY` and `NOREPLICAS` replies counted by the servers. A frozen node cannot be read at the end, and it gives timeouts, not error replies.
  - "Gap" means the longest time without a successful ng reply.
- **Today.** 20 runs on unpatched master `12ecdbc2`. Between it and `d590a62f`, upstream changed only `call.c`.

</details>

<details>
<summary>The fixes PR alone (3 runs per experiment)</summary>

| Experiment | Lost | Deleted back | Stale | Takeover restored / missing | ng failed | Longest gap | `rediswriteerrors` = server count |
|---|---|---|---|---|---|---|---|
| failover, `--redis` alone | 114, 114, 115 of 152 | 3, 3, 3 | 4/4 | 42/114, 42/114, 41/115 | 0 | 0.21–0.22 s | 217 = 217 (`NOREPLICAS`), each run |
| failover, `--redis` + `--redis-write` | 114, 114, 114 of 152 | 3, 3, 3 | 4/4 | 42 / 114 | 0 | 0.21–0.22 s | 617 = 400 `READONLY` + 217 `NOREPLICAS`; 617 = 404 + 213; 612 = 400 + 212 |
| frozen master, `--redis` + `--redis-write` | 115, 115, 115 of 152 | 2, 2, 2 | 4/4 | 41 / 115 | 1, 0, 0 (load-tool artefact) | 1.30–1.32 s | 0 (timeouts only) |
| frozen master, `--redis` alone | 115, 115, 115 of 152 | 2, 2, 2 | 4/4 | 41 / 115 | 1, 0, 1 (artefact) | 1.32–1.33 s | 115 (`READONLY` from the old master once unfrozen and demoted) |

The frozen-master runs pass `--redis-cmd-timeout=1000`. The artefact: an offer delayed by the frozen master, whose call the load tool deleted before answering. A server restarted on the same address (1 run) was used again 0.17 s after it came back, and 0 of the 40 creates made after its return were missing.

</details>

<details>
<summary>Sentinel alone (3 runs per experiment, 1 subscriber run)</summary>

| Experiment | Lost | Deleted back | Stale | Takeover restored / missing | ng failed | Longest gap | `rediswriteerrors` = server count |
|---|---|---|---|---|---|---|---|
| failover | 24, 21, 23 of 152 | 3, 3, 3 | 0/4 | 132/24, 135/21, 133/23 | 0 | 0.21 s | 111 = 111, 97 = 97, 107 = 107 (`NOREPLICAS`) |
| frozen master | 0, 0, 0 of 152 | 2, 2, 2 | 0/4 | 156 / 0 | 1, 1, 0 (artefact) | 1.31–1.33 s | 0 (timeouts only) |
| frozen master, with a subscriber | 0 of 152 | 2 | 0/4 | 156 / 0 | 0 | 1.31 s | 0 (timeouts only) |

- rtpengine uses the new master 1.26–1.65 s after a failover is triggered, and 4.26–4.29 s into a freeze.
- **Publish delay** (6 failover runs, 3 with Sentinel alone and 3 with the repair, from the Sentinel logs): rtpengine publishes the new master 0.10–0.22 s after the leader's `+promoted-slave`, and 0.01–0.15 s after the first `+switch-master` of another Sentinel. The leader logs its own `+switch-master` about 4 s after the promotion.
- The creates lost in a failover were refused with `NOREPLICAS` by the new master until its replica was attached: from 1.44–1.84 s until 6.04–6.48 s after the trigger. The deleted calls that come back are `DEL`s refused in the same window. In a freeze, they are `DEL`s skipped while the master was frozen. Only the repair fixes both.
- **Subscriber** (a second rtpengine subscribed through Sentinel to the first one's DB, master frozen):
  - Sentinel switched at 3.45 s, the subscriber subscribed again on the new master at 3.88 s, and the writer moved at 4.27 s.
  - Acknowledged creates learned: 49 of 49 before the freeze, 16 of 21 between the freeze and the writer's move, and 130 of 130 after it.
  - The subscriber was on the new master before the writer moved, so none was missed after the move.

**Compared with `--redis-resolve-on-reconnect`.** Here a host name that points at the master is rewritten to the new master right after Sentinel reports it. It lost 42 of 152 creates on unpatched master (1 run), and 41–42 with the fixes PR (3 runs), against 21–24 in Sentinel mode and 0 with the repair. The connection stays on the old master until Sentinel closes it at the demotion, because the name is only re-resolved at a reconnect.

</details>

<details>
<summary>Sentinel + repair + resync (7 failover runs, 7 frozen runs, 2 runs with <code>parallel-syncs 2</code>)</summary>

| Experiment | Lost | Deleted back | Stale | Takeover restored / missing | ng failed | Longest gap | `rediswriteerrors` = server count | Resync |
|---|---|---|---|---|---|---|---|---|
| failover ×7 | 0 of 152 | 0 | 0/4 | 156 / 0 | 0 | 0.20–0.22 s | 104–122, equal in each run | 48–49 calls + 12 `DEL`s, done in 5.0–6.0 s |
| frozen master ×7 | 0 of 152 | 0 | 0/4 | 156 / 0 | 0–1 (artefact) | 1.29–1.33 s | 0 (timeouts only) | 41–42 calls + 12 `DEL`s, done in 17–22 ms |
| failover, `parallel-syncs 2` ×2 | 0 of 152 | 0 | 0/4 | 156 / 0 | 0 | 0.23–0.24 s | 118, 135, equal in each run | 48 calls + 12 `DEL`s, done in 6.0 s |

- rtpengine uses the new master 1.26–1.66 s after a failover is triggered, and 4.26–5.06 s into a freeze (in the 5.06 s run, the Sentinels named the new master only at 4.40 s).
- After a failover, the resync takes 5–6 s because the new master refuses writes until its replica is attached.
- The counter includes the repair's own refused attempts, one a second.
- With Sentinel `parallel-syncs 2` instead of 1, the new master refused writes until 6.44 s and 7.24 s after the trigger. There was no gain in this group, which has one replica to reconfigure at the switch.
- In every run with the options, `redisrepairedwrites` and the three resync counters equal the sums of the matching log lines, and `rtpengine-ctl list totals` shows the same values as the ng `statistics` command.

</details>

<details>
<summary>Real partitions (Sentinel + repair + resync)</summary>

A helper container with `iptables` joins the target's network namespace (`docker run --net container:<target> --cap-add NET_ADMIN`) and inserts `DROP` rules. Packets are dropped silently: no RST, no ICMP, no ACK (unlike `docker pause`, where the kernel of the frozen container still accepts connections).

- **rtpengine cut from the master:** in rtpengine's namespace, `INPUT` from and `OUTPUT` to the master dropped at t = 10 s, removed 22.6 s later. The master keeps its replicas and the Sentinels: no failover, and the Sentinels keep naming it.
- **Master isolated:** in the master's namespace, everything but loopback dropped at t = 10 s, removed 34.0–34.5 s later.

| ng latency during the cut | Runs | p50 | p90 | p99 | max | over 500 ms | Longest gap | Lost / deleted back / takeover |
|---|---|---|---|---|---|---|---|---|
| rtpengine cut from the master | 5 | 3.1–7.6 ms | 103–108 ms | 733–933 ms | 1033–2037 ms | 17–21 | 1.30–1.33 s | 0 / 0 / 156 |
| same, attempt under the lock (commits 1–4 of the fixes PR) | 3 | 3.1–3.4 ms | 619–641 ms | 1609–1635 ms | 2009–2035 ms | 63–69 | 2.20–2.23 s | 0 / 0 / 156 |
| Master isolated | 3 | 1.5–1.7 ms | 6.2–16.3 ms | 811–832 ms | 2010–2032 ms | 20–21 | 1.30–1.33 s | 0 / 0 / 156 |

- **Cut from the master.** Each reconnect attempt ends with "Connection timed out" after the 1 s connect timeout, then the 1 s pause. The other requests wait at most `--redis-reconnect-slow` (p90 103–108 ms). What remains is the request whose own thread makes the attempt, once per pause, and the first command on the established connection at the start of the cut (its 1 s command timeout, under the lock). The skipped writes are marked, and the repair writes them once the cut is removed (3–86 per run). No resync: the master did not change.
- **Master isolated.** The Sentinels switch 3.66–4.21 s after the cut, and rtpengine follows at 4.15–4.85 s. The resync rewrites the own calls and the recent `DEL`s on the new master (41 calls + 13 `DEL`s, in 17–23 ms). Nothing is lost.

</details>

<details>
<summary><code>min-replicas-to-write 0</code> (Sentinel + repair + resync, 3 failover runs, 3 frozen runs)</summary>

`CONFIG SET min-replicas-to-write 0` on the three data nodes for the run, set back to 1 after it.

- **Results.** 0 lost, 0 deleted back, 0/4 stale, takeover 156 / 0 in all 6 runs. No `NOREPLICAS` (`rediswriteerrors` 0).
- **What the resync is for here.** The new master accepts writes at once, but so does the old one. Between the promotion and rtpengine's move, 9 operations per failover run (4 creates, 4 re-offers, 1 delete) were acknowledged by the old master only. When the old master becomes a replica, those writes are gone. The resync rewrites them on the new master (50 calls + 13 `DEL`s, in 14–17 ms). With `min-replicas-to-write 1`, the same window gives 52–56 operations refused with `NOREPLICAS` per failover run, which the repair writes again.
- **Follow time.** 2.05–2.06 s after the trigger, against 1.26–1.66 s with `min-replicas-to-write 1`. Inferred from the code: without a `NOREPLICAS` reply, nothing wakes the Sentinel thread before its next poll (`--redis-sentinel-interval`).
- **Frozen master.** The frozen node acknowledges nothing, so nothing lands on it only. Resync 41 calls + 12 `DEL`s in 15–19 ms.

</details>

<details>
<summary>Valkey 8.1.10 (Sentinel + repair + resync: 3 failover runs, 3 frozen runs; Sentinel alone: 1 failover run)</summary>

A separate group with the same settings: `valkey/valkey:8` (Valkey 8.1.10), three data nodes and three `valkey-sentinel`.

| Experiment | Lost | Deleted back | Stale | Takeover restored / missing | ng failed | Longest gap | `rediswriteerrors` = server count | Resync |
|---|---|---|---|---|---|---|---|---|
| failover ×3 | 0 of 152 | 0 | 0/4 | 156 / 0 | 0 | 0.21 s | 95, 108, 130, equal in each run | 48–49 calls + 12 `DEL`s, done in 4.0–6.0 s |
| frozen master ×3 | 0 of 152 | 0 | 0/4 | 156 / 0 | 1 each (artefact) | 1.31–1.33 s | 0 (frozen node) | 41 calls + 12 `DEL`s, done in 17–26 ms |
| failover, Sentinel alone | 22 of 152 | 3 | 0/4 | 134 / 22 | 0 | 0.22 s | 106 = 106 | — |

Same `INFO` fields, same Sentinel replies and epochs, same `NOREPLICAS` window, same results as Redis 7.2. A 10 000-call resync took 7.8 s (10 048 calls + 12 `DEL`s), with a window p50 / p99 / max of 1.9 / 8.5 / 17.6 ms and 0 failed requests.

</details>

<details>
<summary>Steady-state ng latency with and without Sentinel (no trigger, 60 s, 2 runs each)</summary>

Same load (1281 ng requests per run), no failover, no repair or resync. "Sentinel" is `--redis=PW@sentinel/5` with the Sentinel options; "plain" is `--redis=PW@<master>:6379/5`. Both with `--redis-cmd-timeout=1000`.

| Code, mode | Run 1 p50 / p99 / max | Run 2 p50 / p99 / max | Failed | Longest gap |
|---|---|---|---|---|
| this branch, Sentinel | 1.6 / 9.6 / 26.0 ms | 1.6 / 8.3 / 29.6 ms | 0 | 0.21 s |
| this branch, plain | 1.7 / 9.7 / 38.1 ms | 1.4 / 9.7 / 28.2 ms | 0 | 0.22–0.23 s |
| fixes PR, plain | 1.5 / 11.6 / 42.9 ms | 1.5 / 9.5 / 18.1 ms | 0 | 0.21–0.22 s |

No difference above run-to-run noise: the medians are equal within 0.3 ms, the p99 ranges overlap, and the maxima are in this lab's usual tail. The longest gap is the load's own 0.2 s tick.

</details>

<details>
<summary>Resync of 10 000 calls under load (after a failover: 4 runs, and 1 without a failover; after a restart: 1 run)</summary>

10 000 long SDES calls were created over ng. Then the usual load ran for 40 s, with `SENTINEL FAILOVER` at 10 s, or no trigger. The window runs from "resync queued" to "resync done", at the same offsets in the runs without a failover.

| Run | Resync | ng p50 / p99 / max in the window | Longest gap in the window | ng p50 / p99 / max, whole load | Failed |
|---|---|---|---|---|---|
| failover 1 | 10 048 calls + 12 `DEL`s in 8.56 s | 1.4 / 8.7 / 34.0 ms | 0.205 s | 1.5 / 10.8 / 129.4 ms | 0 |
| failover 2 | 10 049 + 12 in 8.59 s | 1.6 / 5.9 / 9.3 ms | 0.204 s | 1.6 / 8.7 / 19.0 ms | 0 |
| failover 3 | 10 048 + 12 in 8.91 s | 1.7 / 14.5 / 22.9 ms | 0.212 s | — | 0 |
| failover 4 | 10 050 + 12 in 8.62 s | 1.5 / 7.7 / 11.6 ms | 0.214 s | — | 0 |
| no failover | — | 1.3 / 11.4 / 30.9 ms | 0.224 s | 1.4 / 8.5 / 30.9 ms | 0 |

- **Duration.** About 5 s of each resync is the new master's `NOREPLICAS` window, with one attempt a second. The rest is the flush, at 3420–4035 writes a second. Failover 3 ran right after a CI job had loaded the host (5-minute load 2.2); its flush was the slowest (3420–3606 writes a second) and its p99 the highest.
- **Latency.** Over the window, the medians (1.4–1.7 ms) are close to that of the run without a failover (1.3 ms); p99 is 5.9–14.5 ms against 11.4 ms, and the maxima 9.3–34.0 ms against 30.9 ms. The 129.4 ms request of failover 1 came 2.9 s before the failover, with the repair queue empty. No failure and no gap.
- **The longest gap** is the load's own 0.2 s tick.
- **Not measured:** the replication traffic of such a resync.

**After a plain master restart** (1 run). A Redis 7.2 server without persistence, with `--redis` alone, the repair and the resync, 10 000 own calls and the usual load. `SHUTDOWN NOSAVE` at 10 s, and the server is back empty on the same address 2 s later, with a new replication ID.

| Resync | ng p50 / p99 / max in the window | Longest gap | ng p50 / p99 / max, whole load | Failed | Afterwards |
|---|---|---|---|---|---|
| 10 043 calls + 13 `DEL`s in 2.32 s | 1.8 / 7.5 / 7.5 ms (47 requests) | 0.20 s | 1.4 / 7.5 / 25.4 ms | 0 | all 10 156 calls in Redis, 0 deleted back |

The resync starts at the reconnect, 2.25 s after the shutdown, and writes about 4300 calls a second.

</details>

### Known limits

- **One writer per database.** A repaired or replayed `DEL` assumes it, or call-ids that are not reused.
- **ACL users** (`AUTH user pass`) are not supported for Sentinels, as today for the data nodes.
- **The repair has no rate option.** It writes one call at a time, giving way to signalling.
- **A master cut from rtpengine only.** The Sentinels keep naming it, so rtpengine stays on it and pays one failed attempt per pause until the path comes back (see the fixes PR).
- **The lab is one host.** `docker pause` and `iptables` rules stand in for a dead host and a partition.
- **Optional TLS** is the third part (below).

### Open questions

1. **Option shape.** Is the `sentinel` keyword in the existing endpoint options right? Or would you prefer per-role flags, or one switch for all roles?
2. **Repair.** Should `--redis-repair-writes` stay opt-in, and should it be one option with the resync or two? It writes back to back, giving way to waiting signalling writes (at least about ten writes a second). Would you rather have a rate limit?
3. **Resync after a restart.** A Redis restart on the same address rewrites every own call (10 000 calls in about 2.3 s in the lab). Is that acceptable, or should the resync run only when the address or the Sentinel master changed?
4. **Defaults and tests.** Sentinel mode warns without `--redis-cmd-timeout`: should it refuse to start? Should `--redis` alone drop the connection on `READONLY` without `--redis-require-master`? And are `perl/NGCP/Rtpengine/FakeRedis.pm`, a forked fake Sentinel and about 45 s of new daemon tests acceptable in your CI?

The series can be split or reshaped if you prefer: `--redis-require-master` alone, or Sentinel without the repair, both stand on their own, and the option names are not fixed.

## 3. TLS (branch `redis-tls`)

### Summary

Optional TLS for every Redis and Redis Sentinel connection. A server that only accepts TLS (a managed service, or `tls-port` with `port 0`) cannot be used today; asked on the mailing list in 2022: https://groups.google.com/g/rtpengine/c/gO2hWMKxCNA/m/DNH37Z54BQAJ

Built on the Sentinel part (branch `redis-sentinel`), which is built on the fixes; the two commits of this part:

1. `MT#55283 redis: optional TLS for the Redis connections` (12 files, +550 −5)
2. `MT#55283 redis: TLS for the Redis Sentinel connections` (4 files, +87 −8)

The two commits on their own: https://github.com/kbinetruy/rtpengine/compare/redis-sentinel...redis-tls

Nothing changes without the new options. Builds without `hiredis_ssl` compile the TLS code out.

### Why its own name check

Released hiredis (1.2.0 up to 1.4.1) checks only the chain; `verify_name` exists only on its master branch. With `redisCreateSSLContext`, any certificate from the trusted CA would pass for any Redis address. So the name is checked here, with OpenSSL, in the subjectAltName only (the subject's CN is not used):

- for an IP address, `X509_VERIFY_PARAM_set1_ip_asc`: the certificate needs an IP entry, and no SNI is sent (RFC 6066 does not allow addresses in SNI);
- for a host name, `X509_VERIFY_PARAM_set1_host` with `X509_CHECK_FLAG_NO_PARTIAL_WILDCARDS`, plus SNI. A wildcard is accepted only as a whole first label (`*.example.com`).

### Options and defaults

The vocabulary is the MQTT one. As for `--mqtt-cafile`, giving a CA turns TLS on: there is no separate switch.

| Option | Default | Notes |
|---|---|---|
| `--redis-cafile=FILE`, `--redis-capath=PATH` | unset | Either one turns on TLS for every Redis connection. Only these trust anchors are used. |
| `--redis-certfile=FILE`, `--redis-keyfile=FILE` | unset | Client certificate, for `tls-auth-clients` on the server. Both or neither. |
| `--redis-tls-server-name=NAME` | unset | The name to send (SNI) and to expect from every data node, instead of each node's host or address. For example, the name of a managed service. |

- TLS 1.2 is the minimum version. Files are PEM, and password-protected keys are not supported (as for MQTT).
- One `SSL_CTX` is made at startup; an unreadable file stops the startup.
- These are refused at startup:
  - a client certificate or a server name without a CA;
  - a certificate without a key, or a key without a certificate;

On a build without `hiredis_ssl`, the options do not exist: `--redis-cafile` is an unknown option, as `--mqtt-host` is on a build without MQTT, `--help` does not list them, and a `redis-cafile` key in the configuration file is ignored.

### What uses TLS, and which name is checked

| Connection | Name expected in the certificate |
|---|---|
| `--redis`, `--redis-write`, `--redis-subscribe`, the restore copies, every reconnect | the host as configured, or the address that Sentinel gave; or `--redis-tls-server-name` |
| Async: keyspace subscription, async deletions | same |
| Sentinel queries (commit 2) | the host as given in `--redis-sentinel` (`--redis-tls-server-name` does not apply) |
| The Sentinel thread's `ROLE` check of a master | the address that the Sentinels gave, or `--redis-tls-server-name` |

The Sentinels share the TLS options with the data nodes, as Redis configures both with the same `tls-*` directives.

A refused certificate or a handshake that times out is a failed connect, logged with the reason (`TLS: certificate of X not accepted: <OpenSSL reason>`, `TLS: handshake with X timed out`), on the subscription too.

### Locking and timeouts

- **Sync connections.** The handshake runs in `redis_connect()`, after `redis_set_timeout()`. So it is bounded by `--redis-cmd-timeout`, like any command. With no command timeout, the connect timeout bounds it, and is removed after the handshake. On a reconnect, it runs in the thread that makes the attempt, without `r->lock` (fixes PR); other threads wait for it at most `--redis-reconnect-slow`. There is no new sleep.
- **Async connections.** TLS starts right after `redisAsyncConnect()`; the handshake runs in the event loop, not under a lock.
- **Sentinel connections.** The handshake runs in the Sentinel thread, bounded by its own command timeout, never under `r->lock`.
- **A refused certificate** is a failed attempt, paced like any other (`--redis-reconnect-pause`, `--redis-reconnect-slow`).

### Build

- `utils/gen-common-flags` adds `-DHAVE_HIREDIS_SSL` when `pkg-config --exists hiredis_ssl`, the MQTT pattern. On Debian trixie and Ubuntu noble, `libhiredis-dev` already ships it: no new build dependency for the daemon.
- On Debian bookworm (hiredis 0.14.1), the build has no warning, and the TLS options are not available (unknown option). The test suite was not run on bookworm.
- `debian/control`: `libio-socket-ssl-perl` is added to Build-Depends, for the test's fake TLS servers.

### Tests

- **New: `t/auto-daemon-tests-redis-tls.pl`**, 126 checks (105 after commit 1), about 7 s. Fake Redis and fake Sentinels run over `IO::Socket::SSL` (helpers in `FakeRedis.pm`), with certificates made by the `openssl` tool at test time. It covers each row of the table above, every refusal (another CA, another address or name, partial wildcard, CN only), SNI, a client certificate required by the server, a handshake with no answer (given up after the 1 s connect timeout), and every startup refusal.
- **Red on the base.** The test fails on the parent (unknown option).
- **The other 14 Redis daemon test files** pass with the changed `FakeRedis.pm`.
- **Mutants**, each made red by the tests: no IP check, no host-name check, no `SSL_VERIFY_PEER`, the handshake timeout missing or left in place, no TLS on async or Sentinel connections, `--redis-tls-server-name` ignored or applied to the Sentinels, a CN fallback, partial wildcards, and the two missing log reasons. One survives, and is equivalent here: using the address instead of a configured host name (the tests use IP addresses; host names are covered through `--redis-tls-server-name`).
- **`make asan-check`** on the tip passed in Debian trixie and Ubuntu noble: exit 0, 14/14 unit tests, 49 daemon test plans, 9503 `ok`, 0 `not ok`, no ASan or LSan report.
- **The patches** apply with `git am` on master `3865f896` after the two earlier PRs.

<details>
<summary>Lab: everything over TLS, Sentinel + repair + resync (3 failover runs, 3 frozen-master runs, 1 plain-TCP run)</summary>

**Setup.**

- Redis 7.2 with `port 0`, `tls-port`, `tls-auth-clients yes`, `tls-replication yes`: one master, two replicas, three Sentinels over TLS.
- One EC P-256 CA, and a certificate per node with its IPv4 address as subjectAltName. rtpengine has a client certificate.
- rtpengine (Debian trixie, hiredis 1.2.0) ran with:

```
--redis=PW@sentinel/5 --redis-sentinel=SPW@192.0.2.11:26379,192.0.2.12:26379,192.0.2.13:26379
--redis-sentinel-master=mymaster --redis-cmd-timeout=1000
--redis-cafile=/certs/ca.pem --redis-certfile=/certs/rtpengine.pem --redis-keyfile=/certs/rtpengine.key
--redis-repair-writes --redis-resync-on-master-change
```

- Load: SDES calls over ng at 5 a second for 40 s; at 10 s, `SENTINEL FAILOVER` or a 30 s freeze of the master; then a takeover node restores over TLS. The plain-TCP run uses the same build and servers without TLS.

**Results.**

| | Failover, TLS ×3 | Frozen master, TLS ×3 | Failover, plain TCP ×1 |
|---|---|---|---|
| Lost / deleted back | 0 / 0 | 0 / 0 | 0 / 0 |
| Takeover restored / missing | 156 / 0 | 156 / 0 | 156 / 0 |
| ng failed | 0 | 0, 1, 0 (load-tool artefact) | 0 |
| Longest gap | 0.22 s | 1.41 s | 0.21 s |
| Offer p50 | 2.7–2.8 ms | 2.6–2.8 ms | 2.4 ms |
| Reconnect to the new master | 3.4–4.6 ms | 3.3–4.3 ms | 4.3 ms |
| Resync | 47–48 calls + 12 `DEL`s, 5.0–5.1 s | 41–42 + 12, 21–32 ms | 48 + 12, 5.0 s |
| `rediswriteerrors` | 107, 118, 113 | 0 | 108 |

- "Reconnect" covers the TCP connect, the TLS handshake, `AUTH`, `SELECT` and `INFO`. It is paid once per reconnect, by the thread that makes the attempt. After a failover it took 3.4–4.6 ms over TLS, against 4.3 ms over plain TCP: no measurable difference.
- The load-tool artefact is the create whose offer stalled behind the frozen master.

**Refusals on the lab**, each stopping the startup with its reason:

- another CA: `self-signed certificate in certificate chain`;
- a wrong `--redis-tls-server-name`: `IP address mismatch`;
- a node whose name is only in the CN: `hostname mismatch`;
- Sentinels with another CA: refused for each Sentinel, then `no Sentinel answered`;
- no client certificate: `Server closed the connection`. With TLS 1.3 the server's refusal arrives at the first command.

</details>

### Limits

- **A missing client certificate** shows as "Server closed the connection" (TLS 1.3).
- **Not supported:** password-protected keys, CRLs, a cipher choice (as for MQTT).
- **One `--redis-tls-server-name`** for every data node. TLS cannot be on for some roles and off for others.
- **EL:** `hiredis_ssl` in `hiredis-devel` not checked.
- **Not measured:** a partition during the handshake (the test covers a silent peer), and certificate rotation.

### Open questions

1. Is `libio-socket-ssl-perl` as a build dependency for one test file acceptable, or should the test skip when the module is missing?
2. Should TLS be turned on by `--redis-cafile` (as for MQTT), or by an explicit `--redis-tls` switch?
3. Should the TLS options be shared by the Sentinels and the data nodes, or should the Sentinels have their own `--redis-sentinel-*` options?
4. Should we keep this name check, or wait for a hiredis release with `verify_name` and use `redisCreateSSLContext`?
5. Do you want a CI job on a build without TLS (for example bookworm), to keep the guard compiling?

The two commits can be split or reshaped if you prefer: TLS for the data nodes alone stands on its own.
