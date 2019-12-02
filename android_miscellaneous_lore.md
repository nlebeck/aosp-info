# Miscellaneous Lore

This page mentions a few random details about AOSP that I discovered the hard way. Maybe they will save you some time and heartache.

 - The Android ActivityManager only allows 18 apps to run concurrently by default. If another app starts up, the ActivityManager will start killing apps, even if there's plenty of available memory. The relevant source files are in the `frameworks/base` [repo][frameworks-base-repo], in `services/core/java/com/android/server/am`. On line 21022 of `ActivityManagerService.java`, there is a call to kill an app if the number of cached apps exceeds the value of `cachedProcessLimit`. The ActivityManager calculates that value by starting with `ProcessList.MAX_CACHED_APPS` on line 150 of `ProcessList.java`. The value of 32 for `ProcessList.MAX_CACHED_APPS` means that `cachedProcessLimit` ends up being 16. If you want to increase the number of apps that can run concurrently, you can increase the value of `ProcessList.MAX_CACHED_APPS`.

- When you flash an AOSP build onto a device, the `userdata` partition has a hardcoded size, rather than letting you use all of the available disk space like you might expect. That hardcoded size is 10GB by default for Android 7.1 running on a Pixel XL. The size is dictated by the constant `BOARD_USERDATAIMAGE_PARTITION_SIZE` on line 89 of `marlin/BoardConfig.mk` in the `device/google/marlin` [repo][marlin-repo]. Increase that value if you're flashing AOSP onto a device with a larger hard disk and want access to more of its space. (Credit goes to [Ali Razeen][ali-homepage] for finding this solution.)

[frameworks-base-repo]: https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-7.1.1_r57

[marlin-repo]: https://android.googlesource.com/device/google/marlin/+/refs/tags/android-7.1.1_r57/

[ali-homepage]: https://www.cs.ubc.ca/people/ali-razeen
