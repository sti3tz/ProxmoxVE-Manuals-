
# Proxmox VE – ZFS Backup Storage Management

> Praxisanleitung zur Verwaltung, Überwachung und Erweiterung eines bestehenden ZFS-Backup-Datasets unter Proxmox VE.

**Kategorie:** Storage Management  
**Plattform:** Proxmox VE / OpenZFS  
**Schwierigkeitsgrad:** Einsteiger bis Fortgeschrittene  
**Version:** 1.0  
**Sprache:** Deutsch

---

## Inhaltsverzeichnis

1. [Zielsetzung](#1-zielsetzung)
2. [Grundlagen und Architektur](#2-grundlagen-und-architektur)
3. [Voraussetzungen](#3-voraussetzungen)
4. [Bestehenden Storage überprüfen](#4-bestehenden-storage-überprüfen)
5. [ZFS-Dataset um 20 GiB erweitern](#5-zfs-dataset-um-20-gib-erweitern)
6. [Storage-Konfiguration validieren](#6-storage-konfiguration-validieren)
7. [Backup-Funktion testen](#7-backup-funktion-testen)
8. [ZFS-Einstellungen optimieren](#8-zfs-einstellungen-optimieren)
9. [Monitoring und Wartung](#9-monitoring-und-wartung)
10. [Fehlerbehebung](#10-fehlerbehebung)
11. [Rollback](#11-rollback)
12. [Checkliste](#12-checkliste)
13. [Befehlsverzeichnis](#13-befehlsverzeichnis)
14. [Glossar](#14-glossar)

---

## 1. Zielsetzung

Diese Anleitung beschreibt, wie ein bereits eingerichteter ZFS-Backup-Speicher unter Proxmox VE überprüft, verwaltet und erweitert werden kann.

**Wichtig:** Das ZFS-Dataset existiert bereits. Eine erneute Erstellung ist nicht erforderlich.

Nach Abschluss dieser Anleitung kannst du:

- Den Zustand eines ZFS-Pools kontrollieren.
- Die Auslastung eines Backup-Datasets analysieren.
- Die Speicherbegrenzung (Quota) verändern.
- Die Einbindung in Proxmox überprüfen.
- Die Funktionsfähigkeit des Backup-Speichers testen.
- Grundlegende ZFS-Einstellungen bewerten.
- Fehler systematisch eingrenzen.

### Anwendungsbeispiel

Ein Proxmox-Administrator verwendet ein bestehendes ZFS-Dataset zur Speicherung automatischer VM-Backups.

Die bisherige Quota beträgt 80 GiB.

Da künftig weitere virtuelle Maschinen gesichert werden sollen, wird die Quota um 20 GiB auf 100 GiB erhöht.

Dabei soll weder das Dataset neu erstellt noch die vorhandene Backup-Konfiguration verändert werden.

---

## 2. Grundlagen und Architektur

### 2.1 Beispielumgebung

| Komponente | Beispielwert |
|---|---|
| Hypervisor | Proxmox VE |
| Node | `pve` |
| ZFS-Pool | `zfspool` |
| Dataset | `zfspool/backups` |
| Mountpoint | `/zfspool/backups` |
| Proxmox-Storage-ID | `zfs-backups` |
| Storage-Typ | Directory |
| Ursprüngliche Quota | 80 GiB |
| Ziel-Quota | 100 GiB |
| Komprimierung | LZ4 |
| Atime | Deaktiviert |
| Deduplication | Deaktiviert |

Die Namen sind Beispiele und müssen an die eigene Infrastruktur angepasst werden.

### 2.2 Speicherarchitektur

```text
Proxmox VE
│
├── ZFS-Pool: zfspool
│   │
│   ├── VM-Volumes
│   │   ├── vm-100-disk-0
│   │   └── ...
│   │
│   └── Dataset: zfspool/backups
│       │
│       ├── Mountpoint: /zfspool/backups
│       │
│       └── dump/
│           ├── vzdump-qemu-100-....vma.zst
│           ├── vzdump-lxc-101-....tar.zst
│           └── ...
│
└── Proxmox Storage
    └── zfs-backups
        └── Directory: /zfspool/backups
```

### 2.3 Was ist eine ZFS-Quota?

Eine Quota definiert, wie viel Speicherplatz ein Dataset maximal verwenden darf.

Beispiel:

```text
Physischer ZFS-Pool:       461 GiB
Backup-Dataset-Quota:       80 GiB
Gewünschte Erweiterung:    20 GiB
Neue Dataset-Quota:       100 GiB
```

Die Quota stellt keine physische Speicherreservierung dar.

Eine Erweiterung von 80 GiB auf 100 GiB bedeutet lediglich, dass das Dataset künftig bis zu 100 GiB des vorhandenen Pool-Speichers verwenden darf.

---

## 3. Voraussetzungen

Vor Beginn müssen folgende Voraussetzungen erfüllt sein:

- Proxmox VE ist installiert und erreichbar.
- Ein funktionsfähiger ZFS-Pool ist vorhanden.
- Das Backup-Dataset existiert bereits.
- Das Dataset ist in Proxmox als Storage eingebunden.
- Administratorzugriff auf den Proxmox-Host ist vorhanden.
- Ausreichend freier Pool-Speicher ist verfügbar.

### Sicherheitshinweise

Die folgenden Befehle dürfen nicht ohne Prüfung des jeweiligen Pool- und Dataset-Namens übernommen werden.

Insbesondere darf kein vorhandenes Dataset gelöscht oder neu erstellt werden.

Befehle wie:

```bash
zfs destroy
zpool destroy
```

sind für die hier beschriebene Erweiterung nicht erforderlich.

---

## 4. Bestehenden Storage überprüfen

Vor jeder Änderung muss der Ausgangszustand ermittelt werden.

### 4.1 ZFS-Pool überprüfen

**Ziel:** Feststellen, ob der ZFS-Pool betriebsbereit ist.

```bash
zpool status -v zfspool
```

**Erwartetes Ergebnis (Beispiel):**

```text
pool: zfspool
state: ONLINE

NAME       STATE    READ WRITE CKSUM
zfspool    ONLINE      0     0     0

errors: No known data errors
```

**Bewertung:**

| Wert | Bedeutung |
|---|---|
| ONLINE | Pool ist betriebsbereit |
| DEGRADED | Redundanz oder Geräteverfügbarkeit eingeschränkt |
| FAULTED | Schwerwiegendes Problem |
| READ | Erkannte Lesefehler |
| WRITE | Erkannte Schreibfehler |
| CKSUM | Prüfsummenfehler |

Bei `DEGRADED`, `FAULTED` oder auffälligen Fehlerzählern sollte zuerst die Ursache untersucht werden.

### 4.2 Dataset identifizieren

**Ziel:** Prüfen, ob das richtige Dataset vorhanden ist.

```bash
zfs list
```

Für das konkrete Dataset:

```bash
zfs list -o name,used,avail,refer,quota,mountpoint zfspool/backups
```

**Erwartetes Ergebnis (Beispiel):**

```text
NAME             USED  AVAIL  REFER  QUOTA  MOUNTPOINT
zfspool/backups  10G   70G    10G    80G    /zfspool/backups
```

Die tatsächlichen Werte hängen von der Speicherbelegung ab.

**Interpretation:**

- `USED`: Durch das Dataset einschließlich gegebenenfalls vorhandener abhängiger Daten belegter Speicher.
- `AVAIL`: Unter Berücksichtigung der ZFS-Grenzen verfügbarer Speicher.
- `REFER`: Direkt durch das Dataset referenzierter Speicher.
- `QUOTA`: Maximale erlaubte Speicherbelegung.
- `MOUNTPOINT`: Einhängepunkt im Dateisystem.

### 4.3 Mountpoint überprüfen

**Ziel:** Sicherstellen, dass Backup-Dateien tatsächlich auf dem ZFS-Dataset gespeichert werden.

```bash
findmnt -T /zfspool/backups
```

**Erwartetes Ergebnis:**

```text
SOURCE           FSTYPE
zfspool/backups  zfs
```

**Warum ist diese Prüfung wichtig?**

Wenn der konfigurierte Backup-Pfad falsch geschrieben oder das Dataset nicht eingehängt ist, können Backups versehentlich auf dem Root-Dateisystem landen.

Dies kann dazu führen, dass der Proxmox-Systemdatenträger vollständig belegt wird.

### 4.4 Proxmox-Storage überprüfen

```bash
pvesm status
```

**Erwartetes Ergebnis:**

```text
Name          Type  Status
zfs-backups   dir   active
```

Der Storage muss aktiv sein.

---

## 5. ZFS-Dataset um 20 GiB erweitern

### 5.1 Ausgangssituation

| Parameter | Wert |
|---|---|
| Aktuelle Quota | 80 GiB |
| Erweiterung | 20 GiB |
| Ziel-Quota | 100 GiB |

### 5.2 Verfügbaren Pool-Speicher prüfen

```bash
zfs list -o name,used,avail zfspool
```

**Ziel:** Sicherstellen, dass ausreichend Speicherplatz für die zusätzliche Nutzung vorhanden ist.

Eine Quota-Erweiterung ist nur dann betrieblich sinnvoll, wenn der Pool über ausreichende Reserven verfügt.

### 5.3 Aktuelle Quota kontrollieren

```bash
zfs get quota zfspool/backups
```

**Erwartetes Ergebnis:**

```text
NAME             PROPERTY  VALUE
zfspool/backups  quota     80G
```

### 5.4 Quota verändern

```bash
zfs set quota=100G zfspool/backups
```

**Was passiert?**

ZFS verändert die maximal zulässige Speicherbelegung des Datasets.

Vorhandene Dateien bleiben bestehen.

Es erfolgt:

- Keine Neuerstellung des Datasets
- Keine Formatierung
- Keine Migration
- Kein Neustart
- Keine Änderung des Mountpoints

### 5.5 Änderung überprüfen

```bash
zfs get quota zfspool/backups
```

**Erwartetes Ergebnis:**

```text
NAME             PROPERTY  VALUE
zfspool/backups  quota     100G
```

Zusätzlich:

```bash
zfs list -o name,used,avail,quota zfspool/backups
```

**Erfolgsbedingung:**

Die neue Quota wird angezeigt und das Dataset bleibt verfügbar.

---

## 6. Storage-Konfiguration validieren

### 6.1 Proxmox-Konfiguration prüfen

```bash
cat /etc/pve/storage.cfg
```

Beispiel:

```text
dir: zfs-backups
    path /zfspool/backups
    content backup
    nodes pve
    shared 0
```

Der entscheidende Punkt ist:

```text
path /zfspool/backups
```

Der Pfad muss exakt mit dem tatsächlichen ZFS-Mountpoint übereinstimmen.

### 6.2 Storage-Status prüfen

```bash
pvesm status
```

**Erwartetes Ergebnis:**

`zfs-backups` wird als `active` angezeigt.

### 6.3 Backup-Dateien kontrollieren

```bash
ls -lh /zfspool/backups/dump/
```

Die bereits vorhandenen Sicherungen müssen weiterhin sichtbar sein.

---

## 7. Backup-Funktion testen

Nach der Erweiterung sollte die Funktionalität des Storage geprüft werden.

### Test 1 – Mountpoint

```bash
findmnt -T /zfspool/backups
```

**Erwartung:** Das Dataset wird als ZFS-Dateisystem erkannt.

### Test 2 – Schreibzugriff

```bash
touch /zfspool/backups/.storage-test
```

Datei kontrollieren:

```bash
ls -l /zfspool/backups/.storage-test
```

Anschließend entfernen:

```bash
rm /zfspool/backups/.storage-test
```

**Erwartung:** Die Datei lässt sich erstellen, anzeigen und entfernen.

### Test 3 – Backup erstellen

Beispiel einer virtuellen Maschine mit der VMID `100`:

```bash
vzdump 100 --storage zfs-backups --mode snapshot --compress zstd
```

**Was wird getestet?**

- Proxmox kann auf den Storage zugreifen.
- Der Backup-Prozess kann gestartet werden.
- Die VM kann gesichert werden.
- Das Backup-Archiv kann geschrieben werden.

**Erwartetes Ergebnis:**

```text
TASK OK
```

Ein erfolgreicher Backup-Task bestätigt die Durchführung, nicht automatisch die Wiederherstellbarkeit.

### Test 4 – Backup-Datei kontrollieren

```bash
pvesm list zfs-backups --content backup
```

**Erwartung:** Das neu erstellte Backup wird aufgelistet.

### Test 5 – Wiederherstellbarkeit

Ein vollständiger Restore-Test sollte regelmäßig auf einer separaten Test-VM erfolgen.

Detaillierte Anweisungen hierzu enthält die zweite Anleitung.

---

## 8. ZFS-Einstellungen optimieren

Für ein Backup-Dataset können folgende Einstellungen sinnvoll sein.

### 8.1 Komprimierung

```bash
zfs set compression=lz4 zfspool/backups
```

LZ4 bietet eine schnelle, ressourcenschonende Komprimierung.

Bereits mit ZSTD komprimierte Backup-Archive lassen sich dadurch normalerweise kaum weiter verkleinern.

### 8.2 Access Time deaktivieren

```bash
zfs set atime=off zfspool/backups
```

Dadurch werden unnötige Aktualisierungen des letzten Dateizugriffs reduziert.

### 8.3 Einstellungen prüfen

```bash
zfs get compression,atime,dedup zfspool/backups
```

**Erwartetes Ergebnis:**

```text
compression  lz4
atime        off
dedup        off
```

### 8.4 Deduplication

Für kleinere Backup-Umgebungen wird die Aktivierung von ZFS-Deduplizierung normalerweise nicht empfohlen.

Deduplication benötigt zusätzliche Ressourcen und bietet bei bereits komprimierten Backup-Archiven häufig wenig Nutzen.

---

## 9. Monitoring und Wartung

### 9.1 ZFS-Gesundheit

```bash
zpool status -v zfspool
```

### 9.2 Speicherauslastung

```bash
zfs list -o name,used,avail,quota zfspool/backups
```

### 9.3 Automatische Wartung

Unter Debian-basierten Proxmox-Systemen können ZFS-Scrubs und TRIM-Aufgaben über Cron oder systemd organisiert werden.

Cron-Konfiguration prüfen:

```bash
cat /etc/cron.d/zfsutils-linux
```

Timer prüfen:

```bash
systemctl list-timers --all | grep -i zfs
```

**Wichtig:** Vor dem Einrichten zusätzlicher Wartungsaufgaben prüfen, ob bereits automatische Jobs existieren.

### 9.4 SMART-Gesundheitsprüfung

Beispiel für eine SATA-SSD:

```bash
smartctl -H /dev/sda
```

Erweiterte Informationen:

```bash
smartctl -a /dev/sda
```

Der Gerätename muss vor der Ausführung überprüft werden.

---

## 10. Fehlerbehebung

| Problem | Mögliche Ursache | Vorgehensweise |
|---|---|---|
| Storage inactive | Dataset nicht verfügbar | ZFS-Pool und Mountpoint prüfen |
| No space left | Quota oder Pool voll | Speicherbelegung analysieren |
| Backup schlägt fehl | Storage oder VM-Problem | Task-Log auswerten |
| Backup liegt auf Root | Falscher Pfad | Mountpoint und Storage-Konfiguration prüfen |
| Quota unverändert | Befehl fehlgeschlagen | Berechtigungen und Dataset-Namen prüfen |
| Schreibtest schlägt fehl | Dataset nicht beschreibbar | Mount- und Poolstatus prüfen |

---

## 11. Rollback

Soll die Quota wieder auf 80 GiB zurückgesetzt werden:

```bash
zfs set quota=80G zfspool/backups
```

Vorher sicherstellen, dass die tatsächliche Speicherbelegung die Reduzierung zulässt.

Anschließend:

```bash
zfs get quota zfspool/backups
```

Eine Quota-Reduzierung löscht vorhandene Dateien nicht automatisch. Eine zu niedrig gesetzte Quota kann jedoch weitere Schreibvorgänge verhindern.

---

## 12. Checkliste

### Vor der Änderung

- [ ] Richtigen ZFS-Pool identifiziert
- [ ] Poolstatus ONLINE
- [ ] Dataset vorhanden
- [ ] Mountpoint korrekt
- [ ] Aktuelle Quota dokumentiert
- [ ] Ausreichend freier Pool-Speicher vorhanden

### Nach der Änderung

- [ ] Neue Quota korrekt
- [ ] Dataset weiterhin eingehängt
- [ ] Proxmox-Storage aktiv
- [ ] Bestehende Backups vorhanden
- [ ] Schreibtest erfolgreich
- [ ] Backup-Test erfolgreich
- [ ] Keine neuen Fehlermeldungen

---

## 13. Befehlsverzeichnis

| Befehl | Beschreibung |
|---|---|
| `zpool status` | Zeigt den Zustand aller ZFS-Pools |
| `zpool status -v zfspool` | Zeigt detaillierte Poolinformationen und bekannte Fehler |
| `zfs list` | Listet ZFS-Datasets und Volumes |
| `zfs get quota` | Zeigt die Quota eines Datasets |
| `zfs set quota=100G` | Setzt eine neue Quota |
| `zfs get compression` | Zeigt die Komprimierung |
| `zfs set compression=lz4` | Aktiviert LZ4 für neue Schreibvorgänge |
| `zfs set atime=off` | Deaktiviert Access-Time-Aktualisierungen |
| `findmnt -T` | Ermittelt das Dateisystem eines Pfades |
| `pvesm status` | Zeigt den Status der Proxmox-Storages |
| `pvesm list` | Listet Inhalte eines Proxmox-Storages |
| `ls -lh` | Zeigt Dateien mit lesbaren Größen |
| `touch` | Erstellt eine leere Testdatei |
| `rm` | Entfernt Dateien |
| `cat` | Gibt den Inhalt einer Datei aus |
| `vzdump` | Erstellt Proxmox-Backups |
| `smartctl` | Liest SMART-Daten eines Datenträgers aus |
| `systemctl` | Verwaltet und überprüft systemd-Dienste |

---

## 14. Glossar

| Begriff | Erklärung |
|---|---|
| ZFS | Dateisystem und Volume-Manager mit integrierten Integritätsprüfungen |
| ZFS-Pool | Gemeinsamer Speicherbereich aus einem oder mehreren Datenträgern |
| Dataset | Logisch verwalteter Bereich innerhalb eines ZFS-Pools |
| Zvol | Blockgerät innerhalb eines ZFS-Pools |
| Quota | Maximale Speicherbelegung eines Datasets |
| Mountpoint | Pfad, unter dem ein Dateisystem verfügbar ist |
| Storage | In Proxmox registrierter Speicherbereich |
| LZ4 | Schneller Komprimierungsalgorithmus |
| Atime | Zeitpunkt des letzten Dateizugriffs |
| Deduplication | Vermeidung mehrfacher Speicherung identischer Datenblöcke |
| Scrub | Prüfung der gespeicherten Daten anhand ihrer Prüfsummen |
| TRIM | Information an SSDs über nicht mehr benötigte Speicherbereiche |
| SMART | System zur Überwachung des Zustands von Datenträgern |
| VZDump | Backup-Werkzeug von Proxmox VE |
| Retention | Aufbewahrungsregel für Sicherungen |
| Restore | Wiederherstellung eines gesicherten Systems |
| Rollback | Rücknahme einer Konfigurationsänderung |

---

## Weiterführende Dokumentation

- [Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/)
- [OpenZFS Documentation](https://openzfs.github.io/openzfs-docs/)

## Lizenz

Diese Anleitung kann als Grundlage für eigene Home-Lab- und Proxmox-Dokumentationen verwendet und angepasst werden.
