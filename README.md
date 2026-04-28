# webos10-homescreen-customization
How to change the picuture &amp; get rid of elements on the homescreen of your LG TV

# Customizing the LG WebOS 10 Homescreen

A guide to removing unwanted UI elements and replacing the hero banner image on WebOS 10 (Rockhopper / Starfish), using a bind-mount overlay — no permanent changes to the read-only filesystem.

> **Tested on:** LG WebOS 10.2.2 (Rockhopper), EU region  
> **Requirements:** Root access, active SSH connection, Homebrew Channel with `webosbrew` init.d support

---

## Background

On WebOS 10, the Home app has been rewritten in **Flutter** (unlike older versions which used QML). The layout is controlled by XML files inside the app's Flutter assets directory:

```
/usr/palm/applications/com.webos.app.home/data/flutter_assets/assets/
```

This directory is on a read-only, cryptographically signed filesystem, so we can't edit files directly. Instead, we use a **bind mount overlay**: copy the assets to a writable location in `/tmp`, apply our modifications, and mount that over the original directory. The original files are never touched.

---

## Directory Structure

Create your working directory on the writable developer partition:

```sh
mkdir -p /media/developer/apps/usr/palm/applications/tld.my.customhome/assets/images/hd
mkdir -p /media/developer/apps/usr/palm/applications/tld.my.customhome/assets/images/2k
mkdir -p /media/developer/apps/usr/palm/applications/tld.my.customhome/assets/images/4k
```

This is the directory we'll be working in for all modifications.

---

## The apply.sh Script

This script does all the heavy lifting. It copies the original assets to `/tmp`, overlays our modifications, mounts the result over the original directory, and restarts the Home app.

```sh
cat > /media/developer/apps/usr/palm/applications/tld.my.customhome/apply.sh << 'EOF'
#!/bin/sh

set -e -x

ASSETS_DIR=/usr/palm/applications/com.webos.app.home/data/flutter_assets/assets
OVERRIDE_DIR=/media/developer/apps/usr/palm/applications/tld.my.customhome/assets

umount "$ASSETS_DIR" 2>/dev/null || true
rm -rf /tmp/weboshome-merged
mkdir /tmp/weboshome-merged
cp -R --no-dereference "$ASSETS_DIR"/. /tmp/weboshome-merged/
cp -R "$OVERRIDE_DIR"/. /tmp/weboshome-merged/

mount --bind /tmp/weboshome-merged "$ASSETS_DIR"
pkill -f com.webos.app.home || true
EOF

chmod +x /media/developer/apps/usr/palm/applications/tld.my.customhome/apply.sh
```

> The `umount` at the start ensures the script is safe to run multiple times without errors.

---

## Removing Unwanted UI Elements

The layout is defined in two XML files. Copy them into your override directory first:

```sh
cp /usr/palm/applications/com.webos.app.home/data/flutter_assets/assets/home.xml \
   /media/developer/apps/usr/palm/applications/tld.my.customhome/assets/home.xml

cp /usr/palm/applications/com.webos.app.home/data/flutter_assets/assets/home_layoutShelfView.xml \
   /media/developer/apps/usr/palm/applications/tld.my.customhome/assets/home_layoutShelfView.xml
```

### Remove the Recommended Shelf

The `recommendedShelf` is the large content recommendation strip at the bottom of the screen (531px tall). It bothered me to always see the tiles which i don't cate about.

```sh
sed -i '/<item id="recommendedShelf"/d' \
    /media/developer/apps/usr/palm/applications/tld.my.customhome/assets/home.xml
```

### Remove the Q-Card List

The `qcardList` is the horizontal card strip shown above the app list.

```sh
sed -i '/<item id="qcardList"/d' \
    /media/developer/apps/usr/palm/applications/tld.my.customhome/assets/home.xml
```

### Move the App List Down

The app list position is controlled by `margin` elements stacked above it in `home.xml`. To push it further down, increase the height of the margin directly above the `appList` item. Open the file and adjust the value to your liking — for example, changing `itemHeight="72"` to `itemHeight="300"` or more.

After your edits, `home.xml` should look something like this:

```xml
<?xml version="1.0" encoding="utf-8"?>
<home version="2.0">
<layout windowType="overlay" pageType="none" pageCount="1" defaultPage="0">
<page pageBodyType="container">
<item id="margin" itemWidth="3840" itemHeight="45" focusType="none"/>
<item id="container" hasChildren="true" itemWidth="3840" itemHeight="900" focusType="scope">
<item id="globalline" itemWidth="300" itemHeight="900" focusType="scope" autoFocus="false"/>
<item id="herobanner" itemWidth="3492" itemHeight="900" focusType="scope" autoFocus="false"/>
</item>
<item id="margin" itemWidth="3840" itemHeight="74" focusType="none"/>
<item id="margin" itemWidth="3840" itemHeight="600" focusType="none"/>
<item id="appList" itemWidth="3840" itemHeight="248" focusType="scope" autoFocus="true"/>
<item id="margin" itemWidth="3840" itemHeight="48" focusType="none"/>
<item id="quickGuide" itemX="0" itemY="0" itemWidth="0" itemHeight="0" focusType="none"/>
</page>
</layout>
</home>
```

> **Note:** Do not remove the `herobanner` item — it controls the background image display. Removing it results in a black screen.

---

## Replacing the Hero Banner Image

The hero banner background image is stored in three resolutions:

```
assets/images/hd/bg_banner_img.png
assets/images/2k/bg_banner_img.png
assets/images/4k/bg_banner_img.png
```

Prepare your replacement image in the correct resolution and copy it to all three folders. Run this from your PC:

```sh
scp your_image.png root@<TV-IP>:/media/developer/apps/usr/palm/applications/tld.my.customhome/assets/images/4k/bg_banner_img.png
scp your_image.png root@<TV-IP>:/media/developer/apps/usr/palm/applications/tld.my.customhome/assets/images/2k/bg_banner_img.png
scp your_image.png root@<TV-IP>:/media/developer/apps/usr/palm/applications/tld.my.customhome/assets/images/hd/bg_banner_img.png
```

> The TV will pick the appropriate resolution based on your display. Providing all three ensures compatibility.

---

## Testing

Apply your changes manually:

```sh
sh /media/developer/apps/usr/palm/applications/tld.my.customhome/apply.sh
```

The Home app will restart automatically. If something goes wrong (black screen, crash), simply reboot the TV — the bind mount is not persistent across reboots, so everything returns to its original state.

---

## Making Changes Persistent (Autostart)

Once you're happy with the result, register the script with the `webosbrew` init system so it runs automatically on every boot:

```sh
ln -sf /media/developer/apps/usr/palm/applications/tld.my.customhome/apply.sh \
       /var/lib/webosbrew/init.d/49-custom-homescreen
```

Then reboot to verify:

```sh
reboot
```

---

## Final Directory Layout

```
/media/developer/apps/usr/palm/applications/tld.my.customhome/
├── apply.sh
└── assets/
    ├── home.xml                    (modified layout)
    ├── home_layoutShelfView.xml    (modified shelf layout)
    └── images/
        ├── hd/
        │   └── bg_banner_img.png
        ├── 2k/
        │   └── bg_banner_img.png
        └── 4k/
            └── bg_banner_img.png
```

---

## Notes & Limitations

- The **hero banner text** ("Experience the new with webOS") and its CTA button are rendered by the Flutter app on top of the background image and loaded from LG's servers. They cannot currently be removed via XML modifications alone.
- The exact XML element names and file structure may differ between TV models, regions, and WebOS versions. Always inspect the original files on your specific TV first.
- The overlay is applied in `/tmp` and is lost on reboot — that's intentional and makes this approach safe to experiment with.
