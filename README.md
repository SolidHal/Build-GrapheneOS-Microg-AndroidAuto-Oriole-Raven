# Build Graphene for Pixel 6/6 plus with MicroG and Android Auto


## setup the envrionment
mostly just follow https://grapheneos.org/build

some parts are pulled out here for brevity, and my own reference

```
sudo apt install signify-openbsd yarn
alias signify=signify-openbsd
```

```
mkdir grapheneos-13
cd grapheneos-13
repo init -u https://github.com/GrapheneOS/platform_manifest.git -b 13
repo sync -j14
```

prebuilt vendors repo:
https://releases.grapheneos.org/oriole-vendor-2022011423.tar.zst
https://releases.grapheneos.org/raven-vendor-2022011423.tar.zst

they just need to be placed at vendor/google_devices/{oriole,raven}

### Extract vendors repo

Requirements:
- yarn
- simg2img
- recent node version (anything less than 13 is too old)
- python3 protobuf

Example graphene os release version: `TQ2A.230405.003.E1.2023041100`
So need factory image `TQ2A.230405.003.E1`

build tools:
```
yarn install --cwd vendor/adevtool/
source script/envsetup.sh
m aapt2
```

extract vendors:
```
DEVICE=oriole
BUILD_ID=TQ2A.230405.003.E1
vendor/adevtool/bin/run download vendor/adevtool/dl/ -d $DEVICE -b $BUILD_ID -t factory ota
sudo vendor/adevtool/bin/run generate-all vendor/adevtool/config/$DEVICE.yml -c vendor/state/$DEVICE.json -s vendor/adevtool/dl/$DEVICE-${BUILD_ID,,}-*.zip
sudo chown -R $(logname):$(logname) vendor/{google_devices,adevtool}
vendor/adevtool/bin/run ota-firmware vendor/adevtool/config/$DEVICE.yml -f vendor/adevtool/dl/$DEVICE-ota-${BUILD_ID,,}-*.zip
```

## Add FDroidPrivilegedExtension
https://gitlab.com/fdroid/privileged-extension

add
```
<?xml version="1.0" encoding="UTF-8"?>
<manifest>

  <remote name="fdroid" fetch="https://gitlab.com/fdroid/" />
  <project path="packages/apps/F-DroidPrivilegedExtension"
           name="privileged-extension.git" remote="fdroid"
           revision="refs/tags/0.2.13" />

</manifest>
```
to `<checkout>/.repo/local_manifests/manifests.xml`
create the `local_manifests` dir if necessary

then do a `repo sync -j 12`

Finally, add the following to `build/target/product/base_system.mk`
```
PRODUCT_PACKAGES += F-DroidPrivilegedExtension
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
PRODUCT_PACKAGES += auroraservices
```

to add your own packages, make a repo like:
https://github.com/SolidHal/android_prebuilts_solidhal
https://github.com/SolidHal/android-auto-stub

and modify the instructions above


## Apply Patches for Android Auto

from your graphene os checkout,

```
cd frameworks/base
git am ~/<this-repo>/patches/android-auto-13.patch
```


# keys

following the grapheneos docs create keys in:
$HOME/android/keys/$DEVICE

then link to the build system
ln -s $HOME/android/keys/ $(pwd)/keys


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
