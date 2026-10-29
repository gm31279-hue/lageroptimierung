# Projekthandbuch: Lageroptimierung Stölner

| | |
|---|---|
| **Projekt** | Datengetriebene Lageroptimierung (Bestandssteuerung nach Meldebestand/EOQ) |
| **Auftraggeber** | Stölner GesmbH, Günther Maurer |
| **Konzept** | Ernst (Anforderungspapier „Lageroptimierung", PDF, Juli 2026) |
| **Status** | Produktiv (v1), Weiterentwicklung nach Roadmap (§8) |
| **Live-System** | https://gm31279-hue.github.io/lageroptimierung/ |
| **Quellcode** | https://github.com/gm31279-hue/lageroptimierung (Branch `main`, öffentlich) |
| **Stand dieses Handbuchs** | 29.10.2026 |

> **Arbeitsregel:** Jede Änderung wird zuerst als Change beschrieben (Anlass → Entscheidung → Ergebnis), dann umgesetzt und im Change-Log (§7) protokolliert. Vorgehen angelehnt an OpenSpec: Spezifikation vor Code.

---

## 1. Projektziel

Gleichgewicht zwischen minimaler Kapitalbindung und maximalem Service-Level (Lieferbereitschaft) finden. Ablösung der Bauchgefühl-Bestellung durch ein datengetriebenes, stochastisches Modell auf Basis der WinLine-ERP-Daten (Mesonic WinLine, Edition 2025).

**Ergebnisartefakt im Tagesbetrieb:** eine Bestellvorschlagsliste je Lagerartikel mit ABC/XYZ-Klasse, Sicherheitsbestand, Meldebestand, optimaler Bestellmenge und Ampelstatus (🔴 = Bestand unter Meldebestand → bestellen).

---

## 2. Spezifikation

### 2.1 Fachliche Anforderungen

Quelle: Anforderungspapier Ernst. Nummerierung R1–R12 entspricht der dortigen Zusammenfassung.

| ID | Anforderung | Umsetzung | Status |
|---|---|---|---|
| R1 | Verkaufshistorie (24–36 Monate, je SKU) | WinLine-Verkaufsstatistik-Export (CSV, ab 01.01.2023, Belegebene) | ✅ umgesetzt |
| R2 | Kanaltrennung Händler/Endkunde | Spalte „Kundentyp" im VKST-Export (Händler, Key Account, Gastronomie, Sonstige …) | ✅ umgesetzt |
| R3 | Fehlmengen / Out-of-Stock-Zeiten | Wöchentlicher Export der WinLine-Rückstandsliste (Momentaufnahmen → Zeitreihe); Rückrechnung über Artikeljournal | 🔶 teilweise (Prozess definiert, Auswertung offen) |
| R4 | Wiederbeschaffungszeit μL | Planwert „Beschaffungszeitraum" aus Artikelstamm; real aus Einkaufsbelegen (Bestellung↔Wareneingang) | 🔶 Planwert umgesetzt, Echtwert offen |
| R5 | Lieferzeitschwankung σL | aus Einkaufsbelegen | ⬜ offen |
| R6 | Bestellrestriktionen MOQ/Losgröße, VPE/Colli | Artikelstamm-Export (List-Assistent „ARTSTAMM Einstandspreise MOQ VPE") | ✅ umgesetzt |
| R7 | Einstandspreise | letzter/allg. EP aus Artikelstamm-Export | ✅ umgesetzt |
| R8/R11 | Lagerhaltungskostensatz i | Parameter in der App, Startwert 20 % p.a. | ✅ umgesetzt (Parameter) |
| R9 | Bestellkosten S | Parameter in der App, Startwert 50 €/Bestellung | ✅ umgesetzt (Parameter) |
| R10 | Ziel-Lieferbereitschaftsgrad je Klasse | Parameter: A 98 % / B 95 % / C 90 % → z-Faktor über inverse Normalverteilung | ✅ umgesetzt |
| R12 | Lagerhaltungskosten pro Stück H = C·i | berechnet in der App | ✅ umgesetzt |

**Scope-Ausnahme:** Kundenspezifischer Sonderbau (Artikelbezeichnungen „Niro – …", „Studio – …", „NST – …", „… divers") wird auftragsbezogen beschafft und aus der Lageroptimierung herausgefiltert (abschaltbar).

### 2.2 Nicht-funktionale Anforderungen

| ID | Anforderung | Umsetzung | Status |
|---|---|---|---|
| N1 | Datenschutz: ERP-Daten verlassen den Arbeitsplatz nicht, keine ERP-Daten im Repository | Lokale Verarbeitung im Browser (FileReader), Repo-Regel (C09) | ✅ |
| N2 | Keine Installation, Betrieb im Standardbrowser (Chrome/Edge) | Einzelne HTML-Dateien, Hosting über GitHub Pages | ✅ |
| N3 | Import Verkaufsstatistik (~13 MB) in Sekunden, ohne Einfrieren | Slice-Parser, Chunking, Fortschrittsanzeige (C12/C13) | ✅ verifiziert |
| N4 | Verständliche Fehlermeldungen statt stillem Abbruch | ZIP- und CDN-Hinweise (C13/C14) | ✅ |
| N5 | Stölner-CI | Magenta `#E5007D`, Anthrazit `#575756`, Logo | ✅ |

---

## 3. Systemarchitektur

**Prinzip: Zero-Backend.** Reine Client-Anwendungen als einzelne, in sich geschlossene HTML-Dateien (HTML+CSS+JS inline). Hosting als statische Seiten über GitHub Pages. Es gibt keinen Server, keine Datenbank, kein Login.

**Datenschutz-Designentscheidung:** ERP-Exporte (Umsätze, Einstandspreise) werden nie ins Repository gelegt und nie übertragen. Die Apps lesen die Dateien ausschließlich lokal im Browser (Drag & Drop).

```
WinLine (ERP) --Export CSV/XLS--> lokaler Ablageordner
      --Drag & Drop--> Browser-App (lokale Verarbeitung) --> Bestellvorschlag (Anzeige + CSV-Download)
      --manuell--> Bestellung in WinLine
```

**Externe Abhängigkeiten (CDN, nur Bibliotheken):** SheetJS 0.18.5 (XLS/XLSX-Import im Rechner), Chart.js 4.4.1 (Diagramme im Analyse-Tool), jeweils von cdnjs.cloudflare.com. Ist das CDN blockiert (Firewall), funktioniert der CSV-Import weiter; die App meldet den Ausfall explizit.

**Deployment:** Commit auf `main` → GitHub Pages baut automatisch (~1–2 min) → live.

---

## 4. Komponenten

| Datei | Typ | Zweck |
|---|---|---|
| `index.html` | Startseite | Einstieg, verlinkt alle Komponenten, Schnellstart |
| `Lageroptimierung_Rechner.html` | **App (Kern)** | Import VKST + Artikelstamm → ABC/XYZ, SB, Meldebestand, EOQ, Ampel, CSV-Export |
| `Lageroptimierung_Analyse_Tool.html` | App | Nachfrageanalyse: Pareto, ABC-XYZ-Matrix, Kanalverteilung, μ/σ je Artikel |
| `Leitfaden_Datenbedarf_WinLine.html` | Doku | Einseiter: welche Daten, wie exportieren, wie auswerten |
| `WinLine_Exportanleitung_Lageroptimierung.html` | Doku | Klickanleitung Exporte A–E inkl. Menüpfade, Checkliste |
| `Workbook_Lageroptimierung_Datenauswertung.html` | Schulung | Alle Formeln erklärt (Einsteigerniveau), Übungen mit Lösungen |
| `Handbuch_Einkauf_Innendienst.html` | Prozess | Wochen-/Monats-/Quartalsroutine, Bestellregeln, Zuständigkeiten |
| `PROJEKTHANDBUCH.md` | Projektdoku | Dieses Dokument: Spezifikation, Architektur, Datenmodell, Change-Log, Roadmap |
| `README.md` | Doku | Repo-Übersicht, Verwendung, Methodik-Kurzfassung |
| `logo.png` | Asset | Stölner-Logo „high five" – liegt nur im lokalen Paket; im Repo fehlt es, die Live-Seite nutzt daher das Fallback-Logo von stoelner.at |

---

## 5. Datenmodell (Eingangsdaten)

### 5.1 Export A – Verkaufsstatistik (Pflicht, CSV)

WinLine-Statistikauswertung, eine Zeile pro Belegposition, ohne Kopfzeile, Felder in Anführungszeichen, Komma-getrennt, Windows-1252-kodiert (UTF-8 wird automatisch erkannt). Relevante Spalten (0-basiert):

| Index | Inhalt |
|---|---|
| 1 / 2 | Artikelnummer / Bezeichnung |
| 9 | Warengruppe (Ebene 1) |
| 18 | Kundentyp (= Kanaltrennung, R2) |
| 35 | Belegdatum `JJJJ-MM-TT` (App prüft Index 35 zuerst, sonst Suche) |
| 36 | Menge (negativ = Gutschrift, wird saldiert) |
| 38 | Umsatz netto |
| 40 | Deckungsbeitrag |

Referenzdatei: `VKST23-26.csv` (~13 MB, ~22.600 Zeilen, 2023–2026).

### 5.2 Export B – Artikelstamm (empfohlen, XLS/CSV)

List-Assistent-Liste „ARTSTAMM Einstandspreise MOQ VPE", mit Kopfzeile. Die App mappt Spalten automatisch per Header-Erkennung (manuell korrigierbar): Artikelnummer, Bezeichnung, allg./niedrigster/letzter Einkaufspreis, Colli Einkauf (VPE), Losgröße Einkauf (MOQ), Lagerstand, Mindest-/Sollbestand, Beschaffungszeitraum (Tage).

**Artikel-Ausprägungen:** Suffixe `,P` `,W` `,S` `,X` an Artikelnummern werden beim Abgleich standardmäßig ignoriert (Basisnummer); Lagerstände mehrerer Ausprägungen werden summiert.

### 5.3 Exporte C–E (definiert, Auswertung noch offen)

- **C Einkaufsbelege** (Belegjournal: Bestellung + Wareneingang) → reale μL/σL
- **D Rückstandsliste** (wöchentlich, Dateikonvention `RUECKSTAND_JJJJMMTT.xlsx`) → Fehlmengen-Zeitreihe
- **E Artikeljournal** (Lagerbuchungen ab 2023) → Lagerstands-Rückrechnung, Out-of-Stock-Tage

---

## 6. Berechnungslogik (Rechner)

1. **Nachfrage:** Monatsmengen je Artikel über den gesamten Datenzeitraum (Nullmonate zählen mit) → μ (Mittel), σ (Standardabweichung, Population), CV = σ/μ.
2. **ABC:** Artikel absteigend nach Umsatz; kumuliert ≤ 80 % → A, ≤ 95 % → B, Rest → C.
3. **XYZ:** CV ≤ 0,5 → X; ≤ 1,0 → Y; > 1,0 → Z.
4. **z-Faktor:** aus Ziel-Servicelevel je ABC-Klasse via inverser Normalverteilung (Acklam-Approximation).
5. **Lieferzeit L:** Beschaffungszeitraum aus Artikelstamm (Tage), sonst Parameter „Standard-Lieferzeit" (Default 21 Tage); intern L/30 Monate.
6. **Sicherheitsbestand:** `SB = aufrunden( z · σ · √(L/30) )` – vereinfachte Formel, Erweiterung um σL siehe Roadmap P6.
7. **Meldebestand:** `MB = aufrunden( μ · L/30 + z · σ · √(L/30) )`.
8. **EOQ:** `√( 2 · D · S / H )` mit D = μ·12, S = Bestellkosten, H = EP · i. Bestellmenge = max(EOQ, MOQ), aufgerundet auf volle VPE.
9. **Ampel:** Lagerstand (aus Export B) < MB → 🔴. Offene Lieferantenbestellungen sind derzeit **nicht** berücksichtigt (manuelle Prüfung lt. Prozesshandbuch; siehe Roadmap P5).
10. **Filter:** Sonderbau-Regex `^(Niro|Studio|NST)\s*-|-\s*divers` (abschaltbar). Parameter werden im Browser (`localStorage`) gemerkt.

---

## 7. Change-Log

| # | Datum | Change | Anlass / Entscheidung / Ergebnis |
|---|---|---|---|
| C01 | 10.07.2026 | Anforderungsanalyse | Konzept-PDF (Ernst) auf WinLine-Datenquellen gemappt; Exporte A–E definiert. Ergebnis: Exportanleitung. |
| C02 | 10.07.2026 | Export A durchgeführt | Verkaufsstatistik 2023–2026 als CSV (22.598 Zeilen). |
| C03 | 10.07.2026 | Analyse-Tool v1 | Entscheidung Zero-Backend/Browser-App: keine Installation nötig, ERP-Daten bleiben lokal. Ergebnis: ABC/XYZ-Schnellanalyse. |
| C04 | 10.07.2026 | Workbook (Schulung) | Zielgruppe Einsteiger ohne Statistikwissen; alle Formeln mit Beispielen und Übungen. |
| C05 | 10.07.2026 | Rückstandsliste geprüft | Befund: Großteil des Rückstandswerts ist Sonderbau/Projektgeschäft → Scope-Ausnahme Sonderbau beschlossen; Altfälle (Liefertermine 2024/25) zur Bereinigung markiert. |
| C06 | 10.07.2026 | Rechner v1 (Kern-App) | Import A+B, Parameter, SB/MB/EOQ, Ampel, CSV-Export, Spalten-Automapping, Ausprägungs-Logik. |
| C07 | 10.07.2026 | GitHub-Repo angelegt | Dateien via Web-Upload committet. |
| C08 | 12.07.2026 | CI-Umstellung | Alle Komponenten auf Stölner-CI (Magenta/Anthrazit) + Logo; Leitfaden (Einseiter) und Prozesshandbuch Einkauf/Innendienst ergänzt. |
| C09 | 12.07.2026 | Datenschutz-Regel | Entscheidung: Repository bleibt öffentlich (Live-Seite für alle Mitarbeiter); dafür gilt: keine ERP-Exportdateien (Umsätze, Preise, Kunden) im Repo. Daten werden lokal gehalten und separat weitergegeben. |
| C10 | 12.07.2026 | Übergabepaket | ZIP (Apps + Doku + Daten + Logo) für die Weitergabe an Ernst erstellt. |
| C11 | 07.09.2026 | GitHub Pages aktiviert | Live-URL für alle Mitarbeiter; ersetzt die ZIP-Verteilung (Ursache „Seite nicht gefunden": lokal geöffnete, nicht entpackte ZIPs). |
| C12 | 07.09.2026 | Performance: CSV-Parser | Anlass: „Laden dauert ewig". Zeichenweiser Parser ersetzt durch zeilen-/slice-basierten (`parseLine`); Datumsspalte mit Fast-Path auf Index 35. |
| C13 | 07.09.2026 | UX: Import | Chunked Parsing (3.000 Zeilen/Block) mit Live-Fortschrittsanzeige (%), kein Einfrieren; explizite Meldung bei blockiertem CDN (SheetJS). Verifiziert per End-to-End-Test auf der Live-Seite: 12,4 MB / 23.000 Zeilen in Sekunden inkl. Berechnung. |
| C14 | 07.09.2026 | Root-Cause „Datei lädt nicht" | Befund via Screenshot: Datei wurde direkt aus einem ZIP-Fenster gezogen (Temporärdatei, FileReader-Fehler). Fix: Klartext-Hinweis „ZIP zuerst entpacken" in beiden Apps. |
| C15 | 29.10.2026 | Projekthandbuch eingeführt | Anlass: Partner arbeitet nach OpenSpec und braucht nachvollziehbare Projektdoku statt reinem Source Code. Entscheidung: Markdown-Handbuch (Spezifikation, Architektur, Datenmodell, Change-Log, Roadmap), versioniert im Repo; ab jetzt wird jeder Change hier protokolliert. Ergebnis: `PROJEKTHANDBUCH.md`. |
| C16 | 29.10.2026 | Lokaler Projektordner + Übergabe | Ordner `Lageroptimierung_Stoelner` am Desktop: alle Komponenten, Logo, Projekthandbuch; ERP-Daten (`Daten`) und Konzept-PDF (`Konzept`) nur lokal. Übergabe-ZIP an Ernst (Code, Doku, Projekthandbuch, Testdaten) inkl. vorbereiteter E-Mail. |

---

## 8. Roadmap (priorisiert)

### Phase 1 – Pilot & Baseline (sofort, 2–4 Wochen)

| ID | Schritt | Warum | Aufwand |
|---|---|---|---|
| P1 | **KPI-Baseline erheben:** Lagerwert, Lieferbereitschaft (Anteil sofort lieferbarer Auftragspositionen), Anzahl und Wert der Rückstände, Lagerumschlag | Ohne Ausgangswert kein Erfolgsnachweis | gering |
| P2 | **Pilotlauf:** Rechner mit echten Daten; Einkauf prüft die Top-20 A-Artikel auf Plausibilität; Parameter (i, S, Servicelevel) kalibrieren | Vertrauen der Anwender, Fehler früh finden | gering |
| P3 | **Stammdaten der A-Artikel pflegen:** Beschaffungszeitraum, MOQ, VPE in WinLine | Fehlen die Felder, rechnet der Rechner mit Standardwerten (21 Tage, MOQ/VPE = 1) | mittel |
| P4 | **Rückstandsliste ab sofort wöchentlich als Excel ablegen**, Altfälle bereinigen | Die Fehlmengen-Historie entsteht nur durch Sammeln – jede Woche zählt | gering |

### Phase 2 – Genauigkeit (4–8 Wochen)

| ID | Schritt | Warum | Aufwand |
|---|---|---|---|
| P5 | **Verfügbarer Bestand in der Ampel:** Lagerstand + offene Lieferantenbestellungen − offene Kundenaufträge | Verhindert Fehlalarme („rot, obwohl schon bestellt") | mittel |
| P6 | **Echte Lieferzeiten** aus Einkaufsbelegen (Export C) → μL/σL je Lieferant; erweiterte Formel `SB = z · √(L·σD² + μD²·σL²)` | Sicherheitsbestand bildet das Lieferantenrisiko ab | mittel |
| P7 | **Sonderbau-Kennzeichen in WinLine** (Artikelgruppe/Statistikfeld) statt Namensmuster | Robuster Filter | gering |

### Phase 3 – Ausbau (ab 3 Monaten)

| ID | Schritt | Warum | Aufwand |
|---|---|---|---|
| P8 | Nachfrage um Fehlmengen korrigieren (gesammelte Rückstandslisten, R3) | Echte statt verkaufte Nachfrage | mittel |
| P9 | Saisonalität (B2C) gesondert modellieren | Bessere Prognose saisonaler Artikel | hoch |
| P10 | Automatischer Datenabzug (ODBC/SQL auf die WinLine-Datenbank) statt manueller Exporte | Spart Routinearbeit, aktuellere Daten | hoch |
| P11 | Wirkungsmessung: KPIs nach 3 Monaten gegen Baseline (P1) | Erfolgsnachweis, Parameter nachschärfen | gering |

**Kleinigkeiten:** README auf aktuellen Stand bringen · `logo.png` ins Repo (Live-Seite zeigt dann das CI-Logo) · optional Excel-Rechenvorlage · optional OpenSpec-Ordnerstruktur (`openspec/specs`, `openspec/changes`) für die Weiterentwicklung.

---

## 9. Betrieb (Kurzfassung, Details im Prozesshandbuch)

- **Freitags:** Rückstandsliste exportieren; Rechner mit aktueller VKST + Artikelstamm füttern; 🔴-Liste abarbeiten; Abweichungen im Bestellkommentar begründen; Ergebnis-CSV archivieren.
- **Monatlich:** VKST- und Einkaufsbeleg-Export aktualisieren.
- **Quartalsweise:** Artikelstamm-Export, Stammdaten-Stichprobe, ABC/XYZ-Verschiebungen sichten.
- **Jährlich:** Parameter (i, S, Servicelevel) durch die Geschäftsführung freigeben.
- **Grundregel:** „Hausverstand schlägt Ampel" – die Liste ist Vorschlag, nicht Automat; Übersteuern ist erlaubt, aber dokumentiert.

---

## 10. Glossar

μ/σ = Mittel/Streuung der Monatsnachfrage · CV = σ/μ (Planbarkeit) · SB = Sicherheitsbestand · MB/ROP = Meldebestand · EOQ = optimale Bestellmenge · MOQ/Losgröße = Mindestbestellmenge · VPE/Colli = Verpackungseinheit · L = Wiederbeschaffungszeit · z = Servicefaktor · EP = Einstandspreis · i = Lagerhaltungskostensatz · Sonderbau = kundenspezifische Fertigung (außerhalb Scope).
