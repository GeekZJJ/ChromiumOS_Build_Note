

# 1. Building a specific branch of Chromium OS

## Setup environment on Ubuntu 20.04


### Resize rootfs of build host


``` shell
sudo lvextend -L +800G /dev/mapper/ubuntu--vg-ubuntu--lv # extend the rootfs partition to meet the minimal disk size requirement
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv # resize the rootfs filesystem
```

###  Install dependencies

I followed the official tutorial, but found that  `python-virtualenv`  `python-oauth2client ` is missing. Solution:

``` shell
sudo apt-get install git-core gitk git-gui curl lvm2 thin-provisioning-tools \
     python-pkg-resources python3-venv python3-oauth2client xz-utils
```

### Install depot_tools

``` shell
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
echo 'export PATH=$PATH:'`realpath depot_tools/` >> ~/.bashrc
```

### Setup git

``` shell
git config --global user.name "xxx"
git config --global user.email "xxx"
```

### Make sudo a little more permissive

``` shell
cd /tmp
cat > ./sudo_editor <<EOF
#!/bin/sh
echo Defaults \!tty_tickets > \$1          # Entering your password in one shell affects all shells
echo Defaults timestamp_timeout=180 >> \$1 # Time between re-requesting your password, in minutes
EOF
chmod +x ./sudo_editor
sudo EDITOR=./sudo_editor visudo -f /etc/sudoers.d/relax_requirements # put above script to /etc/sudoers.d to re-requesting sudo password interval
```

### Verify the default file permission setting

``` shell
umask 022
touch umask_test
ls -la umask_test
-rw-r--r-- 1 fydeos fydeos 0 May 27 23:25 umask_test
```

## Get the source

```shell
mkdir -p ~/chromiumos
cd ~/chromiumos
repo init -u https://chromium.googlesource.com/chromiumos/manifest.git -b release-R96-14268.B --repo-url https://chromium.googlesource.com/external/repo.git # get the src of release-R96-14268.B branch
repo sync -j4
```

### Fix the issue: stuck at some project during repo sync
The `repo sync` command getting stuck at `Fetching: 93% (216/232) weave/libweave`. I tried removing the following directory and running  `repo sync` again, but still no lucky.

``` shell
./repo/projects/src/weave/libweave.git
./repo/project-objects/weave/libweave.git
```

I Google and found a possible solution in  [this post](https://groups.google.com/g/android-building/c/glKFGMG_BJw), which saying that`sysctl -w net.ipv4.tcp_window_scaling=0`may help.


## Setup GoogleAPI token and Google Storage (GS) buckets

According to the [offical instuctions](https://www.chromium.org/developers/how-tos/api-keys/) and found the 8th step saying that `In the "Application type" section check the "Other" option and give it a name in the "Name" text box, then click "Create"` while the "Other" in OAuth Client ID in Google is missing.

Googled that and found [this post](https://stackoverflow.com/questions/61984699/how-to-create-application-type-other-in-oauth-client-id-in-google), saying that the `Other` option have been change to `Desktop App`.

Get the API key and OAuth Client ID, then fill into the `~/.googleapikeys`

```
GOOGLE_API_KEY=your_api_key
GOOGLE_DEFAULT_CLIENT_ID=your_client_id
GOOGLE_DEFAULT_CLIENT_SECRET=your_client_secret
```

``` shell
~/chromiumos/chromite/scripts/gsutil config # follow the interactions by this program
```

## Start building and run in qemu

### Create a chroot

``` shell
cd ~/chromiumos
(outside) cros_sdk # bootstrap or Enter the chroot
```

### Select a board and build test image

``` shell
(inside) export BOARD=amd64-generic # see all known boards in ~/trunk/src/overlays
(inside) setup_board --board=${BOARD} --force 
# sets up the board target with a default sysroot of /build/${BOARD}.
(inside) ./set_shared_user_password.sh # set the shared user account "chronos" password for ssh/cmdline with root access
# If building a test image, the password will be ignored and "test0000" will be the password instead. 
(inside) ./build_packages --board=${BOARD} # build packages that will install to the chromium os
(inside) ./build_image --board=${BOARD} --noenable_rootfs_verification test 
# build a flashable image base on previous built packages
(inside) ./image_to_vm.sh --board=${BOARD} --test # Building an image to run in qemu-kvm
```

- Note for the 2nd command: passing the `--force` flag to cleanup previous build before start a new one,  which is helpful to a new build if you ever build success before (If not do so, it always stuck at compiling  'chrome-icu' during compiling for several hours but no cpu load, however git is fetching/indexing  something.) I have waste a couple of hours without passing the flag and  seeing it stuck there but no idea what's happening.

### Run in qemu-kvm

After successful building or image converting success, there are some tips.

``` shell
13:20:33 INFO    : Test image created as chromiumos_test_image.bin
13:20:33 INFO    : To copy the image to a USB key, use:
13:20:33 INFO    :   cros flash usb:// ../build/images/amd64-generic/R96-14268.84.2022_05_28_1312-a1/chromiumos_test_image.bin
13:20:33 INFO    : To flash the image to a Chrome OS device, use:
13:20:33 INFO    :   cros flash YOUR_DEVICE_IP ../build/images/amd64-generic/R96-14268.84.2022_05_28_1312-a1/chromiumos_test_image.bin
13:20:33 INFO    : Note that the device must be accessible over the network.
13:20:33 INFO    : A base image will not work in this mode, but a test or dev image will.
13:20:33 INFO    : To run the image in a virtual machine, use:
13:20:33 INFO    :   cros_vm --start --image-path=../build/images/amd64-generic/R96-14268.84.2022_05_28_1312-a1/chromiumos_test_image.bin --board=amd64-generic


Creating final image
Created image at /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_28_1312-a1
You can start the image with:
cros_vm --start --board amd64-generic --image-path /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_28_1312-a1/chromiumos_qemu_image.bin
```

So I use `cros_vm --start --board amd64-generic --image-path /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_28_1312-a1/chromiumos_qemu_image.bin` to run on my local machine.

### Connect to the test image via SSH or VNC

After the `cros_vm --start` command finishes, it can be ssh into the image by `ssh root@localhost -p9222` and `vnc://localhost:5900` to access ssh/vnc.(cros_vm is a wrapper which start qemu-kvm and map 9222 and 5900 to localhost for ssh/vnc)

# 2. Update Chromium OS kernel to mainline

Take a look at the default kernel version in this branch.

``` shell
localhost ~ # uname -a
Linux localhost 4.14.275-18376-gcb9161426216 #1 SMP PREEMPT Sun May 28 20:02:53 UTC 2022 x86_64 Intel Xeon E312xx (Sandy Bridge) GenuineIntel GNU/Linux
```

### May be the wrong way, although it works

In order to update kernel from default v4.14 to v5.10, I found it define in the file `chromiumos/src/overlays/overlay-amd64-generic/profiles/base/make.defaults`. Make some modification as follows and rebuild image.

``` diff
$  repo diff
project src/overlays/
diff --git a/overlay-amd64-generic/profiles/base/make.defaults b/overlay-amd64-generic/profiles/base/make.defaults
index 15853a8045..0a1147f2ba 100644
--- a/overlay-amd64-generic/profiles/base/make.defaults
+++ b/overlay-amd64-generic/profiles/base/make.defaults
@@ -21,7 +21,7 @@ USE=""

 USE="${USE} containers kvm_host crosvm-gpu virtio_gpu"

-USE="${USE} legacy_keyboard legacy_power_button sse kernel-4_14"
+USE="${USE} legacy_keyboard legacy_power_button sse kernel-5_10"

 USE="${USE} direncryption"

@@ -81,3 +81,5 @@ USE="${USE} vtpm_proxy"

 # No support for zero-copy on virtual machines.
 USE="${USE} -video_capture_use_gpu_memory_buffer"
+
+USE="${USE} direncription_allow_v2"
```

The lastest diff line is add to solve build error after kernel replacement.

Wait the build success and run it in qemu.  SSH into it and check the kernel version

```shell
localhost ~ # uname -a
Linux localhost 5.10.109-12118-g6ab58e5b541c #1 SMP PREEMPT Sun May 29 00:41:01 UTC 2022 x86_64 Intel Xeon E312xx (Sandy Bridge) GenuineIntel GNU/Linux
```

### May be the elegant way, but failed to build

Follow these two  posts ([link1](https://groups.google.com/a/chromium.org/g/chromium-os-dev/c/32bkgRzXn1o) [link2](https://groups.google.com/a/chromium.org/g/chromium-os-dev/c/ph60qWH6Cso) ) and try to use the elegant way which works as the offical tutorial says `Keep the tree green`.


``` shell
(inside) cros_workon --board amd64-generic start chromeos-kernel-5_10 
03:52:54.318: INFO: Started working on 'sys-kernel/chromeos-kernel-5_10' for 'amd64-generic'
(inside) FEATURES="noclean" cros_workon_make --board=${BOARD} --install chromeos-kernel-5_10
(inside) emerge-amd64-generic  --unmerge chromeos-kernel-4_14
(inside) ./build_image --board=${BOARD} --noenable_rootfs_verification test
```
After the `build_image` command, there is a error. Still no idea to solve it.

``` shell
emerge: there are no binary packages to satisfy "sys-kernel/chromeos-kernel-4_14" for /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_29_0901-a1/rootfs/.
(dependency required by "virtual/linux-sources-1-r27::chromiumos" [binary])
(dependency required by "virtual/target-chromium-os-1-r185::chromiumos" [binary])
(dependency required by "virtual/target-os-1-r5::chromiumos" [binary])
(dependency required by "virtual/target-os" [argument])
Entering /mnt/host/source/src/scripts/mount_gpt_image.sh --unmount --from /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_29_0901-a1/chromiumos_base_image.bin --rootfs_mountpt /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_29_0901-a1/rootfs --stateful_mountpt /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_29_0901-a1/stateful --esp_mountpt /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_29_0901-a1/esp --delete_mountpts
09:01:34 INFO    : Unmounting image from /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_29_0901-a1/stateful and /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_29_0901-a1/rootfs
09:01:34 INFO    : Unmounting temporary rootfs /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_29_0901-a1/rootfs//build/rootfs.
Cleaning up /usr/local symlinks for /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_29_0901-a1/stateful/dev_image
09:01:34 INFO    : Running sync -f /mnt/host/source/src/build/images/amd64-generic/R96-14268.84.2022_05_29_0901-a1/chromiumos_base_image.bin
An error occurred in your build so your latest output directory is invalid.
Would you like to delete the output directory (y/N)? n
```


# 3. The devserver

``` shell
(inside) $ start_devserver
Traceback (most recent call last):
  File "/usr/lib/devserver/devserver.py", line 1314, in <module>
    main()
  File "/usr/lib/devserver/devserver.py", line 1269, in main
    os.makedirs(cache_dir)
  File "/usr/lib64/python3.6/os.py", line 210, in makedirs
    makedirs(head, mode, exist_ok)
  File "/usr/lib64/python3.6/os.py", line 220, in makedirs
    mkdir(name, mode)
PermissionError: [Errno 13] Permission denied: '/usr/lib64/devserver/static'

(inside) $ sudo start_devserver
[29/May/2022:02:47:30] ENGINE Listening for SIGTERM.
INFO:cherrypy.error:[29/May/2022:02:47:30] ENGINE Listening for SIGTERM.
[29/May/2022:02:47:30] ENGINE Listening for SIGHUP.
INFO:cherrypy.error:[29/May/2022:02:47:30] ENGINE Listening for SIGHUP.
[29/May/2022:02:47:30] ENGINE Listening for SIGUSR1.
INFO:cherrypy.error:[29/May/2022:02:47:30] ENGINE Listening for SIGUSR1.
[29/May/2022:02:47:30] ENGINE Bus STARTING
INFO:cherrypy.error:[29/May/2022:02:47:30] ENGINE Bus STARTING
[29/May/2022:02:47:30] ENGINE Serving on http://:::8080
INFO:cherrypy.error:[29/May/2022:02:47:30] ENGINE Serving on http://:::8080
[29/May/2022:02:47:30] ENGINE Bus STARTED
INFO:cherrypy.error:[29/May/2022:02:47:30] ENGINE Bus STARTED
```

Test the devserver.

``` shell
$  curl localhost:8080
Welcome to the Dev Server!<br>
<br>
Here are the available methods, click for documentation:<br>
<br>
<a href=doc/build>build</a><br>
<a href=doc/check_health>check_health</a><br>
<a href=doc/controlfiles>controlfiles</a><br>
<a href=doc/is_staged>is_staged</a><br>
<a href=doc/latestbuild>latestbuild</a><br>
<a href=doc/list_image_dir>list_image_dir</a><br>
<a href=doc/list_suite_controls>list_suite_controls</a><br>
<a href=doc/locate_file>locate_file</a><br>
<a href=doc/setup_telemetry>setup_telemetry</a><br>
<a href=doc/stage>stage</a><br>
<a href=doc/symbolicate_dump>symbolicate_dump</a><br>
<a href=doc/update>update</a><br>
<a href=doc/xbuddy>xbuddy</a><br>
<a href=doc/xbuddy_capacity>xbuddy_capacity</a><br>
<a href=doc/xbuddy_translate>xbuddy_translate</a>
```

# 4. Apply OTA update from the image

## Generate the OTA package and host on devserver

### Generate OTA package

I haven't figured out how to do this.

### Start devserver and specify the static_dir

``` shell
(inside) sudo start_devserver --port=80 --static_dir= ../build/images/amd64-generic/R96-14268.84.2022_05_29_0930-a1/
```

## Change the devserver address of target image 

Get local ipaddr with `ip addr` and that is  `192.168.77.60`

### The runtime method

ssh into the build out image, do as follows:

``` shell
mount -o rw,remount / # remount the / for /etc/lsb-release modification
vim /etc/lsb-release # replace the following lines 
# CHROMEOS_AUSERVER to http://192.168.77.60:8080/update
# CHROMEOS_DEVSERVER to http://192.168.77.60:8080
```

### The build time method

Before previous `(inside) setup_board --board=${BOARD} --force ` command, export the environment variable as follows:

``` shell
export CHROMEOS_VERSION_DEVSERVER="http://192.168.77.60" 
export CHROMEOS_VERSION_AUSERVER="http://192.168.77.60/update" 
```

## Apply update package from the old image

``` shell
update_engine_client --update
```

