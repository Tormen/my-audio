# my-audio

Sets the macOS default audio output by **rules** -- "when this device is
present, play through that one" -- and shows the audio status.

- An HDMI or DisplayPort display is plugged in -> play through it.
- A Bluetooth speaker connects -> play through a Multi-Output Device.

A LaunchAgent (`my-audio watch`) applies the rules each time the set of
audio devices changes, and **never in between**: an output you pick by hand
stays until the next plug or unplug.

Python 3, standard library only -- CoreAudio is called through `ctypes`.

## Commands

    my-audio              status: current output, the rule that picked it,
                          volume and mute, the agent's state
    my-audio -V           ... and every device: transport, uid, output streams
    my-audio apply        apply the rules now, once
    my-audio watch        the loop the LaunchAgent runs
    my-audio install      write the LaunchAgent and load it
    my-audio uninstall    what removing it would do; `uninstall go` does it

    --create-config [FILE]   the default config, to stdout or a new FILE
    --config FILE            use FILE
    --run-tests [FILTER]     the self-test
    --version                version, git describe, build id
    -Q / -V / -D [PATH]      quiet / verbose / debug

Status, for example:

     >>> output: Multi-Output Device  (rule 3: blue:* => Multi-Output Device -- 'Speaker' present)
        > volume 45%, not muted
        > agent: running [pid 1234] (local.my-audio)

## Rules

`AUDIO_RULES` in the config, one rule per line, read top to bottom:

    <transport>:<device-name glob> => <target device name | @>

    AUDIO_RULES='
    hdmi:*  => @                     # an HDMI display or receiver: play through it
    dprt:*  => @                     # a DisplayPort display: play through it
    blue:*  => Multi-Output Device   # any Bluetooth audio device
    '

- Each time the set of present devices changes, the **first** rule whose
  device is present picks the output. No rule matches -> `OUTPUT_FALLBACK`
  (`""` leaves the output as it is).
- `@` is the matched device itself; only devices with outputs count for it,
  so a Bluetooth microphone never becomes the output.
- A target is found by **name** only: a Multi-Output Device reports 0
  output streams and is still a valid target.
- Transports are CoreAudio's: `bltn usb hdmi dprt blue blea airp grup virt
  thun pci 1394 eavb ccwd fwrk unkn`, or `*`. `my-audio -V` shows each
  present device's.
- A target that is not present is an `ERROR` line in the log; `watch` goes on.

## How `watch` notices a change

CoreAudio calls a listener as soon as a device comes or goes; the loop then
waits `SETTLE_TIME` seconds (a device often arrives in steps) and applies
the rules. Without a wake-up it re-reads the device list every
`RECHECK_INTERVAL` seconds anyway -- one code path, the listener only makes
it early.

## Config

Found at `$MY_AUDIO_CONFIG`, `--config FILE`, else the first of
`/LINKS/default/my-audio.conf`, `/LINKS/default/my-audio`,
`~/.my-audio.conf`, `/etc/my-audio.conf`, `/usr/local/etc/my-audio.conf`.
Shell syntax, read with `.`. `my-audio --create-config` prints every key with
its explanation: `AUDIO_RULES`, `OUTPUT_FALLBACK`, `RECHECK_INTERVAL`,
`SETTLE_TIME`, `LAUNCHD_LABEL`, `LOG_FILE`.

## Install

    my-audio --create-config > ~/.my-audio.conf     # then edit the rules
    my-audio                                        # check what they decide
    my-audio install

`install` writes `~/Library/LaunchAgents/<LAUNCHD_LABEL>.plist` -- loaded in
the GUI session only (`LimitLoadToSessionType Aqua`), kept alive, output to
`LOG_FILE` -- and runs it with the Python that ran `install`, so launchd's
short `PATH` cannot pick another one.

The plist carries one line after its DOCTYPE saying who installed it, when
and how -- `<!-- installed by my-audio install (<user>) on <host>, <time>:
generated, build <id> -->` -- so `my-rollout` counts the agent as my-audio's
job, not as one nobody installed. `install` ignores that line when it
compares, and writes again a plist that lacks it.

## Tests

`my-audio --run-tests [FILTER]` -- TAP output. CoreAudio is replaced by a
fake device list, so plugging and unplugging is simulated; one read-only test
reads the real device list.
