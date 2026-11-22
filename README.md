# waipu-tv-Root


Bevor ich Code abwerfe, die unbequeme Wahrheit zuerst: **Ein reines `adb shell`-Root-Script gibt es für diesen Stick nicht.** Der Build ist `user/release-keys`, `adbd` läuft nicht als root, es gibt kein `su` im System, und `adb root` scheitert. Root bekommst du nur über die Bootloader-Ebene. Der Stick ist ein SEI700WP (SEI Robotics, Amlogic S905Y4, Android 11) – das ist die v1 von 2021. Falls du die v2 (2025) oder die Android-14-Box hast, greift das hier nicht.

Der einzige aktuell funktionierende Weg (Stand heute, bootloader `01.01.240422.151641`): OEM-Unlock über den Amlogic-DNL-Modus → TWRP nach `boot_a` → **root adb-Shell in TWRP**. Dauerhaftes Magisk im laufenden Android ist noch WIP, weil das Gerät in einen Bootloop läuft.

Deshalb zwei Dateien: ein Host-Script (steuert Unlock, Backup, Flash, Rückweg) und ein On-Device-Script, das per `adb shell` als root ausgeführt wird. Beide gehören ins selbe Verzeichnis.

## Voraussetzungen

- Linux-PC mit `adb`, `fastboot`, `python3`, `pip install adnl`, `sudo`
- **USB-A-auf-USB-A-Kabel** (Männchen/Männchen) – ohne das kein DNL, ohne DNL kein Unlock
- `dnl_send.py` im selben Ordner (steckt im `waipu-unlock-tools.zip` aus dem [XDA-Thread](https://xdaforums.com/t/wip-guide-oem-unlock-waipu-tv-4k-stick-sei700wp-oppen-s905y4-no-soldering.4809559/))
- `twrp_SEI700WP_boot_poweroffhook_avb.img` aus dem [TWRP-Release 0.0.1](https://github.com/Ben-bit-code208/waipu-twrp/releases/tag/0.0.1)
- Debugging an, ADB autorisiert, **nichts Wichtiges auf dem Stick** (Unlock löscht `userdata`)
- Nur Slot `a`. Slot `b` bootloopt selbst mit Stock-Image

## Script 1 – `waipu-root.sh` (Host)

```bash
#!/usr/bin/env bash
# =====================================================================
#  waipu-root.sh - Unlock / TWRP / Root-Werkzeug
#  Ziel : waipu.tv 4K Stick v1 (SEI700WP, S905Y4, Android 11)
#  Test : Bootloader 01.01.240422.151641
#  Nutzung: ./waipu-root.sh <doctor|backup|unlock|twrp|shell|debloat|stock|restore>
# =====================================================================
set -euo pipefail

EXPECT_FP='waiputv/SEI700WP/SEI700WP:11/'
TWRP_IMG='twrp_SEI700WP_boot_poweroffhook_avb.img'
TWRP_SHA='31295c621d702c655064ea765d587f2ac85c0bc95af5c126176878a14dfc8704'
STOCK_BOOT_SHA='9e52620ce0c060aebed5bdf3d2d423b983e6791f49b84cd4f886c76aa9181ce8'
DNL_ID='1b8e:c004'
FB_ID='18d1:0d02'
DEV_SCRIPT="$(dirname "$0")/waipu-rootfs.sh"

YES=0; OUT=''
c_r=$'\e[31m'; c_g=$'\e[32m'; c_y=$'\e[33m'; c_b=$'\e[1m'; c_0=$'\e[0m'
say(){  printf '%s==>%s %s\n' "$c_b" "$c_0" "$*"; }
ok(){   printf '%s ok %s %s\n' "$c_g" "$c_0" "$*"; }
warn(){ printf '%s !! %s %s\n' "$c_y" "$c_0" "$*"; }
die(){  printf '%s xx %s %s\n' "$c_r" "$c_0" "$*" >&2; exit 1; }
confirm(){ [ "$YES" = 1 ] && return 0; printf '%s [j/N] ' "$1"; read -r a
           case "$a" in j|J|y|Y) return 0;; *) die 'abgebrochen';; esac; }

need_adb(){ adb start-server >/dev/null 2>&1 || true
            adb get-state >/dev/null 2>&1 || die 'kein ADB-Gerät verbunden'; }
need_root_pc(){ [ "$(id -u)" = 0 ] || command -v sudo >/dev/null || die 'sudo fehlt'; }

check_device(){
  local fp mf
  fp=$(adb shell getprop ro.build.fingerprint 2>/dev/null | tr -d '\r')
  mf=$(adb shell getprop ro.product.manufacturer 2>/dev/null | tr -d '\r')
  [ -n "$fp" ] || die 'Fingerprint nicht lesbar'
  case "$fp" in
    "$EXPECT_FP"*) ok "Gerät passt: $fp" ;;
    *) warn "unerwarteter Fingerprint: $fp"
       warn "Hersteller: $mf (erwartet SEI)"
       confirm 'Trotzdem weitermachen?' ;;
  esac
}

fix_clock(){ adb shell "date @$(date +%s)" >/dev/null 2>&1 || true; }   # kein RTC, sonst 1970

state_of(){
  echo "state      : $(adb get-state 2>/dev/null)"
  echo "fingerprint: $(adb shell getprop ro.build.fingerprint | tr -d '\r')"
  echo "slot       : $(adb shell getprop ro.boot.slot_suffix | tr -d '\r')"
  echo "verified   : $(adb shell getprop ro.boot.verifiedbootstate | tr -d '\r')"
  echo "vbmeta     : $(adb shell getprop ro.boot.vbmeta.device_state | tr -d '\r')"
  echo "secure     : $(adb shell getprop ro.boot.secure | tr -d '\r')"
  echo "adb uid    : $(adb shell id | tr -d '\r')"
  echo "su vorhanden: $(adb shell 'command -v su || echo NEIN' | tr -d '\r')"
}

# ---------------------------------------------------------------- doctor
doctor(){
  need_adb; say 'Zustand (es wird nichts geändert)'; state_of
  echo; echo '--- Partitionen ---'
  adb shell 'ls -l /dev/block/by-name/ 2>/dev/null' | tr -d '\r' || true
  echo; echo '--- super/mapper ---'
  adb shell 'ls -l /dev/block/mapper/ 2>/dev/null' | tr -d '\r' || true
}

# ---------------------------------------------------------------- backup
backup(){
  need_adb; check_device; fix_clock
  OUT="${OUT:-./waipu-backup-$(date +%Y%m%d-%H%M%S)}"; mkdir -p "$OUT"
  say "Sicherung nach $OUT"
  adb shell getprop          > "$OUT/getprop.txt" 2>/dev/null || true
  adb shell 'cat /proc/cmdline' > "$OUT/cmdline.txt" 2>/dev/null || true
  adb shell 'ls -l /dev/block/by-name/' > "$OUT/by-name.txt" 2>/dev/null || true

  local p dev sha local_sha
  for p in boot_a vendor_boot_a misc vbmeta vbmeta_system dtbo_a; do
    dev="/dev/block/by-name/$p"
    say "lese $p"
    if ! adb shell "dd if=$dev of=/data/local/tmp/bk_$p.img bs=1M" 2>/dev/null; then
      warn "$p nicht lesbar - übersprungen"; continue; fi
    sha=$(adb shell "sha256sum /data/local/tmp/bk_$p.img" | tr -d '\r' | awk '{print $1}')
    adb pull "/data/local/tmp/bk_$p.img" "$OUT/$p.img" >/dev/null 2>&1 || { warn "pull $p fehlgeschlagen"; continue; }
    local_sha=$(sha256sum "$OUT/$p.img" | awk '{print $1}')
    if [ "$sha" = "$local_sha" ]; then ok "$p  $sha"; echo "$sha  $p.img" >> "$OUT/SHA256SUMS"
    else warn "$p Hash-Abweichung ($sha != $local_sha)"; fi
    adb shell "rm -f /data/local/tmp/bk_$p.img" || true
  done

  if [ -f "$OUT/boot_a.img" ]; then
    sha=$(sha256sum "$OUT/boot_a.img" | awk '{print $1}')
    [ "$sha" = "$STOCK_BOOT_SHA" ] && ok 'boot_a ist bit-identisch zum bekannten Stock-Image' \
      || warn "boot_a entspricht NICHT dem Referenz-Stock ($STOCK_BOOT_SHA) - evtl. schon gepatcht"
  fi
  ok "fertig: $OUT"
}

# ---------------------------------------------------------------- unlock
unlock(){
  need_adb; check_device; need_root_pc
  [ -f "$(dirname "$0")/dnl_send.py" ] || die 'dnl_send.py fehlt (aus waipu-unlock-tools.zip)'
  command -v adnl >/dev/null || die 'adnl fehlt: pip install adnl'
  warn 'Unlock löscht userdata/metadata und setzt AVB auf orange. Nicht rückgängig ohne Relock.'
  confirm 'OEM-Unlock jetzt durchführen?'

  say 'Neustart in DNL (USB-Burn-Mode)'
  adb reboot update || true
  for _ in $(seq 30); do sleep 1; lsusb 2>/dev/null | grep -qi "$DNL_ID" && break; done
  lsusb 2>/dev/null | grep -qi "$DNL_ID" \
    || die "DNL nicht gefunden (USB-A/A-Kabel? Port ok? nochmal 'adb reboot update')"
  ok 'DNL erkannt'

  say 'Secure-Boot-Whitelist umgehen, oemlock setzen (das && ist Pflicht)'
  sudo python3 "$(dirname "$0")/dnl_send.py" \
    'oem printenv && setenv oemlock unlock && saveenv' || true
  sudo adnl reboot bootloader
  for _ in $(seq 30); do sleep 1; fastboot devices 2>/dev/null | grep -q . && break; done
  fastboot devices | grep -q . || die 'kein Fastboot-Gerät'
  ok 'U-Boot-Fastboot erkannt'

  fastboot flashing unlock || warn 'unlock meldet Fehler - Status prüfen'
  fastboot getvar unlocked 2>&1 | tail -1
  fastboot getvar secure   2>&1 | tail -1
  fastboot reboot
  sleep 25
  need_adb; fix_clock
  ok "verifiedbootstate: $(adb shell getprop ro.boot.verifiedbootstate | tr -d '\r') (soll: orange)"
}

# ---------------------------------------------------------------- twrp
twrp(){
  need_adb; check_device
  [ -f "$TWRP_IMG" ] || die "$TWRP_IMG fehlt"
  [ "$(sha256sum "$TWRP_IMG" | awk '{print $1}')" = "$TWRP_SHA" ] || die 'TWRP-Hash falsch'
  ok 'TWRP-Image verifiziert'
  [ "$(adb shell getprop ro.boot.verifiedbootstate | tr -d '\r')" = orange ] \
    || warn 'Gerät ist nicht orange/unlocked - TWRP wird evtl. abgelehnt'
  confirm 'TWRP nach boot_a schreiben?'

  fix_clock
  adb push "$TWRP_IMG" /data/local/tmp/twrp_new.img >/dev/null
  adb shell 'dd if=/data/local/tmp/twrp_new.img of=/dev/block/by-name/boot_a bs=4M'
  adb shell sync
  adb shell sha256sum /dev/block/by-name/boot_a | tr -d '\r' | grep -q "$TWRP_SHA" \
    && ok 'boot_a verifiziert' || die 'Hash auf dem Gerät weicht ab - NICHT rebooten, erst prüfen'

  adb reboot
  say 'warte auf TWRP'
  for _ in $(seq 60); do sleep 1; adb get-state >/dev/null 2>&1 && break; done
  sleep 3
  adb shell 'setprop sys.usb.config adb' >/dev/null 2>&1 || true
  adb shell getprop ro.twrp.version | tr -d '\r' | grep -q . && ok 'TWRP läuft' || warn 'TWRP-Prop fehlt'
}

# ---------------------------------------------------------------- shell
shell(){
  need_adb
  [ "$(adb shell id -u | tr -d '\r')" = 0 ] || die 'kein root adb - erst ./waipu-root.sh twrp'
  [ -f "$DEV_SCRIPT" ] || die 'waipu-rootfs.sh fehlt'
  adb push "$DEV_SCRIPT" /data/local/tmp/waipu-rootfs.sh >/dev/null
  adb shell 'sh /data/local/tmp/waipu-rootfs.sh'
  say 'Root-Shell (TWRP). Verlassen mit exit.'
  adb shell
}

# ---------------------------------------------------------------- debloat
debloat(){
  need_adb; fix_clock
  say 'Waipu-Zwangsservices stilllegen (kein Root nötig)'
  for p in com.nes.skywayclient com.nes.coreservice \
           com.nes.leanbacklauncher.partnercustomizer com.nes.remoteota.waipu; do
    adb shell "pm suspend $p" 2>/dev/null && ok "$p suspendiert" || warn "$p fehlgeschlagen"
    adb shell "cmd appops set $p RUN_ANY_IN_BACKGROUND ignore" 2>/dev/null || true
  done
  warn 'skywayclient reaktiviert alles wieder - deshalb ist es oben mit dabei'
  say 'Eigenen Launcher als Home setzen (Beispiel Projectivy):'
  echo '  adb shell cmd package set-home-activity com.spocky.projengmenu/.ui.home.MainActivity'
}

# ---------------------------------------------------------------- zurück
stock(){
  need_adb; fix_clock
  [ -n "$OUT" ] && [ -f "$OUT/boot_a.img" ] || die 'OUT=<backupordner> setzen'
  confirm 'Stock-boot_a zurückschreiben?'
  adb push "$OUT/boot_a.img" /data/local/tmp/stock_boot.img >/dev/null
  adb shell 'dd if=/data/local/tmp/stock_boot.img of=/dev/block/by-name/boot_a bs=4M'
  adb shell sync
  adb shell sha256sum /dev/block/by-name/boot_a | tr -d '\r'
  adb reboot
}

restore(){
  need_adb; fix_clock
  [ -n "$OUT" ] || die 'OUT=<backupordner> setzen'
  for p in boot_a vendor_boot_a misc vbmeta vbmeta_system; do
    [ -f "$OUT/$p.img" ] || { warn "$p fehlt"; continue; }
    confirm "$p zurückschreiben?"
    adb push "$OUT/$p.img" /data/local/tmp/rs_$p.img >/dev/null
    adb shell "dd if=/data/local/tmp/rs_$p.img of=/dev/block/by-name/$p bs=1M"
    adb shell sync
    adb shell "sha256sum /dev/block/by-name/$p" | tr -d '\r'
  done
}

case "${1:-}" in
  doctor) doctor ;;
  backup) backup ;;
  unlock) unlock ;;
  twrp)   twrp ;;
  shell)  shell ;;
  debloat) debloat ;;
  stock)  stock ;;
  restore) restore ;;
  *) sed -n '2,8p' "$0"; exit 1 ;;
esac
```

## Script 2 – `waipu-rootfs.sh` (läuft auf dem Stick, in TWRP)

```sh
#!/system/bin/sh
# Läuft als root in TWRP. Sichert die kritischen Partitionen und prüft den Zustand.
set -u
export PATH=/sbin:/system/bin:/system/xbin:$PATH
D=/data/local/tmp/wbr
L(){ echo "[rootfs] $*"; }

[ "$(id -u)" = 0 ] || { L "kein root - abbrechen"; exit 1; }
L "root bestätigt: $(id)"

# Kein RTC: Uhr stellen, sonst alles auf 1970
date @"$(date +%s)" >/dev/null 2>&1 || true

twrp set tw_enable_adb 1 2>/dev/null || true
twrp set tw_enable_fastboot 0 2>/dev/null || true
twrp set tw_no_screen_timeout 1 2>/dev/null || true

mkdir -p "$D"

L "--- Zustand ---"
L "slot        : $(getprop ro.boot.slot_suffix 2>/dev/null)"
L "twrp        : $(getprop ro.twrp.version 2>/dev/null)"
L "graphics    : $(grep -m1 'graphics' /tmp/recovery.log 2>/dev/null)"

L "--- Sicherung nach $D (von dort per adb pull holen) ---"
for p in boot_a vendor_boot_a misc vbmeta vbmeta_system dtbo_a; do
  src="/dev/block/by-name/$p"
  [ -e "$src" ] || { L "$p nicht vorhanden"; continue; }
  if dd if="$src" of="$D/$p.img" bs=1M 2>/dev/null; then
    L "$p $(sha256sum "$D/$p.img" | awk '{print $1}')"
  else
    L "$p LESEFEHLER"
  fi
done

L "--- misc muss unangetastet bleiben ---"
L "misc sha256: $(sha256sum /dev/block/by-name/misc 2>/dev/null | awk '{print $1}')"
L "erwartet   : 1c16ac591e620d5553b4ab0ef839ad11843c8fee4490462e5f6252b674eb53da"

L "--- Partitionen (Info) ---"
ls -l /dev/block/mapper/ 2>/dev/null
L "system bleibt read-only: EXT4-RW-Mount scheitert an Feature 0x4000 (bekannt)"

L "--- Hinweis ---"
L "Sichere jetzt vom PC aus:  adb pull $D ./waipu-twrp-dump"
L "Zurück zu Android:         ./waipu-root.sh stock OUT=<backupordner>"
```

## Ablauf

```bash
chmod +x waipu-root.sh waipu-rootfs.sh
./waipu-root.sh doctor                 # Zustand ansehen
./waipu-root.sh backup                 # boot_a/vendor_boot_a/misc/vbmeta sichern
./waipu-root.sh unlock                 # DNL -> oemlock -> fastboot flashing unlock
./waipu-root.sh twrp                   # TWRP nach boot_a, Hash verifiziert
./waipu-root.sh shell                  # root adb-Shell in TWRP + Sicherung
```

Danach vom PC: `adb pull /data/local/tmp/wbr ./waipu-twrp-dump`

## Wichtige Grenzen, damit du nicht stolperst

- **`/system` lässt sich in TWRP nicht rw mounten.** Das Recovery-Kernel scheitert an einem `ro_compat`-Feature (`EXT4-fs (dm-0): couldn't mount RDWR ... 4000`). Lesen ja, schreiben nein. Für echte Systemänderungen musst du `system.img` am PC entpacken, per `debugfs` patchen und neu flashen.
- **Magisk bootet derzeit nicht.** Gepatchtes `boot_a` → Bootloop. Rückweg über DNL + Fastboot mit Stock-Image ist möglich, aber fummelig. Deshalb ist TWRP-in-`boot_a` der belastbare Root-Zugang, nicht Magisk.
- **`/sdcard` ist in TWRP nur tmpfs** (`/data` ist FBE-verschlüsselt). TWRP-Backups überleben keinen Reboot → immer per `adb pull` auf den PC.
- **`fastboot boot boot.img` wird ignoriert**, DNL-Memory-Boot ist gesperrt. Nur echtes Flashen hilft.
- **`misc` niemals wipschen** (`boot-recovery` @0, `_a`/`BCAB` @0x800).
- **Slot `b` ist kaputt**, auch mit Stock-Image. Nur `a`.
- Der Stick **sperrt sich gelegentlich selbst wieder** zu. Dann `./waipu-root.sh debloat` bzw. `pm suspend com.nes.skywayclient`.
- Kein RTC: jede ADB-Session braucht `adb shell date @$(date +%s)`, sonst landen Dateien auf 1970 und TLS/auth zickt.

## Ohne Root, wenn dir das reicht

`./waipu-root.sh debloat` killt die vier Waipu-Zwangsdienste (`skywayclient`, `coreservice`, `partnercustomizer`, `remoteota`) und setzt den AppOps. Damit läuft ein eigener Launcher dauerhaft, und `coreservice` blockiert keine Fremd-APKs mehr. Das deckt den Großteil dessen ab, was die meisten eigentlich wollen – ohne Unlock und ohne Datenverlust.

Wenn du willst, baue ich dir als Nächstes den `system.img`-Patch-Weg (debugfs + `su`-Binary + `avbctl disable-verification`) – das ist der einzige Pfad zu echtem, dauerhaftem Root im laufenden Android auf dieser Kiste.