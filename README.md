# up

Update all your developer tools with one command.

`up` runs your update commands together, shows the progress on a live board,
and then tells you exactly which packages changed.

```
  Updating your tools   00:24

  ✔  Claude skills   00:04  2 changed
  ⠹  npm globals     00:24  working
  ✔  Homebrew index  00:06  Already up-to-date.
  ·  Homebrew apps          waiting

  ███████████████░░░░░░░░░░░░░  2/4
```

At the end you get the change list:

```
  All updates are done.   total 01:24

  Claude skills 2 changed
    ↑ code-review
    ↑ tdd

  npm globals 2 changed
    ↑ githits   0.11.4 → 0.12.0
    + newtool   1.0.0  (new)

  Homebrew apps 3 changed
    ↑ git       2.43.0 → 2.44.0
    ↑ jq        1.7 → 1.7.1
    − oldpkg    (removed)
```

## What it updates

| Job | Command | Lane |
| --- | --- | --- |
| Claude skills | `npx --yes skills update -g -y` | node |
| npm globals | `npm update -g` | node |
| Homebrew index | `brew update` | brew |
| Homebrew apps | `brew upgrade` | brew |

## Install

```sh
git clone git@github.com:iler/up.git ~/projects/up
mkdir -p ~/.local/bin
ln -s ~/projects/up/up ~/.local/bin/up
```

Then type `up`. If `~/.local/bin` is not in your `PATH`, use an alias instead:

```sh
alias up=~/projects/up/up
```

## Requirements

- bash 3.2 or later. This is the version that macOS supplies.
- `jq`, for the change list of the skills and the npm globals. Without `jq`
  those two lists stay empty. The updates still run.

The script has no other dependencies. This is on purpose: an updater must not
depend on the packages that it updates.

## Options

| Option | Effect |
| --- | --- |
| `-s`, `--sequential` | Run all jobs one after the other. |
| `-v`, `--verbose` | Show the raw output. This turns off the live board. |
| `-o`, `--only <name>` | Run only the jobs that match `<name>` or lane `<name>`. |
| `-n`, `--dry-run` | Show the jobs and the commands. Do not run them. |
| `-l`, `--list` | List the jobs and stop. |
| `--color <when>` | `always`, `never` or `auto`. The default is `auto`. |
| `-h`, `--help` | Show the help. |
| `--version` | Show the version. |

Examples:

```sh
up                 # run all updates
up --only brew     # run only the Homebrew lane
up -v --only npm   # update the npm globals and show the raw output
```

## How it works

### Lanes

Each job belongs to a lane. Lanes run at the same time. The jobs in one lane
run in sequence. The `node` lane and the `brew` lane are separate, because two
npm processes must not write to the global directory together.

If a job fails, the script skips the remaining jobs of that lane. `brew upgrade`
therefore never runs after a failed `brew update`. This is the same rule as
`brew update && brew upgrade`.

### The change list

The script does not read the output of the update commands. Output text changes
between tool versions. Instead each job has a snapshot function. The function
prints one line for each installed item:

```
<name> <version>
```

The script runs the function before and after the job, and then compares the two
lists. The version source is:

| Job | Source |
| --- | --- |
| Claude skills | `skillFolderHash` in `~/.agents/.skill-lock.json` |
| npm globals | `npm ls -g --depth=0 --json` |
| Homebrew apps | `brew list --versions` |

This also works for `npm update -g`, which only reports *"changed 3 packages"*
and never says which ones.

Skills have no version number, only a content hash. Those lines therefore show
the name alone.

### The board

The board redraws in place. It reads the terminal size from `stty size`, because
`tput cols` cannot see the terminal inside a command substitution and reports 80.
The board also turns the line wrap off while it runs. A long line can then never
become two screen lines and break the redraw.

The layout follows the window height. The full board needs `jobs + 5` rows.
With fewer rows the script drops the blank lines. With fewer than `jobs + 3`
rows it shows one status line only. The four default jobs therefore give a full
board at 9 rows, a compact board at 7, and one line at 6.

Without a terminal — in `cron`, or through a pipe — the script prints plain
lines instead of the board.

## Add your own job

Add one line to `register_jobs`:

```sh
add_job "<name>" <lane> "<necessary program>" "<command>" [snapshot function]
```

For example:

```sh
add_job "Rust" rust "rustup" "rustup update" snap_rust
```

Use a new lane name if the job can run at the same time as the others. Give the
job a snapshot function if it installs packages. The snapshot function is
optional.

## Logs

Each job writes its own log file to `~/.local/state/update-script/<timestamp>/`.
The script keeps the last 10 runs. Set `UP_LOG_DIR` to change the location.

If a job fails, the script prints the end of that log and exits with status 1.

## Notes

The jobs get no keyboard input. A job that asks for a password therefore fails
with a clear message. It does not stop and wait. This can happen with a Homebrew
cask that needs `sudo`.

`npx skills update` normally asks for the scope. The script uses `-g -y` to stop
the question and to always update the global skills. This also matches the lock
file that the change list reads. Remove the `-g` in `register_jobs` if you
prefer the project scope.

## Licence

MIT. See [LICENSE](LICENSE).
