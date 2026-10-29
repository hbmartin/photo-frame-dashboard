# Dromart MI86 (Frameo, rk312x, Android 6.0.1): root and network ADB from an SD card

Record of one unit, 2026-10-28, macOS host. Observations are kept separate from guesses.
Serials, MAC addresses, network addresses, Wi-Fi details and key material are omitted.

**Short version:** nothing on this frame enumerated over USB, and TCP 5555 was closed. Frameo's
manufacturer SD utility ran a root shell from the filename of a dummy APK. A second script on the
same card authorized this computer's ADB key and turned on ADB on port 5555. The connection
authenticated. The boot image and the other firmware partitions were not written by the SD utility. A later
session verified root, captured the firmware, installed Home HQ and WebView 106, and reconnected
authenticated network ADB and root after a reboot. After physical navigation and automatic
dashboard startup were verified, Frameo was removed. A final reboot confirmed Home HQ startup
and authenticated ADB with root, as recorded in section 11.

Match the fingerprint below before using the card. The same filename trick was reported for a
Frameo 159K and is still unreviewed on that hardware
([#126](https://github.com/wesbos/photo-frame-dashboard/issues/126)). This page is the MI86.

The mechanism is the one in
[dusty-du/frameo-159k-jailbreak-builder](https://github.com/dusty-du/frameo-159k-jailbreak-builder),
from [this comment on #126](https://github.com/wesbos/photo-frame-dashboard/issues/126#issuecomment-6460047808).
That report was a 159K, RK3326, Android 8.1, Frameo 1.30.19. The script published here is the
Android 6 adaptation that ran on this MI86.

## 1. Match this unit

Stop at the first difference. `scripts/biuframe-setup.sh` will refuse this fingerprint; leave it
unused.

| | |
| --- | --- |
| About / Support | Frameo. Support screen showed 1280×800 |
| Frameo before replacement | `net.frameo.frame` **v1.32.16\_-\_OTA_test**, versionCode **80321**, under `/data/app` |
| Factory Frameo before replacement | v1.27.5, versionCode **54800**, `/system/app/Frameo`. A photographed manufacturer task during the first probe showed 1.27.5 while the updated package was already on the device |
| Firmware | `wifi photo frame_20250822` |
| Fingerprint | `rockchip/rk312x/rk312x:6.0.1/MXC89K/user.zcx.20250822.060141:user/release-keys` |
| Brand / model | `ro.product.brand=BND`, `ro.product.model=wifi photo frame` |
| Platform | `ro.board.platform=rk312x`, `ro.hardware=rk30board`, `armeabi-v7a` |
| Android | 6.0.1, API 23, `ro.adb.secure=1`, `ro.secure=1`, `ro.debuggable=0` |
| FOTA platform string | `RK3126_6.0` (a firmware property; the case was not opened) |
| SELinux | Permissive |
| `su` on disk | `/system/xbin/su`, 133,472 bytes, dated 2025-08-22. The probe listed it and did not run it |

The frame clock read 31 December 1969 during the probe and a real 2026 date during the later
script. Use the computer's clock for the order of events.

## 2. What was tried before the card

Observed on this unit. These are reasons the SD card was the route, and they are records of what
happened with the cables and buttons that were used.

- **USB stayed absent.** `adb devices` and `fastboot devices` were empty, and the macOS USB
  inventory had no Android or Rockchip device. Checked in normal Frameo, in the red Frameo
  Recovery screen, and in Android Recovery. Also checked with the RESET button held and with
  POWER held, with the barrel supply connected and with USB as the only lead. The cable chain
  used for those tests was USB-C to USB-A to USB-C.
- **TCP 5555 was closed.** A full scan of the frame found one open high port. It accepted a TCP
  connection, returned no banner, reset an HTTP request, reset an ADB handshake, and timed out a
  TLS handshake. After the first card script failed, that port refused connections.
- **Android Recovery** (Supported API 3) offered apply update from ADB, apply update from
  external storage, update rkimage from external storage, apply update from cache, wipe
  data/factory reset, wipe cache partition, and recovery system from backup. Any button cycled
  the highlight; a long press activated it. A short press of Power on the highlighted "reboot
  system now" returned to Android. Touch, swipe, and an attached USB-C keyboard did not move the
  highlight. ADB sideload was not entered. The menu reported `failed to mount /mnt/external_sd`
  while the card was out.
- **Red Frameo Recovery** showed v1.27.5 (54800), with Backup to external storage, Export photos,
  Update Frameo, Install update from external storage, Reset frame, and Hide all photos. USB was
  still absent. No wipe was selected.
- **Android Settings were reachable on the glass**, including Developer options with the master
  switch on. One screen showed Select USB configuration as Charging. A later system log had
  `adb_enabled=1` and `persist.sys.usb.config=mtp,adb`, while the kernel USB state in that same
  log was DISCONNECTED. That is one snapshot of an unplugged or undetected gadget.
- A Jackpal Terminal Emulator APK sitting on the card, plus a macOS `._` copy of it, was removed
  before the utility test. It is a terminal package, so Frameo's update task has nothing to do
  with it.

## 3. What the card changes

Two cards were used, in this order. Both use the same three-file layout.

| Run | Writes on the frame | Result on this unit |
| --- | --- | --- |
| Probe | None. The script writes `mi86-probe.txt` and `mi86-probe.done` on the card | `uid=0(root) gid=0(root) context=u:r:toolbox:s0` |
| ADB enable | Replaces `/data/misc/adb/adb_keys`, sets `service.adb.tcp.port` and `persist.adb.tcp.port` to `5555`, restarts `adbd` | Authenticated `adb` on TCP 5555. Shell uid 2000. A root command in that session returned uid 0, context `u:r:toolbox:s0`. The exact root command was not saved |

The first ADB script (the one that looked for a standalone `base64` command) also ran as root,
then stopped before it replaced the key file or set either port. Section 6 is the script that
finished.

Frameo itself, on card insertion, already runs a root command that appends one line to
`adb_keys`. The line is a key with a comma added in front of `QAAAA`, then `chmod 640` and
`chown system:shell`, then `adbd` is restarted. The card script drops lines with that comma
prefix and keeps the original bytes on the card for undo. The public key and the hostname comment
inside it stay in the private log.

No factory reset was done. At the end of the SD-card work, Frameo was left in place and there
was no partition backup. The later backup and application replacement are recorded in sections
8 and 11; they follow the usual [rules](../README.md#rules-we-learned-the-hard-way).

## 4. How the utility runs the script

Observed in the system log of the failed first enabler, which is the same launch path as the
probe:

1. Mounting the card starts `net.frameo.frame`'s manufacturer utility
   (`AManufacturerUtilities`), then `ASequentialTaskExecutor`.
2. The task looks in `Frameo_utilities` for one APK, checks the package name, and accepts a
   higher versionCode ("APK File valid with newer version code").
3. The install step is a root `chmod 777` of the APK path, wrapped in single quotes. A quote
   inside the filename closes that quote. The next part of the filename is
   `sh $(find /storage -maxdepth 2 -name p_frameo_jb)`, with `${IFS}` standing in for each
   space. `#` comments out the rest of the line, including the trailing `.apk'`.
4. `chmod` then looks for a file named `x` and fails. The shell still runs `sh` on
   `p_frameo_jb`. Frameo reports the exit code of that whole line. Exit 0 becomes the on-screen
   text "Successfully installed APK". A non-zero exit from the script becomes error **1103**
   ("Failed to install APK"). The log of 1103 still shows `id`, `find` and `sh` running as
   `u:r:toolbox:s0`.
5. The active package stayed versionCode 80321. The success text means the shell line exited 0.

The tested sequence was: boot normal Frameo with the card out, open Frameo Settings, insert the
card, wait about 30 seconds. The task draws a **COMPLETE TASK** button.

Inserting the card also delivered `MEDIA_MOUNTED` to `com.adups.fota` and to Rockchip's update
receiver (`RKUpdateReceiver` / `RKUpdateService`). Keep the frame off the internet until those
updaters are identified and disabled over ADB. See [app-updates.md](app-updates.md).

### Card format that the frame read

MBR, one FAT32 partition of about 31 GB, starting at byte offset 1,048,576 (1 MiB), volume label
`FRAMECARD`. The physical card was a 512 GB card. The kernel logged one SD timeout (`-110`) and
then detected the SDXC card. A later read-only `fsck_msdos -n` on the Mac exited 0. The card was
not reformatted.

Exactly one name ending in `.apk` may sit in `Frameo_utilities`. A macOS AppleDouble file
(`._something.apk`) counts. `dot_clean -m` on the card, or `cp -X` when copying, clears those.

```text
FRAMECARD/
├── p_frameo_jb
└── Frameo_utilities/
    ├── frameo_utilities.config
    └── (one APK; the name is shell syntax and is created by prepare_card.py)
```

`frameo_utilities.config` is this line and a newline:

```properties
auto_run=action_update_version
```

The APK is a dummy `net.frameo.frame` package with `hasCode=false`, `minSdkVersion` 23, a v1 JAR
signature from a disposable local certificate, and a versionCode higher than the Frameo already
installed. On this unit that was **80322**, one above 80321. A newer OTA needs a higher
`--version-code`. The certificate is local; it is a signature check, not a Frameo signature.

## 5. Probe first

The probe is [`scripts/dromart-mi86/probe.sh`](../scripts/dromart-mi86/probe.sh)
(1,270 bytes, SHA-256 `421ffd9c25a9ecb74a2fcf7cafbe31bcaff537d38bf73fcadf2f8d8927913c62`).
It records `id`, a short property list, the path of `su`, and SELinux. It does not call `su`,
does not start ADB, and does not change a setting. `mi86-probe.done` makes a second insertion
leave the first log alone.

Stage a directory (this does not touch a card):

```sh
python3 -I scripts/dromart-mi86/prepare_card.py --mode probe --out ./mi86-card
```

Copy the staged files to the FAT32 root. On the Mac used here, `cp -X` avoids new AppleDouble
files. Then check there is still one APK, and eject from Finder.

```sh
cp -X -R ./mi86-card/. "<card-mount-point>/"
dot_clean -m "<card-mount-point>"
find "<card-mount-point>/Frameo_utilities" -name '*.apk' -o -name '._*'
```

Replace `<card-mount-point>` with the mounted card's location before running these commands.
Boot Frameo, open Settings, insert the card, wait for the task, tap **COMPLETE TASK**, shut the
frame down, and read the card on the computer.

| `mi86-probe.txt` | Meaning |
| --- | --- |
| `uid=0(root)` and `context=u:r:toolbox:s0` | The utility ran the script as root. This unit did |
| Some other uid | The script ran as that user. The ADB script in section 6 stops unless `id` is uid 0 |
| No file | The mount, the config, the APK count, the versionCode, or the filename did not reach `sh` |

This unit's probe also recorded `init.svc.adbd=running`, empty `service.adb.tcp.port` and
`persist.adb.tcp.port`, and SELinux `Permissive`.

Remove `mi86-probe.done` only when you intend the probe to log again. Leave the log in your
private diagnostics folder.

## 6. Turn on network ADB

This is a write: one file under `/data/misc/adb/`, two properties, and an `adbd` restart. It
needs the owner's approval for that write. Do it only after the probe's `id` line is root and
the fingerprint still matches.

The script is [`scripts/dromart-mi86/enable-adb.sh`](../scripts/dromart-mi86/enable-adb.sh)
(16,420 bytes, SHA-256 `701a03c339f4c9f8030c0e1e87be99d3e424b0b0d8892a3edc905ff9cc179f7d`).
It is the upstream `p_frameo_jb` template
([blob `479eca15adf773080c792be0e481e72723945936`](https://github.com/dusty-du/frameo-159k-jailbreak-builder/blob/479eca15adf773080c792be0e481e72723945936/templates/p_frameo_jb))
with the changes this frame forced:

- Android 6 has no `base64` command. AOSP builds the applet into toybox and ships no `base64`
  alias. The script self-tests `aGk=` → `hi` with standalone `base64`, then `toybox base64`,
  then `busybox base64`, and keeps the decoder's stderr. This unit printed
  `Base64 implementation: toybox`.
- A key file whose only line is the comma-prefixed vendor line is an acceptable empty baseline.
  The raw bytes are still saved. Any other invalid line aborts before a change.
- `init.svc.adbd` may be `stopping` because Frameo just restarted it. The script waits up to ten
  seconds for `running` or `stopped`. This unit's snapshot was `running`.
- Logs, the public key, and the rollback copies live in `mi86-adb-v2/` on the card, so an older
  `mi86-adb/started` marker cannot block the run.
- The script deletes `Frameo_utilities/frameo_utilities.config` as soon as it starts, so the
  task does not fire again. `mi86-adb-v2/started` and `mi86-adb-v2/done` also block a repeat.
- After replacing the key it sets both TCP properties to 5555, restarts `adbd`, and looks for a
  `LISTEN` state on port 5555 (`15B3` in `/proc/net/tcp` or `tcp6`). A failed check after the
  key file changes rolls the snapshot back.

Put the computer's **public** key on the card. `~/.android/adbkey` stays on the computer.

```sh
python3 -I scripts/dromart-mi86/prepare_card.py --mode adb --replace \
  --pubkey "$HOME/.android/adbkey.pub" --out ./mi86-card
```

`--replace` overwrites `p_frameo_jb`, the config, and the utility APK in that directory. Copy to
the card the same way as the probe. The private key must not be on the card. Eject, boot Frameo,
open Settings, insert the card, wait for the task, tap **COMPLETE TASK**, and leave the frame on
Wi-Fi.

Error 1103 means the shell line exited non-zero. On the first enabler that was the `base64`
failure, and the key file was untouched. Shut down, read `mi86-adb-v2/result.txt`, and keep the
`attempt.*` directory. A `ROLLBACK FAILED` line means the next boot needs that directory before
anyone edits `adb_keys` by hand.

On the run that worked, `result.txt` contained uid 0, `Base64 implementation: toybox`, and
`local keys, persistent TCP port 5555 and listening adbd verified`. The script's own last line
says remote authentication was still untested at that moment. From the computer:

```sh
adb connect <frame-ip>:5555
adb shell id
```

This unit's first packet was an AUTH challenge (`ro.adb.secure=1`). The following connection
authenticated. `adb shell id` was uid 2000 (`u:r:shell:s0`). A root command in the same session
was uid 0 (`u:r:toolbox:s0`); the verification note did not record whether that was `su -c id`
or `su 0 id`. Try a read-only `id` through both forms before assuming either works. Another
computer's key was not tried.

`getprop dhcp.wlan0.ipaddress` was empty while the script ran. The frame was still reached at
the address it already had on the LAN. DHCP can change that address.

`persist.adb.tcp.port` was read back as `5555` before any reboot. The later activation reboot
in section 8 confirmed persistence. Until a reboot is confirmed on another unit, treat network
ADB as a property set in the running system. Anyone
on the LAN who later gets a line into `adb_keys` gets a shell, and
`su` is on the device, so keep the frame on an isolated network.

The installed key file on this unit was the one public key from the card. Ownership stayed
uid 1000, gid 2000, mode 640, label `u:object_r:adb_keys_file:s0` (system:shell, the mode Frameo
had just set). The previous file's only line was the comma-prefixed vendor line, so the cleaned
baseline was empty. The original bytes are on the card at `mi86-adb-v2/adb_keys.before`.

## 7. Undo

Do this from a root shell, or by running a one-shot card script you have read. The copies are on
the card from the successful run.

| Change | Undo |
| --- | --- |
| `/data/misc/adb/adb_keys` replaced with this computer's public key | Put `mi86-adb-v2/adb_keys.before` back, then `chown 1000:2000`, `chmod 640`, and `chcon u:object_r:adb_keys_file:s0`. That backup is the comma-prefixed vendor line, which is what was there |
| `service.adb.tcp.port` and `persist.adb.tcp.port` set to `5555` | `setprop` both back to an empty value and `stop adbd` / `start adbd`. Both were empty before |
| `adbd` restarted | The property undo restarts it again |
| `frameo_utilities.config` removed from the card | Write the one-line config back when you mean the task to run again. Delete `mi86-adb-v2/started` and `mi86-adb-v2/done` or the script returns immediately |

## 8. Verified after access

Later checks on the same unit, 2026-10-28:

| Check | Observed result |
| --- | --- |
| Root | Both `adb shell su 0 id` and `adb shell su -c id` returned uid 0 |
| SoC | `/proc/device-tree/compatible` is `rockchip,rk3126` |
| Kernel / RAM | Linux 3.10.0, armv7l; `MemTotal` 1,021,876 KiB |
| Touch / Wi-Fi | `gslX680`; loaded `aic8800_fdrv` and `aic8800_bsp` modules |
| Display | 1280×800 at density 160; framebuffer native mode 800×1280 |
| Vendor layout | `/vendor` links to `/system/vendor`; no separate vendor partition |
| Network ADB after reboot | Reconnected using the same authorized computer key; `ro.adb.secure=1`, `persist.adb.tcp.port=5555`, root still worked. Runtime `service.adb.tcp.port` was empty after boot, while ADB listened using the persistent setting |
| Firmware backup | Captured the eMMC prefix through the start of userdata, 1,600,126,976 bytes. Exact length, SHA-256 and gzip readback passed; no aligned `0xCC` filler. Boot, recovery and other static regions matched independent reads |
| Live-system caveat | The first system capture had changing ext4 journal/counters and SuperSU inode timestamps. File contents matched. A later complete 1,207,959,552-byte system image, with the `daemonsu` init restart loop stopped and system read-only, matched full-partition hashes before and after transfer |
| Frameo backup | Factory and updated APKs, native libraries, original WebView, shared photo paths, settings snapshots and complete Frameo private data backed up. Static files matched independent live reads; every private regular file matched an independent read |
| Recovery / restore | USB Loader/Maskrom access and raw partition restoration remain untested. No bootloader, kernel, boot, recovery or resource image was written |

The old Android tar skipped a long crash-log directory. A host-created PAX archive of all private
app-data entries supplements the larger archive, including named FIFOs as metadata. Frameo must
be stopped while that snapshot is taken. First put Android Settings in front: force-stopping the
current Home app while it is in front caused Android to launch it again. Keep vendor APKs,
private data, keys and firmware images out of version control.

## 9. Guesses

- The single high TCP port belonged to Frameo. The Denver PFF-1024BMK2, on Frameo v1.32.16, had
  one high port owned by `net.frameo.frame` and no listeners once Frameo was disabled. The owner
  of this unit's port was not read from `/proc`.
- USB data is missing because of the cable, the connector, or the gadget configuration. The log
  showed USB debugging enabled and the kernel gadget disconnected. That does not choose among
  those causes.

## 10. Another unit

Send the fingerprint, both Frameo versionCodes, whether `mi86-probe.txt` shows uid 0, and which
`Base64 implementation:` line `result.txt` printed. Leave out the serial, MAC, IP, volume id, and
the key file. A different fingerprint or a different versionCode check is a different frame:
stop and write that down, including a failed probe.

## 11. Launcher and dashboard replacement

The owner authorized replacement of Frameo after backup, with a working replacement and a
successful ADB reconnect after reboot required before removal.

- Installed Home HQ from this repository, minimum API 23 / target 28, using compile SDK 35
  and build-tools 36.1.0. It displayed the repo's demo data; no live calendar, weather or
  family account was connected.
- All seven dashboard views rendered on the frame, fonts loaded, and the displayed sample
  images had nonzero decoded dimensions.
- The factory Launcher3 manifest has MAIN with DEFAULT and MONKEY categories, with HOME
  removed. Enabling its package does not make it an ordinary Home option. Installed the pinned
  Rootless Pixel Launcher 3.9.1 from the F-Droid URL and SHA-256 in `biuframe-setup.sh` instead.
  Its Home screen and the floating Home HQ navigation overlay were visually checked.
- WebView 44 was replaced with the repo-pinned LineageOS WebView 106.0.5249.126, including its
  three ARM native libraries in `lib/arm`; the original APK also remains as `webview.apk.bak`.
  All installed files matched their host hashes, and system was remounted read-only. Home HQ
  failed to initialize before reboot while Android still had the old package metadata; it
  started successfully once the reboot activated WebView 106.
- Enabled the native Menu button and Home HQ navigation service, with its optional
  WRITE_SECURE_SETTINGS grant. Back and Dashboard controls appear outside Home HQ. The owner
  physically opened Menu → Android settings → Apps and returned with Dashboard successfully.
- This firmware overwrites the ordinary Home preference at boot, even with Frameo disabled:
  logs show it adding `net.frameo.frame/.ui.activities.AFirstScreen` and removing that entry as
  dangling. Choosing Home HQ with **Always** therefore does not survive a restart reliably.
  Setting a persistent preferred activity through PackageManager's
  `addPersistentPreferredActivity` API instead resolved MAIN / HOME / DEFAULT to
  `com.wesbos.homehq/.Home`, user 0. Android 6 requires the caller to be **system UID 1000**;
  root UID 0 alone is rejected. No device-owner policy or framework patch was installed.
- Package settings are written asynchronously. Wait for the entry to appear under
  `persistent-preferred-activities` in `/data/system/users/0/package-restrictions.xml` before
  rebooting; the observed delay was 10 seconds. The persistent preference must be cleared
  with `clearPackagePersistentPreferredActivities("com.wesbos.homehq", 0)` as UID 1000 before
  selecting a different default Home or removing Home HQ.
- With Frameo reversibly disabled, a reboot launched Home HQ directly, rendered the
  dashboard, retained the navigation service, and reconnected authenticated ADB with root in
  50.6 seconds. The boot log's initial HOME launch was Home HQ, with no chooser interaction.
- After that successful check, removed only the backed-up factory `/system/app/Frameo` and
  updated `/data/app/net.frameo.frame-1` code directories. Shared photo paths were not deleted.
  A final reboot again launched Home HQ directly and rendered the dashboard, with authenticated
  ADB, root, secure ADB authorization, the persistent Home preference and accessibility service
  enabled. Both APK directories were absent; `pm path` and package listings, including `-u`,
  returned no Frameo package. Android also cleaned its private data directory during package
  reconciliation; the complete private-data backup was retained. Shared photo
  directories remained present, and `/system` was read-only. WebView remote debugging was off.
