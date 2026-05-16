# libaccu_sleep.bash

[日本語版](README.ja.md)

A somewhat reliable Bash timer helper for periodic loops.

`libaccu_sleep.bash` is a small Bash library that sleeps against an accumulated
target schedule instead of sleeping for a fresh relative interval on every loop.
This helps reduce drift when the loop body itself takes time.

## Requirements

- Bash 5 or later is recommended.
- A POSIX-like environment with `sleep`, `stty`, and `/dev/tty` if key input
  hooks are used

Accumulated scheduling uses Bash's `EPOCHREALTIME`. If `EPOCHREALTIME` is
unavailable, the library prints a warning when sourced and `ACCU_SLEEP` falls
back to a plain `sleep` call. In that fallback mode, accumulated scheduling,
drift correction, and key input hooks are not available.

## Usage

Source the library, then call `ACCU_SLEEP` with an interval in microseconds.

```bash
#!/usr/bin/env bash

source ./libaccu_sleep.bash

while true; do
    printf '%s\n' "$EPOCHREALTIME"

    # Sleep until the next 1-second schedule point.
    ACCU_SLEEP 1000000
done
```

`ACCU_SLEEP` keeps an internal next-target timestamp. If the loop body takes
100 ms and the interval is 1 second, the next sleep is shortened so the loop
continues to aim at the original schedule.

When `EPOCHREALTIME` is unavailable, `ACCU_SLEEP` sleeps for the requested
relative interval and does not keep an accumulated schedule.

Use `ACCU_SLEEP_RESET` when starting a new schedule.

```bash
ACCU_SLEEP_RESET
```

## API

### `ACCU_SLEEP <interval_us>`

Sleeps until the next accumulated schedule point.

- `interval_us` must be a non-negative integer number of microseconds.
- Returns `2` and prints an error if the interval is invalid.

### `ACCU_SLEEP_RESET`

Clears the accumulated schedule. The next `ACCU_SLEEP` call starts from the
current time.

## Key Input Hook

If a function named `ACCU_SLEEP_ON_KEY` is defined, the library reads one
character at a time from `ACCU_SLEEP_TTY` while waiting. The default input is
`/dev/tty`.

```bash
function ACCU_SLEEP_ON_KEY() {
    local key=$1

    case "$key" in
        q)
            ACCU_SLEEP_RESTORE_TTY
            exit 0
            ;;
    esac
}

trap 'ACCU_SLEEP_RESTORE_TTY' EXIT
```

You can override the input path before sourcing or using the library.

```bash
ACCU_SLEEP_TTY=/dev/tty
source ./libaccu_sleep.bash
```

TTY access is optional. If no hook is defined, or if the configured TTY is not
readable, `ACCU_SLEEP` falls back to `sleep`.

## Sample

Run the sample program to print per-tick timing information.

```bash
./libaccu_sleep.sample
```

The sample requires `EPOCHREALTIME` because it prints timing statistics.

For Bash 3.x environments, use the date-based sample instead.

```bash
./libaccu_sleep.bash3.sample
```

Controls for `libaccu_sleep.sample`:

- `Ctrl-x`: print current statistics and continue
- `Ctrl-c`: print statistics and exit

The Bash 3 compatible sample does not handle key input. It uses `date +%s.%N`
when GNU date nanoseconds are available, falls back to `date +%s` otherwise,
and prints statistics every 100 samples and when it exits.

## Development

Syntax check:

```bash
bash -n libaccu_sleep.bash
bash -n libaccu_sleep.sample
bash -n libaccu_sleep.bash3.sample
/bin/bash -n libaccu_sleep.bash3.sample
```

ShellCheck:

```bash
shellcheck -x libaccu_sleep.bash libaccu_sleep.sample libaccu_sleep.bash3.sample
```

Use `-x` for the sample so ShellCheck follows the sourced library file.

## License

GPL-3.0. See [LICENSE](LICENSE).
