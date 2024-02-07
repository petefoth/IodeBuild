# IodeBuild

Docker microservice for building [IodéOS](https://iode.tech/) ROMs

This project is a fork of [LineageOS for microG's](https://lineage.microg.org/) [`docker-lineage-cicd` project](https://github.com/lineageos4microg/docker-lineage-cicd/). If you are not familiar with that project, please start by reading their [`README.md` documentation](https://github.com/lineageos4microg/docker-lineage-cicd/blob/master/README.md). In particular, read the sections giving some background on Docker
- [Why Docker?](https://github.com/lineageos4microg/docker-lineage-cicd/blob/master/README.md#why-docker)
- [How do I install Docker?](https://github.com/lineageos4microg/docker-lineage-cicd/blob/master/README.md#how-do-i-install-docker)

## What does Docker build

The main build artefact is the main ROM `.zip` file (e.g `iode-4.9-20240116-<device-name>.zip`). This file can be flashed from recovery as described below

The other build artefacts are dependent on settings passed to `docker` in environment variables:
1. By default, a custom recovery image (e.g. `iode-4.9-20240116-<device-name>-recovery.img`, and any other images needed for installing the ROM over the devices stock ROM.
2. If the `ZIP_UP_IMAGES` variable is set `true`, all the images mentioned in 1. are compressed into a single `-images.zip` file (e.g. `iode-4.9-20240116-<device-name>images.zip` )
3. If the `MAKE_IMG_ZIP_FILE` variable is set `true`, then a flashable `-img.zip` file will be built. This file contains ***all*** the images needed to flash the ROM using `fastboot update...` or `fastboot flash`.

All build artefacts are copied or moved to the `ZIP_DIR`directory.

## How do I use this project to build IodéOS?

Before you start, make sure you have the latest version of the Docker image:
```
docker pull petefothl4m/iodebuild
```
The requirements for building IodéOS are roughly the same as for [building LineageOS](https://wiki.lineageos.org/devices/sunfish/build):
- A relatively recent x86_64 computer:
  - Linux, macOS, or Windows - these instructions are only tested using LInux Mint and Ubuntu 20.04 LTS, so we recommend going with one of those.
  - A reasonable amount of RAM (16 GB to build up to lineage-17.1, 32 GB or more for lineage-18.1 and up). The less RAM you have, the longer the build will take. Enabling ZRAM can be helpful. If builds fail because of lack of memory, you can sometimes get over the problem by increasing the amount of swap, but this will be at the expense of slower build times.
  - A reasonable amount of Storage (~300 GB for IodéOS v4). You might require more free space for enabling ccache,or building for multiple devices, . Using SSDs results in considerably faster build times than traditional hard drives.
- A decent internet connection and reliable electricity. :)
- Some familiarity with basic Android operation and terminology. It may be useful to know some basic command line concepts such as cd, which stands for “change directory”, the concept of directory hierarchies, and that in Linux they are separated by /, etc.

### Using Settings and enviroment variables to configure your build
This Docker image contains a great number of settings, to allow you to fully
customize your build. They are all listed in the [Dockerfile][dockerfile], with their default values. Look at the the [Examples](#examples) to see how to set different values using `-e ...` clauses in your `docker run...` command.

The most important settings are
- `BRANCH_NAME` default `v4-staging`: which IodéOS branch to build. See [here][iodeos-branches] for the list of currently supported branches
- `DEVICE_LIST`: comma-separated list of devices to build

#### Volumes

You also have to provide Docker some volumes, where it'll store the source, the
resulting builds, the cache and so on. The volume names used internally by the docker engine volumes are:

- `/srv/src`, for the LineageOS sources
- `/srv/zips`, for the output builds
- `/srv/logs`, for the output logs
- `/srv/ccache`, for the ccache
- `/srv/local_manifests`, for custom manifests (optional)
- `/srv/userscripts`, for the user scripts (optional)
- `/srv/keys`, for the signing keys if `SIGN_BUILDS` is `true`
- `/srv/tmp`, for temporary files if `BUILD_OVERLAY` is `true`

They are mapped to actual directories on your computer `-v ...` clauses in your `docker run...` command.

#### Settings to control 'switchable' build steps

Some of the the steps in the build process (e.g `repo sync`, `mka`) can take a long time to complete. When working on a build, it may be desirable to skip some of the steps. The following environment variables (and their default values) control whether or not each step is performed
```
# variables to control whether or not tasks are implemented
ENV INIT_MIRROR true
ENV SYNC_MIRROR true
ENV RESET_VENDOR_UNDO_PATCHES true
ENV CALL_REPO_INIT true
ENV CALL_REPO_SYNC true
ENV CALL_GIT_LFS_PULL false
ENV APPLY_PATCHES true
ENV PREPARE_BUILD_ENVIRONMENT true
ENV CALL_BREAKFAST true
ENV CALL_MKA true
ENV ZIP_UP_IMAGES false
ENV MAKE_IMG_ZIP_FILE false
```

To 'switch' an operation, change the default value of the the variable in a `-e clause` in the `docker run` command e.g.
` -e "CALL_REPO-SYNC=false" \`

The `ZIP_UP_IMAGES` and `MAKE_IMG_ZIP_FILE` variables control how the `.img` files created by the buid are handled:
- by default, the `img` files are copied - unzipped - to the `zips` directory
- if `ZIP_UP_IMAGES` is set `true`, the images are zipped and the resulting `...images.zip` is copied to the `zips` directory
- if `MAKE_IMG_ZIP_FILE` is set `true`, a flashsable `...-img.zip` file is created, which can be installed using `fastboot flash` or `fastboot update`

#### Other settings

Other useful settings are:
- `CCACHE_SIZE (50G)`: change this if you want to give more (or less) space to
ccache
- `BUILD_TYPE (userdebug)`: type of your builds, see [Android docs](https://source.android.com/docs/setup/build/building#choose-a-target)
- `BUILD_OVERLAY (false)`: normally each build is done on the source tree, then
the tree is cleaned with `mka clean`. If you want to be sure that each build
is isolated from the others, set `BUILD_OVERLAY` to `true` (longer build
time). Requires `--cap-add=SYS_ADMIN`.
- `CRONTAB_TIME (now)`: instead of building immediately and exit, build at the
specified time (uses standard cron format)
- `ZIP_SUBDIR (true)`: Move the resulting zips to $ZIP_DIR/$codename instead of $ZIP_DIR/
- `PARALLEL_JOBS`: Limit the number of parallel jobs to run (`-j` for `repo sync` and `mka`).
By default, the build system should match the number of parallel jobs to the number of cpu
cores on your machine. Reducing this number can help keeping it responsive for other tasks.
- `RETRY_FETCHES`: Set the number of retries for the fetch during `repo sync`. By default, this value is unset (default `repo sync` retry behavior). Positive values greater than 0 are allowed.

The full list of settings, including the less interesting ones not mentioned in this guide, can be found in the [Dockerfile][dockerfile].

#### Settings that don't do very much building IodéOS
The following settings are present in the Dockerfile, but they are not very useful when building IodéOS
- `LOCAL_MIRROR (false)`: I'm it sure how useful it is to create a mirror of the LineageOS source when building IodéOS. Setting this `true` will do it though
- `RELEASE_TYPE (UNOFFICIAL)`: change the release type of your builds. This doesn't add the type text to the name of the main ROM `.zip` file, though it does to the other build artefacts: this is probably a bug ;)

The following settings, present in `lineageos4microg/docker-lineage-cicd/` have been removed, as they are not releveant when building IodéOS
- `WITH_GMS`: IodéOS already includes the microG components
- `SIGNATURE_SPOOFING`: IodéOS allows 'unrestricted' signature spoofing, equivalent to `SIGNATURE_SPOOFING=true` in `lineageos4microg/docker-lineage-cicd/`.

#### Signing

By default, builds are signed with the Android test keys. If you want to sign
your builds with your own keys (**highly recommended**):

 - `SIGN_BUILDS (false)`: set to `true` to sign the builds with the keys
    contained in `/srv/keys`; if no keys are present, a new set will be generated

### Further information

See the [`lineageos4microg/docker-lineage-cicd` documentation](https://github.com/lineageos4microg/docker-lineage-cicd/blob/master/README.md) for further information on
- [Additional custom apps](https://github.com/lineageos4microg/docker-lineage-cicd/blob/master/README.md#additional-custom-apps)
- [Proprietary files](https://github.com/lineageos4microg/docker-lineage-cicd/blob/master/README.md#proprietary-files)
- [Over the Air updates](https://github.com/lineageos4microg/docker-lineage-cicd/blob/master/README.md#over-the-air-updates)

### Examples

#### Build for a device officially supported by LineageOS
(In this example, Google Pixel 4a `sunfish`).
- Proprietary files are available from `TheMuppets`, and device and kernel sources are in the LineageOS github repos, so no need for a manifest file.
- Build signed with my own keys which I have placed in `/home/pete/srv/keys`. If there are no keys in `/home/user/keys`, a new set will be generated in that directory before starting the build,  (and will be used for every subsequent build).
- `docker run` command
```sh
docker run \
  -v "/home/pete/srv/iodeOS4:/srv/src" \
  -v "/home/pete/srv/iodeOS4/tmp:/srv/tmp" \
  -v "/home/pete/srv/iodeOS4/ccache:/srv/ccache" \
  -v "/home/pete/srv/iodeOS4/zips:/srv/zips" \
  -v "/home/pete/srv/iodeOS4/logs:/srv/logs" \
  -v "/home/pete/srv/keys:/srv/keys" \
  -v "/home/pete/srv/iodeOS4/local_manifests:/srv/local_manifests" \
  -v "/home/pete/srv/iodeOS4/userscripts:/srv/userscripts" \
  -e "BRANCH_NAME=v4-staging" \
  -e "RELEASE_TYPE=UNOFFICIAL" \
  -e "SIGN_BUILDS=true" \
  -e "INCLUDE_PROPRIETARY=false" \
  -e "CLEAN_AFTER_BUILD=false" \
  -e "DEVICE_LIST=sunfish" \
  -e "CALL_GIT_LFS_PULL=true" \
  -e "ZIP_UP_IMAGES=true" \
  -e "MAKE_IMG_ZIP_FILE=true" \
  petefothl4m/iodebuild
```
#### Build for a device not officially supported by LineageOS
(In this example, Sony Xperia XZ1 Compact `lilac`)
- Signed with test keys
- Needs a manifest (a file with the extension `.xml`) in `/srv/local_manifests`, specifying where to find device and kernel sources, and proprietary blobs.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
      <!-- kernel sources-->
      <project name="whatawurst/android_kernel_sony_msm8998" path="kernel/sony/msm8998" remote="github" revision="lineage-20" />

      <!-- device sources -->
      <project name="whatawurst/android_device_sony_yoshino-common" path="device/sony/yoshino-common" remote="github" revision="lineage-20" />
      <project name="whatawurst/android_device_sony_lilac" path="device/sony/lilac" remote="github" revision="lineage-20" />

      <!-- proprietary blobs for lilac -->
      <project name="whatawurst/android_vendor_sony_lilac" path="vendor/sony/lilac" remote="github" revision="lineage-20" />
      <project name="whatawurst/android_vendor_sony_yoshino-common" path="vendor/sony/yoshino-common" remote="github" revision="lineage-20" />
</manifest>
```
- `docker run` command
```sh
docker run \
  -v "/home/pete/srv/iodeOS4:/srv/src" \
  -v "/home/pete/srv/iodeOS4/tmp:/srv/tmp" \
  -v "/home/pete/srv/iodeOS4/ccache:/srv/ccache" \
  -v "/home/pete/srv/iodeOS4/zips:/srv/zips" \
  -v "/home/pete/srv/iodeOS4/logs:/srv/logs" \
  -v "/home/pete/srv/keys:/srv/keys" \
  -v "/home/pete/srv/iodeOS4/local_manifests:/srv/local_manifests" \
  -v "/home/pete/srv/iodeOS4/userscripts:/srv/userscripts" \
  -e "BRANCH_NAME=v4-staging" \
  -e "SIGN_BUILDS=false" \
  -e "CLEAN_AFTER_BUILD=false" \
  -e "DEVICE_LIST=lilac" \
  -e "ZIP_UP_IMAGES=true" \
  -e "MAKE_IMG_ZIP_FILE=true" \
  petefothl4m/iodebuild
```

## How do I install the IodéOS for MicroG ROM

Follow the LineageOS installation instructions for your device, which can be accessed from the [LineageOS Devices wiki pages](https://wiki.lineageos.org/devices/). If the LineageOS installation instructions require or refer to any `.img` files, these images can be obtained by unzipping the `-images.zip` or `-img.zip` file mentioned [above](https://github.com/petefoth/IodeBuild/tree/doc-changes-2402#what-does-docker-build) in the previous section.

### 'Clean' and 'dirty' flashing

A 'clean' flash is when the data partition is wiped and/or formatted before the ROM is installed. This will remove all user-installed apps and data. It is sometimes referred to as a 'fresh installation'.

A 'dirty flash' is when the data partition ***is not*** wiped and/or formatted before the ROM is installed. Normally this will result in all user-installed apps and data still being present after the installation.

Newer versions of the LineageOS for MicroG ROM can usually be 'dirty flashed' over older versions ***with the same Android version***.

Dirty flashing is ***sometimes*** possible over
- older versions of the LineageOS for MicroG ROM ***with an earlier** Android version***;
- the official LineageOS ROM (without microG)

In both these cases, problems may be encountered with app permissions, both for user-installed apps and for the pre-installed apps. These problems can sometimes be fixed by manually changing the app permissions.

If you are 'dirty' flashing, it is a good idea to backup your user-installed apps and data in case the 'dirty' flash fails.

## Troubleshooting and support

To be written

[iodeos-branches]: https://github.com/LineageOS/android/branches
[signature-spoofing]: https://github.com/microg/GmsCore/wiki/Signature-Spoofing
[microg]: https://microg.org/
[blobs-pull]: https://wiki.lineageos.org/devices/bacon/build#extract-proprietary-blobs
[blobs-extract]: https://wiki.lineageos.org/extracting_blobs_from_zips.html
[blobs-themuppets]: https://github.com/TheMuppets/manifests
[blobs-the-muppets]: https://gitlab.com/the-muppets/manifest
[dockerfile]: Dockerfile
[iodeos-branches]: https://gitlab.com/iode/os/public/manifests/android/-/branches
