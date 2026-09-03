# Queuefix für mehrere gleichzeitige RustDesk-Sitzungen

## Zweck und Stand

Dieser Fork basiert auf RustDesk 1.4.9. Der Testbuild trägt die Version 1.4.10.
Er behebt einen unbegrenzten Speicheraufbau auf dem gesteuerten Rechner, der
bei mehreren gleichzeitigen Fernwartungssitzungen sichtbar werden kann.

Der Fork ist unabhängig und kein offizielles RustDesk-Release. Der erzeugte
Windows-Installer ist nicht signiert und muss vor einem produktiven Rollout im
eigenen Prozess geprüft und signiert werden.

## Fehlerbild

Für jede eingehende Verbindung gab es im Hostprozess einen unbegrenzten
Tokio-MPSC-Kanal für Videonachrichten. Erzeugte der Host neue Videoframes
schneller, als eine Verbindung sie senden konnte, blieben alte Frames in der
Warteschlange. Jeder Eintrag hielt die dazugehörige `Arc<Message>` und damit
den Frame-Speicher fest.

Mehrere langsame oder gleichzeitig verbundene Zuschauer erhöhten das Risiko:
Jede Verbindung besaß ihre eigene unbegrenzte Warteschlange. Unter anhaltendem
Rückstau konnten deshalb der private beziehungsweise zugesicherte Speicher des
Windows-Prozesses und anschließend die Auslagerungsdatei stark wachsen.

## Technische Änderung

Die Änderung liegt in `src/server/connection.rs` und ersetzt ausschließlich
den Videokanal einer Verbindung durch eine spezialisierte Warteschlange:

1. Pro Verbindung werden höchstens zwei ausstehende Videonachrichten gehalten.
2. Wartet bereits ein `VideoFrame`, ersetzt ein neuer Frame diesen alten Frame.
   Der Empfänger bekommt dadurch das aktuellste Bild statt veraltete Bilder
   nachzuarbeiten.
3. Ein neues `SwitchDisplay` verwirft veraltete Frames und ältere
   Displaywechsel. Danach kann genau ein aktueller Frame folgen.
4. Beim Schließen des Empfängers wird die Warteschlange geleert. Weitere
   Sendungen werden abgewiesen.
5. Steuerung, Eingabe, Zwischenablage, Audio, Dateiübertragung und sonstige
   Nicht-Videonachrichten verwenden unverändert den bisherigen Kanal.

Die Obergrenze ist als `MAX_PENDING_VIDEO_MESSAGES = 2` festgelegt. Damit ist
der durch wartende Videonachrichten belegbare Speicher pro Verbindung
begrenzt, unabhängig davon, wie lange ein Zuschauer nicht nachkommt.

## Warum nicht einfach Frames blockierend senden?

Ein blockierender oder vollständig gepufferter Versand würde die
Bildschirmaufnahme beziehungsweise andere Verbindungen ausbremsen. Für eine
Live-Übertragung ist ein alter, noch nicht gesendeter Frame wertlos, sobald ein
neuerer Frame vorliegt. Das Ersetzen veralteter Frames begrenzt den Speicher
und reduziert zugleich die sichtbare Verzögerung.

`SwitchDisplay` wird gesondert behandelt, weil der folgende Frame zur neuen
Anzeige gehören muss. Ein einfacher Kanal mit Kapazität zwei würde diese
Reihenfolge nicht zuverlässig ausdrücken.

## Zugehörige Commits

- `02c5f94` – Queuefix, Tests und Versionswechsel auf 1.4.10
- `74dc9a2` – automatischer Build auf der Queuefix-Branch
- `89d981a` – Erzeugung eines Windows-x64-MSI in GitHub Actions

## Automatisierte Prüfungen

Die Unit-Tests in `src/server/connection.rs` prüfen:

- nur der neueste wartende Frame bleibt erhalten;
- ein Displaywechsel bleibt vor dem neuesten Frame erhalten;
- ein neuer Displaywechsel verwirft veraltete Videonachrichten;
- nach dem Schließen des Empfängers werden keine Nachrichten angenommen.

Der Workflow `.github/workflows/build-queuefix-windows.yml` prüft zusätzlich
die Rust-Formatierung, baut den Windows-x64-Client und erzeugt ein MSI. Der
Build zu Commit `89d981a` wurde erfolgreich ausgeführt.

## Praktischer Belastungstest

Der Patch sollte auf einem separaten Testrechner geprüft werden, nicht zuerst
auf einem produktiven Digital-Signage-Gerät.

1. Testbuild auf einem Windows-Rechner installieren.
2. Zwei oder mehr Fernwartungssitzungen gleichzeitig öffnen.
3. Die Sitzungen mindestens zehn Minuten aktiv lassen und Bildänderungen
   erzeugen.
4. Den privaten Speicher aller RustDesk-Prozesse wiederholt erfassen:

```powershell
Get-Process rustdesk | Select-Object Id,@{N='RAM_MB';E={[math]::Round($_.WorkingSet64/1MB,1)}},@{N='Privat_MB';E={[math]::Round($_.PrivateMemorySize64/1MB,1)}}
```

Für eine laufende Anzeige alle zwei Sekunden:

```powershell
while ($true) { Clear-Host; Get-Date; Get-Process rustdesk | Select-Object Id,@{N='RAM_MB';E={[math]::Round($_.WorkingSet64/1MB,1)}},@{N='Privat_MB';E={[math]::Round($_.PrivateMemorySize64/1MB,1)}}; Start-Sleep 2 }
```

Eine geringe Zunahme beim Aufbau neuer Sitzungen ist normal. Entscheidend ist,
dass sich der private Speicher anschließend einpendelt und nicht dauerhaft
linear oder explosionsartig wächst. Zusätzlich sollten Verbindungswechsel,
Monitorwechsel, Eingabe, Zwischenablage und Sitzungsabbau geprüft werden.

## Bekannte Grenzen

- Die Änderung begrenzt nur die hostseitige Video-Warteschlange. Andere
  Speicherprobleme werden dadurch nicht automatisch behoben.
- Die bisherige Prüfung mit mehreren gleichzeitigen Sitzungen ist ein
  Belastungstest, aber noch kein Langzeitnachweis.
- Das MSI aus GitHub Actions ist nicht codesigniert.
- Bei einer Übernahme auf einen neueren Upstream-Stand müssen Konflikte und
  Änderungen im Verbindungsmodul erneut geprüft werden.

## Rückkehr zum offiziellen Verhalten

Zum Vergleich kann ein unveränderter Upstream-Build aus Tag `1.4.9` erstellt
werden. Ein Rollback des Fixes erfolgt durch Rücknahme von Commit `02c5f94`;
dabei werden auch die Versionsänderungen und die zuerst eingeführte
Queuefix-Workflowdatei zurückgenommen.
