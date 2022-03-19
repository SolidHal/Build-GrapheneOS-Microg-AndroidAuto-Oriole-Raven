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

### Extract vendors repo

Example graphene os release version: `S2B3.220205.007.A1.2022031103`
So need factory image `S2B3.220205.007.A1`

These can be found on the us, or chinese sites (depending on where google chooses to upload them)

https://developer.android.google.cn/about/versions/12/download-ota?hl=zh-cn
https://developers.google.com/android/images

And need the OTA image for `S2B3.220205.007.A1` 

like the factory images, some of these are on the US site, some on the chinese site
https://developer.android.google.cn/about/versions/12/download-ota?hl=zh-cn


Requirements:
- yarn
- simg2img
- recent node version (anything less than 13 is too old)
- python3 protobuf


#### build adevtool
```
cd vendor/adevtool/ && yarn install && cd ../..
```

#### extract the vendors
place the zip in `vendor/adevtool/dl/`

then run
```
sudo vendor/adevtool/bin/run generate-all vendor/adevtool/config/DEVICE.yml -c vendor/state/DEVICE.json -s vendor/adevtool/dl/DEVICE-BUILD_ID-*.zip
```

##### NOTE: aapt2
if you don't have an existing build, `generate-all` will complain about missing aapt2
to build it

example:
```
sudo vendor/adevtool/bin/run generate-all vendor/adevtool/config/oriole.yml -s vendor/adevtool/dl/oriole-s2b3.220205.007.a1-factory-6549410e.zip -c vendor/state/oriole-state-output-file.json
```

next:

```
vendor/android-prepare-vendor/execute-all.sh -d oriole -b s2b3.220205.007.a1 -o vendor/android-prepare-vendor -i vendor/adevtool/dl/oriole-s2b3.220205.007.a1-factory-6549410e.zip --ota ~/android/oriole-ota-s2b3.220205.007.a1-cc557a1a.zip
```


finally:

```
cp -r vendor/android-prepare-vendor/oriole/s2b3.220205.007.a1/vendor/google_devices/oriole/radio/* vendor/google_devices/oriole/firmware/
```


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

# Installing

decompress the factory zip in `out/release-oriole-$BUILD_NUMBER`
use `./flash-all.sh`

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

# Updating

grab the update zip in `out/release-oriole-$BUILD_NUMBER`

follow:
https://grapheneos.org/usage#updates-sideloading




## Very Important, Uninstalling GrapheneOS
Replacing GrapheneOS with the stock OS

Installation of the stock OS via the stock factory images is the same process described above. However, before locking, there's an additional step to fully revert the device to a clean factory state.

The GrapheneOS factory images flash a non-stock Android Verified Boot key which needs to be erased to fully revert back to a stock device state. After flashing the stock factory images and before locking the bootloader, you should erase the custom Android Verified Boot key to untrust it:

```
fastboot erase avb_custom_key
```
