# Compiling and Deploying AOSP

The Android Open Source Project (AOSP) [website][setup] describes how to download and compile the Android source code, but its instructions are a bit outdated in places and have a lot of extra information, so here is a walkthrough that aims to focus on the important parts and provide clarification.

These instructions specifically describe how to compile AOSP version `android-7.1.1_r57` and run it on a Pixel XL, but hopefully they are useful even if you're targeting a different version or device.

## Compiling AOSP

1.  Find a computer running Ubuntu with well over 100GB of free hard drive space, at least 16GB of RAM, and as many CPU cores as you can get your hands on. In my experience, AOSP compilation can be a bit finicky and break on the wrong version of Ubuntu---as of September 2018, I believe Ubuntu 16.04 could compile AOSP 7.1.1, but Ubuntu 18.04 could not.

2.  Install OpenJDK 8 (`apt-get install openjdk-8-jdk`).

3.  Install the big list of packages on the [Establishing a Build Environment][establishing-environment] page under “Installing required packages (Ubuntu 14.04).”

4.  Install the `repo` tool according to the instructions on the [Downloading the Source][downloading] page.

5.  Create a working directory in which to download the AOSP source code.

6.  Change directories to your working directory and run `repo init`, using the `-b` argument to specify the tag of your chosen version, as described on the [Downloading the Source][downloading] page:

		repo init -u https://android.googlesource.com/platform/manifest -b android-7.1.1_r57

7.  Run `repo sync` to download the source code. Like the website says, this command can take up to an hour to complete.

8.  (I'm not sure if this step is necessary, and I'm also not sure if these instructions are correct.) Download the proprietary binaries for your device. [This page][drivers] has a big list of binaries for Google-made devices. To figure out exactly which binaries to download, look at [this list][source-tags] of build names and version tags, match your chosen version tag to its build name, and find the latest build name in the binaries list that is before your build. Since we're running `android-7.1.1_r57` on a Pixel XL, I would first see that `android-7.1.1_r57` corresponds to build `NGI77B`, and then I would note that the latest build before `NGI77B` in the binaries list for the Pixel XL is `NOF27D`.

9.  Extract the proprietary binaries into your downloaded source tree according to the instructions at the bottom of the [Downloading the Source][downloading] page.

10.  Run `source build/envsetup.sh` to load various environment variables required by the build process into your shell session.

11.  Run the `lunch` command to choose which device and debug level to target with the build. I’m using a Pixel XL, which has the codename “marlin,” so I run `lunch aosp_marlin-userdebug`.

12.  Run `make -jN` to compile the code! Like the “Preparing to Build” page says, you’ll want to use a number N that is between 1 and 2 times the number of hardware threads on your machine.

## Deploying your compiled AOSP build

The [Flashing Devices][flashing-devices] page of the AOSP documentation describes the basics of how to deploy a build, but it is a bit incomplete, in that it doesn't say what to do if you can't plug your Android device directly into your development machine.

You’ll first need to unlock the bootloader of your device as described on the [Flashing Devices][flashing-devices] page.

Once you’ve unlocked the bootloader of your device, if you happen to have compiled AOSP on a machine into which you can easily plug your Android device, then you can simply run the `adb reboot bootloader` and `fastboot flashall -w` commands mentioned on the [Flashing Devices][flashing-devices] page to flash your build onto your device.

### Manually flashing the image

 If you compiled AOSP on a machine to which you don't have direct physical access, you’ll need to do a little bit more work to get your build onto your device, described below.

The ultimate product of an Android build is a bunch of `.img` files located in the `out/target/product/<codename>` directory in the source tree. Each `.img` file corresponds to a named system partition, and you use the `fastboot flash` command to flash a file onto its partition on the device. One way to figure out the partition name corresponding to each `.img` file is by downloading a factory image for your device [here][factory-images], running its `flash-all.sh` script, and seeing what partition names that script uses.

The specific number of `.img` files and their names seem to differ slightly across devices.  For the Pixel XL, there are files named `boot.img`, `ramdisk.img`, `ramdisk-recovery.img`, `system.img`, `system_other.img`, `userdata.img`, and `vendor.img`. Of these files, the ones you need to flash onto your Pixel XL when deploying a new build are `boot.img`, `system.img`, `system_other.img`, and `userdata.img`.

Flashing `userdata.img` will wipe out the installed apps and data on the phone, and from what I can tell, it’s not necessary to flash it if you are flashing a new build compiled from source onto a device that is already running a build compiled from source---you’ll just want to flash it if you are flashing a build compiled from source onto a device running a factory image (or vice versa). As far as I can tell, there’s no need to do anything with `ramdisk.img`, `ramdisk-recovery.img`, and `vendor.img` when flashing a new build.

One more complication is that the Pixel XL has two different "boot slots" named A and B. My understanding is that these multiple slots are used by Android's [over-the-air updates][ota-updates] to install an updated system image while leaving a functional copy of the current image around, in case the update process runs into any errors. Each boot slot has its own set of named partitions, and each device has a "preferred boot slot" at any one time. The partition names corresponding to some of the `.img` files differ depending on the preferred boot slot (e.g., `system.img` now corresponds to either `system_a` or `system_b` instead of just `system`). If you run `fastboot flash` with a partition name that does not include the boot slot, fastboot will automatically use the corresponding partition on the preferred boot slot. You can view the preferred boot slot of a specific Pixel XL by booting it into bootloader mode and looking at the status information it prints on-screen.

Here are the steps to manually flash the `.img` files onto your Pixel XL:

1. Find a local computer that you can directly plug your Android device into. Make sure it has the Android SDK installed.

2.  Copy the `.img` files from your development machine (in the `out/target/product/marlin` subdirectory of the source tree) onto your local computer. Do all subsequent steps on the local computer.

3.  Reboot your phone into bootloader mode by running `adb reboot bootloader`. If your device isn’t connected to ADB, there should be a button combination that you can hold down immediately after powering on the device to boot into bootloader mode. On the Pixel XL, hold down the power and volume down buttons.

4.  Run `fastboot flash <partition-name> <img-file>` for each `.img` file to flash it onto your device. Here are the fastboot commands, including partition names, for the Pixel XL, assuming the preferred boot slot is slot A (note that when you use partition name `boot`, fastboot will translate it to `boot_a` automatically):

		fastboot flash boot boot.img
		fastboot flash system_a system.img
		fastboot flash system_b system_other.img
		fastboot flash userdata userdata.img
    
    If the preferred boot slot on your device is slot B, you should instead flash `system.img` onto `system_b` and `system_other.img` onto `system_a`.

5.  Run `fastboot reboot` to reboot your device into your newly flashed build.

## Restoring a factory image

If something goes wrong while flashing your build, or if you flash a build that crashes during boot, you can just boot your device back into bootloader mode and flash a different build on. If you don’t have a working build handy, you can use a [factory image][factory-images] for your device. The factory images come with a `flash-all.sh` script that automates flashing the different `.img` files.

[setup]: https://source.android.com/setup
[establishing-environment]: https://source.android.com/setup/build/initializing
[downloading]: https://source.android.com/setup/build/downloading
[drivers]: https://developers.google.com/android/drivers
[source-tags]: https://source.android.com/setup/start/build-numbers.html#source-code-tags-and-builds
[ota-updates]: https://source.android.com/devices/tech/ota/
[factory-images]: https://developers.google.com/android/images
[flashing-devices]: https://source.android.com/setup/build/running
