# How_to_build_android_kernel_to_load_modules_dynamically
This a tutorial which shows the steps for: 
1: unlock bootloader, 
2: download kernel and cross-compiler sources
3: compile android kernel with "loadable module support", 
4: flash the boot.img, 
5: root the android,
6: compile your own kernel module and insert it


## == Unlock the Bootloader ==

Downlad **adb** and **fastboot** from your package manager or following [this guide](https://source.android.com/setup/build/running#building-fastboot-and-adb) .

    sudo apt-get install adb fastboot
    
Start the *adb daemon*, verify you are able to see the phone with *adb* and restart it in bootloader mode (fastboot mode).

    adb start-server
    adb devices
    adb reboot bootloader

Verify you are able to see the phone with **fastboot** and unlock the bootloader

    fastboot devices
    fastboot oem unlock
    (or fastboot flashing unlock for newest phones)

You can also follow [this official guide](https://source.android.com/setup/build/running#unlocking-the-bootloader)  to start manually in fastboot mode and to unlock the bootloader.

You can find others **fastboot commands** [here](https://forum.xda-developers.com/android/help/adb-fastboot-commands-bootloader-kernel-t3597181).




## == Download a Cross-Compiler ==

I first tried to use the latest stable NDK from [the official repo](https://developer.android.com/ndk/downloads/) .

(DO NOT DOWNLOAD IT !! It doesn't work)

    export PATH=$PATH:/home/user/ndk/android-ndk-r17b/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/
    export CROSS_COMPILE=arm-linux-androideabi-

But I had very strange warnings at compilation which blocked the final lincking.

So after a a few researches I found [here](https://forum.xda-developers.com/google-nexus-5/help/solved-stock-kernel-build-problems-t2532147) it was better to use the **arm-eabi-gcc** rather than the **arm-linux-androideabi-gcc**.

Therefore I downloaded the latest version of [arm-eabi-4.8](https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/arm/arm-eabi-4.8/+/master).

==> extract it to /home/user/ndk/arm-eabi-4.8

    export PATH=$PATH:/home/user/ndk/arm-eabi-4.8/bin
    export CROSS_COMPILE=arm-eabi-

Note: I can confirm the choice of **arm-eabi-4.8** because it is the cross-compiler defined in the kernel image itself in *build.config*:
```
    ARCH=arm
    BRANCH=android-msm-shamu-3.10
    CROSS_COMPILE=arm-eabi-
    DEFCONFIG=shamu_defconfig
    EXTRA_CMDS=''
    KERNEL_DIR=private/msm-moto
    LINUX_GCC_CROSS_COMPILE_PREBUILTS_BIN=prebuilts/gcc/linux-x86/arm/arm-eabi-4.8/bin
    FILES="
    arch/arm/boot/zImage-dtb
    vmlinux
    System.map
    "
```



## == Build Kernel ==

In order to insert our own kernel module into the Nexus 6 kernel, we need to have the kernel config option **loadable module support** activated.
If you already have rooted your phone, you can verify it taping, otherelse trust me:
```
  adb shell su -c "lsmod"
  or
  adb shell su -c "insmod /data/local/tmp/glitchmin.ko"
```
These function are not authorized by default in Android kernels. Therefore, we will flash a new kernel with the option *loadable module support* enabled.

The Nexus 6 (shamu) we received was configured as follow:
* Android: 6.0.1
* Kernel: 3.10.40 - g557ba38 Nov 4 2015 armv7l
* Security Update: Dec 2015
* Build: MMB29K
* Board Platform: msm 8084
* [Every other info here](https://www.androiddevice.info/submission/31783/show)

In according with [this list](https://source.android.com/setup/build/building-kernels#figuring-out-which-kernel-to-build), 
I downloaded [this official Google kernel android-msm-shamu-3.10-marshmallow](https://android.googlesource.com/kernel/msm/+/android-msm-shamu-3.10-marshmallow).  

==> extract it to /home/user/images_kernel/msm-android-msm-shamu-3.10-marshmallow
  export MYKERNEL=/home/user/images_kernel/msm-android-msm-shamu-3.10-marshmallow
  cd $MYKERNEL

You can also use the [CyanogenMod kernel android_kernel_moto_shamu] (https://github.com/CyanogenMod/android_kernel_moto_shamu)  (<-- Finally this one does not boot)




Then you can build your kernel following the next steps (inspired from [this guide](https://source.android.com/setup/build/building-kernels#building) or [this guide](http://marcin.jabrzyk.eu/posts/2014/05/building-and-booting-nexus-5-kernel) or [this one](https://source.android.com/setup/build/devices or [this last one](https://developer.sony.com/develop/open-devices/guides/kernel-compilation-guides/how-to-build-and-flash-a-linux-kernel-for-aosp-supported-devices#ManuallyLinux32)).

* Configure the Makefile .config:

In $MYKERNEL/arch/arm/configs/shamu_config, substitute

`# CONFIG_MODULES is not set`

by

`CONFIG_MODULES=y` 

Then
```
  cp arch/arm/configs/shamu_config .config
  make ARCH=arm SUBARCH=arm CROSS_COMPILE=arm-eabi- shamu_defconfig
```
* Verify if the option **Enable loadable module support** is well activated with:
  `make ARCH=arm SUBARCH=arm CROSS_COMPILE=arm-eabi- menuconfig`     
==> if not select *Enable loadable module support*, press **SPACE** on your keyboard and then select **Save** and **Exit**
    
* Finally compile the whole kernel (it takes a lot of time...):
  `make ARCH=arm SUBARCH=arm CROSS_COMPILE=arm-eabi- -j4`

(Note: -jN means it will use N processor in parallele)

You must obtain at the end something like :
```
  Kernel: arch/arm/boot/Image is ready
  DTC     arch/arm/boot/dts/apq8084-shamu-p2.dtb
  AS      arch/arm/boot/compressed/head.o
  GZIP    arch/arm/boot/compressed/piggy.gzip
  CC      arch/arm/boot/compressed/misc.o
  CC      arch/arm/boot/compressed/decompress.o
  CC      arch/arm/boot/compressed/string.o
  SHIPPED arch/arm/boot/compressed/hyp-stub.S
  SHIPPED arch/arm/boot/compressed/lib1funcs.S
  SHIPPED arch/arm/boot/compressed/ashldi3.S
  AS      arch/arm/boot/compressed/hyp-stub.o
  AS      arch/arm/boot/compressed/lib1funcs.o
  AS      arch/arm/boot/compressed/ashldi3.o
  AS      arch/arm/boot/compressed/piggy.gzip.o
  LD      arch/arm/boot/compressed/vmlinux
  OBJCOPY arch/arm/boot/zImage
  Kernel: arch/arm/boot/zImage is ready
  CAT     arch/arm/boot/zImage-dtb
  Kernel: arch/arm/boot/zImage-dtb is ready
```

You can find your **Image** , **zImage** and **zImage-dtb** in **$MYKERNEL/arch/arm/boot/**

You can find the device tree blob **.dtb** at **$MYKERNEL/arch/arm/boot/dts/*.dtb**

You have your kernel ready. On most embedded systems that will be the end of your work. Usually you'll copy the kernel to SD card or NFS location, and the board will boot. But on Android it's different. You need to prepare special boot partition which then you can boot using fastboot. 

More info about kernel and boot image on [this link](https://source.android.com/devices/bootloader/partitions-images#images).





## == Update the kernel in the boot image ==

So you need to start from downloading the Android image for your phone from Google sites. Go to Nexus Factory Images site and download the factory image that matches to Android version that's on your phone.


Download the **factory image** for the Nexus6 (shamu) Android 6.0.1 (MMB29K) [here](https://developers.google.com/android/images#shamu).

==> extract it to /home/user/factory/shamu-mmb29k

    export FACTORY=/home/user/factory/shamu-mmb29k
    cd $FACTORY
    mkdir unziped_img
    unzip ./image-shamu-mmb29k.zip -d unziped_img
    cd unziped_img

Or download the **boot.img** [from this link](https://desktop.firmware.mobi/device:16/firmware:694).


### === Solution 1 (Worked) ===

Inspired from [this guide](http://rex-shen.net/android-unpackpack-factory-images/.

    sudo apt-get install abootimg

* Unpack

Go into unziped factory image file or downloaded boot.img file,

    mkdir boot
    cd boot
    abootimg -x ../boot.img
    # after the successful execution of the last command, we will have initrd.img (= ramdisk), zImage (= kernel) and bootimg.cfg (= kernel config)
    # inside the boot folder.

If you want to inspect the root file system **rootfs** from the **ramdisk**:

    file initrd.img
    # the output should be similar to the following line:
    # initrd.img: gzip compressed data, from Unix

    mkdir ramdisk
    cd ramdisk
    # "gunzip -c" means unpack to standard output, and "cpio -i" convert standard output in files
    gunzip -c ../initrd.img|cpio -i

* Repack
```
    # to create an image:
    abootimg --create boot.img -f bootimg.cfg -k $MYKERNEL/arch/arm/boot/zImage-dtb -r initrd.img [-s <secondstage>] -c bootsize=[SIZE_IT_WANTS]

    # to update an existing image:
    abootimg -u boot.img -f bootimg.cfg -k $MYKERNEL/arch/arm/boot/zImage-dtb -r initrd.img [-s <secondstage>] -c bootsize=[SIZE_IT_WANTS]
```

Note: the option *"-r <ramdisk>"*, where ramdisk is initrd.img, not the ramdisk folder.

Note: It my case, it works only with **zImage-dtb**, not with **zImage** nor **Image



### === Solution 2 ===

Download [the official google tool](https://android.googlesource.com/platform/system/core/+/master/mkbootimg/).      

==> extract it to /home/user/bootimgtool

    mkdir $FACTORY/unpacked_img
    export BOOTIMGTOOL=/home/user/bootimgtool
    cd $BOOTIMGTOOL

* Unpack
```
    ./unpack_bootimg --boot_img $FACTORY/unziped_img/boot.img --out  $FACTORY/unpacked_img
```
* Repack
```
    cp $MYKERNEL/arch/arm/boot/zImage-dtb $ $FACTORY/unpacked_img/kernel
    ./mkbootimg --kernel $FACTORY/unpacked_img/kernel --ramdisk $FACTORY/unpacked_img/ramdisk -o $FACTORY/boot.img
```


### === Solution 2 bis ===

Download [this tool](https://github.com/bzyx/bootimg-tools).

    git clone https://github.com/bzyx/bootimg-tools
    export BOOTIMGTOOL=/home/user/bootimg-tools

Then you need to unpack the .img from your factory image. Insert it your custom kernel and repack it.

    cd $BOOTIMGTOOL
    make
    cd mkbootimg

* Unpack
``` 
    ./unmkbootimg -i $FACTORY/unziped_img/boot.img
    cp $MYKERNEL/arch/arm/boot/zImage-dtb $ $FACTORY/unpacked_img/kernel
```
* Repack
```
    ./mkbootimg --base 0 --pagesize 2048 --kernel_offset 0x00008000 --ramdisk_offset 0x02000000 --second_offset 0x00f00000 --tags_offset 0x01e00000 --cmdline 'console=ttyHSL0,115200,n8 androidboot.console=ttyHSL0 androidboot.hardware=shamu msm_rtb.filter=0x37 ehci-hcd.park=3 utags.blkdev=/dev/block/platform/msm_sdcc.1/by-name/utags utags.backup=/dev/block/platform/msm_sdcc.1/by-name/utagsBackup coherent_pool=8M' --kernel kernel --ramdisk ramdisk.cpio.gz -o $FACTORY/boot.img
```




## == Flash the boot image into the phone bootloader ==

### === Solution 1 ===

* First set your phone in **fastboot mode** taping:
`adb reboot bootloader`

or pressing manually **POWER** and **SOUND LOWER DOWN** for a few seconds.

Then taping:
`fastboot devices`              
Your device must appear in the list

* If you want to test quickly if your new custom **boot.img** is well functional try:
`fastboot boot boot.img`

* If it does not return you **FAILED**, you can try to flash it in memory using:
```
fastboot flash boot boot.img
fastboot reboot
```

* Otherelse, you will understand the pain of compiling an android kernel .... Google is your friend!

### === Solution 2 ===

* First you need to rezip your new **boot.img** in the **image-shamu-mmb29k.zip
```
cd $FACTORY
zip -r ../image-shamu-mmb29k.zip boot.img
```

* Connect your phone in **fastboot mode** 
```
fastboot devices              ==> device must appear in the list
```

* Execute either the factory script
`./flash-all.sh`
or
```
fastboot -w update image-shamu-mmb29k.zip
fastboot reboot
```



If you want to learn more about the differents partitions, the android system, the fastboot commands, etc..., cast a glance [this slides here](https://fr.slideshare.net/chrissimmonds/android-bootslides20).




## == Root the phone ==

You can use a automatic script like [Cf-Auto-Root](https://desktop.firmware.mobi/device:16/firmware:694) for it.

Or follow this next steps inspired from [this guide](https://dottech.org/202068/how-to-root-google-nexus-6-on-android-7-0-with-supersu/).

* Download **TWRP** for Nexu 6 Shamu [here](https://eu.dl.twrp.me/shamu/) .
* Download the latest **SuperSU** zip file [from this link]http://www.supersu.com/download) or [from this one](https://www.teamandroid.com/2017/05/31/download-supersu-282/).
* Download **fastboot** and **adb** from:
  sudo apt-get install adb fastboot
* Connect the Google Nexus 6 smartphone to the computer with the USB cable that is used for charging the battery of the device.
* Copy the SuperSU.zip file over to the internal storage SD card folder for the Google Nexus 6 smartphone. (you may need to choose usb mode into MTP to transfer file)
* Type the following command to get the Google Nexus 6 smartphone into the Bootloader Mode that is a required before flashing the custom recovery image on the smartphone.
  adb reboot bootloader
* Type the following  command to have the custom recovery image flashed onto the Google Nexus 6 smartphone.
  fastboot flash recovery twrp-3.XXXX-shamu.img
* Navigate in the bootloader menu with sound hardware keys and select ''RECOVERY MODE'' to boot directly into the Recovery Mode now and the custom recovery boots up.
* Tap on the **Install** button and then select the SuperSu.zip file.
* Choose the **reboot option**, then slides,then tap **no install** to reboot the system back into the normal mode and continue using the device as you typically would but this time you now have the root access and the custom recovery image installed.




## == Compile ClkScrew mymodule.ko module ==

* Write your kernel module hello-1.c
```
/*  
 *  hello-1.c - The simplest kernel module.
 */
#include <linux/module.h>	/* Needed by all modules */
#include <linux/kernel.h>	/* Needed for KERN_INFO */

int init_module(void)
{
	printk(KERN_INFO "Hello world 1.\n");

	/* 
	 * A non 0 return means init_module failed; module can't be loaded. 
	 */
	return 0;
}

void cleanup_module(void)
{
	printk(KERN_INFO "Goodbye world 1.\n");
}
```
  

* Write the kernel module Makefile

```
    ccflags-y += -fno-stack-protector -fno-pic -Wno-unused-function
    obj-m += module.o
    module-objs := hello-1.o hello-2.o hello-3.o
    CROSS_COMPILE=/home/user/ndk/arm-eabi-4.8/binarm-eabi-
    KERNEL_DIR=/home/user/images_kernel/msm-android-msm-shamu-3.10-marshmallow
    TMP_BUILD ?= /tmp/build
    MODULE_NAME = module.ko
    mkfile_path := $(abspath $(lastword $(MAKEFILE_LIST)))
    current_dir := $(patsubst %/,%,$(dir $(mkfile_path)))
    all:
            mkdir -p $(TMP_BUILD)
            cp ./*.c ./*.h ./Makefile $(TMP_BUILD)
            make -C ${KERNEL_DIR} M=$(TMP_BUILD) ARCH=arm PLATFORM=shamu CROSS_COMPILE=${CROSS_COMPILE} modules V=1
            cp $(TMP_BUILD)/$(MODULE_NAME) $(PWD)/$(MODULE_NAME)
            rm -rf $(TMP_BUILD)
    clean:
            make -C ${KERNEL_DIR} M=$(PWD) clean
```
* Now compile it.

 `make`

It should create the module.ko

* Finally insert your kernel module

```
adb push module.ko /data/local/tmp
adb shell su -c "lsmod"
adb shell su -c "insmod /data/local/tmp/module.ko"
adb shell su -c "lsmod"
```

You can have more info with:
```
adb shell su -c "cat /proc/kmsg"
```

More info about kernel module programmation [here](https://linux.die.net/lkmpg/c119.html).
