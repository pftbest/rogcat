[![Build Status](https://travis-ci.org/flxo/rogcat.svg)](https://travis-ci.org/flxo/rogcat)
[![Build status](https://ci.appveyor.com/api/projects/status/ng8npy7ym6l8lsy0?svg=true)](https://ci.appveyor.com/project/flxo/rogcat)
[![crates.io](https://img.shields.io/crates/v/rogcat.svg)](https://crates.io/crates/rogcat)

# rogcat

...is a `adb logcat` wrapper. The `Android Debugging Bridge` (`adb`) is the default way to interact with a Android
device during development. The `logcat` subcommand of `adb` allows access to `Androids` internal log buffers. `rogcat`
tries to give access to those logs in a convenient way including post processing capabilities. The main feature probably
is a painted and reformatted view. `rogcat` can read logs from

* running `adb logcat` (default)
* a custom command (`stdout` and `stderr`)
* one or multiple files
* `stdin`
* a serial port
* TCP host and port

The processing steps within a `rogcat` run include parsing of the input stream and applying filters (if provided).
`rogcat` comes with a set of implemented in and output formats:

* `csv:` Comma separated values
* `raw:` Record (line) as received
* `html:` A static single page html with a static table. This option cannot be used as input format. The page layout needs some love...
* `human:` A human friendly colored column based format. See screenshot
* `json:` Single line JSON

Except the `human` and `html` format the output of `rogcat` is parseable by `rogcat`.

![Screenshot](/screenshot.png)

## Examples

The following examples show a subset of `rogcats` features. *Please read `--help`!*

### Live

Capture logs from a connected device and display unconditionally. Unless configurated otherwise, `Rogcat` runs `adb logcat -b main -b events -b crash -b kernel`:

`rogcat`

Write captured logs to `testrun.log`:

`rogcat -o testrun.log`

Write captured logs to file `./trace/testrun-XXX.log` by starting a new file every 1000 lines. If `./trace` is not present
it is created:

`rogcat -o ./trace/testrun.log -n 1000` or `rogcat -o ./trace/testrun.log -n 1k`

### stdin

Process the `stdout/stderr` output of `somecommand`:

`rogcat somecommand` or `somecommand | rogcat -`

### Filter

Display logs from `adb logcat` and filter on records where the tag matches `^ABC.*` along with *not* `X` and the message includes `pattern`:

`rogcat -t "^ADB.*" -t \!X -m pattern`

The Read all files matching `trace*` in alphanumerical order and dump lines matching `hmmm` to `/tmp/filtered`:

`rogcat -i trace* -m hmmm  -o /tmp/filtered`

Check the `--message` and `--highlight` options in the helptext.

### Serial

Open and read `/dev/ttyUSB0` with given settings and process:

`rogcat -i serial:///dev/ttyUSB0@115200,8N1`

...and on `Windows` this would look like this:

`rogcat -i serial://COM0@115200,8N1`

The `,8N1` part is optional and default ;-).

### TCP

To connect via TCP to some host run something like:

`rogcat tcp://traceserver:1234`

### Bugreport

Capture a `Android` bugreport. This only works for `Android` version prior 7:

`rogcat bugreport`

Capture a `Android` bugreport and write (zipped) to `bugreport.zip`:

`rogcat bugreport -z bugreport.zip`

### Log

Write message "some text" into the device log buffer (e.g annotations during manual testing):

`rogcat log "some text"`

Set level and tag or read from `stdin`:

```
rogcat-log
Add log message(s) log buffer

USAGE:
    rogcat log [OPTIONS] [MESSAGE]

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
    -l, --level <LEVEL>    Log on level [values: trace, debug, info, warn, error, fatal, assert, T, D, I, W, E, F, A]
    -t, --tag <TAG>        Log tag

ARGS:
    <MESSAGE>    Log message. Pass "-" to capture from stdin'
```

### Profiles

List available profiles (see Profiles chapter):

`rogcat profiles --list`

Live trace with profile `app`:

`rogcat -p app`

## Installation

With a working/recent `Rust` and `cargo` setup run

```
cargo install rogcat
```

or use Homebrew by running

```
brew install flxo/tap/rogcat
```

or grab one of the [binary releases](https://github.com/flxo/rogcat/releases) on the GitHub page.

## Configuration

When `rogcat` runs without any command supplied it defaults to running `adb logcat -b all`. The following options
can be overwritten in the `rogcat` config file `config.toml`. The location of the config file is platform specific:

* MacOS: `$HOME/Library/Preferences/rogcat/config.toml`
* Linux: `$HOME/.config/rogcat/config.toml`
* Windows: `%HOME%/AppData/Roaming/rogcat/config.toml`

### Restart

By default `rogcat` restarts `adb logcat` when that one exits. This is intentional behavior to make `rogcat` reconnect
on device power cycles or disconnect/reconnects. A `Windows 7` bug prevents `rogcat` from restarting `adb`.  Place
`restart = false` in the configuration file mentioned above to make `rogcat` exit when `adb` exits.

### Buffer

The default behavior of `rogcat` is to dump `all` logcat buffers. This can be overwritten by selecting specific buffers in
the `rogcat` configuration file. e.g:

```
buffer = ["main", "events"]
```

### Terminal settings

Some parameters of the `human` format are adjustable via the config file:

```
terminal_tag_width = 20
terminal_shorten_tag = true
terminal_show_time_diff = true
terminal_show_date = false
terminal_time_diff_width = 10
terminal_hide_timestamp = true
terminal_color = never
terminal_no_dimm = true
```

## Profiles

Optionally `rogcat` reads a (`toml` formated) configuration file if present. This configuration may include tracing profiles
('-p') and settings. The possible options in the configuration file are a subset of the command line options. The configuration
file is read from the location set in the environment variable `ROGCAT_PROFILES` or a fixed pathes depending on your OS:

* MacOS: `$HOME/Library/Preferences/rogcat/profiles.toml`
* Linux: `$HOME/.config/rogcat/profiles.toml`
* Windows: `%HOME%/AppData/Roaming/rogcat/profiles.toml`

The environment variable overrules the default path. See `rogcat profiles --help` or `rogcat profiles --examples`.

Example:

```
[profile.B]
comment = "Messages starting with B"
message = ["^B.*"]

[profile.ABC]
extends = ["A", "B"]
comment = "Profiles A, B plus the following filter (^C.*)"
message = ["^C.*"]

[profile."Comments are optional"]
tag = ["rogcat"]

[profile.complex]
comment = "Profiles can be complex. This one is probably very useless."
highlight = ["blah"]
message = ["^R.*", "!^A.*", "!^A.*"]
tag = ["b*", "!adb"]

[profile."W hitespace"]
comment = "Profile names can contain whitespaces. Quote on command line..."

[profile.A]
comment = "Messages starting with A"
message = ["^A.*"]

[profile.rogcat]
comment = "Only tag \"rogcat\""
tag = ["^rogcat$"]
```

To check your setup, run `rogcat profiles --list` and select a profile for a run by passing the `-p/--profile` option.

## Usage

```
rogcat 0.2.15-pre
Felix Obenhuber <felix@obenhuber.de>
A 'adb logcat' wrapper and log processor. Your config directory is "/Users/felix/Library/Application Support/rogcat".

USAGE:
    rogcat [FLAGS] [OPTIONS] [COMMAND] [SUBCOMMAND]

FLAGS:
    -c, --clear             Clear (flush) the entire log and exit
    -d, --dump              Dump the log and then exit (don't block)
        --help              Prints help information
        --hide-timestamp    Hide timestamp in terminal output
        --no-dimm           Use white as dimm color
        --overwrite         Overwrite output file if present
    -r, --restart           Restart command on exit
        --shorten-tags      Shorten tags by removing vovels if too long for human terminal format
        --show-date         Show month and day in terminal output
        --show-time-diff    Show the time difference between the occurence of equal tags in terminal output
    -s, --skip              Skip records on a command restart until the last received last record is received again. Use
                            with caution!
    -V, --version           Prints version information

OPTIONS:
    -b, --buffer <buffer>...
            Select specific (logcat) log buffers. Defaults to main, events, kernel and crash (logcat default)

        --color <color>                          Terminal coloring option [values: auto, always, never]
    -a, --filename-format <filename_format>
            Select a format for output file names. By passing 'single' the filename provided with the '-o' option is
            used (default).'enumerate' appends a file sequence number after the filename passed with '-o' option
            whenever a new file is created (see 'records-per-file' option). 'date' will prefix the output filename with
            the current local date when a new file is created [values: single, enumerate, date]
    -f, --format <format>
            Output format. Defaults to human on stdout and raw on file output [values: csv, html, human, json, raw]

    -H, --head <head>                            Read n records and exit
    -h, --highlight <highlight>...
            Highlight messages that match this pattern in RE2. The prefix '!' inverts the match

    -i, --input <input>...
            Read from file instead of command. Use 'serial://COM0@115200,8N1 or similiar for reading a serial port

    -l, --level <level>
            Minimum level [values: trace, debug, info, warn, error, fatal, assert, T, D, I, W, E, F, A]

    -m, --message <message>...                   Message filters in RE2. The prefix '!' inverts the match
    -o, --output <output>                        Write output to file
    -p, --profile <profile>                      Select profile
    -P, --profiles-path <profiles_path>          Manually specify profile file (overrules ROGCAT_PROFILES)
    -n, --records-per-file <records_per_file>    Write n records per file. Use k, M, G suffixes or a plain number
    -t, --tag <tag>...                           Tag filters in RE2. The prefix '!' inverts the match
    -T, --tail <tail>                            Dump only the most recent <COUNT> lines (implies --dump)

ARGS:
    <COMMAND>    Optional command to run and capture stdout from. Pass "-" to d capture stdin'. If omitted, rogcat
                 will run "adb logcat -b all" and restarts this commmand if 'adb' terminates

SUBCOMMANDS:
    bugreport      Capture bugreport. This is only works for Android versions < 7.
    completions    Generates completion scripts
    devices        Show list of available devices
    help           Prints this message or the help of the given subcommand(s)
    log            Add log message(s) log buffer
    profiles       Show and manage profiles
```

## Bugs

There are plenty. Please report on GitHub. Patches are welcome!

## Licensing

Rogcat is open source software and comes with no warranty. See ``COPYING`` for details.
