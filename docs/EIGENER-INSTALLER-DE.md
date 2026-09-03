# Queuefix in einen eigenen RustDesk-Installer übernehmen

Diese Anleitung beschreibt, wie ein Kollege den öffentlichen Fork als Quelle
für einen eigenen, vorkonfigurierten Windows-x64-Installer verwendet. Keine
Serveradresse, kein öffentlicher Serverschlüssel und erst recht kein Passwort
sollten fest in Git-Commits eingetragen werden, wenn sie nicht bewusst
öffentlich sein dürfen.

## Variante A: direkt auf diesem Fork aufbauen

```bash
git clone --recurse-submodules https://github.com/kamohy/rustdesk.git
cd rustdesk
git checkout codex/bounded-video-queue-1.4.9
git submodule update --init --recursive
```

Danach unter GitHub den Bereich **Actions** öffnen, den Workflow
**Build queuefix Windows x64** auswählen und **Run workflow** auf der Branch
`codex/bounded-video-queue-1.4.9` starten.

Der Workflow erzeugt zwei Artefakte:

- `rustdesk-1.4.10-queuefix-windows-x64`: vollständiges Programmverzeichnis;
- `rustdesk-1.4.10-queuefix-msi-x64`: nicht signierter MSI-Installer.

Mit der GitHub CLI kann der letzte erfolgreiche Lauf auch so abgerufen werden:

```bash
gh run list --repo kamohy/rustdesk --workflow build-queuefix-windows.yml --limit 5
gh run download RUN_ID --repo kamohy/rustdesk --dir artifacts/RUN_ID
```

`RUN_ID` wird durch die Nummer des gewünschten erfolgreichen Laufs ersetzt.

## Variante B: den Fix in einen bestehenden eigenen Fork übernehmen

Im vorhandenen RustDesk-Fork des Kollegen:

```bash
git remote add queuefix https://github.com/kamohy/rustdesk.git
git fetch queuefix codex/bounded-video-queue-1.4.9
git checkout -b queuefix-integration
git cherry-pick 02c5f94
```

Damit werden der eigentliche Codefix, die Unit-Tests, die Version 1.4.10 und die
Grundversion des Buildworkflows übernommen. Soll der fertige MSI-Schritt
ebenfalls übernommen werden:

```bash
git cherry-pick 74dc9a2 89d981a
```

Bei einem Fork, der nicht mehr auf RustDesk 1.4.9 basiert, sollte der Codefix
nicht ungeprüft per `cherry-pick` erzwungen werden. Stattdessen die Änderung in
`src/server/connection.rs` portieren und die Semantik mit den vier enthaltenen
Queue-Tests erneut bestätigen.

## Eigenes Branding und eigene Konfiguration

Der Queuefix selbst benötigt keine Serverdaten. Branding, Serveradresse und
öffentlicher Server-Schlüssel gehören in eine getrennte Anpassungsschicht oder
in den bestehenden Custom-Client-Prozess. Dadurch bleibt der Speicherfix auch
bei Änderungen der Infrastruktur wiederverwendbar.

Empfohlene Trennung:

1. **Queuefix-Commit:** ausschließlich Warteschlangenlogik und Tests.
2. **Produktkonfiguration:** Rendezvous-/Relay-Adresse, öffentlicher Schlüssel,
   erlaubte Optionen und gegebenenfalls feste Client-ID.
3. **Branding:** Programmname, Symbole, Herstellerangaben und Dateinamen.
4. **Packaging:** MSI-Metadaten, UpgradeCode/ProductCode, Installation und
   Deinstallation.
5. **Signierung:** MSI und ausführbare Dateien mit dem eigenen Zertifikat
   signieren; private Signierschlüssel nur als geschützte CI-Secrets ablegen.

Geheime Werte werden als GitHub-Actions-Secrets hinterlegt und im Workflow nur
zur Buildzeit gelesen. Sie gehören weder in die README noch in ein öffentliches
Repository. Ein öffentlicher RustDesk-Serverschlüssel ist kein privater
Signierschlüssel, sollte aber trotzdem nur dann fest eingebaut werden, wenn die
zugehörige Serveradresse ebenfalls für die Zielgruppe bestimmt ist.

## Installer anpassen

Die vorhandene MSI-Erzeugung liegt unter `res/msi`. Der Queuefix-Workflow führt
im Wesentlichen diese Schritte aus:

```powershell
python3 .\build.py --portable --flutter --skip-portable-pack --hwcodec --vram
Move-Item .\flutter\build\windows\x64\runner\Release .\rustdesk-custom-windows-x64
Push-Location .\res\msi
python preprocess.py --arp -d ..\..\rustdesk-custom-windows-x64
nuget restore msi.sln
msbuild msi.sln -p:Configuration=Release -p:Platform=x64 /p:TargetVersion=Windows10
Pop-Location
```

Für einen eigenen Installer müssen mindestens Produktname, Hersteller,
Version, Upgrade-Verhalten und Signierung geprüft werden. Der `UpgradeCode`
muss für dieselbe Produktlinie stabil bleiben; ein fremder oder offizieller
RustDesk-Installer darf nicht unbeabsichtigt ersetzt werden.

## Abnahme vor einem Rollout

1. Erfolgreichen CI-Lauf und Prüfsumme des MSI dokumentieren.
2. Installation, Update und vollständige Deinstallation auf einer Testmaschine
   prüfen.
3. Konfiguration nach Neustart und Benutzerwechsel kontrollieren.
4. Mindestens fünf gleichzeitige Sitzungen aufbauen und den privaten Speicher
   über einen längeren Zeitraum beobachten.
5. Monitorwechsel, Zwischenablage, Dateiübertragung, Tastatur und Maus testen.
6. Erst anschließend signieren und in einer kleinen Pilotgruppe verteilen.

## Pflege bei neuen RustDesk-Versionen

Bei einem neuen Upstream-Release:

```bash
git remote add upstream https://github.com/rustdesk/rustdesk.git
git fetch upstream --tags
git checkout -b queuefix-next upstream/NEUER_TAG
git cherry-pick 02c5f94
```

Nach Konfliktauflösung müssen die Tests und der Windows-Build erneut laufen.
Vor dem Beibehalten des Patches außerdem prüfen, ob Upstream das Problem
inzwischen selbst und gegebenenfalls anders gelöst hat.
