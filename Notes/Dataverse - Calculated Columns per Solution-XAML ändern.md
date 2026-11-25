---
categories:
  - "[[D365]]"
  - "[[Power Platform]]"
  - "[[Help]]"
  - "[[BBU]]"
tags:
  - Dataverse
  - Solution
  - Calculated Column
  - XAML
created: 2026-11-25
---
# Dataverse - Calculated Columns per Solution-XAML ändern

Fallbeispiel aus dem BBU-Projekt (Tabelle `bbu_Bewertung`, Standalone-Solution `temp_rs_standalone`): Berechnete Spalten zeigen nichts an, solange nicht alle Eingabewerte gefüllt sind. Die Absicherung (`null` → `0`) wurde per XAML in der Solution-ZIP versucht. Dabei hat der Import mehrfach Formeln abgelehnt – die Fehler und das Learning daraus stehen hier.

## Problem

`Summe Qualitätspunkte`, `Summe Preispunkte` und `Gesamtpunkte` bleiben leer, sobald **ein** Eingabewert leer ist. Gewünscht: leere Eingaben zählen als 0.

Die Spalten bauen aufeinander auf:

| Spalte | Berechnung |
|---|---|
| `Punktemax15TEST` | Jahre Berufserfahrung > 15 → 15, sonst der Wert selbst |
| `Summe Preispunkte` | `niedrigster Angebotspreis / Preis für eine Stunde * 50` |
| `Summe Qualitätspunkte` | `Punkte_Supervisorausbildung + Punktemax15TEST + Punkte_Supervisionsleistungen + Punkte_sonstige_Erfahrung` |
| `Gesamtpunkte` | `Summe Qualitätspunkte + Summe Preispunkte` |

Rollup-Spalten gibt es in der Solution keine, nur diese 4 berechneten Spalten (`SourceType 1`) und 3 Business Rules, die die Punkte-Spalten setzen.

## Ursache

- Eine berechnete Spalte liefert **null, sobald ein Operand null ist**. Das setzt sich nach oben fort: Ist `Punkte_sonstige_Erfahrung` leer, ist `Summe Qualitätspunkte` leer und damit auch `Gesamtpunkte`.
- Der Designer packt jeden Operanden in `SelectFirstNonNull(x)`. Das sieht nach Coalesce aus, hat aber **keinen Default-Wert** und hilft deshalb nicht.
- Die Business Rules setzen die Punkte-Spalten nur, wenn sie laufen. Bei alten Datensätzen bleiben die Eingaben `null`, bis jemand speichert.
- `Preis für eine Stunde` und `Niedrigster Angebotspreis` hatten keine Absicherung – leer, und bei Stundenpreis 0 auch noch eine Division durch 0.

## Entscheidung: Portal oder XAML?

**Immer zuerst im Maker-Portal ändern (DEV) und dann exportieren.** Der Designer prüft beim Tippen, und der Export enthält genau die XAML, die der Import akzeptiert. XAML von Hand zu schreiben lohnt sich nur, wenn eine vorbereitete ZIP gebraucht wird – und dann sind mehrere Import-Runden einzuplanen.

Portal-Variante für `Summe Preispunkte` (Wenn / Sonst im Berechnungs-Editor):

- **Wenn** `Niedrigster Angebotspreis` enthält Daten **und** `Preis für eine Stunde` enthält Daten
  - **Dann** `Niedrigster Angebotspreis / Preis für eine Stunde * 50`
- **Sonst** `Summe Preispunkte` = 0

Für `Punktemax15TEST`: Wenn Jahre > 15 → 15, sonst wenn Jahre enthält Daten → Jahre, sonst → 0.

## Verlauf der Import-Versuche

Alle Versuche waren Importe von `temp_rs_standalone`. Stand: 2026-11-25.

| Version | Inhalt | Ergebnis |
|---|---|---|
| 1.0.0.5 | 3 Änderungen auf einmal (`Punktemax15TEST`, `Summe Preispunkte` mit 3 Bedingungen inkl. `> 0`, Business Rule) | Fehler bei `bbu_summepreispunkte` (nur die Meldung „Error while importing Calculated fields name ..."). Vermutete Ursache siehe unten, nicht bestätigt. |
| 1.0.0.6 | wie 1.0.0.5, aber `Summe Preispunkte` im Original | **Import OK** → `Punktemax15TEST` (NotNull-Zweig + Auffangzweig) und die Business-Rule-Erweiterung sind erprobt. |
| 1.0.0.7 (minimal) | `Gesamtpunkte`, `Summe Qualitätspunkte` mit `SelectFirstNonNull(x, 0)` (0 als Default-Wert) | Fehler **0x80060424** „The bbu_gesamtpunkte field has an invalid XAML formula definition" |
| 1.0.0.8 (minimal) | `Gesamtpunkte` (beide / nur Qualität / nur Preis / sonst 0), `Summe Preispunkte` (beide Preise / sonst 0), jeweils mit NotNull-Zweigen | Fehler **0x80060417** „FormulaDefinition is not valid Xaml" bei `bbu_gesamtpunkte`. Ursache: `xmlns:mxsq` fehlte im Kopf (siehe unten). |
| 1.0.0.8 (korrigiert) | wie oben, `xmlns:mxsq` im Kopf ergänzt | **Import OK** → bestätigt: `NotNull` + `And` (2 Bedingungen) funktioniert, sobald `xmlns:mxsq` im Kopf steht. |

### Fehler 1: 0x80060424 „invalid XAML formula definition"

Die XAML war ladbar, aber der Inhalt unzulässig. Auslöser: ein zweiter Parameter bei `SelectFirstNonNull` (0 als Default) plus ein zusätzlicher `CreateCrmType`-Schritt. Der Importer akzeptiert nur Bausteine in der Form, wie der Designer sie selbst erzeugt.

### Fehler 2: 0x80060417 „FormulaDefinition is not valid Xaml"

`Gesamtpunkte` und `Summe Preispunkte` hatten vorher keine Bedingung und deklarierten deshalb im `<Activity>`-Kopf **kein** `xmlns:mxsq` (`clr-namespace:Microsoft.Xrm.Sdk.Query;assembly=Microsoft.Xrm.Sdk, ...`). Die neuen Zweige verwenden `mxsq:ConditionOperator` und `mxsq:LogicalOperator` aber in `x:TypeArguments`.

Warum wurde das nicht vorher bemerkt: Ein XML-Parser prüft Namespace-Präfixe nur bei Element- und Attributnamen, **nicht** in Attributwerten. Die Datei war „wohlgeformt", für den XAML-Loader aber ungültig. Sehr wahrscheinlich war das auch der Grund für den ersten Fehler bei `bbu_summepreispunkte` (1.0.0.5) – bestätigt ist das nicht.

## Was der Importer akzeptiert

| Baustein | Status |
|---|---|
| Zweig mit `NotNull`-Bedingung + Auffangzweig (`true`), der eine Konstante setzt | akzeptiert (1.0.0.6) |
| Zusätzlicher Auffangzweig in einer Business Rule | akzeptiert (1.0.0.6) |
| `SelectFirstNonNull(x, 0)` mit Default | **abgelehnt** (0x80060424) |
| `NotNull` + `And` (2 Bedingungen) mit deklariertem `mxsq` (`Gesamtpunkte`, `Summe Preispunkte`) | akzeptiert (1.0.0.8 korrigiert) |
| `And` aus 3+ Bedingungen, `GreaterThan` gegen Dezimalkonstante | nicht bestätigt |

Regeln, die daraus folgen:

- Nur Bausteine verwenden, die der Designer selbst erzeugt – aus einer vorhandenen exportierten Formel kopieren, nichts erfinden.
- Führt man in einer bisher bedingungslosen Formel die erste Bedingung ein, den Namespace `mxsq` im Kopf ergänzen.
- Eine berechnete Spalte arbeitet wie if / else-if / else: Der **erste** wahre Zweig gewinnt. „Sonst" ist ein Zweig mit der Bedingung `true`.
- Bei n optionalen Summanden braucht man 2^n Zweige für „leer → 0". Besser: die Eingänge sicher füllen (Business Rule mit Else-Zweig, einmaliger Backfill alter Datensätze) oder Hilfsspalten, statt einer 16-Zweige-Formel.
- Dateien ASCII und mit CRLF belassen, nur die geänderten Formeln anfassen.

## Minimal-Import (nur die geänderten Spalten)

Ein Voll-Import überschreibt Formulare, Ansichten und alles andere in der Zielumgebung. Deshalb eine reduzierte Kopie bauen:

- `customizations.xml`: unter `Entities/Entity/EntityInfo/entity/attributes` nur Primärschlüssel, Name-Spalte und geänderte Spalten behalten. `FormXml`, `SavedQueries`, `RibbonDiffXml` der Entity entfernen, ebenso die Kinder von `Workflows` und `EntityRelationships`.
- `Formulas/`: nur die geänderten `.xaml`-Dateien, `[Content_Types].xml` bleibt.
- `solution.xml`: `<Version>` erhöhen (pro Versuch neu), `<MissingDependencies>` leeren; das Root-Component der Entity (`type="1"`) kann stehen bleiben.
- ZIP mit `customizations.xml` und `solution.xml` zuerst packen.
- Die Spalten, die die Formeln lesen, müssen in der Zielumgebung schon existieren.
- Business Rules sind Workflow-Komponenten – sie gehören nicht zur Spaltenliste und müssen bei Bedarf separat in die ZIP.

Beobachtung: Entity und Ansichten einer solchen reduzierten ZIP liefen ohne Fehler durch; die Formeln werden danach geprüft (bei ca. 71 % Fortschritt).

## Import-Log lesen

„Protokoll herunterladen" im Solution-Verlauf liefert eine **Excel-XML-Datei** (SpreadsheetML), keinen Text. Interessant sind die Zeilen mit `ErrorCode` und `ErrorText`:

- Die Meldung auf Solution-Ebene nennt nur die Spalte („Error while importing Calculated fields name bbu_x").
- Die Zeile vom Typ `Feld` enthält Code und Text.
- Gemeldet wird nur die **erste** fehlerhafte Spalte – mehrere kaputte Formeln erscheinen deshalb nacheinander.

Zum Auslesen gibt es ein kleines Skript im Projekt (siehe unten).

## Vor jedem Import prüfen

- Alle XAML-Dateien der ZIP (auch die unveränderten) durch den Check laufen lassen: Präfixe im Kopf deklariert, Variablen deklariert und nicht doppelt, CRLF.
- Neue Versionsnummer, neues ZIP.
- Die Portal-Schritte als Fallback bereithalten.

## Manueller Test nach dem Import

1. Neuen Datensatz nur mit Namen anlegen → `Summe Qualitätspunkte`, `Summe Preispunkte`, `Gesamtpunkte` zeigen **0**.
2. Jahre Berufserfahrung = 20 → `Punktemax15TEST` = 15; 8 → 8; leer → 0.
3. Angebotspreis 40, Stundenpreis 50 → `Summe Preispunkte` = 40 / 50 × 50 = 40; Stundenpreis leeren → 0.
4. Supervisionsstunden setzen und wieder leeren → `Punkte Supervisionsleistungen` jeweils 0.
5. Alten Datensatz mit leeren Punkte-Feldern öffnen → keine leere berechnete Spalte.

## Manuelle Anpassungen in der Umgebung (nicht Teil der ZIP)

- `Preis für eine Stunde` und `Niedrigster Angebotspreis`: „Geschäftlich erforderlich".
- `Preis für eine Stunde`: Mindestwert 0,01 (der Standard-Mindestwert ist -100000000000). Die Formel für `Summe Preispunkte` ohne `> 0`-Bedingung fängt Stundenpreis = 0 nicht ab.

## Offene Punkte

- [x] Import von 1.0.0.8 (korrigiert): durchgegangen (2026-11-25), manueller Test in Dynamics steht noch aus.
- [ ] `Summe Qualitätspunkte` mit 3 Optionen → 8 Zweige (`And` aus 3 Bedingungen) – Machbarkeit offen.
- [ ] Alte Datensätze: `Punkte_Supervisionsleistungen` bleibt leer, bis gespeichert wird (Backfill nötig, falls viele).

## Links

- Projekt-Skill: `.claude/skills/dataverse-calculated-column-xaml/` im Repo Avanade.BBU.CSM (Regeln, XAML-Bausteine, `check-formula-xaml.py`, `read-import-log.py`)
- [[PL-400 - Dataverse Solution Layers]]
- [[Dataverse]] · [[Calculated Column]] · [[Business Rule]] · [[Solution]]
