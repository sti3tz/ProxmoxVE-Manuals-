
# Proxmox VE – VM & LXC Backup Management

> Schritt-für-Schritt-Anleitung zur Einrichtung automatisierter Backups, Durchführung manueller Sicherungen und Überprüfung der Wiederherstellbarkeit von virtuellen Maschinen und LXC-Containern.

**Kategorie:** Backup & Recovery  
**Plattform:** Proxmox VE  
**Schwierigkeitsgrad:** Einsteiger bis Fortgeschrittene  
**Version:** 1.0  
**Sprache:** Deutsch

---

## Inhaltsverzeichnis

1. [Zielsetzung](#1-zielsetzung)
2. [Backup-Grundlagen](#2-backup-grundlagen)
3. [Beispielumgebung](#3-beispielumgebung)
4. [Vorbereitende Prüfungen](#4-vorbereitende-prüfungen)
5. [Backup-Strategie definieren](#5-backup-strategie-definieren)
6. [Automatische Backups konfigurieren](#6-automatische-backups-konfigurieren)
7. [Manuelle Backups erstellen](#7-manuelle-backups-erstellen)
8. [LXC-Besonderheiten](#8-lxc-besonderheiten)
9. [Backup-Ergebnisse prüfen](#9-backup-ergebnisse-prüfen)
10. [Wiederherstellung testen](#10-wiederherstellung-testen)
11. [Backup-Monitoring](#11-backup-monitoring)
12. [Fehlerbehebung](#12-fehlerbehebung)
13. [Checkliste](#13-checkliste)
14. [Befehlsverzeichnis](#14-befehlsverzeichnis)
15. [Glossar](#15-glossar)

---

## 1. Zielsetzung

Diese Anleitung zeigt, wie virtuelle Maschinen und LXC-Container in Proxmox VE automatisiert gesichert werden können.

Dabei wird ein bereits vorhandener Backup-Storage verwendet.

**Ein neues ZFS-Dataset muss nicht erstellt werden.**

Nach Abschluss kannst du:

- Bestehende VMs und Container identifizieren.
- Backup-Jobs erstellen.
- Zeitpläne konfigurieren.
- Aufbewahrungsregeln festlegen.
- Manuelle Backups durchführen.
- Backup-Ergebnisse kontrollieren.
- Wiederherstellungen testen.
- Fehler systematisch analysieren.

### Praxisbeispiel

Ein Proxmox-Host betreibt mehrere Systeme:

- Home Assistant als virtuelle Maschine
- Paperless-ngx als LXC-Container
- PatchMon als LXC-Container

Home Assistant wird bereits täglich gesichert.

Nun sollen zusätzliche Systeme in das Backup-Konzept aufgenommen werden.

---

## 2. Backup-Grundlagen

### 2.1 Was ist ein Backup?

Ein Backup ist eine separate Sicherung von Daten oder Systemzuständen, die zur Wiederherstellung nach einem Fehler verwendet werden kann.

Typische Szenarien:

- Fehlgeschlagene Updates
- Beschädigte Konfigurationen
- Versehentlich gelöschte Daten
- Fehlerhafte Änderungen
- Defekte virtuelle Maschinen
- Hardwareausfälle, sofern Backups unabhängig gespeichert sind

### 2.2 Backup versus Snapshot

| Eigenschaft | Snapshot | Backup |
|---|---|---|
| Zweck | Zustand zu einem Zeitpunkt festhalten | Wiederherstellbare Sicherung |
| Typischer Speicherort | Ursprünglicher Storage | Definierter Backup-Storage |
| Unabhängigkeit | Häufig abhängig vom Ursprung | Als Archiv unabhängig nutzbar |
| Langfristige Aufbewahrung | Nur eingeschränkt geeignet | Geeignet |
| Schutz vor Storage-Ausfall | Allein nicht ausreichend | Nur bei getrenntem Speicher |

**Wichtig:** Der VZDump-Modus `snapshot` beschreibt die Sicherungsmethode. Das Ergebnis ist trotzdem ein eigenständiges Backup-Archiv.

### 2.3 RPO und RTO

**RPO – Recovery Point Objective**

Maximal akzeptabler Datenverlust.

Beispiel:

Bei einem täglichen Backup beträgt das theoretische RPO bis zu 24 Stunden, sofern die Sicherungen erfolgreich und regelmäßig erfolgen.

**RTO – Recovery Time Objective**

Maximal akzeptierte Zeit bis zur Wiederherstellung eines Systems.

Beispiel:

Home Assistant soll nach einem Ausfall innerhalb von zwei Stunden wieder betriebsbereit sein.

Ob dieses Ziel erreicht werden kann, muss durch Restore-Tests überprüft werden.

---

## 3. Beispielumgebung

| Komponente | Beispielwert |
|---|---|
| Proxmox-Node | `pve` |
| Backup-Storage | `zfs-backups` |
| Backup-Verzeichnis | `/zfspool/backups/dump` |
| ZFS-Dataset | `zfspool/backups` |
| Quota | 80 GiB |
| Komprimierung | ZSTD |
| Backup-Modus | Snapshot |
| Aufbewahrung | 7 Sicherungen |

### Beispielsysteme

| System | Typ | Beispiel-ID |
|---|---|---|
| Home Assistant | VM | 100 |
| Paperless-ngx | LXC | 101 |
| PatchMon | LXC | 102 |

Die IDs sind Beispiele und müssen vor der Verwendung geprüft werden.

---

## 4. Vorbereitende Prüfungen

Vor der Erstellung neuer Backup-Jobs muss der aktuelle Zustand geprüft werden.

### 4.1 Virtuelle Maschinen anzeigen

```bash
qm list
```

**Erwartetes Ergebnis (Beispiel):**

```text
VMID  NAME             STATUS
100   homeassistant    running
103   test-vm          stopped
```

### 4.2 LXC-Container anzeigen

```bash
pct list
```

**Erwartetes Ergebnis (Beispiel):**

```text
VMID  STATUS   NAME
101   running  paperless
102   running  patchmon
```

### 4.3 Storage überprüfen

```bash
pvesm status
```

**Erwartung:**

Der Storage `zfs-backups` ist aktiv.

### 4.4 Freien Speicherplatz prüfen

```bash
zfs list -o name,used,avail,quota zfspool/backups
```

**Ziel:**

Feststellen, ob ausreichend Platz für zusätzliche Sicherungen vorhanden ist.

### 4.5 Bestehende Backup-Jobs prüfen

In der Proxmox-Weboberfläche:

**Datacenter → Backup**

Vor dem Anlegen eines neuen Jobs kontrollieren:

- Existiert bereits ein Backup-Job für das System?
- Wird die VM möglicherweise durch einen Sammeljob gesichert?
- Welche Aufbewahrungsregeln gelten?
- Wann werden bestehende Jobs ausgeführt?

Doppelte Sicherungen können unnötigen Speicherverbrauch verursachen.

---

## 5. Backup-Strategie definieren

Eine Backup-Strategie sollte vor der technischen Einrichtung festgelegt werden.

### 5.1 Beispielstrategie

| System | Intervall | Retention | Zeitfenster |
|---|---|---|---|
| Home Assistant | Täglich | 7 | 03:00 |
| Paperless-ngx | Täglich | 7 | 03:30 |
| PatchMon | Täglich | 7 | 04:00 |
| Test-VM | Wöchentlich | 4 | Sonntag |

Die angegebenen Zeiten sind Beispiele.

### 5.2 Speicherbedarf kalkulieren

Beispiel:

| System | Größe je Backup | Anzahl | Bedarf |
|---|---:|---:|---:|
| Home Assistant | 6,6 GiB | 7 | 46,2 GiB |
| Paperless-ngx | 2 GiB | 7 | 14 GiB |
| PatchMon | 1 GiB | 7 | 7 GiB |
| Gesamt | | | 67,2 GiB |

Die Werte sind Schätzungen und müssen anhand realer Backups validiert werden.

Zusätzlich berücksichtigen:

- Historische Sicherungen
- Geschützte Backups
- Temporären Speicherbedarf
- Größenwachstum
- Zusätzliche manuelle Sicherungen

**Empfehlung:** Speicher nicht bis an die Quota-Grenze auslasten.

---

## 6. Automatische Backups konfigurieren

### 6.1 Backup-Verwaltung öffnen

Proxmox-Weboberfläche:

**Datacenter → Backup → Add**

### 6.2 Backup-Job erstellen

Beispiel für einen zusätzlichen LXC-Container:

| Einstellung | Beispiel |
|---|---|
| Node | `pve` |
| Storage | `zfs-backups` |
| Selection Mode | Include selected VMs |
| VMID | `101` |
| Schedule | `03:30` |
| Mode | Snapshot |
| Compression | ZSTD |
| Enabled | Ja |

Bei einem täglichen Zeitplan muss die entsprechende tägliche Ausführung im Scheduler ausgewählt werden.

### 6.3 Aufbewahrung konfigurieren

Unter Retention:

```text
Keep Last: 7
```

Damit sollen die sieben neuesten Sicherungen des jeweiligen Systems aufbewahrt werden.

Die tatsächliche Bereinigung hängt von der jeweiligen Job- und Storage-Konfiguration ab.

### 6.4 Backup-Notizen definieren

Beispiel:

```text
VM {{vmid}} | {{guestname}} | Node {{node}} | Automatisches Backup
```

### 6.5 Konfiguration speichern

Nach dem Speichern prüfen:

- Richtiger Node
- Richtige VMID
- Richtiger Storage
- Zeitplan korrekt
- Komprimierung korrekt
- Aufbewahrung korrekt
- Job aktiviert

---

## 7. Manuelle Backups erstellen

Manuelle Backups eignen sich zur Funktionsprüfung und vor größeren Änderungen.

### 7.1 VM-Backup

Beispiel VM 100:

```bash
vzdump 100 --storage zfs-backups --mode snapshot --compress zstd
```

**Was passiert?**

1. Proxmox startet den Backup-Prozess.
2. Der Zustand der VM wird entsprechend dem Backup-Modus erfasst.
3. Die Daten werden gelesen.
4. Das Backup wird komprimiert.
5. Das Archiv wird auf dem Backup-Storage gespeichert.

**Erwartetes Ergebnis:**

```text
TASK OK
```

Typischer Dateiname:

```text
vzdump-qemu-100-YYYY_MM_DD-HH_MM_SS.vma.zst
```

### 7.2 LXC-Backup

Beispiel Container 101:

```bash
vzdump 101 --storage zfs-backups --mode snapshot --compress zstd
```

**Erwartetes Ergebnis:**

```text
TASK OK
```

Typischer Dateiname:

```text
vzdump-lxc-101-YYYY_MM_DD-HH_MM_SS.tar.zst
```

### 7.3 Backup-Modi

| Modus | Beschreibung |
|---|---|
| Snapshot | Sicherung anhand eines konsistenten Sicherungszeitpunkts, soweit unterstützt |
| Suspend | Bei LXC mit zeitweiser Unterbrechung beziehungsweise Synchronisierung |
| Stop | Gast wird für die Sicherung gestoppt |

Die technische Umsetzung unterscheidet sich zwischen QEMU-VMs und LXC-Containern.

Bei LXC hängt die Snapshot-Unterstützung unter anderem vom verwendeten Storage ab.

---

## 8. LXC-Besonderheiten

Bei Containern muss vor dem Backup geprüft werden, wo die Anwendungsdaten gespeichert sind.

### 8.1 Container-Konfiguration anzeigen

```bash
pct config 101
```

**Ziel:**

Prüfen, welche Dateisysteme und Mountpoints verwendet werden.

Beispiel:

```text
rootfs: zfspool:subvol-101-disk-0,size=8G
mp0: zfspool:subvol-101-disk-1,mp=/data,backup=1
```

Das zusätzliche Volume ist in diesem Beispiel für die Sicherung aktiviert.

### 8.2 Bind-Mounts

Beispiel:

```text
mp0: /mnt/documents,mp=/documents
```

Bei Bind-Mounts werden externe Host-Verzeichnisse eingebunden.

Diese Daten sind nicht automatisch vollständig Bestandteil eines VZDump-Backups.

**Wichtig:** Vor dem Backup muss geklärt werden, wie diese Daten separat gesichert werden.

### 8.3 Anwendungskonsistenz

Ein erfolgreiches Container-Backup garantiert nicht automatisch eine anwendungskonsistente Datenbanksicherung.

Bei Datenbanken können zusätzliche Maßnahmen erforderlich sein.

Beispiele:

- Datenbank-Dump
- Anwendungsseitiger Backup-Prozess
- Geplanter Wartungszeitpunkt
- Wiederherstellungstest

### 8.4 Beispiel Paperless-ngx

Zu prüfen sind:

- Dokumentenverzeichnis
- Datenbank
- Konfiguration
- Zusätzliche Mountpoints
- Benutzerberechtigungen

### 8.5 Beispiel PatchMon

Zu prüfen sind:

- Anwendungsdateien
- Datenbank
- Konfiguration
- API-Einstellungen
- Externe Speicherpfade

---

## 9. Backup-Ergebnisse prüfen

### 9.1 Proxmox-Weboberfläche

Navigiere zu:

**Node → zfs-backups → Backups**

Prüfe:

- Backup vorhanden
- Richtige VMID
- Richtiges Datum
- Plausible Größe
- Task erfolgreich

### 9.2 Backup-Dateien anzeigen

```bash
ls -lh /zfspool/backups/dump/
```

### 9.3 Backup-Inhalte über Proxmox anzeigen

```bash
pvesm list zfs-backups --content backup
```

### 9.4 Task-Protokoll

Im Task-Protokoll sollte erscheinen:

```text
TASK OK
```

Zusätzlich kontrollieren:

- Keine unerwarteten Warnungen
- Keine ausgelassenen Daten
- Keine Speicherplatzprobleme
- Keine Fehler beim Schreiben

### 9.5 Bewertung

| Prüfschritt | Erwartung |
|---|---|
| Backup gestartet | Ja |
| Backup abgeschlossen | TASK OK |
| Archiv vorhanden | Ja |
| Dateigröße plausibel | Ja |
| Protokoll ohne kritische Fehler | Ja |

Ein Backup gilt damit als technisch erstellt.

Die Wiederherstellbarkeit muss separat getestet werden.

---

## 10. Wiederherstellung testen

Ein Restore-Test ist die wichtigste Prüfung eines Backup-Konzepts.

**Sicherheitsregel:** Niemals eine produktive VM oder einen produktiven Container ungeprüft überschreiben.

### 10.1 Test vorbereiten

1. Backup auswählen.
2. Freie Test-VMID bestimmen.
3. Ausreichend Speicherplatz prüfen.
4. Isolierte Testumgebung vorsehen.
5. Produktive Netzwerk- und Dienstkonflikte vermeiden.

### 10.2 Freie VMID prüfen

```bash
qm list
pct list
```

Beispiel:

```text
Test-VMID: 900
Test-CTID: 901
```

Die IDs müssen tatsächlich frei sein.

### 10.3 VM wiederherstellen

Beispiel:

```bash
qmrestore /zfspool/backups/dump/DATEINAME.vma.zst 900 --storage zfspool
```

`DATEINAME` muss durch den tatsächlichen Backup-Dateinamen ersetzt werden.

**Was passiert?**

- Proxmox liest das Backup-Archiv.
- Eine neue VM mit der ID 900 wird erstellt.
- Die VM-Datenträger werden wiederhergestellt.
- Die VM-Konfiguration wird übernommen.

**Erwartetes Ergebnis:**

Die neue VM erscheint in Proxmox.

### 10.4 LXC wiederherstellen

Beispiel:

```bash
pct restore 901 /zfspool/backups/dump/DATEINAME.tar.zst --storage zfspool
```

**Erwartetes Ergebnis:**

Der Container wird mit der neuen ID 901 wiederhergestellt.

### 10.5 Vor dem Start prüfen

Besonders wichtig:

- Netzwerkschnittstellen
- IP-Adressen
- Hostnamen
- Automatische Dienste
- Externe Mountpoints
- Verbindungen zu produktiven Datenbanken
- Automatische Startoptionen

Eine wiederhergestellte VM kann produktive Konfigurationen enthalten.

Deshalb sollte der erste Start in einer isolierten Testumgebung erfolgen.

### 10.6 Anwendungstest

Nach dem Start prüfen:

| Prüfung | Erwartung |
|---|---|
| Gast startet | Erfolgreich |
| Betriebssystem verfügbar | Ja |
| Anwendung startet | Ja |
| Daten vorhanden | Ja |
| Konfiguration vollständig | Ja |
| Keine kritischen Fehler | Ja |

### 10.7 Home-Assistant-Beispiel

Prüfen:

- Weboberfläche startet.
- Integrationen werden geladen.
- Automationen sind vorhanden.
- Entitäten sind sichtbar.
- Historische Daten sind vorhanden, soweit gesichert.

Bei einer Testwiederherstellung sollte verhindert werden, dass Automationen produktive Geräte oder Dienste beeinflussen.

### 10.8 Paperless-ngx-Beispiel

Prüfen:

- Weboberfläche erreichbar
- Dokumente vorhanden
- Dokumente lassen sich öffnen
- Datenbank verfügbar
- Benutzer vorhanden
- Berechtigungen korrekt

### 10.9 PatchMon-Beispiel

Prüfen:

- Anwendung erreichbar
- Datenbank vorhanden
- Geräteinformationen sichtbar
- API funktioniert
- Keine kritischen Anwendungsfehler

### 10.10 Testsystem entfernen

Nach erfolgreichem Restore-Test kann das Testsystem über die Proxmox-Weboberfläche entfernt werden.

Vorher sicherstellen, dass ausschließlich das Testsystem ausgewählt wurde.

Produktive Systeme dürfen nicht verändert werden.

---

## 11. Backup-Monitoring

### 11.1 Tägliche Kontrollen

- Backup-Jobs erfolgreich?
- Fehlgeschlagene Tasks vorhanden?
- Storage erreichbar?
- Genügend Speicherplatz vorhanden?

### 11.2 Wöchentliche Kontrollen

- Alle vorgesehenen Systeme gesichert?
- Backup-Größen plausibel?
- Retention funktioniert?
- Ungewöhnliche Größenänderungen?

### 11.3 Monatliche Kontrollen

- Restore-Test durchführen
- Storage-Gesundheit prüfen
- SMART-Werte kontrollieren
- Dokumentation aktualisieren

### 11.4 Externe Sicherungen

Backups sollten nicht ausschließlich auf demselben physischen Datenträger liegen wie die produktiven Systeme.

Empfohlen wird eine an die Anforderungen angepasste 3-2-1-Backup-Strategie:

- Drei Kopien der relevanten Daten
- Zwei unterschiedliche Speichermedien beziehungsweise unabhängige Speicherziele
- Eine Kopie außerhalb des primären Standorts

Zusätzlich sollten unveränderliche oder offline aufbewahrte Sicherungen erwogen werden.

---

## 12. Fehlerbehebung

| Problem | Mögliche Ursache | Vorgehensweise |
|---|---|---|
| Backup startet nicht | Job deaktiviert | Job-Einstellungen prüfen |
| Storage inactive | Storage nicht verfügbar | `pvesm status` |
| No space left | Quota erreicht | `zfs list` |
| Snapshot nicht möglich | Storage unterstützt Modus nicht | Backup-Modus prüfen |
| LXC-Daten fehlen | Mountpoint ausgeschlossen | `pct config` prüfen |
| Backup-Archiv fehlt | Job fehlgeschlagen | Task-Protokoll prüfen |
| Restore schlägt fehl | Storage oder Archivproblem | Restore-Log analysieren |
| Anwendung startet nicht | Daten oder Abhängigkeiten fehlen | Anwendungskonsistenz prüfen |

---

## 13. Checkliste

### Vorbereitung

- [ ] VM oder LXC identifiziert
- [ ] Bestehende Backup-Jobs geprüft
- [ ] Storage aktiv
- [ ] Speicherplatz ausreichend
- [ ] Backup-Strategie festgelegt
- [ ] Retention definiert

### Konfiguration

- [ ] Richtige VMID ausgewählt
- [ ] Zeitplan korrekt
- [ ] Backup-Modus geeignet
- [ ] Komprimierung konfiguriert
- [ ] Aufbewahrung festgelegt
- [ ] Job aktiviert

### Backup-Test

- [ ] Manueller Backup-Lauf gestartet
- [ ] Task erfolgreich
- [ ] Backup-Archiv vorhanden
- [ ] Dateigröße plausibel
- [ ] Keine kritischen Fehler

### Restore-Test

- [ ] Freie Test-ID verwendet
- [ ] Testsystem isoliert
- [ ] Restore erfolgreich
- [ ] Anwendung startet
- [ ] Daten vorhanden
- [ ] Testergebnis dokumentiert
- [ ] Testsystem bereinigt

---

## 14. Befehlsverzeichnis

| Befehl | Beschreibung |
|---|---|
| `qm list` | Listet virtuelle Maschinen auf |
| `qm config <VMID>` | Zeigt die Konfiguration einer VM |
| `pct list` | Listet LXC-Container auf |
| `pct config <CTID>` | Zeigt die Konfiguration eines Containers |
| `pvesm status` | Zeigt den Status der Proxmox-Storages |
| `pvesm list` | Listet Inhalte eines Storages |
| `vzdump` | Erstellt VM- und Container-Backups |
| `vzdump <ID>` | Sichert einen bestimmten Gast |
| `--storage` | Definiert das Backup-Ziel |
| `--mode snapshot` | Verwendet den Snapshot-Backup-Modus |
| `--compress zstd` | Aktiviert ZSTD-Komprimierung |
| `qmrestore` | Stellt ein QEMU-VM-Backup wieder her |
| `pct restore` | Stellt ein LXC-Backup wieder her |
| `zfs list` | Zeigt ZFS-Datasets und Speicherbelegung |
| `zpool status` | Zeigt den Gesundheitszustand eines ZFS-Pools |
| `ls -lh` | Zeigt Dateien und lesbare Größen |
| `findmnt` | Prüft Dateisystem-Mounts |

---

## 15. Glossar

| Begriff | Erklärung |
|---|---|
| Proxmox VE | Open-Source-Virtualisierungsplattform |
| VM | Virtuelle Maschine mit eigenem Gastbetriebssystem |
| LXC | Linux-Container mit gemeinsam genutztem Host-Kernel |
| VMID | Eindeutige ID eines virtuellen Gasts in Proxmox |
| CTID | Container-ID, technisch ebenfalls eine VMID |
| VZDump | Proxmox-Werkzeug zur Erstellung von Backups |
| Snapshot | Zustandsaufnahme zu einem bestimmten Zeitpunkt |
| Backup | Wiederherstellbare Datensicherung |
| Restore | Wiederherstellung aus einer Sicherung |
| Retention | Aufbewahrungsregel für Backups |
| RPO | Maximal akzeptabler Datenverlust |
| RTO | Maximal akzeptierte Wiederherstellungsdauer |
| ZSTD | Komprimierungsalgorithmus |
| ZFS | Dateisystem und Volume-Manager |
| Storage | Speicherziel innerhalb von Proxmox |
| Bind-Mount | Einbindung eines Host-Verzeichnisses in einen Container |
| Mountpoint | Einhängepunkt eines Dateisystems |
| Anwendungskonsistenz | Zustand, in dem Anwendungsdaten zuverlässig wiederherstellbar sind |
| Quota | Begrenzung der Speicherbelegung |
| Backup-Job | Automatisch ausgeführte Sicherungsaufgabe |
| Task-Log | Protokoll einer Proxmox-Operation |
| 3-2-1-Regel | Backup-Strategie mit mehreren unabhängigen Kopien |

---

## Weiterführende Dokumentation

- [Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/)
- [Proxmox Backup and Restore](https://pve.proxmox.com/pve-docs/chapter-vzdump.html)
- [OpenZFS Documentation](https://openzfs.github.io/openzfs-docs/)

## Lizenz

Diese Anleitung kann als Grundlage für eigene Proxmox- und Home-Lab-Dokumentationen verwendet und angepasst werden.

Für eine öffentliche Veröffentlichung empfiehlt sich eine explizite Repository-Lizenz, beispielsweise MIT oder CC BY 4.0.
