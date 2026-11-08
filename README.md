# Retroflag GPi Case 1 + Raspberry Pi Zero 2 W + RetroPie: Kein Ton – dokumentierte Fehlerbehebung

**Status:** Erfolgreich am beschriebenen Gerät getestet; ALSA-Geräte nach Neustart vorhanden.  
**Stand:** 8. November 2026  
**Sprache:** Deutsch  
**Hinweis:** Diese Anleitung beschreibt einen konkreten Fehlerfall, keine universelle Lösung für alle GPi-Case-Images.

## 1. Ausgangslage

| Komponente | Ermittelter Stand |
|---|---|
| Gehäuse | Original Retroflag GPi Case 1 |
| Rechner | Raspberry Pi Zero 2 W Rev 1.0 |
| Betriebssystem | RetroPie, ältere Installation (ca. 2022) |
| Kernel | `5.10.103-v7+` |
| Firmware | 1. Dezember 2021 |
| Bildschirm | DPI, 320 × 240, funktionierte bereits |
| Audio | Kein Ton; EmulationStation zeigte 0 % / On-board audio disabled |

**Wichtig:** Die folgenden Befehle sind nur für Installationen mit vergleichbarem Kernel, Treiber und Audioaufbau gedacht. Andere Images können andere Overlay- und Audioverfahren verwenden.

## 2. Fehlerbild

```bash
aplay -l
```

Ergebnis vor der Reparatur:

```text
aplay: device_list:272: no soundcards found...
```

Weitere Befunde:

- `/proc/asound/cards` meldete keine Soundkarten.
- `snd_bcm2835` war als Kernelmodul geladen, aber es gab keine ALSA-Playback-Geräte.
- Der Treiber war an `bcm2835_audio` gebunden.
- Der Firmware-Bootlog bestätigte das Laden von `dpi24` und `pwm-audio-pi-zero`.
- `dtoverlay -l` zeigte `No overlays loaded`. Das ist kein Nachweis, dass die Boot-Overlays fehlten: Das Kommando zeigt Laufzeit-Overlays an.
- In `/boot/overlays` waren `dpi24.dtbo` und `pwm-audio-pi-zero.dtbo` vorhanden; `audremap.dtbo` und `pwm-2chan.dtbo` fehlten.
- `dmesg` enthielt `bcm2835_audio bcm2835_audio: there is not valid maps for state default`. **Nicht nachgewiesen**, dass diese Meldung den Fehler verursachte.

## 3. Entscheidender Diagnosebefund

```bash
cat /proc/cmdline
```

In der laufenden Kernel-Befehlszeile stand unter anderem:

```text
snd_bcm2835.enable_compat_alsa=0 snd_bcm2835.enable_hdmi=1
```

Auch nach vorherigen Konfigurationsversuchen waren die aktiven Parameter:

```text
enable_compat_alsa: N
enable_hdmi: Y
enable_headphones: N
```

Mit `modprobe -c` und `sudo modprobe --show-depends snd_bcm2835` wurden widersprüchliche Einstellungen sichtbar, unter anderem `enable_compat_alsa=1` und `enable_compat_alsa=0`. Die Herkunft der zusätzlich vom Kernel übernommenen Werte wurde **nicht abschließend nachgewiesen**; sie standen nicht in der ausgelesenen `/boot/cmdline.txt`.

## 4. Erfolgreicher Soforttest

Der entscheidende Befehl war:

```bash
sudo modprobe -r snd_bcm2835 && sudo modprobe snd_bcm2835 enable_compat_alsa=1
```

**Achtung:** Das Entladen scheitert mit `Module snd_bcm2835 is in use`, wenn ein Programm das Audiogerät geöffnet hält. Nicht gewaltsam entladen; zunächst Audioanwendungen beenden. Dieser Test verändert nicht dauerhaft die Bootkonfiguration.

Danach zeigte:

```bash
aplay -l
```

unter anderem:

```text
card 0: ALSA [bcm2835 ALSA], device 0: bcm2835 ALSA
card 1: b1 [bcm2835 HDMI 1], device 0: bcm2835 HDMI 1
card 2: Headphones [bcm2835 Headphones], device 0: bcm2835 Headphones
```

Ein Testton wurde über den GPi-Case-Lautsprecher **hörbar**:

```bash
speaker-test -D hw:0,0 -c 2 -t sine -l 1
```

Die tatsächlich aktive Einstellung wurde mit folgendem Befehl kontrolliert:

```bash
cat /sys/module/snd_bcm2835/parameters/enable_compat_alsa
```

Ergebnis: `Y`.

**Einordnung:** Der Test beweist, dass das Aktivieren des ALSA-Kompatibilitätsmodus auf dieser Installation die Audio-Geräte verfügbar machte und der Lautsprecher funktionierte. Er beweist nicht, dass dies die einzige mögliche Ursache für jeden GPi-Case-Tonausfall ist.

## 5. Dauerhafte Änderung auf dem getesteten Gerät

Vor der Änderung wurde eine Sicherung erstellt:

```bash
sudo cp /boot/cmdline.txt /boot/cmdline.txt.audio-backup
```

Danach wurde der Parameter am Ende der vorhandenen einzigen Zeile von `/boot/cmdline.txt` ergänzt:

```bash
sudo sed -i 's/$/ snd_bcm2835.enable_compat_alsa=1/' /boot/cmdline.txt
```

**Nur einmal ausführen:** Bei wiederholter Ausführung wird der Parameter mehrfach angehängt. Zuerst mit `cat /boot/cmdline.txt` kontrollieren, ob er bereits vorhanden ist. Die Datei muss eine einzige Zeile bleiben.

Die bestätigte Datei lautete nach der Änderung:

```text
console=serial0,115200 console=tty1 root=PARTUUID=aa23456d-02 rootfstype=ext4 fsck.repair=yes rootwait loglevel=3 consoleblank=0 plymouth.enable=0 snd_bcm2835.enable_headphones=1 snd_bcm2835.enable_compat_alsa=1
```

**Nicht blind kopieren:** `root=PARTUUID=...` und die übrigen Werte sind installationsspezifisch. Auf einem anderen Gerät nur den Audio-Parameter ergänzen, nicht die gesamte Beispielzeile ersetzen.

Zusätzlich wurde folgende Modul-Konfiguration angelegt:

```bash
echo 'options snd_bcm2835 enable_compat_alsa=1' | sudo tee /etc/modprobe.d/99-gpi-audio.conf
```

Eine zuvor testweise angelegte Konfigurationsdatei wurde durch Umbenennen deaktiviert:

```bash
sudo mv /etc/modprobe.d/snd-bcm2835.conf /etc/modprobe.d/snd-bcm2835.conf.bak
```

**Wichtig:** Die Modul-Konfiguration allein hatte den Parameterkonflikt zunächst nicht beseitigt. Erst nach dem zusätzlichen Eintrag in `/boot/cmdline.txt` wurde neu gestartet und die Erkennung erneut geprüft. Der jeweilige Einzelbeitrag beider dauerhaften Einstellungen wurde nicht isoliert getestet.

## 6. Neustart und bestätigtes Ergebnis

```bash
sudo reboot
```

Nach dem Neustart war das System wieder erreichbar und:

```bash
aplay -l
```

zeigte weiterhin `card 0: ALSA [bcm2835 ALSA]`, außerdem HDMI- und Headphones-Geräte.

**Bestätigt:** ALSA-Soundkarten bleiben nach dem Neustart verfügbar.  
**Noch nicht separat bestätigt:** Ein zweiter hörbarer Testton nach genau diesem Neustart; der hörbare Testton wurde vor dem Neustart durchgeführt.

Für die abschließende Kontrolle nach dem Neustart:

```bash
cat /sys/module/snd_bcm2835/parameters/enable_compat_alsa
speaker-test -D hw:0,0 -c 2 -t sine -l 1
```

## 7. Was nicht geholfen hat und was man vermeiden sollte

1. **Nur `dtparam=audio=on`**: War bereits gesetzt; trotzdem wurden keine ALSA-Soundkarten erkannt.
2. **Boot-Overlays als fehlend interpretieren**: `dtoverlay -l` allein reicht dafür nicht. Der Firmware-Bootlog zeigte die geladenen Overlays.
3. **Nicht vorhandene Overlays eintragen**: `audremap` und `pwm-2chan` waren auf dieser Installation nicht vorhanden. Nicht ungeprüft eintragen.
4. **`enable_headphones=1` allein**: Führte nicht zur Wiederherstellung der Soundkarten.
5. **Widersprüchliche Moduloptionen**: `enable_compat_alsa=1` und `enable_compat_alsa=0` gleichzeitig erschwerten die Diagnose.
6. **`kernel=zImage` aktivieren**: Auf diesem Gerät führte ein Verweis auf die fehlende Kerneldatei zu einem Bootproblem. Der Eintrag blieb auskommentiert: `#kernel=zImage`.
7. **Unkontrollierte Paket-/Firmware-Upgrades**: Die ältere Distribution hatte nicht mehr problemlos erreichbare Paketquellen. Ein Upgrade war für die erfolgreiche Audio-Reparatur nicht erforderlich.

## 8. Wiederherstellung bei Problemen

Wenn das Gerät noch per SSH erreichbar ist, kann die gesicherte Kernel-Befehlszeile wiederhergestellt werden:

```bash
sudo cp /boot/cmdline.txt.audio-backup /boot/cmdline.txt
```

Die zusätzlich angelegte Modul-Konfiguration kann deaktiviert werden:

```bash
sudo mv /etc/modprobe.d/99-gpi-audio.conf /etc/modprobe.d/99-gpi-audio.conf.disabled
```

**Hinweis:** Diese Befehle sind Rücknahmeoptionen für die beschriebenen Änderungen. Vor dem Neustart kontrollieren, ob die ursprüngliche Datei korrekt ist. Die vorhandene Displaykonfiguration in `/boot/config.txt` nicht verändern.

## 9. Kurzfassung für Forum / GitHub

**Problem:** Original Retroflag GPi Case 1 mit Raspberry Pi Zero 2 W und älterem RetroPie: kein Ton, `aplay -l` meldet `no soundcards found`, obwohl `snd_bcm2835` geladen ist.

**Diagnose:** `cat /sys/module/snd_bcm2835/parameters/enable_compat_alsa` zeigte `N`. Ein Test mit `sudo modprobe -r snd_bcm2835 && sudo modprobe snd_bcm2835 enable_compat_alsa=1` stellte ALSA-Geräte wieder her; `speaker-test -D hw:0,0 -c 2 -t sine -l 1` war hörbar.

**Dauerhafte Lösung auf diesem Gerät:** `snd_bcm2835.enable_compat_alsa=1` am Ende der einzigen Zeile von `/boot/cmdline.txt` ergänzen; zusätzlich wurde `/etc/modprobe.d/99-gpi-audio.conf` mit `options snd_bcm2835 enable_compat_alsa=1` angelegt. Nach Neustart erkannte `aplay -l` die Soundkarten weiterhin.

**Warnung:** Vorher sichern, nicht die komplette `cmdline.txt` eines fremden Systems übernehmen, keine funktionierende DPI-Displaykonfiguration verändern. Auf anderen Kernel-/Image-Versionen können andere Schritte nötig sein.
