# Build Graphene for Pixel 6/6 plus with MicroG and Android Auto


## setup the envrionment
mostly just follow https://grapheneos.org/build

some parts are pulled out here for brevity, and my own reference

```
sudo apt install signify-openbsd
alias signify=signify-openbsd
```

```
mkdir grapheneos-12
cd grapheneos-12
repo init -u https://github.com/GrapheneOS/platform_manifest.git -b 12
repo sync -j14
```

prebuilt vendors repo:
https://releases.grapheneos.org/oriole-vendor-2022011423.tar.zst
https://releases.grapheneos.org/raven-vendor-2022011423.tar.zst

they just need to be placed at vendor/google_devices/{oriole,raven}


## Add Custom packages (app and priv-app)

checkout the repo
```
cd AOSP_SRC

# microg apps, fdroid, auroraservices, etc
mkdir solidhal
cd solidhal
git clone git@github.com:SolidHal/android_prebuilts_solidhal.git .
```

Add the following to `build/target/product/base_system.mk`
```
PRODUCT_PACKAGES += GmsCore GsfProxy FakeStore FDroid FDroidPrivilegedExtension auroraservices
```

to add your own packages, make a repo like:
https://github.com/SolidHal/android_prebuilts_solidhal
https://github.com/SolidHal/android-auto-stub

and modify the instructions above 


## Apply Patches for microG and Android Auto


from your graphene os checkout, 

```
cd frameworks/base
git am ~/<this-repo>/patches/sig_spoofing/android_frameworks_base-S.patch
git am ~/<this-repo>/patches/android-auto-12.patch

cd device/google/raviole
git am ~/<this-repo>/patches/sig_spoofing/deviceFrameworkOverlay.patch

cd packages/modules/Permission
git am ~/<this-repo>/patches/sig_spoofing/permissionControllerPatch.patch
```

# keys

following the grapheneos docs create keys in:
$HOME/android/keys/$DEVICE

then link to the build system
ln -s $HOME/android/keys/ $HOME/android/grapheneos-12/keys


# BUILD
```
source script/envsetup.sh
choosecombo release oriole user
m vendorbootimage target-files-package
m otatools-package
script/release.sh oriole
```


# Setup
- open microg, give it all the permissions
- install some location backends into microg
- install the AndroidAuto app from aurora store
- install the Gapp stub and speech services stubs from https://github.com/SolidHal/android-auto-stub
- open AndroidAuto once, give it all the permissions
- in settings, also give AndroidAuto permission to detect nearby devices (needed for bluetooth)
- The first time you setup AA with a car's headunit you usually need to reboot, when you first plug in it'll mention there's no apps that can handle the device, once you reboot AA should be available.


TODO:
- why is fdroid priv extension not working?




## Very Important, Uninstalling GrapheneOS
Replacing GrapheneOS with the stock OS

Installation of the stock OS via the stock factory images is the same process described above. However, before locking, there's an additional step to fully revert the device to a clean factory state.

The GrapheneOS factory images flash a non-stock Android Verified Boot key which needs to be erased to fully revert back to a stock device state. After flashing the stock factory images and before locking the bootloader, you should erase the custom Android Verified Boot key to untrust it:

```
fastboot erase avb_custom_key
```
