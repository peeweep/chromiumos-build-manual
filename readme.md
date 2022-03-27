# Intro

This article will describe how to compile Chromium OS.

In this article, `(outside)` mean you need run command outside the `cros_sdk`, `(inside)` mean that you need run command in `cros_sdk`.

## 0: Prepare

1. Use Linux:
    Follow the Chromium OS compilation guide, preferably Ubuntu 18.04, my choice is 18.04.6 LTS
2. Connect to network:
    Make sure your network can access Google, I have configured a proxy client on the router, and the entire subnet under the router can access the Internet smoothly.

## 1: Get the Chromium OS code and start building

### Install dependencies

``` shell
# (outside)
sudo apt-get install git gitk git-gui curl lvm2 xz-utils python3-pkg-resources python3-virtualenv python3-oauth2client python3.6 vim
```

Use `python3 --version` to confirm that the version of python3 is greater than 3.6. If it is less than 3.6, please refer to the following commands to switch to the version of python3.6 (and above).

``` shell
# (outside)
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.5 1
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.6 2
sudo update-alternatives --config python3
```

### Configure git

``` shell
# (outside)
git config --global user.name your_name
git config --global user.email your_email
```

### Configure Google API keys

Many features of chromiumOS require the use of api keys. How to apply can follow the [Acquiring Keys](https://www.chromium.org/developers/how-tos/api-keys/) chapter.

Write key to `~/.googleapikeys` like this:
```
google_api_key = "your_api_key"
google_default_client_id = "your_client_id"
google_default_client_secret = "your_client_secret"
```

### Install depot_tools

``` shell
# (outside)
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```
add `export PATH=/path/to/depot_tools:$PATH` to `~/.bashrc`


### Start cros_sdk

* This step will take you around 3-5 hours

1. Enter chromiumos workdir

``` shell
# (outside)
source ~/.bashrc # load PATH env
mkdir -p ~/chromiumos # create chromiumos work directory
cd ~/chromiumos # enter chromiumos dir
```

2. `repo init` with [release-R96-14268.B](https://chromium.googlesource.com/chromiumos/manifest.git/+/refs/heads/release-R96-14268.B) branch

``` shell
# (outside)
repo init -u https://chromium.googlesource.com/chromiumos/manifest.git --repo-url https://chromium.googlesource.com/external/repo.git -b release-R96-14268.B # repo init with release-R96-14268.B branch
repo sync -j4 # repo sync first, better with j4
repo sync -j16  # sync again
cros_sdk # This is the same command used to create the chroot, but if the chroot already exists, it will just enter.
```

`repo sync` will download about 20G of code.

`cros_sdk` will use Google's precompiled sdk, you can also use `cros_sdk --bootstrap` to build it from source.

### Build all package


1. Setup `BOARD`, use `amd64-generic` here

``` shell
# (inside)
export BOARD=amd64-generic
setup_board --board=${BOARD}
```

2. Build all packages
``` shell
# (inside)
./build_packages --board=${BOARD}
```

* This step will also take you about 3-5 hours, most of the time is spent on downloading chromium-src, you can choose not to upgrade chrome-icu and chromeos-chrome.

<details>
  <summary>Click show details</summary>

Check the installed version
``` shell
# (inside)
emerge-${BOARD} --ask chromeos-base/chrome-icu
```

```
Calculating dependencies... done!
[ebuild     U  ] chromeos-base/chrome-icu-96.0.4664.204_rc-r1 [96.0.4657.0_rc-r1] to /build/amd64-generic/
```

``` shell
# (inside)
sudo su
echo ">=chromeos-base/chrome-icu-96.0.4657.0_rc-r1"  >> /build/amd64-generic/etc/portage/package.mask
echo ">=chromeos-base/chromeos-chrome-96.0.4657.0_rc-r1"  >> /build/amd64-generic/etc/portage/package.mask
exit
./build_packages --board=${BOARD}
```
</details>

3. Build test image

``` shell
# (inside)
./build_image --board=${BOARD} --noenable_rootfs_verification test
```

4. Install kvm and check kvm is ok.

``` shell
# (outside)
sudo apt-get install qemu-kvm
sudo kvm-ok
```

5. Convert test image to vm image

``` shell
# (inside)
./image_to_vm.sh --board=${BOARD} --test_image
```

6. Start vm

``` shell
# (inside)
cros_vm --start --image-path=../build/images/${BOARD}/latest/chromiumos_qemu_image.bin --board=${BOARD}
```

connect by ssh, test image's password is `test0000`: 
``` shell
# (outside)
ssh root@localhost -p 9222
```
 
connect by vnc:
``` shell
# (outside)
sudo apt-get install tigervnc-viewer
vncviewer localhost:5900
```

You can use `cros_vm --stop --board=${BOARD}` to stop vm.

## 2: Kernel replacement

### Check the current kernel

``` shell
# (inside)
cros_vm --start --image-path=../build/images/${BOARD}/latest/chromiumos_qemu_image.bin --board=${BOARD}
```

``` shell
# (outside)
ssh root@localhost -p 9222
```

```
localhost ~ # uname -a
Linux localhost 4.14.273-18374-gf939ed477069 #1 SMP PREEMPT Sat Mar 26 22:48:59 CST 2022 x86_64 Intel Xeon E312xx (Sandy Bridge) AuthenticAMD GNU/Linuxlocalhost ~ # uname -r
4.14.273-18374-gf939ed477069
localhost ~ # exit
```

``` shell
# (inside)
cros_vm --stop --board=${BOARD}
```

### Way 1: change virtual/linux-sources USE

`kernel-5_10` and `kernel-4_14` are in `virtual/linux-sources`'s USE, you can install 5.10 kernel by `virtual/linux-sources`.

``` shell
# (inside)
emerge-${BOARD} --unmerge chromeos-kernel-4_14
USE="kernel-5_10 -kernel-4_14" emerge-${BOARD} virtual/linux-sources
```

This method can only take effect temporarily, because the USE of 4.14kernel is used in the base profile, it will take effect on `linux-sources`, and `./build_packages --board=${BOARD}` will still switch back to the 4.14 kernel when rerun it, if you want to make it permanent, we can use method 2.

### Way 2: change global USE

There is such a log when `./build_packages --board=${BOARD}`

```
02:05:44.250: INFO: Setting up portage in the sysroot.
02:05:44.879: INFO: Selecting profile: /mnt/host/source/src/overlays/overlay-amd64-generic/profiles/base for /build/amd64-generic
02:05:44.963: INFO: Updating toolchain.
```

We can see that the 4.14 kernel is defined from `/mnt/host/source/src/overlays/overlay-amd64-generic/profiles/base/make.defaults`

`USE="${USE} legacy_keyboard legacy_power_button sse kernel-4_14"`

We can change the USE in `make.conf` to override this USE

``` shell
# (inside)
vim /build/amd64-generic/etc/make.conf
```

``` diff
-USE=""
+USE="kernel-5_10 -kernel-4_14"
```

Rebuild package

``` shell
# (inside)
./build_packages --board=${BOARD}
./build_image --board=${BOARD} --noenable_rootfs_verification test
```

### Way 2: use cros_workon

`cros_workon` cloud add single package to BOARD

``` shell
# (inside)
emerge-amd64-generic --unmerge chromeos-kernel-4_14
cros_workon --board=amd64-generic start sys-kernel/chromeos-kernel-5_10
cros_workon --board=amd64-generic --install chromeos-kernel-5_10
emerage-amd64-generic sys-kernel/chromeos-kernel-5_10
./build_packages --board=${BOARD}
./build_image --board=${BOARD} --noenable_rootfs_verification test
```
