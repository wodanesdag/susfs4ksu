## Introduction ##
- An addon root hiding kernel patches and userspace module for KernelSU.

- The userspace tool `ksu_susfs`, as well as the ksu module, require a susfs patched kernel to work.

# Warning #
- This is only experimental code, that said it can harm your system or cause performance hit, **YOU ARE !! W A R N E D !!** already

## Compatibility ##
- The susfs kernel patches may differ for different kernel version or even on the same kernel version, you may need to create your own patches for your kernel.

## Patch Instruction (For non-GKI only) ##
**- Prerequisite -**
1. All susfs patches are based on the original official KernelSU (the one from weishu), so you should clone his repo for a better patching result.
2. After cloning the KernelSU to your kernel source, inside KernelSU folder, apply a revert commit from `https://github.com/tiann/KernelSU/commit/898e9d4f8ca9b2f46b0c6b36b80a872b5b88d899`
3. Also you have to manually implement the NON-KPROBE hooks in the kernel by yourself since `SUS_SU` feature is NOT supported for NON-GKI kernel anymore. See `https://kernelsu.org/guide/how-to-integrate-for-non-gki.html#manually-modify-the-kernel-source`
4. Replace all lines starting with `#ifdef CONFIG_KPROBES` to `#if defined(CONFIG_KPROBES) && 0` for all files within KernelSU folder, this is to tell KernelSU not to use any kprobe hooks.
5. SUSFS now supports AUTO_ADD_ features for Magick Mount KernelSU as long as you have `KSU_SUSFS_HAS_MAGIC_MOUNT` feature enabled.

**- Apply SUSFS patches -**
1. Clone the repo with a tag or release version, as they are more stable in general.
2. Run `cp ./kernel_patches/KernelSU/10_enable_susfs_for_ksu.patch $KERNEL_ROOT/KernelSU/`
3. Run `cp ./kernel_patches/50_add_susfs_in_kernel-<kernel_version>.patch $KERNEL_ROOT/`
4. Run `cp ./kernel_patches/fs/* $KERNEL_ROOT/fs/`
5. Run `cp ./kernel_patches/include/linux/* $KERNEL_ROOT/include/linux/`
6. Run `cd $KERNEL_ROOT/KernelSU` and then `patch -p1 < 10_enable_susfs_for_ksu.patch`
7. Run `cd $KERNEL_ROOT` and then `patch -p1 < 50_add_susfs_in_kernel-4.19.patch`, **if there are failed patches, you may try to patch them manually by yourself.**
8. If you want to make your kernel support other KSU manager variant, you can add its own hash size and hash in `ksu_is_manager_apk()` function in `KernelSU/kernel/apk_sign.c`
9. Make sure again to have `CONFIG_KSU` and `CONFIG_KSU_SUSFS` enabled before building the kernel, some other SUSFS feature may be disabled by default, you may enable/disable them via `menuconfig`, `kernel defconfig`, or change the `default [y|n]` option under each `config KSU_SUSFS_` option in `$KernelSU_repo/kernel/Kconfig` if you build with a new defconfig every time.
10. If your KernelSU manager is using magic mount, then you should enable **`KSU_SUSFS_HAS_MAGIC_MOUNT`** option so that mounts can be handled by AUTO_ADD_ features.
11. For some compilor error, please refer to the section **[Known Compilor Issues]** below.
12. For other building tips, please refer to the section **[Other Building Tips]** below.

## Build ksu_susfs userspace tool ##
1. Run `./build_ksu_susfs_tool.sh` to build the userspace tool `ksu_susfs`, and the arm64 and arm binary will be copied to `ksu_module_susfs/tools/` as well.
2. Now you can also push the compiled `ksu_susfs` tool to `/data/adb/ksu/bin/` so that you can run it directly in adb root shell or termux root shell, as well as in your own ksu modules.

## Build sus_su userspace tool (Deprecated) ##
**--Important Notes--**
- sus_su userspace tool is now deprecated, as newer xiaomi devices are found to have a root detection service running which is named "mrmd" and it is spawned by init process, and since sus_su mounted by overlayfs can't be umounted for process spawned by init process, so it will get detected unless there is a better umount scheme for init spawned process.

**--Instruction for 1st mode (Deprecated)--**
- sus_su userspace tool is an executable aimed to get a root shell by sending a request to a susfs fifo driver, this is exclusive for **"kprobe hook enabled KSU"** only, **DO NOT** use it if your KernelSU has kprobe **disabled**.
- Only apps with root access granted by ksu manager are allow to run 'su'.
- For best compatibility, sus_su requires overlayfs to allow all other 3rd party apps to execute 'su' to get root shell.
- See `service.sh` in module templete for more details.

1. Run `./build_sus_su_tool.sh` to build the sus_su executable, the arm64 and arm binary will be copied to `ksu_module_susfs/tools/`.
2. Uncomment the line `#enable_sus_su` in service.sh to enable sus_su
3. Run `./build_ksu_module.sh` to build the module and flash again.

**--Instruction for 2nd mode--**
- Just run `ksu_susfs sus_su 2` to disable core kprobe hooks and enable inline hooks for su.


## Build susfs4ksu module ##
- The ksu module here is just a demo to show how to use it.
- It will also copy the `ksu_susfs` and `sus_su` tool to `/data/adb/ksu/bin/` as well when installing the module.

1. ksu_susfs tool can be run in any stage scripts, post-fs-data.sh, services.sh, boot-completed.sh according to your own need.
2. Run `./build_ksu_module.sh` to build the susfs KSU module.

## Usage of ksu_susfs and supported features ##
- Run `ksu_susfs` in root shell for detailed usages.
- See `$KernelSU_repo/kernel/Kconfig` for supported features after applying the susfs patches.

## Other Building Tips ##
- To only remove the `-dirty` string from kernel release string, open file `$KERNEL_ROOT/scripts/setlocalversion`, then look for all the lines that containing `printf '%s' -dirty`, and replace it with `printf '%s' ''`
- Alternatively, If you want to directly hardcode the whole kernel release string, then open file `$KERNEL_ROOT/scripts/setlocalversion`, look for the last line `echo "$res"`, and for example, replace it with `echo "-android13-01-gb123456789012-ab12345678"`
- To hardcode your kernel version string, open `$KERNEL_ROOT/scripts/mkcompile_h`, and look for line `UTS_VERSION="$(echo $UTS_VERSION $CONFIG_FLAGS $TIMESTAMP | cut -b -$UTS_LEN)"`, then for example, replace it with `UTS_VERSION="#1 SMP PREEMPT Mon Jan 1 18:00:00 UTC 2024"`
- To hardcode your kernel version string which can be seen from /proc/version, open `$KERNEL_ROOT/scripts/mkcompile_h`, then search for variable name `LINUX_COMPILE_BY` and `LINUX_COMPILE_HOST`, then for example, append `LINUX_COMPILE_BY=build-user` and `LINUX_COMPILE_HOST=build-host` after line `UTS_VERSION="$(echo $UTS_VERSION $CONFIG_FLAGS $TIMESTAMP | cut -b -$UTS_LEN)"`
- To spoof the `/proc/config.gz` with the stock config, 

   1. Make sure you are on the stock ROM and using stock kernel.
   2. Use adb shell or root shell to pull your stock `/proc/config.gz` from your device to PC.
   3. Decompress it using `gunzip` or whatever tools, then copy it to `$KERNEL_ROOT/arch/arm64/configs/stock_defconfig`
   4. Open file `$KERNEL_ROOT/kernel/Makefile`.
   5. Look for line `$(obj)/config_data: $(KCONFIG_CONFIG) FORCE`, and replace it with `$(obj)/config_data: arch/arm64/configs/stock_defconfig FORCE`

## Known Compiler Issues ##
   1. error: no member named 'android_kabi_reservedx' in 'struct yyyyyyyy'

      - Because normally the memeber `u64 android_kabi_reservedx;` doesn't exist in all structs with all kernel version below 4.19, and sometimes it is not guaranteed existed with kernel version >= 4.19 and <= 5.4, and even with GKI kernel, like some of the custom kernels has all of them disabled. So at this point if the susfs patches didn't have them patched for you, then what you need to do is to manually append the member to the end of the corresponding struct definition, it should be `u64 android_kabi_reservedx;` with the last `x` starting from `1`, like `u64 android_kabi_reserved1;`, `u64 android_kabi_reserved2;` and so on. You may also refer to patch from other branches like `kernel-4.14`, `kernel-4.9` of this repo for extra `diff` of the missing kabi members.
 

## Other Known Issues ##
- Some of the File Explorer Apps cannot display a files/directory properly when a specific sub path of '/sdcard' or '/storage/emulated/0' is added to sus_path
    1. Make sure the file explorer app has root allowed by KSU manager, because sus_path is only effective on no root allowed process uid.
    2. It is strongly NOT recommended adding sub path of '/sdcard' or '/storage/emulated/0' to sus_path, because file explorer app is likely using android API to retrieve the list of files/directory, which means the calling uid will be changed to other system media provider app such as the google provider to execute the file lookup operation, and makes sus_path think that it is not a root allowed process uid so as to prevent them from showing up, unless the app obtains the root access first then use root privilege to list the files/directories without using android API.

## Credits ##
- KernelSU: https://github.com/tiann/KernelSU
- KernelSU fork: https://github.com/5ec1cff/KernelSU
- @Kartatz: for ideas and original commit from https://github.com/Dominium-Apum/kernel_xiaomi_chime/pull/1/commits/74f8d4ecacd343432bb8137b7e7fbe3fd9fef189

## Telegram ##
- @simonpunk


## Buy me a coffee ##
- PayPal: kingjeffkimo@yahoo.com.tw
- BTC: bc1qgkwvsfln02463zpjf7z6tds8xnpeykggtgk4kw
