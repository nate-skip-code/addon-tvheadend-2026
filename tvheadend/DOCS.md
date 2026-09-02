# Home Assistant Community Add-on: TVHeadend

TVHeadend is a TV streaming server and recorder supporting:
DVB-S, DVB-S2, DVB-C, DVB-T, DVB-T2, ATSC, ISDB-T, IPTV, SAT>IP and HDHomeRun
as input sources.
TVHeadend offers the HTTP (VLC, MPlayer), HTSP (Kodi, Movian) and SAT>IP streaming.

Multiple EPG sources are supported such as
over-the-air DVB and ATSC including OpenTV DVB extensions, XMLTV, PyXML.

Included along with TVHeadend:

- Streamlink (preinstalled)
- WebGrab+Plus (optional, enable it with the `webgrabplus` option)

## Installation

The installation of this add-on is pretty straightforward and not different in
comparison to installing any other Home Assistant add-on.

1. Add this repository to Home Assistant:
   [![Home Assistant with repository URL pre-filled][my-ha-shield]][my-ha-repo]
1. Search for the "TVHeadend" add-on in the Supervisor add-on store and install it.
1. Start the "TVHeadend" add-on.
1. Check the logs of the "TVHeadend" to see if everything went well.
1. Click the "OPEN WEB UI" button and start using it.
1. Ready to go!

## Configuration

**Note**: _Remember to restart the add-on when the configuration is changed._

Example add-on configuration:

```yaml
webgrabplus: false
system_packages:
  - ffmpeg
init_commands:
  - echo 'Hello World'
```

**Note**: _This is just an example, don't copy and paste it! Create your own!_

### Option: `webgrabplus`

Installs the [WebGrab+Plus][webgrabplus] EPG grabber on the first start of the
add-on and schedules a nightly guide update. Disabled by default.

The grabber is downloaded from webgrabplus.com at runtime and stored in
`/config/tvheadend/wg++/`, so it survives add-on updates. It runs on the Mono
runtime, which is reinstalled on every start because the container filesystem
is rebuilt on each add-on update.

If any of that fails, the add-on logs a warning, starts TVHeadend anyway, and
retries on the next start - a broken grabber never prevents the add-on from
running.

### Option: `system_packages`

Allows you to specify additional [Alpine packages][alpine-packages] to be
installed to the TVHeadend Addon (e.g., `ffmpeg`, `g++`, etc. ).

**Note**: _Adding many packages will result in a longer start-up time for the add-on._

### Option: `init_commands`

Customize your TVHeadend environment even more with the `init_commands` option.
Add one or more shell commands to the list, and they will be executed
every single time this add-on starts.

## Additional Details

- Config files are stored in `/config/tvheadend/`
- Recording files are stored in `/config/tvheadend/recordings/`
- `/dev/dvb/`, `/dev/dri/` would be respectively mapped to
  `/dev/dvb/`, `/dev/dri/` inside the addon.

Consider, backing `/config/tvheadend/` up whenever migrating.

## TV tuners

The add-on maps `/dev/dvb/` into the container, and on start it logs the
adapters it can see:

```txt
Detected 1 DVB adapter(s):
  /dev/dvb/adapter0 (1 frontend(s))
    /dev/dvb/adapter0/frontend0: opened OK
```

`opened OK` means the add-on could actually open the tuner, which is what
TVHeadend needs. If it instead logs `cannot open (Operation not permitted)`,
the device node is mapped into the container but the container is not
permitted to use it - TVHeadend responds to that by listing no adapters at
all, with an empty **TV adapters** tree and nothing in its own log.

That happens when a device is mapped by directory rather than by node. The
add-on's `devices` list therefore names every `/dev/dvb/adapterN/*` node
individually, for adapters 0 through 8; if you somehow have more than nine
tuners, extend that list in `config.yaml`.

If it instead warns that there is no `/dev/dvb` or that no adapters were found,
the problem is on the host, not in TVHeadend. **An add-on cannot load kernel
modules or firmware for the host** - it can only use device nodes the host
kernel has already created. Two things must be true on the host:

1. The kernel driver for the tuner is present and loaded.
2. Any firmware blob the tuner needs is in the host's `/lib/firmware/`.

Check the host's kernel log (`dmesg`) for the tuner. A line such as
`Direct firmware load for dvb-demod-si2168-b40-01.fw failed with error -2`
means the driver bound but the firmware is missing.

Home Assistant OS does ship drivers and firmware for many common tuners - a
Hauppauge Xbox One stick (em28xx + si2168) is detected out of the box on HAOS
18.2 / generic-x86-64, for instance. Coverage is not exhaustive, though, and
requests to broaden it have [been declined][haos-dvb]. If `dmesg` shows the
driver never binding, a Home Assistant Supervised install on a general-purpose
distribution is the fallback.

### The tuner works but no channels are found

If the add-on logs `opened OK`, TVHeadend lists the adapter, and scans run and
tune each frequency, but every mux comes back `FAIL` with zero services, the
problem is the radio path rather than anything in software. In rough order of
how often each is the culprit:

1. **USB 3.0 interference.** USB 3.0 ports and cables emit broadband noise
   across the UHF television band, and a tuner stick plugged straight into
   one can be completely desensitized while looking perfectly healthy. Move
   the tuner to a USB 2.0 port, and preferably use an extension cable to get
   it away from the machine - the whole computer is a noise source. This is
   the single most common cause of "everything is configured right and I
   still get nothing".
2. **Unpowered amplifier.** An amplified antenna whose power injector is not
   connected usually attenuates rather than passes signal, giving total
   silence instead of weak reception.
3. **Amplifier overload.** Close to high-power transmitters, an amplifier can
   drive the tuner's front end into compression. An overloaded tuner locks
   onto nothing, which looks identical to no signal. Try without the amp.
4. **Antenna, cabling or aim.** Check what you should be able to receive at
   your address, and on which RF channels, before assuming reception is
   possible at all.

To tell whether any signal is arriving, open **Status -> Stream** while a scan
is running. The number that matters is **Bandwidth**: a locked ATSC mux carries
roughly 19,000 kb/s. Zero means no lock. Treat the signal-strength and SNR
columns with suspicion - many drivers report a plausible-looking value derived
from AGC even when only noise is present.

A useful way to split the problem in half: connect the same antenna to a
television with a built-in tuner and run its channel scan. If the television
finds nothing either, the antenna or location is at fault; if it finds
channels, the tuner, its USB port or its cabling is.

### Hauppauge Digital TV Tuner for Xbox One

This tuner is an `em28xx` bridge with a Silicon Labs `si2168` (revision B40)
demodulator and an `si2157` tuner. It needs the `em28xx`, `em28xx_dvb`,
`si2168` and `si2157` modules, plus the `dvb-demod-si2168-b40-01.fw` firmware,
which is **not** part of `linux-firmware` - it is distributed by
[Hauppauge][hauppauge-linux] and has to be placed in the host's
`/lib/firmware/` by hand.

Once `dmesg` shows the frontend registering and `/dev/dvb/adapter0` exists on
the host, restart the add-on and it will pick the tuner up.


## Changelog & Releases

This repository keeps a change log using [GitHub's releases][releases]
functionality.

Releases are based on [Semantic Versioning][semver], and use the format
of `MAJOR.MINOR.PATCH`. In a nutshell, the version will be incremented
based on the following:

- `MAJOR`: Incompatible or major changes.
- `MINOR`: Backwards-compatible new features and enhancements.
- `PATCH`: Backwards-compatible bugfixes and package updates.

## Support

Got questions?

You have several options to get them answered:

- The Home Assistant [Community Forum][forum].
- You could also [open an issue here][issue] GitHub.

## Authors & contributors

This repository is maintained by [nate-skip-code][maintainer]. It is a fork of
the TVHeadend add-on originally created by [GauthamVarmaK][gautham].

This has been possible thanks to the community add-ons initiative by [Frenck][frenck]

## License

MIT License

Copyright (c) 2021-2023 GauthamVarmaK

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

[haos-dvb]: https://github.com/home-assistant/operating-system/issues/3653
[hauppauge-linux]: https://www.hauppauge.com/pages/support/support_linux.html
[webgrabplus]: https://www.webgrabplus.com/
[alpine-packages]: https://pkgs.alpinelinux.org/packages
[forum]: https://community.home-assistant.io/
[frenck]: https://github.com/frenck
[gautham]: https://github.com/GauthamVarmaK
[maintainer]: https://github.com/nate-skip-code
[my-ha-shield]: https://my.home-assistant.io/badges/supervisor_add_addon_repository.svg
[issue]: https://github.com/nate-skip-code/addon-tvheadend-2026/issues
[semver]: http://semver.org/spec/v2.0.0.htm
[my-ha-repo]: https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2Fnate-skip-code%2Faddon-tvheadend-2026
[releases]: https://github.com/nate-skip-code/addon-tvheadend-2026/releases
