# BlueALSA HFP-HF / HSP-HS Tools

## Overview

A collection of shell scripts to help create a Bluetooth Handsfree and Headset device using BlueALSA (https://github.com/arkq/bluez-alsa) and Bluez (https://github.com/bluez/bluez/).

Requires BlueALSA features from the latest bluez-alsa source, not included in BlueALSA release 4.0.0 or earlier.

These scripts make use of some of the Bluez test programs that are included in the Bluez source code repository. Copy the `test` directory from the Bluez sources to a suitable location; the scripts here assume `/usr/local/share/bluez/test`, although that can be changed by setting the environment variable `BLUEZ_TEST_DIR`.

Multiple BlueALSA instances are not supported; it is required to run just one instance of the `bluealsa` service, with service name `org.bluealsa`.

It is possible to build a very simple "handsfree" device, where the only user interaction is to enable pairing mode, using just the basic components listed here. In this case all other interaction (including volume control) must be done on the AG device (e.g. the mobile phone). The resulting device would have functionality very similar to a simple Bluetooth speaker supporting HFP and HSP; and by adding the BlueALSA `bluealsa-aplay` utility support for A2DP can be included.

To add more user controls to the device, oFono can be used which would enable full Bluetooth HFP HandsFree support. Alternatively it is possible to implement simpler devices by creating clients using BlueALSA's PCM and RFCOMM APIs.

### Key Features

- HSP and HFP support
- Simple, "Just Works", Bluetooth pairing
- Permits only one audio device to connect at a time
- compatible with `bluealsa-aplay` for A2DP support
- compatible with `oFono` for HFP telephony support

## Basic Components

### bluealsa-simple-agent

Bluez requires a Bluetooth agent for device pairing, and optionally also for
authorizing device connections.
It is possible to use the Bluez `bluetoothctl` utility for interactive pairing; but we want our handsfree unit to be able to enter pairing mode at the press of a button, and to complete the pairing process without further user intervention.
So we use our own agent instead of `bluetoothctl`.

Once paired, a remote device needs to connect each of the services it wants to use.
Bluez can be told to "trust" a device, so that all service connections from that device are accepted without further authorization.
However we want to be able to accept audio service connections from just one device at a time.
So we **must not set the "trusted" attribute** of devices.
When a device is not trusted, Bluez instead requests authorization from the default agent for each service connection request.
In this way the agent can accept or refuse individual service connection requests.

The `bluealsa-simple-agent` service included here implements both the non-interactive pairing and service connection authorization requirements of our hands-free unit.
The agent consists of a bash script, `bluealsa-simple-agent.bash` and a `systemd` service unit to manage it.
The script is essentially a wrapper around the Bluez `simple-agent` test program.

Once enabled in `systemd`, the `bluealsa-simple-agent` service starts when the bluetooth service starts, and stops when bluetooth stops.
On start it ensures that pairing mode is turned off.
When pairing mode is enabled (by some other program) then the agent disconnects any currently connected device.
This is because only one device at a time can be connected.
As soon as one device has succesfully paired, the agent will cancel pairing mode so that the device can immediately connect.

The handsfree unit should not be in pairing mode permanently, so we also require a service that can enable pairing mode on demand.
See the `bluealsa-simple-pairable` service below for that.

The agent bash script needs to know the location of the Bluez test programs and the `python` executable (the Bluez test programs are all python scripts).
These locations can be set by editing the `bluealsa-simple-agent.service` file.
Change the environment variables there as required.
The defaults are:
```
Environment=PAIRABLE_TIMEOUT=180
Environment=PYTHON=/usr/bin/python3
Environment=BLUEZ_TEST_DIR=/usr/local/share/bluez/test
```

The `PAIRABLE TIMEOUT` environment variable is used when the agent starts, to set the default pairing time limit, in seconds.
This value is overridden by the timeout set by the `bluealsa-simple-pairable` service, so is included here only in case the user chooses some other method for pairing.

To use this service, enable it to start with the bluetooth system:
```
systemctl enable bluealsa-simple-agent
```

### bluealsa-simple-pairable

This is a one-shot service that puts the hands-free unit into pairing mode. It consists of just a `systemd` service unit file. This service should not be enabled to run automatically; to switch the unit to pairing mode, run
```
systemctl start bluealsa-simple-pairable
```
This will put the unit into pairing mode for a fixed time limit, then switch off that mode. To terminate the pairing mode before the timeout, run
```
systemctl stop bluealsa-simple-pairable
```
As for the agent above, the location of `python` and the Bluez test programs, and the pairable time limit, are all set by environment variables which can be edited as required. The timeout defined here overrides the value set by the agent.

### bluealsa-handsfree-audio

A service to route the bluetooth HFP and HSP audio streams between BlueALSA and the local sound card. It consists of a bash script, `bluealsa-handsfree-audio.bash`, and a `systemd` service unit file to manage it. When enabled in `systemd` this service is started when the bluetooth service starts.

The script uses `bluealsa-cli`, `aplay`, and `arecord`. It must be started before the first device connects, and there must be at most one device connected at any time.

By default, the script uses the ALSA `default` device for speaker and microphone. To choose different devices, edit the environment variables defined in the `systemd` service unit file.

`BA_HF_SPEAKER` must be a valid ALSA PCM playback device name, for example `plughw:0,0` etc.

`BA_HF_MICROPHONE` must be a valid ALSA PCM capture device name, for example `plughw:0,0` etc.

`BA_HF_LATENCY` is a value, in milliseconds, that is used to select the ALSA device hardware parameters. The default value is 100, which gives good results in most cases. Using a lower value will reduce the delay caused by buffering, but will also increase the likelihood of overruns and underruns, resulting in drop-outs in the audio.

`BA_HF_SOFTVOL` is a string that determines whether `bluealsa-handsfree-audio` sets the speaker audio stream to use `BlueALSA`'s "SoftVolume" volume control. Permitted values are:
* "true" - always use SoftVolume
* "false" - never use SoftVolume
* "hsp" - use SoftVolume only for HSP connections

It defaults to `true` (i.e. always use SoftVolume). In SoftVolume mode, BlueALSA applies volume change requests from the remote device to the PCM stream before passing the stream to `bluealsa-handsfree-audio`. This mode is recommended when using `bluealsa-handsfree-audio` on its own or in combination with `bluealsa-aplay` for A2DP support. When using with oFono, this variable should be set to `hsp` because all HFP volume control is taken over by oFono. To disable volume control (for example when using some other application to operate the sound card controls), set this variable to `false`.

To use this service, enable it to start with the bluetooth system:
```
systemctl enable bluealsa-handsfree-audio
```

### A2DP support

To add support for the A2DP profile, install and enable `bluealsa-aplay` from the bluez-alsa project. It is recommended to use BlueALSA soft-volume volume control.
There is currently no support for the microphone channel of codecs such as SBC Faststream.

## oFono integration

The above components are sufficient to build a complete hands-free unit when used in conjunction with oFono. All user interaction would have to be done through oFono clients, which are out-of-scope for this project.

It is necessary to have the oFono service running before pairing or connecting devices can be done, so it is advisable to add the following to the `[Unit]` section of `bluealsa-simple-agent.service`
```
Requires=ofono.service
After=ofono.service
```

## Additional Buttons

If not using oFono, it is possible to add support for additional buttons by using BlueALSA's own APIs. For example to create a device with similar controls to "ear-bud" devices which would permit:
* Volume control - local "volume up" and "volume down" buttons to control HFP/HSP volume (and A2DP volume if A2DP support is installed)
* Microphone mute
* Incoming call accept
* Call reject / cancel

It is planned to add another service here which will take care of the BlueALSA API interaction and allow the use of command-line utilities to action button-presses for this purpose.

## Installation

The scripts and systemd unit files here are templates that need to be instantiated by substituting the desired installation directories.

A `meson` build specification file is included to simplify the installation. To
use it, do:

```shell
meson setup builddir
sudo meson install -C builddir
```

The default install prefix is `/usr/local`, so that the scripts are installed into `/usr/local/bin` and the systemd unit files into `/usr/local/lib/systemd/system`.

To install on a production system the prefix should be overridden in the `setup` command, and optionally the systemd directory can also be overridden to be outside the prefix. So a typical example is
```shell
meson setup --prefix=/usr -Dunitdir=/lib/systemd/system builddir
sudo meson install -C builddir
```

To install into a local directory that can be used to create a distributable package, set the DESTDIR environment variable for the `meson install` command. For example, to create a directory called "INSTALLATION"
```shell
sudo DESTDIR="$(pwd)/INSTALLATION" meson install -C builddir
```
