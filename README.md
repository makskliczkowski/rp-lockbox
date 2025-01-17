# rp-lockbox
rp-lockbox is a firmware for the [Red Pitaya](https://www.redpitaya.com/) FPGA board which turns it
into a feedback controller (lockbox) optimized for optics experiments.

## Features
* Multiple-Input, Multiple-Output PID controller
* Web interface for configuration
* Automatic relock (e.g. using the transmission signal of a cavity)
* Remote control via Ethernet using SCPI commands
* Autonomous operation (connection to a PC is only required for configuration)

## Installation
Build the software and FPGA configuration from source (see below) or download a binary archive
(rp-lockbox.tar.gz) [here](https://github.com/schmidf/rp-lockbox/releases).

Set up the Red Pitaya following the [official manual](https://redpitaya.readthedocs.io/en/latest/index.html).

Copy the firmware tarball to the Red Pitaya using a [SCP](https://en.wikipedia.org/wiki/Secure_copy)
client:
```
scp rp-lockbox.tar.gz root@$RPHOSTNAME:/root/
```

Connect to the Linux system running on the Red Pitaya using [SSH](https://redpitaya.readthedocs.io/en/latest/developerGuide/os/ssh/ssh.html).

(On the Red Pitaya) unpack the tarball and run the install script:
```
tar xf rp-lockbox.tar.gz
cd rp-lockbox
./install.sh
```

Then start the lockbox SCPI command server and the web interface (this stops the default web server):
```
systemctl start lockbox
systemctl start lockbox-web-interface
```

If you want to automatically start the SCPI server instead of the web server on system boot, execute
the following commands:
```
systemctl disable redpitaya_nginx
systemctl enable lockbox
systemctl enable lockbox-web-interface
```

The following commands revert to the default configuration:
```
systemctl disable lockbox
systemctl disable lockbox-web-interface
systemctl enable redpitaya_nginx
```

## Usage
The lockbox can be configured using the included web interface which runs on the default HTTP port
(80).

It is also possible to remote control the lockbox by sending SCPI commands via a TCP/IP connection
on port 5000.
See the [official documentation](https://redpitaya.readthedocs.io/en/latest/appsFeatures/remoteControl/remoteControl.html)
for examples how to do this in various programming languages.

The available SCPI commands are documented [here](doc/SCPI_commands.rst).

A python module and example GUI application for controlling the lockbox can be found in the
[examples/python](examples/python) folder. A standalone executable version of the GUI for windows
can be downloaded [here](https://github.com/schmidf/rp-lockbox/releases).

### PIDs
The FPGA implements four PID controllers that connect the two inputs of the Red Pitaya with the two
outputs in all possible combinations.

### Output limiting
Global limits can be defined for both outputs of the Red Pitaya. When an output is at its limit, the
integrators of the corresponding PID controllers are frozen in order to avoid integrator windup. If
automatic integrator reset of the PID is enabled, the integrator register is reset to zero.

### Relock
Each of the PID controllers contains an automatic relock feature. When the feature is enabled, it
monitors the voltage on one of the Red Pitaya auxiliary analog inputs.
If the voltage leaves a user-defined window, the PID is considered unlocked. In this case the
internal state of the PID controller is frozen and a triangular voltage sweep with user-defined slew
rate and increasing amplitude is generated on the output. Once the monitored signal re-enters the
specified window, the PID is engaged again.

## How to build
### Architecture
rp-lockbox consists of three core components:

1. The FPGA configuration (lockbox.bit)
2. An API library (liblockbox.so) for reading and modifying the FPGA registers
3. The SCPI command server (lockbox-server)

### Build requirements
Only Linux is supported as a build environment. Further build requirements for the different
components are:

#### FPGA configuration:

* [Xilinx Vivado](https://www.xilinx.com/products/design-tools/vivado.html) 2017.2 (the free WebPACK
edition is sufficient). The build scripts generated by Vivado are not compatible between versions,
so make sure to install exactly this version.

#### API and SCPI server:

* The ARMv7 (armhf) cross compiler toolchain (gcc-arm-linux-gnueabihf in Debian)

### Build process

Set up required environment variables:
```
source settings.sh
```

Check out the scpi-parser submodule:
```
git submodule update --init
```

The three components can be built separately:
```
make fpga
make api
make scpi
```

Or all at once:
```
make all
```

Finally run
```
make install
make tarball
```
to copy the generated files to the `build` subdirectory and generate a compressed archive.
