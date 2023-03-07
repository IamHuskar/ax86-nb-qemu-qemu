
[Android-x86 source ](https://www.android-x86.org/source.html)
```
mkdir android-x86
cd android-x86
repo init -u git://git.osdn.net/gitroot/android-x86/manifest -b q-x86
repo sync --no-tags --no-clone-bundle
```

```
export LC_ALL=C
cd path/to/android-x86/
source ./build/envsetup.sh
lunch android_x86-userdebug
cd path/to/android-x86/external
git clone https://github.com/goffioul/ax86-nb-qemu
cd ax86-nb-qemu
git submodule init
git submodule update
cd path/to/android-x86/external
mmma ax86-nb-qemu/
```

How to generate auto-generated files for libnb-qemu
```
cd qemu/
export ANDROID_NDK=/path/to/ndk/version
./configure \
	--target-list=arm-linux-user \
	--disable-system \
	--disable-slirp \
	--enable-trace-backends=nop \
	--disable-libpmem \
	--disable-nettle \
	--disable-gcrypt \
	--disable-curses \
	--disable-curses \
	--disable-bzip2 \
	--cpu=i686 \
	--with-coroutine=sigaltstack \
	--disable-malloc-trim \
	--disable-modules \
	--disable-plugins \
	--disable-libxml2 \
	--disable-werror \
	--cc=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/i686-linux-android21-clang \
	--cxx=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/i686-linux-android21-clang++ \
	--extra-ldflags=-latomic

make config-host.h qemu-version.h

make -C arm-linux-user config-target.h gdbstub-xml.c
make -C arm-linux-user target/arm/decode-vfp.inc.c target/arm/decode-vfp-uncond.inc.c \
	target/arm/decode-a32.inc.c target/arm/decode-a32-uncond.inc.c \
	target/arm/decode-t32.inc.c target/arm/decode-t16.inc.c
make -C arm-linux-user linux-user/arm/syscall_nr.h

sed -i -e 's,^#define CONFIG_ARM_A64_DIS,// &,' arm-linux-user/config-target.h
```


##  links
[ARM binary code translator](https://groups.google.com/g/android-x86/c/_3HoNJTi_Y0/)  
[Moving away from libhoudini](https://groups.google.com/g/android-x86/c/GZ3tQ0mUIdU/m/TnS9pIqWDgAJ)


It's a lot of DYI at the moment. There are 2 main repos, both based on Android Q:
- https://github.com/goffioul/ax86-nb-qemu for the native side, use it with your Android-x86 repo
- https://github.com/goffioul/ax86-nb-qemu-guest for the emulated side, use it with a stock AOSP and aosp_arm-eng lunch target

There's no integration with enable_nativebridge, and you shouldn't have this qemu bridge enabled at boot, as there's still an issue with that. The way I'm using it at the moment is to have a normal q-x86 build with (non-functional) houdini (from Android 9) deployed in read-write mode (note that in my build, I included /system/etc/houdini9_y.sfs and I preset persist.sys.nativebridge=1, so it's available on boot without download or user action). After boot, I disable houdini: "adb shell umount /system/lib/libhoudini.so; adb shell umount /system/lib/arm". Then I use the "copy-xxx" scripts that are in the above repos, to transfer the compiled bridge and runtime to the q-x86 install.

To compile the native part of the bridge (this assumes you already have an ADB connection to your device):
    mmma path/to/ax86-nb-qemu/libnb-qemu
    ./path/to/ax86-nb-qemu/libnb-qemu/copy-bridge.sh

To compile the emulated runtime (same assumption):
    apply patches from path/to/ax86-ng-qemu-guest/patches
    mmma path/to/ax86-nb-qemu-guest
    ./build/soong/soong_ui.bash --make-mode libnb-qemu-runtime
    ./path/to/ax86-nb-qemu-guest/copy-runtime.sh

The way it works is that I piggyback on a deployed (but non-functional) houdini bridge from SFS image, and I swap the libnb-qemu bridge by bind-mounting libnb-qemu.so to libhoudini.so