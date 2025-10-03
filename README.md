# qubes-storage — Sicherer Datei-Zugriff über Qrexec

Dieses Projekt bietet ein minimales, aber hart abgesichertes Dateispeicher-Interface für Qubes OS.  
Ein **Storage‑Qube** (z. B. `storage-qube`) hält die Daten unter einem festgelegten Wurzelverzeichnis. Andere AppVMs greifen **nur per Qrexec** auf diesen Speicher zu – ohne Netzfreigaben und ohne direkt gemountete Volumes.

Das CLI‑Werkzeug `qubes-storage` in den anfragenden VMs kapselt die Qrexec‑Aufrufe und stellt eine benutzerfreundliche Oberfläche bereit. Auf der Serverseite (im Storage‑Qube) sorgen Qrexec‑Service‑Skripte (`user.Storage…`) für strikte Pfadvalidierung, Größenlimits und atomare Schreibvorgänge.


## Architektur

```
AppVM (Client)                           Storage‑Qube (Server)
-----------------                        ----------------------
qubes-storage  ── qrexec-client-vm ──▶   /etc/qubes-rpc/user.Storage*
 (CLI)                                     (Bash‑Skripte)

Statefile: ~/.cache/qubes-storage-path   Datenwurzel: /home/user/shared
```

- **Client:** `qubes-storage` (Bash), verwaltet einen „Remote‑Arbeitsordner“ und spricht Qrexec‑Dienste an.
- **Server:** Qrexec‑Dienste `user.Storage(List, Ls, Get, Put, Copy, Move, Delete, Mkdir, Rmdir, Stat)`, jeweils als eigenständige Skripte.
- **Dom0‑Policy:** Legt fest, dass *nur* der definierte Storage‑Qube Ziel der Dienste ist.


## Sicherheitsmodell (Kurzfassung)

1. **Pfad‑Jail:** Alle Dienste berechnen `SAFE` mit `realpath -m` relativ zu `BASEPATH` (Standard: `/home/user/shared`) und lehnen alles außerhalb ab (`[[ "$SAFE" != "$BASEPATH"* ]]`). Path‑Traversal (`../`) ist damit blockiert.
2. **Größenlimits:** `Get` und `Put` respektieren `MAX_BYTES` (Umgebungsvariable). Standard im Code:
   - `user.StorageGet`: `MAX_BYTES=104857600` (100 MiB)
   - `user.StoragePut`: `MAX_BYTES=104857600` (100 MiB)
3. **Atomare Writes:** `Put`/`Copy` schreiben in eine temporäre Datei im Zielverzeichnis und `mv`en diese anschließend (Crash‑sicherer, keine Teilzustände).
4. **Minimal‑Oberfläche:** Kein Shell‑Globbing auf Serverseite; Parameter kommen über `stdin` (eine oder zwei Zeilen), wodurch Sonderzeichen robust sind.
5. **Rechte:** `Put` setzt `chmod 0640` auf neue Dateien.
6. **Rückgabecodes:**
   - `2` bei ungültigem Pfad/Eingaben,
   - `3` bei Überschreitung `MAX_BYTES`,
   - `1` bei nicht gefundenen Quellen (z. B. `Move/Copy`).


## Komponenten

### Client: `qubes-storage`

- **Speichert den Remote‑Arbeitsordner** in `~/.cache/qubes-storage-path`.
- **Pfadnormalisierung** via `normalize_remote(base, add)`:
  - absoluter Zusatz (`/foo`) überschreibt `base`,
  - ansonsten `base/add`, Normalisierung via `realpath -m` unter künstlicher Root `/` und Entfernen des führenden `/`.
- **Unterstützte Befehle:**
  - `list [dir]` – zeilenweise Inhalte (Dirs `/`, Links `@`), Quelle: `user.StorageList`
  - `ls [dir]` – `ls -lA` des Remote‑Verzeichnisses, Quelle: `user.StorageLs`
  - `get <remote> [local]` – Datei holen, Quelle: `user.StorageGet`
  - `put <localfile> [remote]` – Datei hochladen, Quelle: `user.StoragePut`
  - `copy <remote-src> <remote-dst>` – Datei kopieren (remote→remote), Quelle: `user.StorageCopy`
  - `move <remote-src> <remote-dst>` – Datei verschieben (remote→remote), Quelle: `user.StorageMove`
  - `delete <remote>` – Datei löschen, Quelle: `user.StorageDelete`
  - `mkdir <remote-dir>` / `rmdir <remote-dir>` – Verzeichnis anlegen/entfernen, Quellen: `user.StorageMkdir`/`user.StorageRmdir`
  - `stat|details <remote>` – `stat` auf Pfad, Quelle: `user.StorageStat`
  - `cd <remote-dir>` / `pwd` – Arbeitsordner verwalten (nur clientseitig)
  - `gui-get` – Dateiauswahl per **rofi** und Download
  - `gui-put` – Upload per Dateidialog (**zenity**)
  - `edit <remote>` – Datei herunterladen, lokalen Editor öffnen (`$EDITOR` oder `nano`), zurückschreiben per `StoragePut`

> Hinweis: `gui-get` nutzt im Skript `user.StorageListFiles`. Dieser Service ist **nicht** Teil des Pakets. In der Praxis kann `user.StorageList` verwendet werden oder ein eigener `user.StorageListFiles`‑Wrapper bereitgestellt werden (siehe Erweiterungen).

### Server: Qrexec‑Dienste (Storage‑Qube)

Standard‑`BASEPATH`: `/home/user/shared` (anpassbar je Skriptkopf).

| Service              | stdin                                   | stdout                              | Beschreibung |
|----------------------|------------------------------------------|-------------------------------------|--------------|
| `user.StorageList`   | optional Pfad (1 Zeile; default `.`)     | Einträge mit Suffix (`/`, `@`)      | Flaches Listing (`find … -maxdepth 1`) |
| `user.StorageLs`     | optional Pfad                            | `ls -lA` Ausgabe                    | Detail‑Listing |
| `user.StorageGet`    | Pfad (1 Zeile)                           | Dateiinhalte                        | Download mit Größenlimit |
| `user.StoragePut`    | Pfad (1 Zeile) + Dateiinhalt (rest)      | `OK`                                | Upload atomar, setzt Modus `0640` |
| `user.StorageCopy`   | Quelle (Zeile 1), Ziel (Zeile 2)         | `OK` (implizit)                     | Kopie im Storage‑Qube (mit Temp‑Datei) |
| `user.StorageMove`   | Quelle (Zeile 1), Ziel (Zeile 2)         | `OK` (implizit)                     | Atomarer Move im selben FS |
| `user.StorageDelete` | Pfad (1 Zeile)                           | `OK`                                | Löschen einer Datei |
| `user.StorageMkdir`  | Verzeichnisname (1 Zeile)                | —                                   | `mkdir -p` |
| `user.StorageRmdir`  | Verzeichnisname (1 Zeile)                | —                                   | `rmdir` (scheitert, wenn nicht leer) |
| `user.StorageStat`   | Pfad (1 Zeile)                           | `stat`‑Ausgabe                      | Metadaten |


## Installation

### 1) Storage‑Qube vorbereiten

1. Qube anlegen, z. B. `storage-qube` (ohne Netz, falls gewünscht).
2. Verzeichnis für Daten anlegen:  
   ```bash
   mkdir -p /home/user/shared
   ```
3. Qrexec‑Services installieren (als `root`):
   ```bash
   install -Dm0755 qrexec-services-for-storage-qube/user.Storage* /etc/qubes-rpc/
   ```
   Alternativ in einem Paket unterbringen. Passen Sie bei Bedarf in jedem Skript `BASEPATH` an.
4. Test lokal im Storage‑Qube (ohne Qrexec) ist nicht sinnvoll; Dienste erwarten Parameter via `stdin`.

### 2) Dom0‑Policy setzen

Erstellen Sie z. B. `/etc/qubes/policy.d/30-qubes-storage.policy` mit folgendem Inhalt und passen Sie den Qube‑Namen an (hier: `storage-qube`):

```
user.StorageCopy  *  storage-qube  allow
user.StorageGet   *  storage-qube  allow
user.StorageLs    *  storage-qube  allow
user.StorageMove  *  storage-qube  allow
user.StorageStat  *  storage-qube  allow
user.StorageDelete*  storage-qube  allow
user.StorageList  *  storage-qube  allow
user.StorageMkdir *  storage-qube  allow
user.StoragePut   *  storage-qube  allow
user.StorageRmdir *  storage-qube  allow
```

> Je nach Qubes‑Version kann die empfohlene Policy‑Syntax variieren. Die oben beigelegte Datei `policy-snippet-for-dom0` entspricht der im Repo verwendeten Syntax und funktioniert in typischen Qubes 4.1/4.2‑Setups.

### 3) Client‑VM(s) einrichten

1. Kopieren Sie `qubes-storage` nach `/usr/local/bin/` und machen Sie es ausführbar:
   ```bash
   install -Dm0755 qubes-storage /usr/local/bin/qubes-storage
   ```
2. Am Skriptkopf `STORAGE_VM` auf den Namen des Storage‑Qubes setzen, z. B.:
   ```bash
   STORAGE_VM="storage-qube"
   ```
3. Optional: Abhängigkeiten für Komfortfunktionen installieren:
   - `rofi` (für `gui-get`),
   - `zenity` (für `gui-put`),
   - `$EDITOR` (für `edit`; Standard ist `nano`).


## Konfiguration

- **STORAGE_VM (Client):** Name des Ziel‑Qubes (Default: `my-data` → anpassen!).
- **BASEPATH (Server):** Datenwurzel; pro Service‑Skript konfigurierbar (Default: `/home/user/shared`).
- **MAX_BYTES (Server):** Upload/Download‑Limit via Umgebungsvariable:
  ```bash
  # Beispiel: 100 MiB Limit für Put und Get
  MAX_BYTES=104857600
  ```
  *Hinweis:* In `user.StoragePut` steht derzeit `MAX_BYTES=1000000` (≈1 MiB) im Code. Passen Sie dies an oder exportieren Sie `MAX_BYTES` in der Service‑Umgebung.


## Verwendung (Beispiele)

```bash
# Arbeitsordner setzen (relativ zur Datenwurzel)
qubes-storage cd projects/demo

# Hochladen
qubes-storage put ./README.md      # → remote: projects/demo/README.md
qubes-storage put ./logo.png assets/logo.png

# Auflisten
qubes-storage list .               # flache Liste, / für Verzeichnisse
qubes-storage ls .                 # Details (ls -lA)

# Download
qubes-storage get assets/logo.png ./logo.png

# Kopieren/Verschieben/Löschen
qubes-storage copy    assets/logo.png assets/logo@2x.png
qubes-storage move    assets/tmp.txt assets/archive/tmp.txt
qubes-storage delete  assets/old.bin

# Metadaten
qubes-storage stat assets/logo.png

# GUI‑Flows
qubes-storage gui-put
qubes-storage gui-get
```

**Exitcodes & Fehlertexte** sind sprechend gehalten (z. B. „Invalid path“, „File too large“). Beim `get` meldet das CLI „Fehler: get fehlgeschlagen“ und gibt Code 1 zurück.

## Erweiterungen / Roadmap

- **`user.StorageListFiles`:** Das CLI referenziert diesen Dienst in `gui-get`. Entweder:
  - `user.StorageList` verwenden (einfacher Drop‑in in der Case‑Branch) oder
  - einen dedizierten Dienst hinzufügen, der nur Dateien (ohne `/`) listet.
- **Write‑Once‑Policy:** Separate Dienste mit restriktiver Dom0‑Policy (nur `Get/List`) für „Reader‑VMs“.
- **Quotas/Rate‑Limits:** Serverseitig via cgroups oder `MAX_BYTES` je Aufruf dynamisch festlegen.
- **Logging/Auditing:** Audit‑Trails in der Storage‑VM, z. B. via `logger` mit `QREXEC_REMOTE_DOMAIN`.
- **Paketierung:** RPM/DEB für einfache Verteilung; systemweiter `DEFAULT_BASEPATH` in `/etc/default/qubes-storage`.


## Testen

In einer beliebigen AppVM (Client):

```bash
# trocken: nur Listen/Stat
qubes-storage list .
qubes-storage stat README.md

# Roundtrip:
echo "hello" > /tmp/hello.txt
qubes-storage put /tmp/hello.txt demo/hello.txt
qubes-storage get demo/hello.txt /tmp/hello.out
diff -u /tmp/hello.txt /tmp/hello.out
```

Serverseitige Fehlerfälle provozieren („Invalid path“, zu große Datei), um Policy und Limits zu verifizieren.


## Troubleshooting

- **`Permission denied`/Policy‑Prompts:** Dom0‑Policy prüfen. Ziel‑Qube‑Name muss zu `STORAGE_VM` passen.
- **`Invalid path`:** Pfad liegt außerhalb `BASEPATH` oder Ziel existiert nicht. Pfade ohne führendes `/` verwenden (Client normalisiert korrekt).
- **`File too large` (Exit 3):** `MAX_BYTES` erhöhen oder Datei segmentieren.
- **`rmdir: Directory not empty`:** Erst Inhalte löschen oder `StorageMove` nutzen, um umzustrukturieren.


## Sicherheitshinweise

- Dienste laufen in der Storage‑VM **ohne Root**. Pflegen Sie restriktive Dateirechte im Datenbaum.
- Dom0‑Policy restriktiv halten (nur gewünschte Aufrufer erlauben, ggf. `ask, default_target=storage-qube`).
- Keine nicht‑validierten Daten an Shell‑Expansion übergeben; dieses Repo nutzt ausschließlich `stdin` und `--`‑Optionen.
- Optional die Storage‑VM vom Netz trennen.


## Lizenz

Creative Commons Attribution-NonCommercial 4.0 International Public License
- freie private, nichtkommerzielle Nutzung,
- Namensnennung ist verpflichtend,
- kommerzielle Nutzung nur mit Zustimmung.


## Dateien im Repository

- `qubes-storage` – Client‑CLI
- `qrexec-services-for-storage-qube/user.Storage*` – Server‑Dienste
- `policy-snippet-for-dom0` – Beispiel‑Policy für Dom0
