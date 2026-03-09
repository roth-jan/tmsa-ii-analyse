# TMSA-II UI Modules & Workflows — Vollständige Analyse

## Core Business Flow

```
Avis Entry (avis)
  → Dispatch Planning (mengenplan) [Avise → Touren zuweisen]
    → Departure Management (abfahrt) [Borderos erstellen, KFZ zuweisen]
      → Post-Processing (nacharbeit) [Korrekturen, Quittungen]
        → Billing (tuabrechnung) [Bewerten → Freigeben → Erzeugen]
          → Reporting (berichtswesen) [Dokumente, Statistiken]
```

## Master Data Dependencies

```
TU (tu) → KFZ (kfz) → KFZ-Kondition
        → Transportbehälter
Route (route) → RouteSuffix
DispoOrt (dispoort) → Mengenplan Spaltenköpfe
DispoRegel (disporegel) → Automatische Disposition
DFÜAutorouting (dfüautorouting) → EDI-Konfiguration
Forecast (forecast) → Kapazitätsplanung
Kondition (kondition) → Preis-/Kostenberechnung
```

---

## 1. tmsa.ui.avis — Avis-Verwaltung

### Zweck
Erfassung und Verwaltung von Avisen (Voranmeldungen). Typen: Vollgut (VG), Leergut (LG), Materialrückführung (MRF), Leergutrückführung (LRF), 3G (Cross-Dock).

### Screens
- **frmAvis** — Haupt-Avisverwaltung (Master-Detail: Avis-Header + Artikelzeilen)
- **frmLeergut** — Leergut-Avise
- **frm3gV3** — Drittgeschäft-Avise
- **frmAvisSuche** — Suche nach Avisnummer, Lade-/Lieferdatum, Versender, Empfänger, OEM
- **frmWarenkorb** — Avis-Vorlagen
- **frmADR** — Gefahrgutdaten pro Artikelzeile

### Entities
- **Avis**: AvisID, Avisnummer, LadeDatum, LieferDatum, LadeZeitVon/Bis, LieferZeitVon/Bis, VersenderKontaktFK, RoutenFK, OEMFK, NiederlassungFK, AvisStatus, AvisArtFK, Bemerkung, Fremdnummer
- **Artikelzeile**: EmpfängerKontaktFK, PackmittelFK, Anzahl, Gewicht, Lademeter, LademeterMega, AbladestelleFK, KFZFK, EinsatzArtFK, TUFK, TransportbehälterFK
- **ArtikelzeileADR**: Gefahrgutdaten pro Zeile

### Business Rules
- **Datumvalidierung**: LadeZeitVon < LadeZeitBis; LieferZeitVon < LieferZeitBis; LadeDatum < LieferDatum
- **Ladedatum-Fenster**: Je nach Tageszeit 0–14 oder 1–14 Tage; Wochenende: Fr+1 = Mo
- **Pflichtfelder VG/LRF**: Abladestelle, Packmittel, Anzahl > 0, Gewicht > 0, OEM, LademeterMega ≥ 0,01
- **Pflichtfelder LG/MRF**: OEM, Versender, Empfänger pro Artikelzeile
- **KFZ-Vordisposition**: Wenn KFZ gesetzt → EinsatzArt erforderlich
- **Sperrzeit-Prüfung**: Gegen WerkSperrzeiten (Abladestelle, Werk, OEM)
- **AvisStatus**: 0=Freigegeben, 1=Gesperrt, 2=Erfasst (braucht Freigabe-Recht), 3=Storniert
- **Berechtigungen**: "Avis löschen", "Avis speichern", "Avis Freigabe", "Avisvorlage überschreiben/löschen"

### Workflow
1. Neu → Avisart wählen (VG/LG) → Versender eingeben → Empfänger, Route, Packmittel, Anzahl, Gewicht, LDM → Artikelzeilen hinzufügen → Speichern
2. Suche → Kriterien → Ergebnis laden
3. Vorlage speichern/laden für wiederkehrende Avise

---

## 2. tmsa.ui.mengenplan — Dispositionsboard (Kern-Modul)

### Zweck
Zentrale Dispositionsplanung. Visualisiert alle Avise als Pivot-Grid (Blockansicht) oder Liste. Disponenten weisen Sendungen Touren zu.

### Screens
- **frmMengenplan** — Haupt-Dispoboard mit:
  - **Blockansicht**: Vertikal = Versender/Empfänger, Horizontal = DispoOrte, Zellen = Artikelzeilen
  - **Listenansicht**: Tabellarische Alternative
  - **Fahrzeugliste**: Tour-Übersicht (TU, KFZ, Route, Kondition, Kosten)
  - **KFZ/TB-Sidebar**: Verfügbare Fahrzeuge und Container
  - **Filterleiste**: Datum, OEM, NL, Region, Gutart, Umschlagspunkte, Einsatzart

### Business Rules
- **VG/MRF**: Versender vertikal, Ziel horizontal
- **LG**: Umgekehrt
- **Tour-Operationen**: Zusammenfassen, Splitten, Löschen, TU/KFZ/Kondition zuweisen
- **Dispositionsregeln**: Automatische Zuweisung via Regelwerk
- **AbrechnungsStatus**: Bei Status=3 keine TU/KFZ-Änderung möglich
- **"Mengenplan nur sehen"**: Nur-Lesen-Zugriff

### Workflow
1. Datum/OEM/Filter wählen → Aktualisieren → Block-/Listenansicht
2. Artikelzeilen per Drag&Drop zu Touren zuweisen
3. Tour bearbeiten: TU, KFZ, Kondition, Einsatzart, Zugnummer setzen
4. Dispositionsregeln anwenden → automatische Zuweisung

---

## 3. tmsa.ui.abfahrt — Abfahrtsverwaltung

### Zweck
Verwaltung von Abfahrten, Borderos (Ladescheine) und dem operativen Versandprozess.

### Screens
- **frmAbfahrt** — Abfahrt suchen, Borderos verwalten, Sendungen zuordnen, RSN-Verwaltung
- **frmBorderoUebersicht** — Bordero-Übersicht
- **frmSendungSuche** / **frmTourSuche** / **frmKFZSuche** — Suchmasken

### Entities
- **Abfahrt**: AbfahrtID, Nummer, Datum, TUFK, KFZFK, RouteFK, AbrechnungsStatus
- **Bordero**: BorderoID, AbfahrtFK, OEMNummer, Gewicht, LDM
- **BorderoDetailBMW**: BMW-spezifisch mit Terminkategorien, Auftragsbereich

### Business Rules
- **Modi**: Start, Anzeige, Bearbeiten, BorderoNeu, leer, Abfahrtabgerechnet
- **Abgerechnete Abfahrt**: Bearbeitung gesperrt
- **Faktura Status zurücksetzen**: Spezialrecht erforderlich
- **Sendung**: Nur Referenznummer editierbar im Grid

---

## 4. tmsa.ui.nacharbeit — Nachbearbeitung

### Zweck
Post-Dispatch Korrekturen. Tourdaten editieren, Quittungen setzen, Leerfahrten markieren, Abrechnungsstopp setzen.

### Screens
- **frmNacharbeit** — Gleiche Fahrzeugliste wie Mengenplan plus:
  - AusrufA/B/G/Q/F Flags (Bordero-Status, Quittung, Gutschrift, Abrechnung)
  - Leerfahrt, Quittung, Quittungdatum, Abrechnungsstopp
- **frmDetailsicht** — Tour-Detailansicht
- **frmHistorie** / **frmBeleghistorie** — Änderungshistorie
- **frmStreckenabschnitt** — Streckenabschnitt-Editor

### Workflow
1. Datum + Filter → Touren laden
2. Inline-Editing: TU, KFZ, Einsatzart, Zugnummer, Kondition
3. Quittung setzen, Leerfahrt markieren, Abrechnungsstopp

---

## 5. tmsa.ui.tuabrechnung — TU-Abrechnung (Billing)

### Zweck
Drei-Phasen-Abrechnungsprozess: Bewerten → Freigeben → Erzeugen.

### Screens
- **TUAbrechnung** — Filter wie Nacharbeit + Aktionsbuttons: Bewerten, Freigeben, Erzeugen
- **Storno-Bereich**: Belegsuche, Storno-Ausführung, Wiederholungsdruck
- **Details** / **frmStornoDetail** — Abrechnungs-/Storno-Details
- **frmWeibudaProtokoll** — WeiBuDa-Protokoll

### Business Rules
- **AbrechnungsStatus**: 0=Offen, 1=Bewertet, 2=Freigegeben, 3=Erzeugt
- **Berechtigungen pro Phase**: bewerten, freigeben, drucken
- **Status-Progression**: Nur sequenziell 0→1→2→3
- **Steuerrechner**: Steuerberechnung integriert
- **Buerotex**: Textvorlagen für Abrechnungsbelege

### Workflow
1. Touren filtern → 2. Bewerten (Kosten berechnen via Kondition) → 3. Freigeben → 4. Erzeugen (Belege generieren)

---

## 6. tmsa.ui.sendungsbildung — Sendungserfassung

### Zweck
Erstellung einzelner Sendungen mit Versender/Empfänger, Verpackung, Gewicht, Gefahrgut.

### Richtungsarten
- **WE** (Wareneingang): Inbound mit Abladestelle
- **WA** (Warenausgang): Outbound mit Ladestelle
- **3G**: Kein Ladeort, OEM deaktiviert

### Workflow
1. Richtung wählen → Versender/Empfänger → Packmittel-Zeilen → Speichern → SLB/PUS-Nummern generiert → Labels drucken

---

## 7–16. Weitere Module (Kurzübersicht)

| Modul | Zweck |
|---|---|
| **tmsa.ui.tu** | Transport-Unternehmer Stammdaten (Adressen, NL-Zuordnung, Lizenzen, KFZ, Einsatzarten) |
| **tmsa.ui.kfz** | Fahrzeug-Stammdaten (Kennzeichen, Fabrikat, Schadstoffklasse, NL-/Kondition-Zuordnung) |
| **tmsa.ui.kondition** | Preiskonditionen (Tour-/Last-/Leerkm-Faktoren) |
| **tmsa.ui.route** | Routen-Stammdaten (OEM-bezogen, Suffixe, Sonderkennzeichen) |
| **tmsa.ui.forecast** | Mengenplanung (MaxFrachtDirekt/Lager pro OEM/NL/Relation) |
| **tmsa.ui.dispoort** | Dispositionsorte (Spaltenköpfe im Mengenplan) |
| **tmsa.ui.disporegel** | Dispositionsregeln (automatische Tour-Zuweisung nach Kriterien) |
| **tmsa.ui.recherche** | Kurzrecherche (Sendungssuche nach diversen Kriterien, Export CSV/Excel) |
| **tmsa.ui.dfüautorouting** | EDI-Routing (Übertragungsnorm pro OEM) |
| **tmsa.ui.berichtswesen** | Plugin-basiertes Berichtswesen (50+ Berichte, dynamisch geladen) |

---

## Querschnittsthemen

### Berechtigungssystem
Jede Aktion ist rechtebezogen via `TmsaInfo.hatRecht("modul|aktion")`. Beispiele:
- "tmsa.ui.tu|bearbeiten", "tmsa.ui.tu|loeschen", "tmsa.ui.tu|entsperren"
- "Avis loeschen", "Avis Freigabe", "Mengenplan nur sehen"
- "tmsa.ui.tuabrrechnung.bewerten/freigeben/drucken"
- "Faktura Status zuruecksetzen"

### UI-Patterns
- **Anzeige/Bearbeiten-Modus**: Jede Form schaltet zwischen Lesen und Editieren
- **Infragistics Controls**: UltraGrid, UltraToolbarsManager, UltraCombo, UltraDropDown
- **Lokalisierung**: Alle Grids und Combos via LokalisierungsManager
- **Grid-Layout-Persistenz**: Benutzerspezifische Layouts via GridLayoutMgr
- **DatenGeändert/DatenSauber-Events**: Steuern den Formular-Status

### Lookup-Tabellen
Zentrale `daNachschlageTabellen` liefert: OEM, Niederlassung, Route, RouteSuffix, TU, KFZ, Packmittel, Abladestelle, EntladeZone, EinsatzArt, LKWTyp, Fabrikat, Schadstoffklasse, Länder, ADR, Frankatur, etc.
