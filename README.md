# Mazda Linux Toolchain

This repository contains configuration files necessary to compile a toolchain
suitable for cross compiling software for the Mazda Connect system, which is
included in newer Mazda cars. It was tested on the 2016 Mazda 6, but the process
should be similar for other models.

These configuration files were created for use with OS X, but but should work
with other platforms after minor adjustments.

## Warning

This project is not for the faint-of-heart. By making modifications to the
Mazda Connect system, you're putting yourself at risk. Regardless of how much
experience you have, if you're not careful, there is a very high chance your
device will get stuck in a boot loop. If you don't understand the full extent
of what the goal of this project is or how it works, I recommend that you stop
here.

This guide makes some assumptions about the knowledge you already have. If you
run into troubles, feel free to file issues. However, most of the steps should
be fairly easy to understand as long as you know what you're doing.

## Setup

1. Make sure you're using a case-sensitive filesystem before continuing. On Mac,
   you can create a sparse image and do all of your work inside of it. Ensure
   the partition you're saving data on is at least 20 GB large. If you don't,
   you may run out of space towards the end of the process.

2. Download this repository:

   ```
   git clone https://github.com/rpendleton/mazda-toolchain.git mazda-toolchain
   cd mazda-toolchain
   ```

3. Download the Freescale Linux kernel. The processor used by Mazda Connect is
   based on the Freescale i.MX6. You can download the proper Linux kernel source
   tree from Freescale's git repository.

   For cmu150_NA_55.00.753A, this is linux-2.6-imx#3.0.35-4.1.0. If you are
   cloning the kernel to a different folder, make sure you update the custom
   Linux kernel path in the ct-ng config file.

   ```shell
   # from mazda-toolchain
   git clone git://git.freescale.com/imx/linux-2.6-imx.git linux-freescale-3.0.35
   cd linux-freescale-3.0.35
   git checkout imx_3.0.35_4.1.0
   ```

4. Obtain the Linux kernel configuration file from your vehicle. You can get the
   config file from an existing Mazda Connect system by loading the `configs.ko`
   kernel module, and then copying `/proc/config.gz`. I've included a copy of
   my config for convenience.

   ```shell
   # from linux-freescale-3.0.35
   ssh root@192.168.53.1
   insmod /lib/modules/3.0.35/kernel/kernel/configs.ko
   exit

   scp root@192.168.53.1:/proc/config.gz config.gz
   gzip -d config.gz
   mv config .config
   ```

5. Update config to add missing values. In some cases, the config file obtained
   from your car may not include all of the values that are present in the
   Freescale repository. You can add any new options by running the following,
   similar to how you would a regular Linux kernel:

   ```shell
   # from linux-freescale-3.0.35
   make ARCH=arm oldconfig
   ```

6. Patch the kernel headers. One of the headers exported by Freescale depends on
   non-exported headers. Rather than exporting some internal headers, it's
   easier to simply un-export the one header.

   Open the `include/linux/Kbuild` file and comment/remove the line that exports
   the `fsl_devices.h` header.

7. Install and check the headers. Make sure that make doesn't emit any errors,
   or you will encounter a build failure a few minutes into your build.

   ```shell
   # from linux-freescale-3.0.35
   # include patch command here
   make ARCH=arm headers_install
   make ARCH=arm headers_check
   ```

8. We're ready to begin compiling the toolchain. This process took thirty
   minutes on my computer, but the time required may vary. If you don't have
   crosstool-ng, install it before running this step.

   ```shell
   # from linux-freescale-3.0.35
   cd ../
   ct-ng build
   ```

9. Transfer shared libraries to your Mazda Connect system. Since glibc isn't
   supported on all platforms (specifically OS X), this toolchain uses musl.
   Because of this, you'll need to transfer a few libraries from `sysroot/lib`
   and `sysroot/usr/lib`. Since you made it this far, I'll assume you can figure
   out which libraries are necessary for your application.

   Alternatively, you can link all of your programs with musl statically, but
   this will increase the size of your programs.

10. Start playing with your new toolchain!
