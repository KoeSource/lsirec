# Linux SAS2008/2108 HBA Crossflash - lsirec

```
IT Firmware = HBA Modus
IR Firmware = RAID Modus
```

Für ZFS somit die IT Firmware verwenden. <br>


## lsiutil  installation

---

Zum Crossflashen wird lsiutil benötigt. <br>
lsiutil hat keine Produkt/Hersteller beschränkungen. <br>
Somit können jegliche Karten geflasht werden.

Beispiel installation Debian 11:

```
cd recipes-support/lsiutil/files/lsiutil-1.72.tar.gz
tar -xvf lsiutil-1.72.tar.gz
cd lsiutil
apt-get install build-essential
make -f Makefile_Linux
cd ../lsirec
make
```

## Vorbereitungen

---

IOMMU deaktivieren:

`vi /etc/default/grub`

Wie folgt bearbeiten: <br>
Alt: `GRUB_CMDLINE_LINUX_DEFAULT="quiet"` <br>
Neu : `GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=off amd_iommu=off `

`update-grub`

Reboot

Kontrolle, IOMMU sollte nicht verfügbar sein.

`dmesg|grep -i iommu`

Tip: Falls dies nicht funktioniert, die Virtualisierungsoptionen im BIOS ausschalten.

***Firmware***

[www.boradcom.com](https://www.broadcom.com/) -> Search

9211-8i P20 -> 2118ir.bin, 2118it.bin <br>
UEFI BSD_P20 -> mptsas2.rom, x64sas2.rom

Unzip
```
apt install unzip
unzip Firmware_9211.zip
```

## Crosflass guide  - Kurz

---

***WICHTIG: Zuerst guide unten lesen!*** <br>

`lsiutil -e`

Backup

```
1.  Upload FLASH section
    1. Complete (all sections)
```

```
67.  Dump all port state
68.  Show port state summary
    (SAS WWID wird später benötigt)
```

Löschen

```
33.  Erase non-volatile adapter storage
    3.  FLASH
    8.  Persistent manufacturing config pages
```

Flashen

```
2.  Download firmware (update the FLASH)
    2118it.bin (HBA)
    oder 2118ir.bin (RAID)
```

```
4.  Download/erase BIOS and/or FCode (update the FLASH)
    BIOS = mptsas2.rom
    FCode -> Return, leer lassen
    UEFI = x64sas2.rom
```

`lspci | grep -i SAS2008`

`./lsirec 0000:01:00.0 readsbr sbr_backup.bin`

`python3 sbrtool.py parse sbr_backup.bin sbr.cfg`

sbr.cfg bearbeiten

`python3 sbrtool.py build sbr.cfg sbr_new.bin`

`./lsirec 0000:01:00.0 writesbr sbr_new.bin`

Neustart oder:

`./lsirec 0000:01:00.0 info` <br>
`./lsirec 0000:01:00.0 reset` <br>
`./lsirec 0000:01:00.0 rescan`

`lsiutil -e`

```
1.   Change SAS WWID
```

## Crossflash guide - Lang

---
<br>

***WICHTIG: ALLE DATENTRÄGER ENTFERNTEN!*** <br>
***NICHT FLASHEN WENN DATENTRÄGER ANGESCHLOSSEN SIND!***

In den Pfad wechseln mit der Firmware rom. lsiutil starten.

`lsiutil -e`

Adapter auswählen, Backup erstellen:

```
1.  Upload FLASH section
    1. Complete (all sections)
```

Folgende Ausgaben in einer Datei Speichern:

```
67.  Dump all port state
68.  Show port state summary
    (SAS WWID wird später benötigt)
```

FLASH und Manufacturing Config Pages löschen, kann paar Minuten dauern:

```
33.  Erase non-volatile adapter storage
    3.  FLASH
    8.  Persistent manufacturing config pages
```
Neue Firmware Flashen:

```
2.  Download firmware (update the FLASH)
    2118it.bin (HBA)
    oder 2118ir.bin (RAID)
```

Wenn benötigt kann nun das BIOS und UEFI Option ROM geflasht werden:
```
4.  Download/erase BIOS and/or FCode (update the FLASH)
    BIOS = mptsas2.rom
    FCode -> Return, leer lassen
    UEFI = x64sas2.rom
```

Hinweis : BIOS und UEFI ROMs können geflasht werden.
Somit werden UEFI und BIOS Boot unterstützt.

Hinweis: Das Booten des Rechners dauer länger wenn man das Option ROM verwendet.
Diesen Schritt kann auch übersprungen werden wenn man den Boot-Support nich benötigt.

lsiutil beenden.

PCIe Adresse auslesen, diese muss ggf. angepasst werden bei den nächsten Schritten:

`lspci | grep -i SAS2008`

SBR Backup erstellen, korrekter PCIe Anschluss verwenden!:

`./lsirec 0000:01:00.0 readsbr sbr_backup.bin`

sbr.cfg anpassen, siehe sample_sbr vorlagen.<br>
SASAddr eintragen `SASAddr = 0xYOUR_SAS_WWID`. <br> 
Subsystem VID/PID anpassen oder gleich lassen.

Hinweis: Die SASAddr wird im sbr sehr selten benötigt. <br>
Der Controller funktioniert auch ohne sie. <br>
Da wir aber ein Backup erstellt haben tragen wir sie wieder ein.

`python3 sbrtool.py parse sbr_backup.bin sbr.cfg`

`python3 sbrtool.py build sbr.cfg sbr_new.bin`

`./lsirec 0000:01:00.0 writesbr sbr_new.bin`

Neustart

Die SAS WWID beim Controller hinterlegen. <br>
Hier wird empfohlen die SAS Adresse zu hinterlegen. <br>
Wenn mehrere Controller im selben System verwendet werden dient diese als ID. <br>
(Ist nicht das gleiche wie der SBR)

`lsiutil -e`

```
18.  Change SAS WWID
```

*NEW*: Anstatt ein neustart können folgende befehle verwendet werden:

`./lsirec 0000:01:00.0 info` <br>
`./lsirec 0000:01:00.0 reset` <br>
`./lsirec 0000:01:00.0 rescan`

`reset` Wird am Anfang fehlschlagen da die neue Firmware gestartet werden muss. <br>
Sobald `IOC is READY` steht ist der Controller bereit.<br>
Kann mit `info` abgefragt werden.

<br>

## Probleme

---

**mmap bar1**

Add `iomem=relaxed` to /etc/default/grub.<br>
`update-grub`

<br>

## Disclaimer

---

***Dieses Tool ist nur für Controller mit einem SAS2x08 Chipset ausgelegt.*** <br>
***Verwendung auf eingene Gefahr***

This has barely been tested a couple of cards. Don't blame me if this bricks or
smokes your HBA.

DO NOT attempt to use this tool on non-SAS2x08 chipsets. It probably won't work
and may do horrible things. This tool deliberately does not check the VID/PID
so it can be used on cards with wacky SBRs, but that means it will happily
try to write the SBR into any random PCI device too.

I have tested this on an LSI SAS2108-based MegaRAID card (Fujitsu D2616) with
MegaRAID firmware and it successfully backed up the SBR, but the action
triggered a PCI error in syslog (though the controller kept working). Your
mileage may vary. I have not yet tried crossflashing it live to IT/IR mode.

## License

---

2-clause BSD. See the LICENSE file.
