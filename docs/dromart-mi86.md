# Dromart MI86 (Frameo, rk312x, Android 6.0.1): root and network ADB from an SD card

Record of one unit, 2026-10-28, macOS host. Observations are kept separate from guesses.
Serials, MAC addresses, Wi-Fi details and key material are left out; they stay in the ignored
`diagnostics/dromart-mi86/` folder.

**Short version:** nothing on this frame enumerated over USB, and TCP 5555 was closed. Frameo's
manufacturer SD utility ran a root shell from the filename of a dummy APK. A second script on the
same card authorized this computer's ADB key and turned on ADB on port 5555. The connection
authenticated. The boot image and the other firmware partitions were not written. A reboot was
not tested, and there is no eMMC backup yet.

Match the fingerprint below before using the card. The same filename trick was reported for a
Frameo 159K and is still unreviewed on that hardware
([#126](https://github.com/wesbos/photo-frame-dashboard/issues/126)). This page is the MI86.

Checked on 2026-10-28: no open issue or pull request named this device. The mechanism is the one
in [dusty-du/frameo-159k-jailbreak-builder](https://github.com/dusty-du/frameo-159k-jailbreak-builder),
from [this comment on #126](https://github.com/wesbos/photo-frame-dashboard/issues/126#issuecomment-6460047808).
That report was a 159K, RK3326, Android 8.1, Frameo 1.30.19. The script published here is the
Android 6 adaptation that ran on this MI86.

## 1. Match this unit

Stop at the first difference. `scripts/biuframe-setup.sh` will refuse this fingerprint; leave it
unused.

| | |
| --- | --- |
| About / Support | Frameo. Support screen showed 1280×800 |
| Running Frameo | `net.frameo.frame` **v1.32.16\_-\_OTA_test**, versionCode **80321**, under `/data/app` |
| Factory Frameo | v1.27.5, versionCode **54800**, `/system/app/Frameo`. A photographed manufacturer task during the first probe showed 1.27.5 while the updated package was already on the device |
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

No factory reset was done. Frameo was left in place. There is no partition backup. Further writes
(`/system`, disabling Frameo, a kiosk install) wait on a backup and on the usual
[rules](../README.md#rules-we-learned-the-hard-way).

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
python3 -I scripts/dromart-mi86/prepare_card.py --mode probe --out /tmp/mi86-card
```

Copy the staged files to the FAT32 root. On the Mac used here, `cp -X` avoids new AppleDouble
files. Then check there is still one APK, and eject from Finder.

```sh
cp -X -R /tmp/mi86-card/. /Volumes/FRAMECARD/
dot_clean -m /Volumes/FRAMECARD
find /Volumes/FRAMECARD/Frameo_utilities -name '*.apk' -o -name '._*'
```

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
  --pubkey "$HOME/.android/adbkey.pub" --out /tmp/mi86-card
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

`persist.adb.tcp.port` was read back as `5555` before any reboot. Whether it is still 5555 after
a reboot was **not** tested. Until that reboot is confirmed, treat network ADB as a property set
in the running system. Anyone on the LAN who later gets a line into `adb_keys` gets a shell, and
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

## 8. After ADB answers

Still open on this unit:

- A verified eMMC backup (byte count, SHA-256, gzip readback, a `0xCC` check). The key-file
  snapshot is the undo for section 6. It is not a firmware backup.
- Read `su -c id` and, if `-c` is rejected, `su 0 id`. The binary is on disk; the probe did not
  execute it.
- Reboot, then `getprop persist.adb.tcp.port` and `adb connect` again.
- `pm list packages -f` and the WebView version. Neither was recorded. `com.adups.fota` and
  Rockchip's update receiver both woke on card mount; identify them before the frame goes online.
- Partition map, RAM, and `ro.config.low_ram`. Unknown here. Do not reuse another RK3126
  frame's offsets.
- Launcher, kiosk, and disabling Frameo, only after the backup and after the owner has opened
  Android Settings → Apps from the panel with Frameo still enabled. Settings were reachable
  before this work. That check has to be repeated by hand before Frameo is turned off.

## 9. Guesses

- The SoC is an RK3126. The evidence is `ro.board.platform=rk312x`, `ro.hardware=rk30board`, and
  the FOTA string `RK3126_6.0`, which is the same family as the other rk312x frames in the
  [device index](devices.md). No loader chip id and no board photo.
- The single high TCP port belonged to Frameo. The Denver PFF-1024BMK2, on Frameo v1.32.16, had
  one high port owned by `net.frameo.frame` and no listeners once Frameo was disabled. The owner
  of this unit's port was not read from `/proc`.
- USB data is missing because of the cable, the connector, or the gadget configuration. The log
  showed USB debugging enabled and the kernel gadget disconnected. That does not choose among
  those causes.
- `persist.adb.tcp.port` will still be 5555 after a reboot, because that is what the property is
  for on other frames in this repo. This unit has not been rebooted to check.

## 10. Another unit

Send the fingerprint, both Frameo versionCodes, whether `mi86-probe.txt` shows uid 0, and which
`Base64 implementation:` line `result.txt` printed. Leave out the serial, MAC, IP, volume id, and
the key file. A different fingerprint or a different versionCode check is a different frame:
stop and write that down, including a failed probe.
