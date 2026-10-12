# arosl/repo_updater: the fork

This fork of
[Dicklesworthstone/repo_updater](https://github.com/Dicklesworthstone/repo_updater)
runs ru's agent sessions on [herdr](https://herdr.dev) through
[nhm](https://github.com/arosl/nhm), in place of tmux and ntm. It is part of
the [ACFS fork](https://github.com/arosl/agentic_coding_flywheel_setup), which
replaces ntm with nhm (ACFS bead acfs-9hkh.29).

The fork follows upstream closely. It carries only the changes below, keeps
them small, and merges upstream regularly. It adds no features of its own.

## The delta

ru calls ntm in three places: `ai-sync` and `dep-update` (through
`spawn_ai_session`), `agent-sweep`, and the review driver `ntm`. Each call
now goes to nhm. The function names (`ntm_*`), the driver name `ntm` and the
`--mode=ntm` flag stay, so the diff stays small and upstream merges stay
easy. The local driver (tmux and stream-json) is upstream's, unchanged.

| Where in `ru` | Upstream | The fork |
|---|---|---|
| `ntm_check_available`, `detect_review_driver`, `doctor`, the `ai-sync` and `dep-update` checks | `ntm`, `ntm --robot-status` | `nhm`, `nhm --robot-status` |
| `ntm_spawn_session` | `ntm --robot-spawn=S --spawn-cc=1 --spawn-wait --spawn-dir=D` | the same flags on `nhm` |
| `ntm_send_prompt` | `--robot-send=S --msg=…`, in 3.5 KB chunks above 4 KB | one `nhm --robot-send=S --type=claude --msg-file=-`, the prompt on stdin. nhm has no tmux SendKeys limit, and each chunk would be a prompt of its own. `ntm_send_prompt_chunked` is gone |
| `ntm_wait_completion` | `--robot-wait=S --condition=idle --transition` | `nhm --robot-ack=S --timeout=…`: it waits for the completion of the session's latest send. ru's exit codes come from its answer: 0 only when every confirmation is `prompt_returned`; 1 `timed_out`, or an agent that only started (`output_started`, which nhm reports with exit 0); 3 an entry in `failed`; 2 otherwise |
| `ntm_interrupt_session`, `ntm_driver_interrupt_session` | `--robot-interrupt=S`, `tmux send-keys C-c` | `nhm interrupt S` (each agent's own interrupt key) |
| `ntm_kill_session`, `ntm_driver_stop_session`, `kill_ai_session` | `ntm kill S -f`, `tmux kill-session` | `nhm kill S -f` (see the gaps) |
| `cleanup_agent_sweep_sessions`, `ntm_driver_list_sessions` | grep on `"name":"ru_sweep_…"`, `.sessions \| keys[]` | jq on `.sessions[].name`: ntm's status lists sessions in an array |
| `capture_pane_output`, `capture_pane_tail` | `tmux capture-pane -t S:0.1` | `nhm --robot-tail=S --lines=N`, slot `0.1`, else the first pane |
| `ntm_driver_get_session_state`, `ntm_driver_stream_events`, the `spawn_ai_session` poll | `--robot-activity=<pane>` `.state`, `ntm health` `.panes[0].activity` | `nhm --robot-activity=S` `.agents[0].state` (ntm's `ActivityOutput`); `WAITING`, `IDLE` and `STALLED` count as done where upstream read `idle` and `stale` |
| `ntm_driver_stream_events` health | `--robot-health=S` `.alerts` | `nhm health S --json`: the session's own health, when it is not `healthy`, is the alert |
| `ntm_driver_start_session` | `--robot-spawn S --dir D --prompt P`, `.panes[0]` | `nhm --robot-spawn=S --spawn-cc=1 --spawn-wait --spawn-dir=D`, then one send of the prompt; the pane is `.agents[0].pane` |
| `ntm_driver_send_to_session` | `--robot-send --pane=P`, then `tmux send-keys` | `nhm --robot-send=S --type=claude --msg-file=-`; it succeeds when `.successful` names the agent. There is no tmux session to fall back to |
| a session exists | `tmux has-session` | `nhm_session_exists`: `nhm --robot-status` lists it |
| `spawn_ai_session` | `ntm spawn S --cc=1 --no-user`, `tmux send-keys "cd …"`, `ntm send S --cc --file F` | `nhm --robot-spawn=S --spawn-cc=1 --spawn-no-user --spawn-wait --spawn-dir=<repo>`, `nhm send S --cc --file F` |
| `error_ntm_spawn_failed` | `ntm doctor`, `ntm list` | `nhm doctor`, `nhm list` |
| `RU_REPO_OWNER` (`self-update`), the robot-docs install line | `Dicklesworthstone` | `arosl`: ru updates from the fork's releases |
| `VERSION=`, `VERSION` | `1.5.0` | `1.5.0-nhm.1`, the fork's release version (see Releases) |

Installer and workflows:

| File | Change |
|---|---|
| `install.sh` | `REPO_OWNER="arosl"`: it installs the fork's releases. `maybe_install_ntm` never installs ntm; it reports nhm, or says that ACFS installs it |
| `.github/workflows/notify-acfs.yml` | the job runs only in upstream's repository; the fork has no `ACFS_DISPATCH_TOKEN`, and the ACFS fork pins ru by tag |

Tests:

| File | Change |
|---|---|
| `scripts/test_bin/nhm` | new: a mock nhm. It answers with the shapes nhm v0.1.2 printed against its own stub herdr, cut to the fields ru reads, and takes its pane text from the tmux mock beside it |
| `scripts/test_e2e_framework.sh` | `e2e_create_mock_ntm` copies the nhm mock in place of the ntm mock |
| `scripts/test_unit_ntm_driver.sh` | the `ntm_check_available` mocks are named `nhm`; nine tests of the driver functions against the nhm mock (ACFS rehearsal step E21.b) |
| `scripts/test_e2e_install.sh` | the mock curl answers for the fork's release URLs |
| `FORK.md` | this file |

## Releases

The fork tags `v<upstream version>-nhm.<n>` (for example `v1.5.0-nhm.1`) on
its `main`. Upstream's `release.yml` requires that `VERSION=` in `ru` and
the `VERSION` file equal the tag, so the version changes in a commit before
the tag. The workflow then publishes `ru`, `install.sh` and their `.sha256`
files. Its release notes still name upstream's URLs. `install.sh` and
`ru self-update` take the fork's latest release. The ACFS fork pins the
tagged `install.sh` in `checksums.yaml` and runs it with `RU_VERSION` set,
so "latest" never decides the version it installs (ACFS bead
acfs-9hkh.64).

## Gaps

These need a later nhm. Until then, the call fails, and ru handles it as it
handles any failed ntm call:

- **Spawn.** nhm 0.1.2 has no `--robot-spawn` (nhm.spawn, ACFS bead
  acfs-9hkh.19). Until a release has it, `agent-sweep`, `ai-sync`,
  `dep-update` and the review driver cannot start a session.
- **Kill.** nhm 0.1.2 has no `kill` (nhm.lifecycle, ACFS bead acfs-9hkh.20).
  ru ignores a failed kill, so a session stays open until that release.
  Close it in herdr.
- **Ownership.** Before ACFS's takeover (`nhm admin takeover`), ACFS's Bash
  owns the herdr server, and nhm refuses to spawn, send or interrupt.
- **gemini.** nhm has no gemini kind. `--agent=gemini` fails in
  `spawn_ai_session` with "unknown agent type". The help text still lists
  gemini, as upstream's does.

## Not changed, on purpose

- **The docs** (README, `SKILL.md`, the plan documents, the help text) still
  name ntm and tmux. Renaming them would conflict with nearly every upstream
  merge. Read "ntm" there as nhm.
- **The names** `ntm_*`, the driver `ntm`, `--mode=ntm` and the mock's
  `NTM_MOCK_*` variables keep upstream's spelling, so scripts that call ru
  keep working.
- **`scripts/test_bin/ntm`** stays, unused, so upstream merges of it apply.

A change to this list gets a bead first, so the delta stays listed.
