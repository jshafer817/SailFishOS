# SailFishOS

Ramdisk\LVM Partitions:
mount_stowaways() {
    echo "########################## mounting stowaways"
    if [ ! -z $DATA_PARTITION ]; then
	echo "[initramfs] Activating LVM..."
        mkdir /boot
        mount -o ro -t ext3 /dev/mmcblk0p13 /boot
        LVM_SYSTEM_DIR=/boot/etc/lvm /boot/usr/sbin/lvm.static vgscan
        sleep 2
        LVM_SYSTEM_DIR=/boot/etc/lvm /boot/usr/sbin/lvm.static vgmknodes
        sleep 2
        LVM_SYSTEM_DIR=/boot/etc/lvm /boot/usr/sbin/lvm.static vgchange -a y
        umount /boot

        data_subdir="$(get_opt data_subdir)"

	mkdir /data
	mkdir /target

	mount $DATA_PARTITION /data
	ln -s /data/media/0/.stowaways /data
	mount --bind /data/${data_subdir}/.stowaways/${DEFAULT_OS} /target
	mkdir /target/data # in new fs
	mount --bind /data/${data_subdir} /target/data
    else
        echo "Failed to mount /target, device node '$DATA_PARTITION' not found!" >> /diagnosis.log
    fi
    mount
}

zypper install strace rfkill

Grahpics:
hwc.patch in cm11
patch hwcomposer-plugin fro Nokius
Be sure debugfs is not mounted!

Wifi:
3.0 Kernel:
depmod -a -v
Add /etc/modulesload.p/ath6kl.conf
compat
cfg80211
ath6kl

3.4 Kernel:
Create service and kernel modules instead of builtin:
[Unit]
Description=ATH6KL
Before=network.target

[Service]
RemainAfterExit=true
ExecStart=/usr/bin/enable_ath6kl

[Install]
WantedBy=multi-user.target
--EOF--

/usr/bin/enable_ath6kl

#!/bin/bash
systemctl stop bluetooth.service
systemctl stop connman.service
insmod /cfg80211.ko
insmod /ath.ko
insmod /ath6kl.ko
sleep 5
rfkill unblock all
/usr/bin/bcattach &
sleep 15
rfkill unblock all
hciconfig hci0 up
systemctl start bluetooth.service
systemctl start connman.service

Bluetooth:
3.4 Only:
bluetooth.service changes.
-ConditionPathIsDirectory=/sys/kernel/debug/bluetooth
/var/lib/connman/settings BlueTooth should be enabled to prevent rfkill.


Sound: Used Hammerheads /etc/pulse directory but renamed  arm_qualcomm_msm_8974_hammerhead_flattened_device_tree_000b.pa to default.pa

Screen Rotation:
Changed /usr/lib/qt5/qml/Sailfish/Silica/ApplicationWindow.qml
property bool _transpose: (screenRotation % 180) != 270
property int _defaultPageOrientations: Orientation.InvertedLandscape

DPI:
dconf write /desktop/sailfish/silica/theme_pixel_ratio 0.9

Browser:
BrowserPage.qml
 Browser.DownloadRemorsePopup { id: downloadPopup }
        portrait: browserPage.isLandscape
    }
    
    
Video Codecs:
ln -s /system/etc/media_codecs.xml /etc
ln -s /system/etc/media_profiles.xml /etc

libgstav:
for module in gstreamer gst-plugins-base gst-plugins-good gst-plugins-bad gst-plugins-ugly gst-libav; do
   git clone git://anongit.freedesktop.org/git/gstreamer/$module ;
 done

cd into libgstav

sb2 -t $VENDOR-$DEVICE-armv7hl -R ./autogen.sh --prefix=/usr --disable-gtk-doc

"evdev_trace -i" (from mce-tools.rpm

Lots of patches to build an image:
So you booted your Sailfish OS rootfs? Congrats. No GUI? Oh dear :)

get connected:
If your host PC is connected to internet via wlan0:
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
echo 1 > /proc/sys/net/ipv4/ip_forward
on device:
route add default gw 192.168.2.2 (<- host's usb0 IP)
echo nameserver 208.67.222.222 > /etc/resolv.conf

lipstick segfaults:
(as you follow steps below, strace any of the binaries that would fail for non-obvious reasons, zypper in strace)
systemctl stop user@100000.service
test simple hwc as root:
EGL_PLATFORM=hwcomposer test_hwcomposer
if you use display-caf, apply this patch: http://pastebin.com/TsfSwdXG
if strace dies after open("/sys/kernel/debug/tracing/trace_marker..., perform
systemctl mask sys-kernel-debug.mount
minimer:
zypper in qt5-qtdeclarative-qmlscene
curl -O http://qtl.me/minimer3.tar.gz
tar -xf minimer3.tar.gz; cd minimer
qmlscene -platform hwcomposer main.qml
if fails as user, try as root
for more info: zypper in gdb
if you get test_hwcomposer, minimer or lipstick segfaults:
if your device uses qcom_display-caf (i.e. TARGET_QCOM_DISPLAY_VARIANT :=   caf in any of the BoardConfig*.mk files in your device repos), you'll need to recompile android hwcomposer*.so for your device after some patching (https://pastee.org/bmmwa for cm-11.0). Recompling can be done in HABUILD_SDK by first checking the module name (hwcomposer*) from make modules output and then running make hwcomposer_module_name
determine if device uses QCOM_BSP, this can be found in BoardConfig.mk or BoardConfigCommon.mk in any of the device repos for the device, the repos included can be determined by looking at the -include device/$VENDOR/*/BoardConfig.mk or device/$VENDOR/*/BoardConfigCommon.mk lines at beginning the .mk files starting from the primary BoardConfig.mk
if you found QCOM_BSP in any of .mk files, doublecheck with your gralloc.h around the first (*allocSize)(...) function, for example https://github.com/CyanogenMod/android_hardware_libhardware/blob/cm-10.1/include/hardware/gralloc.h#L240
if #ifdef says QCOM_HARDWARE, substitute QCOM_BSP below with that one
add to your droid-hal-$DEVICE.spec before the %include line:
%define android_config \
#define QCOM_BSP 1\
%{nil}
add to qt5-qpa-hwcomposer the entry DEFINES += QCOM_BSP after https://github.com/mer-hybris/qt5-qpa-hwcomposer-plugin/blob/098e6fa7158a49e1c61e717c69665b92190391eb/hwcomposer/hwcomposer.pro#L33
rebuild droid-hal-device (mb2 rpm/ ...)
rebuild libhybris
rebuild qt5-qpa-hwcomposer-plugin
if now test_hwcomposer and/or minimer show only first frame and then freeze, try this patch (heavily WIP and can cause memory leaks in long run, so NOT a final solution): this is how the patch looks in qt5-qpa-hwcomposer-plugin: the "if" block at https://gist.github.com/Nokius/2b07dc5bad0fea0b0bb1#file-hwcomposer_backend_v11-cpp-L79 and commenting out L101 -- firstly, apply this patch to test_hwcomposer.cpp of libhybris, then port over to your version of qt5-qpa-hwcomposer-plugin .cpp (hwc version can be seen during minimer launch)
scream on the IRC if this worked for you
If strace indicates something like:
"Waiting for service display.qservice..."
This error is known only on cm-10.1 base, and will be upstreamed to mer-hybris soon, but we need more tests: apply https://github.com/mer-hybris/android_frameworks_native/commit/6ed4a6e834f6c71b2b6bd8ae1134f50b060e70be to this line https://github.com/CyanogenMod/android_frameworks_base/blob/cm-10.1/cmds/servicemanager/service_manager.c#L88 and also apply https://github.com/mer-hybris/android_system_core/commit/34ea48fd3ad7bf47ec0d0524d76bd20e62717773
open("/sys/kernel/debug/tracing/trace_marker", O_WRONLY|O_LARGEFILE) = 
disable debugfs by: https://github.com/mer-hybris/droid-hal-device/commit/8d437fc6f215081d4e1d2baaa6ac23bb94f73154
if it still crashes on gralloc or other gpu related bits, refer to WIP: https://wiki.merproject.org/wiki/Adaptations/libhybris/gpu

SIM card not detected:
ensure rild is launched from /init*.rc scripts
add "disabled" to init scripts, thenlaunch it manually, and observe logcat/strace
provide any missing firmware (mounts, e.g. on /firmware or /modem) if any
replicate /dev/block structure from Android as closely as possible (for rild accessing the modem partition directly) if rild crashes constantly
once rild is happy, ensure ofono is launched and observe journald, and possibly CPU usage using top
disable your PIN lock with a different phone

reboot after bootup:
few seconds into boot: selinux=0 to your kernel cmdline (you'll find it somewhere under find device -name 'BoardConfig*.mk')  and CONFIG_SECURITY_SELINUX_BOOTPARAM=y to defconfig (and post here your device name so we know how big the issue is):
UPDATE: it looks like all >=cm11 devices will need this cmdline option, thanks Tassadar
amami
deb/flo
hammerhead
c8813q 
hwp6_u06 (Huawei) is affected too, stracing droid-hal-init leads to hwnff service restarting and thus timing the droid-hal-init out
~1min into boot: systemctl mask ofono

experimental gstreamer-1.0 support (video and camera):
make hybris-hal from the latest hybris-11.0 manifest (ensure you repo sync at least once since 2015-03-16)
go through hadk and build an image for your device, ensure it boots to UI
Compile and install libhybris locally (instructions 13.8.3 section HADK v1.0.3.0)
perform:
HABUILD_SDK $
make -j8 libdroidmedia minimediaservice minisfservice
MER_SDK $
mkdir -p $MER_ROOT/devel/mer-hybris
cd $MER_ROOT/devel/mer-hybris
sb2 -t $VENDOR-$DEVICE-armv7hl -R -msdk-install ssu ar gst http://repo.merproject.org/obs/home:/sledge:/experimental:/gstdroid/sailfish_latest_armv7hl/
sb2 -t $VENDOR-$DEVICE-armv7hl -R -msdk-install zypper ref
PKG=gst-droid
git clone https://github.com/foolab/$PKG.git -b droidmedia
cd $PKG
mb2 -s rpm/$PKG.spec -t $VENDOR-$DEVICE-armv7hl build
mkdir -p $ANDROID_ROOT/droid-local-repo/$DEVICE/$PKG/
rm -f $ANDROID_ROOT/droid-local-repo/$DEVICE/$PKG/*.rpm
mv RPMS/*.rpm $ANDROID_ROOT/droid-local-repo/$DEVICE/$PKG
createrepo $ANDROID_ROOT/droid-local-repo/$DEVICE
sb2 -t $VENDOR-$DEVICE-armv7hl -R -msdk-install zypper ref
Port these patches to your device:
https://github.com/mer-hybris/droid-hal-device/commit/cb997e62eb46fcea738ca84fb10907561a082f8e - the .xml files might be device specific
https://github.com/mer-hybris/droid-hal-device/commit/905ce9c154dd9fbb9a5dba364d563e293d099804
https://github.com/mer-hybris/droid-hal-device/commit/98e8ade56f30e0b9cc019ef9480b0803b1744509
Rebuild droid-hal-$DEVICE
Produce an image with an extra repository: http://repo.merproject.org/obs/home:/sledge:/experimental:/gstdroid/sailfish_latest_armv7hl/
You should have a (muted) video playback working
to get sound on, as root after burning the image on the phone, 
           zypper in gstreamer1.0-libav
           this is not distributable in the image...
Get cameraplus:
zypper install harbour-cameraplus harbour-cameraplus-tools
Starting 30th March 2015, in the steps below use your device codename under configs/: https://github.com/sledges/cameraplus/commit/73ba58340aceb6e98f4410ab2014a695179e466f#diff-ee0bd87d0f10d1e814d0ce2ca908cbb2
Until codecs are sorted, gut out the audio support for video recordings for your device: https://github.com/foolab/cameraplus/commit/41f0ff12660a1398acbb9ee358213c25a8dbb89f
Run /usr/bin/dump_resolutions from cameraplus-tools
Add couple of resolutions to your device folder like this: https://github.com/sledges/cameraplus/commit/46d80e31754c8b61953825c306fdf592d06aaefc
If camera still segfaults/freezes, try this on your device: https://github.com/mer-hybris/android_device_lge_hammerhead/commit/4903bf1096a3a7e37b2308f101934ee6e8162086
Check for known bugs (search page for 'camera'): http://bit.ly/port-bugs
ping us on #sailfishos-porter and we'll DIT!

