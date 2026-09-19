# Windows 11 Update Loop Fix

> Ein Skript, das die Endlosschleife **„Nach Updates wird gesucht“** unter Windows 11 behebt, indem es alle Windows-Update-Komponenten sauber zurücksetzt.

---

## Das Problem

Nach einer frischen Installation von Windows 11 bleibt Windows Update häufig dauerhaft bei „Nach Updates wird gesucht“ hängen. Dahinter stecken meistens eine dieser Ursachen:

| Ursache | Was passiert |
|---|---|
| Beschädigter Update-Cache | Ein abgebrochener Download blockiert die Warteschlange dauerhaft. |
| Hängende BITS-Aufträge | Der Übertragungsdienst wartet ewig auf einen Auftrag, der nie fertig wird. |
| Falsche Systemzeit | Weicht die Uhr zu stark ab, scheitert der TLS-Handshake zu den Microsoft-Servern lautlos. |
| Veraltete Zertifikatsdatenbank | `catroot2` passt nicht mehr zu den signierten Update-Paketen. |
| Beschädigtes Komponentenlager | Der Update-Stack selbst hat defekte Systemdateien. |
| Übrig gebliebene WSUS-Richtlinie | Der PC fragt einen Firmen-Update-Server, den es im Heimnetz nicht gibt. |

Das Skript adressiert alle sechs Punkte in einem Durchlauf.

---

## Was das Skript tut

1. **Vorab-Diagnose** – Windows-Version, freier Speicherplatz, Systemzeit und aktive Update-Richtlinien werden geprüft und gemeldet.
2. **Hängende Downloads verwerfen** – die BITS-Warteschlange wird geleert.
3. **Dienste anhalten** – `wuauserv`, `bits`, `usosvc`, `dosvc`, `cryptsvc` und `appidsvc`. Die ursprüngliche Startart jedes Dienstes wird vorher gesichert.
4. **Caches zurücksetzen** – `SoftwareDistribution` und `catroot2` werden **umbenannt statt gelöscht**, dazu die BITS-Datenbank und der Cache der Übermittlungsoptimierung.
5. **Netzwerk bereinigen** – ein hängen gebliebener WinHTTP-Proxy wird zurückgesetzt.
6. **Dienste zurückstellen** – exakt auf die vorher gesicherte Startart, nicht auf einen geratenen Standardwert.
7. **Zeit synchronisieren** – per `w32tm` gegen den Zeitserver.
8. **Reparieren** – `DISM /RestoreHealth`, danach `sfc /scannow` (in dieser Reihenfolge, siehe unten).
9. **Update-Suche anstoßen** und eine Zusammenfassung mit Protokollpfad ausgeben.

---

## Was das Skript bewusst **nicht** tut

Viele kursierende Update-Reset-Skripte enthalten diese Zeilen:

```bat
sc.exe sdset bits D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)...
sc.exe sdset wuauserv D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)...
```

**Dieses Skript macht das nicht – und das ist Absicht.** Diese Sicherheitsdeskriptoren stammen aus Windows Vista bzw. Windows 7. Unter Windows 10 und 11 sehen die Standardrechte dieser Dienste anders aus: Sie enthalten unter anderem Einträge für `NT SERVICE`-Konten und AppContainer, die es damals noch nicht gab. Wer die alten Werte setzt, überschreibt die aktuellen Berechtigungen mit einem veralteten Satz und kann damit die Zusammenarbeit zwischen `wuauserv` und dem Update Orchestrator (`usosvc`) beschädigen – also genau das Problem verursachen, das er beheben wollte. Microsoft führt diesen Schritt in der aktuellen Troubleshooting-Dokumentation nicht mehr.

Ebenfalls nicht enthalten ist das massenhafte Neuregistrieren von DLLs per `regsvr32` – ein Großteil der dort genannten Bibliotheken existiert unter Windows 11 gar nicht mehr.

---

## Voraussetzungen

- Windows 11 (das Skript läuft auch unter Windows 10 und weist auf die Abweichung hin)
- Administratorrechte – das Skript fordert sie bei Bedarf selbst per UAC an
- Mindestens 20 GB freier Speicherplatz für nachfolgende Funktionsupdates
- Eine aktive Internetverbindung

---

## Anleitung

1. Die neueste Version [hier herunterladen](https://github.com/exnermax/Windows-11-Update-Loop-Fix/releases/latest).
2. Die `.zip`-Datei entpacken.
3. Alle laufenden Anwendungen beenden. Falls ein früherer Versuch an gesperrten Ordnern gescheitert ist: den PC im **abgesicherten Modus** starten.
4. Rechtsklick auf `Win11UpdateFix.cmd` → **Als Administrator ausführen**.
   *(Ein Doppelklick genügt ebenfalls – das Skript fordert die Rechte dann selbst an.)*
5. Den Anweisungen folgen. `DISM` und `SFC` brauchen zusammen erfahrungsgemäß 15–40 Minuten.
6. **Den PC neu starten.** Erst danach greifen alle Änderungen. Das Skript bietet den Neustart am Ende an.

### Parameter

| Parameter | Wirkung |
|---|---|
| *(keiner)* | Interaktiv mit Rückfrage vor dem Start und vor dem Neustart |
| `/y` | Läuft ohne jede Rückfrage durch, kein Neustart – für Skripte und Deployment |

```bat
Win11UpdateFix.cmd /y
```

---

## Protokoll und Rollback

**Protokoll:** `C:\ProgramData\Win11UpdateFix\Win11UpdateFix.log`

Das Skript löscht die Update-Caches nicht, sondern benennt sie um:

| Original | Sicherung |
|---|---|
| `C:\Windows\SoftwareDistribution` | `C:\Windows\SoftwareDistribution.bak` |
| `C:\Windows\System32\catroot2` | `C:\Windows\System32\catroot2.bak` |

Windows legt beide Ordner beim nächsten Start automatisch neu an. Läuft Windows Update danach wieder, können die `.bak`-Ordner gefahrlos gelöscht werden – sie belegen unter Umständen mehrere Gigabyte.

Wird das Skript mitten im Lauf abgebrochen, bleiben die Update-Dienste deaktiviert zurück. Das ist kein Grund zur Sorge: Beim nächsten Start erkennt das Skript den abgebrochenen Lauf anhand von `services.state` und stellt die gesicherten Startarten automatisch wieder her.

---

## Wenn es danach immer noch nicht läuft

| Symptom | Nächster Schritt |
|---|---|
| Warnung „Dienst lässt sich nicht anhalten“ | Im abgesicherten Modus erneut ausführen. |
| Warnung „SoftwareDistribution ist gesperrt“ | Ebenfalls abgesicherter Modus – ein Virenscanner oder ein laufendes Update hält den Ordner offen. |
| Warnung zu WSUS-Richtlinie | Im Heimnetz unter `gpedit.msc` → *Computerkonfiguration → Administrative Vorlagen → Windows-Komponenten → Windows Update* auf „Nicht konfiguriert“ setzen. |
| DISM meldet einen Fehlercode | Details in `C:\Windows\Logs\DISM\dism.log`. Bei Fehler `0x800f081f` fehlt die Reparaturquelle – dann mit einer ISO als Quelle arbeiten. |
| SFC meldet einen Fehlercode | Details in `C:\Windows\Logs\CBS\CBS.log`. |
| Suche hängt weiterhin | Update-Verlauf mit `Get-WindowsUpdateLog` in PowerShell auswerten; bei Verdacht auf Hardware-/Treiberkonflikt einen sauberen Neustart durchführen. |

---

## Änderungen in Version 2.0

| Bereich | Version 1.0 | Version 2.0 |
|---|---|---|
| Dienst-Berechtigungen | Setzte veraltete Vista-Deskriptoren per `sc sdset` | Entfernt – siehe Begründung oben |
| Startart nach dem Lauf | Fest auf `auto` gesetzt, obwohl der Windows-11-Standard `demand` ist | Wird vor dem Lauf ausgelesen und exakt wiederhergestellt |
| Reihenfolge | DISM lief, **während** Windows Update deaktiviert war | Dienste werden erst wiederhergestellt, dann repariert |
| Reparaturbefehl | Nur `StartComponentCleanup` (räumt auf, repariert nicht) | `RestoreHealth` vor `SFC`, Cleanup danach |
| Berücksichtigte Dienste | 4 Dienste | zusätzlich `usosvc` und `dosvc` – unter Windows 11 die eigentlichen Update-Steuerdienste |
| Caches | Wurden unwiderruflich gelöscht | Werden umbenannt, Rollback bleibt möglich |
| Downloader-Pfad | `%ALLUSERSPROFILE%\Application Data\...` (Legacy-Junction) | `%ProgramData%\Microsoft\Network\Downloader` |
| BITS-Warteschlange | Wurde nicht geleert | `bitsadmin /reset /allusers` vor dem Anhalten |
| Systemzeit | Nicht berücksichtigt | Wird geprüft und synchronisiert |
| Diagnose | Keine | Version, Speicherplatz, Richtlinien und Zeit vorab |
| Abbruch mitten im Lauf | Dienste blieben deaktiviert zurück | Automatischer Rollback beim nächsten Start |
| Administratorrechte | Abbruch mit Hinweis | Werden per UAC selbst angefordert |
| Protokoll | Keins | Vollständiges Protokoll unter `%ProgramData%` |
| Umlaute in der Konsole | Fehlerhaft dargestellt | Codepage wird auf UTF-8 gesetzt und danach zurückgestellt |
| Unbeaufsichtigter Betrieb | Nicht möglich | Schalter `/y` |

---

## Haftungsausschluss

Das Skript verändert Systemdienste und Systemordner. Es wurde so gebaut, dass jeder Schritt umkehrbar ist, dennoch gilt: **Die Nutzung erfolgt auf eigene Verantwortung.** Vor dem Einsatz auf Produktivsystemen empfiehlt sich ein Wiederherstellungspunkt:

```bat
powershell -Command "Checkpoint-Computer -Description 'Vor Win11UpdateFix' -RestorePointType MODIFY_SETTINGS"
```

## Lizenz

MIT – siehe [LICENSE](LICENSE).
