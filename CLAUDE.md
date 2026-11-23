# esp-to-rs232

A port-powered Wi-Fi to RS-232 bridge that plugs directly onto a PC's DE-9 COM
port, running Tasmota where possible. Modelled on
[mithro/esp32-to-433mhz](https://github.com/mithro/esp32-to-433mhz) and
[mithro/esp32-to-gps](https://github.com/mithro/esp32-to-gps); follow their
layout, script style and conventions unless there is a reason not to.

## Git workflow (mandatory)

- **Small, logical commits.** Each commit is one logical unit of work (one
  file, one fix, one feature step). Commit messages use the
  "Area: what changed" style.
- **One branch per logical piece of work, in its own worktree**, created from
  `origin/main` in `.worktrees/<branch>` (use the
  `superpowers:using-git-worktrees` skill), once the repository has a remote.
- Never push, open PRs or merge to `main` without the user asking.

## Project rules

- **Power comes only from the port.** No external supply, no battery. The
  target is **30 mW average draw**. Every design choice is checked against
  the power budget in `docs/`. USB power is an absolute last resort; a
  different Wi-Fi chip and firmware is preferred over it.
- **9600 baud is the required target; 115200 is a stretch goal.**
- **Tasmota is preferred.** Use an ESP32-family or ESP8266 chip if the power
  budget allows. Other chips or firmware need the user's agreement.
- **No off-the-shelf RS-232 transceiver.** Level shifting uses discrete parts.
- **Research claims carry their source.** Every number in `docs/research/`
  names the document it came from; anything not verified is marked as such.
- **Open source tools only** (Python, KiCad, Tasmota/PlatformIO).
- Generated artefacts (boards, drawings) come from scripts in `scripts/` and
  must regenerate byte-identically; CI checks this with `git diff --exit-code`.

## Tooling

- Python via `uv` only (`uv run`, PEP 723 `# /// script` headers on scripts).
- Scratch files go in the project-local `tmp/` (gitignored), never `/tmp`.
- Dates are ISO 8601 (YYYY-MM-DD).
