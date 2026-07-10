# Lageroptimierung – Stölner GesmbH

Werkzeuge und Dokumentation für die datengetriebene Bestandssteuerung: von den WinLine-Exporten bis zum Bestellvorschlag (ABC/XYZ, Sicherheitsbestand, Meldebestand, EOQ).

**Startseite:** [`index.html`](index.html) öffnen – von dort sind alle Werkzeuge verlinkt.

## Inhalt

| Datei | Zweck |
|---|---|
| `index.html` | Startseite mit Links zu allen Werkzeugen |
| `Lageroptimierung_Rechner.html` | **App:** VKST- und Artikelstamm-Export importieren → Bestellvorschlagsliste mit Ampel, CSV-Export |
| `Lageroptimierung_Analyse_Tool.html` | **App:** Schnellanalyse der Verkaufsstatistik (Pareto, ABC-XYZ-Matrix, Kanäle) |
| `Workbook_Lageroptimierung_Datenauswertung.html` | Schulungsunterlage für Mitarbeiter: alle Formeln und Schritte erklärt, mit Übungen |
| `WinLine_Exportanleitung_Lageroptimierung.html` | Anleitung: welche Daten wie aus der WinLine exportiert werden (Exporte A–E) |

## Verwendung

1. Repository klonen oder als ZIP herunterladen (grüner **Code**-Button → *Download ZIP*)
2. `index.html` im Browser öffnen (Doppelklick genügt, keine Installation nötig)
3. In den Apps die WinLine-Exporte per Drag & Drop importieren

Die Apps benötigen Internet nur zum Laden zweier Bibliotheken (Chart.js, SheetJS via CDN). **Alle importierten Daten werden ausschließlich lokal im Browser verarbeitet** – nichts wird hochgeladen.

## Wichtig

- **Keine echten Exportdateien (Verkaufszahlen, Preise, Kundendaten) ins Repository legen.**
- Empfehlung: Repository **privat** halten und Kollegen als Collaborator einladen.

## Methodik (Kurzfassung)

- Nachfrage je Artikel auf Monatsbasis: μ (Durchschnitt), σ (Streuung), CV = σ/μ
- Klassifizierung: ABC nach Umsatz (80/15/5), XYZ nach CV (0,5 / 1,0)
- Sicherheitsbestand: `SB = z × σ × √L` (z aus Ziel-Servicelevel je ABC-Klasse)
- Meldebestand: `MB = μ × L + SB`
- Bestellmenge: `EOQ = √(2·D·S/H)` mit `H = Einstandspreis × Lagerkostensatz`, aufgerundet auf MOQ/VPE

Details und Beispiele: siehe Workbook.
