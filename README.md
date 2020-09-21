---
page_type: sample
languages:
  - c
products:
  - open-enclave
  - trustbox
  - vs-code
---

# TrustBox Open Enclave development

This manual demonstrates how to set up and use Open Enclave development environment for Scalys [TrustBox Edge](https://scalys.com/trustbox-industrial). This development environment was prepared and tested on Ubuntu 18.04. For fast prototyping it also possible to use virtual machine available from Scalys OpenEnclave resources [page](https://scalys.com/downloads). It contains a clean Ubuntu 18.04 installation pre-configured with the Open Enclave development tooling.

# TrustBox setup

TrustBox is an industrial grade confidential computing solution based on an NXP Layerscape CPU. It comes with the pre-installed Ubuntu and allows to use Azure IoT Edge and Open Enclave along with a full hardware hardening to create a really secure edge node.

## Connectivity

The easiest way to have a first connection to the TrustBox is via serial connection. It can be done through the micro USB port, via serial terminal emulator program like PuTTY on Windows platforms or picocom on Linux. From example on the Ubuntu platform:

```
$ sudo apt install picocom
$ sudo picocom -b 115200 /dev/ttyUSB0
```

for more information please refer to the Trustbox Edge Quick Start Guide available at Scalys [webpage](https://scalys.com/downloads).

## TrustBox preparation

To ensure that OpenEnclave environment works properly it's version should correspond to the firmware used by TrustBox. Follow these instructions to update the TrustBox to the latest available firmware.

1. unscrew the case of TrustBox to extract the SD card from the board
2. download the latest available SD card image (img.gz) from Scalys binary releases [page](http://trustbox.scalys.com/pub/releases). For the development purposes it is recommended to select latest available devel image.
**Note**: it is also possible to build this image from sources. For that refer BSP User Guide available at Scalys downloads [page](https://scalys.com/downloads).
3. unpack and flash SD card image:
$ sudo gunzip trustbox-lsdk-2004-main-latest.img.gz
$ sudo dd if=trustbox-lsdk-2004-main-latest.img of=<DEVICE_PATH>
4. insert flashed SD card into TrustBox and connect to it via serial interface
5. right after power on of the TrustBox press any key during the countdown (in bootloader)
6. run the following commands to update the firmware:
```
run update_mmc_uboot_qspi_nor
run update_mmc_ppa_qspi_nor
run update_mmc_pfe_qspi_nor
reset
<once again stop boot during countdown>
env default -a
saveenv
reset
```

# Development environment

This Open Enclave development guide is based on Ubuntu 18.04 and the Visual Studio Code. Following instructions descibe how to create the setup required for both cross-compiling and applicaion debug via emulation.

## Install dependencies

Install general development dependencies:

```
sudo apt install git python-pip gcc-aarch64-linux-gnu g++-aarch64-linux-gnu gdb-multiarch libfdt1 docker.io
sudo snap install --classic code
sudo snap install --classic --channel=3.14/stable cmake
pip install --upgrade iotedgehubdev
git clone --recursive https://github.com/openenclave/openenclave.git && cd openenclave
sudo scripts/ansible/install-ansible.sh
sudo ansible-playbook scripts/ansible/oe-contributors-setup-cross-arm.yml
cd ..
```

Allow current user to control docker.
```
sudo usermod -a -G docker $USER

```
**Note**: it is required to reboot the machine after the command above.

## Prepare arm64 cross compiling in emulation

This process uses qemu aarch64 emulation. Some of the following steps can be skipped on distributions more recent than 18.04.

```
sudo apt install binfmt-support qemu-user-static
sudo mkdir -p /lib/binfmt.d
sudo sh -c 'echo :qemu-arm:M::\\x7fELF\\x01\\x01\\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x02\\x00\\x28\\x00:\\xff\\xff\\xff\\xff\\xff\\xff\\xff\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\xfe\\xff\\xff\\xff:/usr/bin/qemu-arm-static:F > /lib/binfmt.d/qemu-arm-static.conf'
sudo sh -c 'echo :qemu-aarch64:M::\\x7fELF\\x02\\x01\\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x02\\x00\\xb7\\x00:\\xff\\xff\\xff\\xff\\xff\\xff\\xff\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\xfe\\xff\\xff\\xff:/usr/bin/qemu-aarch64-static:F > /lib/binfmt.d/qemu-aarch64-static.conf'
sudo systemctl restart systemd-binfmt.service
sudo sed -i 's/^deb h/deb [arch=amd64,i386] h/g' /etc/apt/sources.list
sudo dpkg --add-architecture arm64
echo 'deb [arch=arm64] http://ports.ubuntu.com/ubuntu-ports bionic main restricted universe multiverse' | sudo tee -a /etc/apt/sources.list
echo 'deb [arch=arm64] http://ports.ubuntu.com/ubuntu-ports bionic-updates main restricted universe multiverse' | sudo tee -a /etc/apt/sources.list
echo 'deb [arch=arm64] http://ports.ubuntu.com/ubuntu-ports bionic-backports main restricted universe multiverse' | sudo tee -a /etc/apt/sources.list
sudo apt update
sudo apt install libssl-dev:arm64 linux-libc-dev:arm64 libc6-dev:arm64
```

## Install Visual Studio Code Open Enclave extension

At this moment the Open Enclave extension available on the marketplace is not in sync with the latest TrustBox firmware. For the latest version this guide refers to the extension distributed by Scalys.

1. download latest openenclave extension (msiot-vscode-openenclave-latest.vsix) from Scalys OpenEnclave [resources](https://scalys.com/downloads/)
2. run VS Code
3. install extension from the downloaded vsix file (Extensions -> Install from VSIX...)
4. download Open Enclave devkit (ms-iot-msiot-vscode-openenclave.tar.gz) from Scalys OpenEnclave [resources](https://scalys.com/downloads/).
5. install the devkit into VS Code location:
```
tar xf ms-iot.msiot-vscode-openenclave-latest.tar.gz -C $HOME/.config/Code/User/globalStorage/
```

# Example build

## Fibonacci example

The following Open Enclave example demonstrates the basic trusted application operation. It executes the trusted application which calculates a certain amount of numbers from Fibonacci sequence.

1. download the example sources:
```
git clone https://github.com/Scalys/EnclaveFibonacci
```
2. open in VS Code the downloaded directory and run "Tasks: Run Build Task" to build the project
3. build products are placed in `<project>/bld/ls-ls1012grapeboard/out/bin`. They should be installed on the TrustBox system by following paths:
```
5d286b7e-ff68-4b4b-b7b8-05f55dbfd0c7.ta -> /usr/lib/optee_armtz/
EnclaveFibonacci -> /root
```
4. boot the TrustBox and run the example:
```
tee-supplicant -d
./EnclaveFibonacci 5d286b7e-ff68-4b4b-b7b8-05f55dbfd0c7
```

## Debug with QEMU

To run debug of the trusted application in AArch64 emulation via qemu in docker:

1. open the project
2. press F1 and run "Tasks: Run Task" -> "Build for QEMU (AArch64/ARMv8-A)"
3. select the debug window and ensure that in the drop-down list of GDB profiles selected "(gdb) Launch QEMU (AArch64/ARMv8-A)"
4. run the example by pressing F5
5. switch to Tab "Terminal" and login into buildroot qemu environment with login "root"
6. within qemu run
```
/mnt/host/bin/EnclaveFibonacci 5d286b7e-ff68-4b4b-b7b8-05f55dbfd0c7
```

## New project

To create a new clean Open Enclave project:

1. press F1 and run "New Open Enclave Solution"
2. press F1 and run "Configure Default Build Task"
3. select "Build for Grapeboard (LS1012)
4. build project with Ctrl+Shift+B

