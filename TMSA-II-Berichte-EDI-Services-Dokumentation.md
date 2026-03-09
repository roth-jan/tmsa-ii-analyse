# TMSA-II Berichte, EDI/VDA-Schnittstellen und Hintergrunddienste

## Dokumentation - Stand: 09.03.2026

---

# Teil 1: Berichtsmodule (tmsa.bericht.*)

Alle Berichte implementieren das Interface `IBerichtWrapper` und werden ueber `IBerichtMetaDaten` klassifiziert. Sie nutzen den WCF-Service `SvcBerichtswesen.BerichtswesenServiceClient` fuer den Datenzugriff. Berichtsanzeige erfolgt ueber DevExpress XtraReports (`BerichtsAnzeigeFormular`).

## Berichtsarten (EBerichtsart)

| Berichtsart | Beschreibung |
|---|---|
| **Abfertigungsbericht** | Berichte fuer den Abfertigungsprozess (Wareneingang/-ausgang) |
| **Dispobericht** | Berichte fuer die Tourenplanung und Disposition |
| **Datenbankanfrage** | Reine Datenexporte (CSV/Excel) ohne visuelle Darstellung |
| **OhneOberfaeche** | Hintergrundberichte, die automatisch verarbeitet werden |
| **Unbekannt** | Sonderkategorie (z.B. Weibuda-Protokolle) |

---

## 1.1 Abfertigungsberichte

### 1. Erloesuebersucht (AbfertigungErloesliste)
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.AbfertigungErloesliste/BerichtWrapper.vb`
- **Name:** "Erlosuebersicht"
- **Typ:** Abfertigungsbericht
- **Format:** CSV-Export (Semikolon-getrennt, UTF-8)
- **Parameter:** NiederlassungFK, Filter, AlleTourenAusgeben, StatusinformationenErmitteln, VersenderAusgeben
- **Datenquelle:** `svc.HoleAbfertigungErloes()` - Erloesuebersucht der Abfertigung
- **Besonderheit:** Reisenummer wird Excel-kompatibel formatiert (`="wert"`)

### 2. Ausfallfrachten
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.ausfallfrachten/BerichtWrapper.vb`
- **Name:** "Ausfallfrachten"
- **Typ:** Abfertigungsbericht
- **Format:** CSV-Export
- **Parameter:** DatumStart, DatumEnde, Filename, Lieferanten, OEM
- **Datenquelle:** `svc.Ausfallfrachten_erstellen()` - Vergleich avisierter vs. tatsaechlicher Mengen
- **Bereinigung:** Entfernt `00:00:00` und `01.01.1900` Platzhalter aus Export

### 3. Ausfallfrachten Tour
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.ausfallfrachtentour/BerichtWrapper.vb`
- **Name:** "Ausfallfrachten Tour"
- **Typ:** Abfertigungsbericht
- **Format:** CSV-Export (Vollgut und Leergut getrennt)
- **Parameter:** DatumStart, DatumEnde, NiederlassungID, TuID, OemID, Typ (V/L), ExportFilename
- **Datenquelle:** `svc.AusfallfrachtenTour_erstellen()`
- **Spalten Vollgut:** Datum, TRNR, Avis Lieferant/Versender, Ort, TU, Empfaenger Ort, Avisiertes/Erfasstes Gewicht+LDM, Differenzen, Fracht, Anteile
- **Spalten Leergut:** Datum, TRNR, Avis Empfaenger, Empfaenger Ort, TU, Versender Ort, Gewicht/LDM, Differenzen, Fracht, Anteile

### 4. Borderodruck
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.borderodruck/BerichtWrapper.vb`
- **Name:** "Borderodruck"
- **Typ:** Abfertigungsbericht
- **Format:** DevExpress XtraReport (Druckvorschau + Druck)
- **Parameter:** NiederlassungID, ListAbfahrtID (List(Of Guid)), BenutzeStandartAnzahlKopien, AnzahlKopien
- **Datenquelle:** `svc.Borderodruck_Holen()` - Bordero-Kopf, Sendungsdaten, Gefahrgutanhang
- **OEM-spezifische Layouts:**
  - **BMW:** Eigenes Layout (`BorderodruckBMW`) mit Sendungszaehler und Seitenberechnung
  - **Daimler:** Eigenes Layout (`BorderodruckDaimler`) + Deckblatt, Sonderfall-Relationen 813/313
  - **Porsche:** Eigenes Layout (`BorderodruckPorsche`)
  - **MAN:** Layout aus Embedded Resource (DE/EN)
  - **Brose:** Layout aus Embedded Resource (DE/EN)
  - **Standard:** Fallback-Layout (DE/EN)
- **Besonderheiten:**
  - Multi-Layout-Report: Pro Bordero wird OEM-spezifisches Layout geladen
  - Eurogitterpaletten-Zaehlung fuer Relation 830
  - Packmittelbegleitschein-Erstellung fuer Daimler bei EmpfaengerOrt "Neu-Ulm" + Tauschmittel=2
  - Kopienanzahl aus NiederlassungOEM-Zuordnung
  - Externe Nummern werden am Bindestrich abgeschnitten

### 5. E+V Kosten (EVKosten)
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.evkosten/BerichtWrapper.vb`
- **Name:** "E+V Kosten"
- **Typ:** Abfertigungsbericht
- **Format:** DevExpress XtraReport + CSV-Export
- **Parameter:** NiederlassungID, DatumStart, DatumEnde
- **Datenquelle:** `svc.EVKosten_Holen()` - Eingangs- und Versandkosten
- **CSV-Spalten:** OEM, Datum, Relation, Gesamt Anzahl/Tonnage, EV Anzahl/Tonnage/Betrag, Nicht EV Anzahl/Tonnage
- **Sortierung:** Datum ASC, Relation ASC

### 6. Frachtbrief
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.frachtbrief/BerichtWrapper.vb`
- **Name:** "Frachtbrief"
- **Typ:** Abfertigungsbericht
- **Format:** DevExpress XtraReport
- **Parameter:** SendungID
- **Datenquelle:** `svc.Frachtbrief_Holen()`

### 7. Gefahrgut-Statistik
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.gefahrtgut_statistik/BerichtWrapper.vb`
- **Name:** "Gefahrgut-Statistik"
- **Typ:** Abfertigungsbericht
- **Format:** CSV-Export
- **Parameter:** DatumStart, DatumEnde, Filename
- **Datenquelle:** `svc.Gefahrgut_Statistik_erstellen()`

### 8. Interne Bemerkung
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.internebemerkung/BerichtWrapper.vb`
- **Name:** "Interne Bemerkung"
- **Typ:** Abfertigungsbericht
- **Format:** CSV-Export
- **Parameter:** NiederlassungFK, SendungDatumVON, SendungDatumBIS
- **Datenquelle:** `svc.InterneBemerkung_Holen()`

### 9. Mistralvorschlag
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.mistralvorschlag/BerichtWrapper.vb`
- **Name:** "Mistralvorschlag"
- **Typ:** Abfertigungsbericht
- **Format:** CSV-Export mit spezifischem Layout
- **Parameter:** datumStart, datumEnde, niederlassungID, filename
- **Datenquelle:** `svc.Mistralvorschlag_erstellen()`

### 10. Offene DFUE (OffeneDFUE)
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.offenedfü/BerichtWrapper.vb`
- **Name:** "OffeneDFUE"
- **Typ:** Abfertigungsbericht
- **Format:** DevExpress XtraReport
- **Parameter:** NiederlassungID, DatumStart, DatumEnde
- **Datenquelle:** `svc.OffeneDFUEPUS_Holen()` und `svc.OffeneDFUEVDA4913_Holen()` - Zwei parallele Abfragen

### 11. Packmittelbegleitschein
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.packmittelbegleitschein/BerichtWrapper.vb`
- **Name:** "Packmittelbegleitschein"
- **Typ:** Abfertigungsbericht
- **Format:** DevExpress XtraReport
- **Parameter:** AbfahrtID
- **Datenquelle:** `svc.PackmittelBegleitschein_Holen()`

### 12. PUS Nachdruck
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.pusnachdruck/BerichtWrapper.vb`
- **Name:** "PUSNachdruck"
- **Typ:** Abfertigungsbericht
- **Format:** DevExpress XtraReport
- **Parameter:** DocumentNumber (List(Of String)), NiederlassungID
- **Datenquelle:** `svc.PUSNachdruck_Holen()`, `svc.PUSNachdruckNachTourID_Holen()`
- **Versandfunktion:** `getVersandReport()` fuer automatisierten Dokumentversand

### 13. Reklamation
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.reklamation/BerichtWrapper.vb`
- **Name:** "Reklamation"
- **Typ:** Abfertigungsbericht
- **Format:** CSV-Export
- **Datenquelle:** Ueber NameWertEintrag-basierte Service-Aufrufe

### 14. Rollkarte
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.rollkarte/BerichtWrapper.vb`
- **Name:** "Rollkarte"
- **Typ:** Abfertigungsbericht
- **Format:** DevExpress XtraReport
- **Parameter:** RollkarteID
- **Datenquelle:** `svc.Rollkarte_Holen()`, `svc.RollkarteID_Holen()`

### 15. Sendungen mit Tauschmittel
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.sendungenmittauschmittel/BerichtWrapper.vb`
- **Name:** "Sendungen mit Tauschmittel"
- **Typ:** Abfertigungsbericht
- **Format:** DevExpress XtraReport + CSV
- **Parameter:** DatumStart, DatumEnde, VG, LG
- **Datenquelle:** `svc.SendungenMitTauschmittel_Holen()`

### 16. Sonderfahrt
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.sonderfahrt/BerichtWrapper.vb`
- **Name:** "Sonderfahrt"
- **Typ:** Abfertigungsbericht
- **Format:** CSV-Export
- **Parameter:** NiederlassungFK, SendungDatumVON, SendungDatumBIS, listOEMID, listKostentraeger, listVerursacher, listVeranlasser
- **Datenquelle:** `svc.Sonderfahrt_Holen()`

### 17. Verbringungsnachweis
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.verbringungsnachweis/BerichtWrapper.vb`
- **Name:** "Verbringungsnachweis"
- **Typ:** Abfertigungsbericht
- **Format:** DevExpress XtraReport
- **Parameter:** NiederlassungID, SendungDatumVon, SendungDatumBis, LieferantenMistralNummer, SendungID, OEMIDs
- **Datenquelle:** `svc.Verbringungsnachweis_Holen()`, `svc.Verbringungsnachweis_Sendung()` (Einzelsendung)

### 18. Weikunda-Protokoll Entladelisten
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.weikundaprotokollentladelisten/BerichtWrapper.vb`
- **Name:** "Weikunda-Protokoll Entladelisten"
- **Typ:** Abfertigungsbericht
- **Format:** DevExpress XtraReport
- **Parameter:** NiederlassungID, SendungListeID, WeikundaReferenzNummer
- **Datenquelle:** `svc.WeikundaProtokollEntladelisten_Holen()`, `svc.EntladelistenWeikundaReferenznummern_Holen()`

---

## 1.2 Dispoberichte

### 19. Ausgangsplan (Ausgangsliste)
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.ausgangsliste/BerichtWrapper.vb`
- **Name:** "Ausgangsplan"
- **Typ:** Dispobericht
- **Format:** DevExpress XtraReport + CSV-Export
- **Parameter:** NiederlassungID, DatumVON, DatumBIS, listZugnummernKreisID, UmschlagpunktID
- **Datenquelle:** `svc.Ausgangsplan_Holen()` - Tour-, Arbeitsablauf-, Streckenabschnitt-Daten
- **CSV-Spalten:** TRNR, Reisenummer, TU, KFZ, Transportbehaelter, Start 1-2, Zielort 1-5
- **Lokalisierung:** Unterstuetzt Mehrsprachigkeit (Culture-Management)

### 20. Avis
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.avis/BerichtWrapper.vb`
- **Name:** "Avis"
- **Typ:** Dispobericht
- **Format:** DevExpress XtraReport
- **Parameter:** AvisNummer, AvisID, NiederlassungID
- **Datenquelle:** `svc.Avis_Holen()` - Avis-Details mit Request/Response-Pattern

### 21. DESADV-Avise
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.DESADVAvise/BerichtWrapper.vb`
- **Name:** "DESADV-Avise"
- **Typ:** Dispobericht
- **Format:** CSV-Export
- **Parameter:** LadeDatum, ExportFilename
- **Datenquelle:** `svc.DESADVDaten_Holen()`
- **CSV-Spalten:** Ladedatum, OEM, Versender, Strasse, Ort, Empfaenger, Werk Externenummer, Fremdnummer, Artikel Anzahl, Packmittel, Gewicht, Lademeter

### 22. DESADV zuruecksetzen
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.desadvreset/BerichtWrapper.vb`
- **Name:** "DESADV zuruecksetzen"
- **Typ:** Dispobericht
- **Format:** Kein Report - Operationsaufruf
- **Parameter:** Lieferscheinnummer
- **Datenquelle:** `svc.DesadvReset()` - Setzt DESADV-Status per Auftragsnummer zurueck

### 23. Fahranweisung
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.fahranweisung/BerichtWrapper.vb`
- **Name:** "Fahranweisung"
- **Typ:** Dispobericht
- **Format:** DevExpress XtraReport mit Subreport (Arbeitsablauf)
- **Parameter:** TourDatum, TourNummerVon, TournummerBis, ZugnummerVon, ZugNummerBis, TourID, Benutzer, listZugnummernKreis
- **Datenquelle:** `svc.Verlademeldung_Holen()` - Verlademeldungsdaten mit Arbeitsablauf

### 24. Fahrzeugliste
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.fahrzeugliste/BerichtWrapper.vb`
- **Name:** "Fahrzeugliste"
- **Typ:** Dispobericht
- **Format:** DevExpress XtraReport
- **Parameter:** NiederlassungID, AbfrageModus, TourID, TourDatum, TourNummerVon/Bis, ZugNummerVon/Bis, listZugnummernkreis, StreckenabschnittIDs
- **Datenquelle:** `svc.Fahrzeugliste_Holen()`
- **Versandfunktion:** `getVersandReport()` fuer automatischen Versand mit Sprachauswahl

### 25. IFCSUM Uebersicht
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.ifcsumuebersicht/`
- **Name:** "IFCSUM Uebersicht"
- **Typ:** Dispobericht

### 26. IFCSUM zuruecksetzen
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.ifcsumreset/`
- **Name:** "IFCSUM zuruecksetzen"
- **Typ:** Dispobericht

### 27. Ladeschein
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.Ladeschein/BerichtWrapper.vb`
- **Name:** "Ladeschein"
- **Typ:** Dispobericht
- **Format:** DevExpress XtraReport
- **Datenquelle:** `svc.Ladestelle_Holen()`

### 28. Lagerliste
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.lagerliste/BerichtWrapper.vb`
- **Name:** "Lagerliste"
- **Typ:** Dispobericht
- **Format:** DevExpress XtraReport
- **Parameter:** DatumStart, DatumEnde, OEMID, UmschlagpunktID, ZulaufKennzeichen, AnschlussKennzeichen, Direkt, NiederlassungFK

### 29. Lieferantenavis
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.lieferantenavis/BerichtWrapper.vb`
- **Name:** "LieferantenAvis"
- **Typ:** Dispobericht
- **Format:** DevExpress XtraReport
- **Parameter:** NiederlassungID, LadeDatum, LieferantID, listOEMID
- **Datenquelle:** `svc.LieferantenAvis_Holen()`
- **Versandfunktionen:** `getVersandReport()`, `VersandRequest()` - Automatisierter Dokumentversand

### 30. Masterplan
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.masterplan/BerichtWrapper.vb`
- **Name:** "Masterplan"
- **Typ:** Dispobericht
- **Format:** DevExpress XtraReport
- **Parameter:** NiederlassungID, TourDatum, TourNummerVon/Bis, ZugnummerVon/Bis, FrachtAnzeigen, listZugnummernkreis
- **Datenquelle:** `svc.Masterplan_Holen()`
- **Besonderheit:** Umschlagspunkt-Gruppierung - aggregiert Strecken mit gleichen Start-/Zielkontakten

### 31. NVDispoplan
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.nvdispoplan/BerichtWrapper.vb`
- **Name:** "NVDispoplan"
- **Typ:** Dispobericht
- **Format:** DevExpress XtraReport
- **Parameter:** NiederlassungID, UmschlagPunktID, TourDatum, NiederlassungAnzeigeText, BerichtName
- **Datenquelle:** `svc.NVDispoplan_Holen()`

### 32. PUS Daten zuruecksetzen
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.pusdatenzurücksetzen/BerichtWrapper.vb`
- **Name:** "PUS Daten zuruecksetzen"
- **Typ:** Dispobericht
- **Format:** Kein Report - Operationsaufruf
- **Parameter:** NiederlassungID, DocumentNumber
- **Datenquelle:** `svc.PUSDatenZuruecksetzen()` - Setzt PUS-Daten zurueck

### 33. Tagesuebersicht Transportunternehmer
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.tagesübersichtTu/BerichtWrapper.vb`
- **Name:** "Tagesuebersicht Transportunternehmer"
- **Typ:** Dispobericht
- **Format:** DevExpress XtraReport
- **Parameter:** NiederlassungID, Tourdatum, TuID, listEinsatzArtID
- **Datenquelle:** `svc.TagesuebersichtTu_Holen()`, `svc.TagesuebersichtTU_VersandDaten_Holen()`
- **Versandfunktion:** `getVersandReport()` mit Versandart-Erkennung pro TU (Fax/EMail)

### 34. Tagesuebersicht TU Voravis
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.tagesübersichtTuVoravis/BerichtWrapper.vb`
- **Name:** "Tagesuebersicht Transportunternehmer Voravis"
- **Typ:** Dispobericht
- **Format:** DevExpress XtraReport
- **Parameter:** NiederlassungID, Tourdatum, TuID
- **Datenquelle:** `svc.TagesuebersichtTuVoravis_Holen()`
- **Versandfunktion:** `getVersandReport()` mit Versandart-Erkennung pro TU

### 35. Verladeplan
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.verladeplan/BerichtWrapper.vb`
- **Name:** "Verladeplan"
- **Typ:** Dispobericht
- **Format:** DevExpress XtraReport
- **Parameter:** NiederlassungID, TourDatum, TourNummerVon/Bis, ZugnummerVon/Bis, UmschlagPunktID, BerichtName, listZugNummernKreis
- **Datenquelle:** `svc.Verladeplan_Holen()`
- **Besonderheit:** Umschlagspunkt-Gruppierung (identisch zum Masterplan-Pattern)

### 36. Ueberhangliste
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.überhangliste/BerichtWrapper.vb`
- **Name:** (ohne MetaDaten)
- **Format:** DevExpress XtraReport
- **Parameter:** NiederlassungID, ListFahrtypID, DatumVON, DatumBIS, Richtungsart
- **Datenquelle:** `svc.Ueberhangliste_Holen()`

---

## 1.3 Datenbankanfragen (reine Exports)

### 37. Anzahl Avise
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.anzahlavise/BerichtWrapper.vb`
- **Name:** "Anzahl Avise"
- **Typ:** Datenbankanfrage
- **Format:** CSV-Export
- **Parameter:** NiederlassungID, DatumVon, DatumBis, ZielPfad
- **Datenquelle:** `svc.AnzahlAvise_Holen()` - Zaehlung der Avise pro Zeitraum

### 38. DFUE Fehler
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.dfüfehler/BerichtWrapper.vb`
- **Name:** "DFUE Fehler"
- **Typ:** Datenbankanfrage
- **Format:** CSV-Export + Excel-Export
- **Parameter:** NiederlassungID, DatumVon, DatumBis, AbfrageTyp, ZielPfad
- **AbfrageTypen:**
  - 0: VDA4913 MAN allgemein
  - 1: VDA4913 VWT fehlende Lieferantenuebersetzung
  - 2: VDA4913 VWT andere Fehler
  - 3: VDA4913 VWT fehlende Werke/ABS
  - 4: IFCSUM Brose Fehler
  - 5: VDA4913 Porsche
  - 6: VDA4913 Daimler
- **Datenquelle:** `svc.DFUEFehler_Holen()`, `svc.DFUFehler_Untyped_Holen()` (fuer IFCSUM)

### 39. DFUE Quote
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.dfüquote/BerichtWrapper.vb`
- **Name:** "DFUE Quote"
- **Typ:** Datenbankanfrage
- **Format:** CSV-Export
- **Parameter:** NiederlassungID, DatumVon, DatumBis, Herkunft, ZielPfad
- **Herkunft-Typen:** VDA4912-Sendung (0), PUS-Sendung (1), Brose-Avis (2), 3G-Avis (3)
- **Datenquelle:** `svc.DFUEQuote_Holen()` - Berechnung der elektronischen Uebermittlungsquote

### 40. Lieferantenexport
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.lieferanten/BerichtWrapper.vb`
- **Name:** "Lieferantenexport"
- **Typ:** Datenbankanfrage
- **Format:** CSV-Export
- **Parameter:** NiederlassungID, MitDunsJeOem, ZielPfad
- **Datenquelle:** Lieferantenstammdaten mit optionaler DUNS-Nummer pro OEM

### 41. Offene LG-Zustellavise
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.offenelgzustellavise/BerichtWrapper.vb`
- **Name:** "Offene LG-Zustellavise"
- **Typ:** Datenbankanfrage
- **Format:** CSV-Export
- **Parameter:** NiederlassungID, AuftragsDatum, ZielPfad
- **Datenquelle:** Offene Leergut-Zustellavise

---

## 1.4 Berichte ohne Oberflaeche (automatisierte Verarbeitung)

### 42. Entladeliste
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.entladeliste/BerichtWrapper.vb`
- **Name:** "Entladeliste" (lokalisierbar)
- **Typ:** OhneOberflaeche
- **Format:** DevExpress XtraReport
- **Parameter:** SendungListeID
- **Datenquelle:** `svc.Entladeliste_Holen()`

### 43. Quittungsanforderung
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.quittungsanforderung/BerichtWrapper.vb`
- **Name:** "Quittungsanforderung"
- **Typ:** OhneOberflaeche
- **Format:** DevExpress XtraReport + automatischer Versand
- **Funktionen:** `PrintReport_AllInOne()`, `sendReport()` - Druckt und versendet Quittungsanforderungen fuer Touren
- **Datenquelle:** `svc.AddQuittungsanforderung()`

### 44. Transportauftrag
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.transportauftrag/BerichtWrapper.vb`
- **Name:** "Transportauftrag"
- **Typ:** OhneOberflaeche
- **Format:** DevExpress XtraReport mit Sprachauswahl
- **Funktionen:** `GetVersandReport()`, `VersandRequest()`, `GetProbedruckReport()`
- **Versandarten:** Unterstuetzt automatischen Versand per Fax/EMail

### 45. TU-Abrechnung
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.tuabrechnung/BerichtWrapper.vb`
- **Name:** "TUAbrechnung"
- **Typ:** OhneOberflaeche
- **Format:** CSV-Export
- **Parameter:** DatumStart, DatumEnde, TUFK, AbrechnungsStatus (Liste), EinsatzArt (Liste)
- **Berechnungen:** Zugehoerigkeitsuebersetzung, Abrechnungsstatusuebersetzung

---

## 1.5 Sonstige

### 46. BMW Versandavis
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.bmwversandavis/`
- (Verzeichnis leer - nur Platzhalter)

### 47. Weibuda-Uebergabeprotokoll
- **Datei:** `/tmp/tmsa-analyse/TMSA-II/lib/tmsa.bericht.weibudaübergabeprotokoll/BerichtWrapper.vb`
- **Name:** "Weibuda-Uebergabeprotokoll"
- **Typ:** Unbekannt
- **Format:** DevExpress XtraReport
- **Parameter:** NiederlassungID, TransferID
- **Datenquelle:** `svc.WeibudaUebergabeProtokoll_Holen()`, `svc.TransferIDs_Holen()`

---

# Teil 2: EDI/VDA-Schnittstellen

## 2.1 NTC.Logistik.Edifact - EDIFACT Parser-Bibliothek

- **Pfad:** `/tmp/tmsa-analyse/NTC.Logistik.Edifact/`
- **Technologie:** C#, generische EDIFACT-Parser-Infrastruktur
- **Unterstuetzte Nachrichtentypen:**
  - **IFTSTA** (International Forwarding and Transport Status message) - Subset D16A
  - **IFCSUM** (International Forwarding and Consolidation Summary) - Import-Verarbeitung
- **Architektur:**
  - Base-Klassen fuer Segmente, Composites, Datenelemente
  - Enumerationen: Absicht, Format, EDIStatus, EDIComponentType, EDIFormat, Quelle, AvailabilityIndicator
  - Exception: `StructureViolationException` fuer Strukturfehler
  - FileViewer-Tool fuer EDIFACT-Dateien
- **UNA-Handling:** Automatisches Parsen des Service String Advice (Separator-Erkennung)

## 2.2 NTC.Logistik.VDA4913.Model - VDA 4913 Datenmodell

- **Pfad:** `/tmp/tmsa-analyse/NTC.Logistik.VDA4913.Model/`
- **Technologie:** C#, WCF DataContracts
- **VDA 4913 Satzarten (128 Byte Festformat):**

| Satzart | Klasse | Beschreibung |
|---|---|---|
| **711** | SatzArt711Base | Vorsatz - Uebertragungskopf (Empfaenger, Sender, Uebertragungsnummer, Datum) |
| **712** | SatzArt712Base | Transportsatz (SLB-Nr, Gewicht, Lademeter, Frachtfuehrer, LKW-Art) |
| **713** | SatzArt713Base | Lieferscheinkopf (LS-Nummer, Versanddatum, Abladestelle, Werk, Empfaenger) |
| **714** | SatzArt714Base | Lieferscheinposition (Sachnummer, Liefermenge, Mengeneinheit, Ursprungsland) |
| **715** | SatzArt715Base | Packmittelsatz (Packmittel-Nr, Anzahl, Fuellmenge, Verpackungsabmessung) |
| **716** | SatzArt716Base | Textsatz zur Position |
| **717** | SatzArt717Base | Einzelpackstuecksatz |
| **718** | SatzArt718Base | Produktionsnummernsatz |
| **719** | SatzArt719Base | Nachsatz - Zaehler aller Satzarten |

- **Variante:** SchenkerVA30MOD (DB-Schenker-spezifische Modifikation, Version 04, Stand Maerz/96)
- **Validierung:** Umfangreiche Feldvalidierung (Pflichtfelder, Numerisch/Alphanumerisch, Laengencheck, Positionseindeutigkeit)

## 2.3 NTC.Logistik.VDA4913.Export - VDA 4913 Export-Service

- **Pfad:** `/tmp/tmsa-analyse/NTC.Logistik.VDA4913.Export/`
- **Technologie:** WCF-Service (C#), Entity Framework
- **Service-Interface:** `IExportService.ExportSchenkerVA30MOD()`
- **Ablauf:**
  1. Empfang des VDA4913-Dokuments via WCF
  2. **Persistierung:** Speicherung aller Satzarten in Datenbank (TMSA2Entities-Kontext)
     - VA30MOD_Dokument -> VA30MOD_SatzArt711 -> VA30MOD_Transport -> VA30MOD_SatzArt712 -> VA30MOD_TransportInhalt -> VA30MOD_SatzArt713/714/715 -> VA30MOD_SatzArt719
  3. **Dateiexport:** Serialisierung in Flatfile (`.mod`-Datei, alle Felder position-sortiert aneinandergereiht)
  4. **Statusupdate:** Abfertigungsservice-Aufruf `VA30MODStatusSetzen()` (Status 2=Erfolg, 3=Fehler)
- **Protokollierung:** NTC.Logistik.Protocol mit DB-Writer (Severity-Levels)
- **Async:** Export laeuft asynchron (`Task.Factory.StartNew`)

## 2.4 NTC.Logistik.VDA4913.Transformation - VDA 4913 Transformation

- **Pfad:** `/tmp/tmsa-analyse/NTC.Logistik.VDA4913.Transformation/`
- **Technologie:** WCF-Service (C#)
- **Service-Interface:** `ITransformationService.TransformToSchenkerVA30MOD()`
- **Aufgabe:** Transformation von TMSA-internen Sendungsdaten in VDA 4913 SchenkerVA30MOD-Format
- **DTO-Objekte:**
  - Sendung/SendungListe - Sendungsdaten
  - VDA4913LieferscheinPosition/LieferscheinPositionListe
  - VDA4913LieferscheinPackstueck/LieferscheinPackstueckListe
  - ScandatenHalle/ScandatenHalleListe
- **Fehlerbehandlung:** Eigene Fault-Contracts (ArgumentNullFault, Fault)

## 2.5 OEM-spezifische VDA 4913 Import-Module

### BMW (tmsa-ii-vda4913bmw)
- **Pfad:** `/tmp/tmsa-analyse/tmsa-ii-vda4913bmw/`
- **Technologie:** VB.NET
- **Komponenten:** Kopierer, Datenquelle, Tester, Importierer (in VWT-Modul)
- **Besonderheit:** BMW-spezifischer VDA4913-Kopierer, Nutzung des VWT-Importierers

### VW Transport (tmsa-ii-vda4913vwt)
- **Pfad:** `/tmp/tmsa-analyse/tmsa-ii-vda4913vwt/`
- **Technologie:** VB.NET, Quartz.NET Scheduler
- **Komponenten:**
  - **Importierer** (`Importierer.vb`): Quartz-IJob, liest VDA4913-Dateien, parsed 128-Byte Saetze (711-719)
  - **Uebersetzer** (`Uebersetzer.vb`): Uebersetzt VDA-Rohdaten in TMSA-interne Strukturen
- **Ablauf:**
  1. Dateien aus Quellverzeichnis lesen (konfigurierbarer Import-Pfad + FileMask)
  2. Zeilenumbrueche entfernen, 128-Byte-Bloecke parsen
  3. Pro Satzart: ParseVDA4913Auftrag/Transport/Lieferschein/Position/Packmittel/Text/EinzelPackstueck
  4. Uebersetzung durch `Uebersetzer.start(ds, con)`
  5. Speicherung in Datenbank
  6. Erfolgreich: Datei nach Backup verschieben; Fehler: nach Error-Verzeichnis
- **Datenmapping:** Codepage 1252, VDA-Datumsformat (JJMMTT), Mengen *1000, Lademeter *10

### Daimler (tmsa-ii-vda4913daimler)
- **Pfad:** `/tmp/tmsa-analyse/tmsa-ii-vda4913daimler/`
- **Technologie:** VB.NET + C# (MailErsteller)
- **Komponenten:** Kopierer, Datenquelle, Importierer, Uebersetzer, MailErsteller, ZeitgeberDienst, Tester
- **Besonderheit:** Eigener MailErsteller fuer Daimler-spezifische Benachrichtigungen

### MAN (tmsa-ii-vda4913man)
- **Pfad:** `/tmp/tmsa-analyse/tmsa-ii-vda4913man/`
- **Technologie:** VB.NET
- **Komponenten:** Kopierer, Datenquelle, Importierer, Uebersetzer, ZeitgeberDienst, Tester

### Porsche (tmsa-ii-vda4913porsche)
- **Pfad:** `/tmp/tmsa-analyse/tmsa-ii-vda4913porsche/`
- **Technologie:** VB.NET + C# (MailErsteller)
- **Komponenten:** Kopierer, Datenquelle, Importierer, Uebersetzer, MailErsteller, ZeitgeberDienst, Tester

### VDA 4927 - VWT (tmsa-ii-vda4927)
- **Pfad:** `/tmp/tmsa-analyse/tmsa-ii-vda4927/`
- **Technologie:** VB.NET
- **Beschreibung:** VDA 4927 Import und Uebersetzung (Packmittelkonto/Leergutsteuerung)
- **Komponenten:** vwtimport.job, vwttranslate.job

## 2.6 DESADV VWT (tmsa-ii-desadvvwt)

- **Pfad:** `/tmp/tmsa-analyse/tmsa-ii-desadvvwt/`
- **Technologie:** C# (Altova MapForce-generiert) + VB.NET
- **Nachrichtentyp:** DESADV (Despatch Advice / Lieferavis) - UN/EDIFACT
- **Aufbau:**
  - **DESADV-Parser:** Auto-generiert durch Altova MapForce 2005, EDIFACT-Scanner mit UNA-Segment-Handling
  - **AltovaFunctions:** Core, Lang, Db, Edifact - Hilfsfunktionen
  - **TMSA2.Schnittstellen.Import.DESADV:** VB.NET-Import-Logik
- **UNA-Segment:** Automatische Separator-Erkennung (Component, Element, Decimal, Release, Segment)

## 2.7 IFCSUM (tmsa-ii-ifcsum)

- **Pfad:** `/tmp/tmsa-analyse/tmsa-ii-ifcsum/`
- **Technologie:** C#, Quartz.NET
- **Nachrichtentyp:** IFCSUM (Forwarding and Consolidation Summary Message) - UN/EDIFACT
- **Komponenten:**
  - **ifcsum.shell:** Konsolenhost
  - **tmsa.ifcsum.import.service:** Windows-Service-Host
  - **tmsa.ifcsum.import.aviscreator:** Quartz-Job, erzeugt Avise via SP `IFCSUM_CreateAvis`
  - **tmsa.ifcsum.import.mailcreator:** Quartz-Job, erzeugt Benachrichtigungs-Mails
- **Avis-Erstellung:** Ruft Stored Procedure `[dbo].[IFCSUM_CreateAvis]` auf
- **Fehlerbehandlung:** Bei Fatal-Error wird Job pausiert; bei SQL-Kommunikationsfehlern automatischer Reschedule (30s)

---

# Teil 3: Hintergrunddienste

## 3.1 AutoFaktura (tmsa-ii-autofaktura)

- **Pfad:** `/tmp/tmsa-analyse/tmsa-ii-autofaktura/`
- **Technologie:** VB.NET, Quartz.NET, WCF
- **Beschreibung:** Automatisierter Faktura-Dienst - erstellt Faktura-Schnittstellendaten fuer uebergebene Abfahrten
- **Ablauf:**
  1. SQL-Abfrage liest Abfahrten mit ausstehender Fakturierung
  2. Fuer jede AbfahrtID: WCF-Service-Aufruf `svc.erstelleFakturaSchnittstelle(AbfahrtID, False, False)`
  3. Logging: AbfahrtID, ExportDate, NiederlassungMatchcode, AbfahrtDatum, AbfahrtNummer
- **Fehlerbehandlung:**
  - Bei Fatal-Error: Job wird pausiert (`context.Scheduler.PauseJob`)
  - Fehler-Mail per SMTP an konfigurierte Empfaenger
  - WCF-Channel-Management: Automatische Wiederherstellung bei Faulted-State
- **Scheduling:** Quartz.NET mit JobKey "Faktura" / "TMSA-II"

## 3.2 Mailempfang (tmsa-ii-mailempfang)

- **Pfad:** `/tmp/tmsa-analyse/tmsa-ii-mailempfang/`
- **Technologie:** VB.NET, Quartz.NET, OpenPop.NET (POP3)
- **Beschreibung:** POP3-Mailempfangs-Dienst - ruft Mails von konfigurierten Postfaechern ab
- **Konfiguration:**
  - POP3-Server, User, Password (aus JobDataMap)
  - DB-Connection fuer Dokumentenspeicherung
- **Funktionalitaet:**
  - POP3-Client liest Mails ab
  - Attachments werden in Dokumenten-Datenbank gespeichert (`daDokument`)
  - Unterstuetzt Interrupt (`IInterruptableJob`)
- **Scheduling:** Quartz.NET mit konfigurierbarem Trigger

## 3.3 Mailversand (tmsa-ii-mailversand)

- **Pfad:** `/tmp/tmsa-analyse/tmsa-ii-mailversand/`
- **Technologie:** VB.NET, Quartz.NET, System.Net.Mail (SMTP)
- **Beschreibung:** SMTP-Mailversand-Dienst - versendet ausstehende Mails aus der Datenbank
- **Konfiguration:**
  - SMTP-Server, Login, Passwort
  - DefaultFromAddress, OverrideToAddress (fuer Test), BccAddress
  - TempSaveAttachments (fuer Debugging)
- **Funktionalitaet:**
  - Liest zu versendende Mails aus Datenbank (`daMailversand`)
  - Erstellt Betreff nach Dokumenttyp (`CreateSubjectByDokumentTyp`)
  - Erstellt Body (`CreateBody`)
  - SMTP-Authentifizierung (Credentials)
  - Abuse-Report-Adresse: `abuse@asp-central.de`
- **Deployment:** Topshelf-basierter Windows-Service oder klassischer Windows-Dienst

## 3.4 Datenimport (tmsa-ii-datenimport)

- **Pfad:** `/tmp/tmsa-analyse/tmsa-ii-datenimport/`
- **Technologie:** VB.NET, Windows Forms (GUI-Tool, kein Hintergrunddienst)
- **Beschreibung:** Interaktives Import-Tool fuer Stammdaten
- **Importierbare Objekte:**
  - Abladestellen
  - Werke
  - Lieferanten
  - Packmittel
  - Avise (mit Artikelzeilen, Routen)
- **Features:**
  - Multi-Datenbank-Unterstuetzung (Datenbankauswahl)
  - Import-Protokoll mit Statusanzeige
  - Anzeige nicht importierter Datensaetze (fehlende OEM, Werke, Lieferanten, Abladestellen, Packmittel)
  - Dezimaltrennzeichen-Konfiguration

## 3.5 TourStatus-Dienst (tmsa-ii-tourstatus)

- **Pfad:** `/tmp/tmsa-analyse/tmsa-ii-tourstatus/`
- **Technologie:** VB.NET, Quartz.NET
- **Beschreibung:** Automatisierter Dienst zur Tour-Status-Aktualisierung
- **Logik-Module:**
  - **TourLogik:** Hauptlogik fuer Tourstatus-Auswertung
  - **BroseArtikelLogik:** Entfernt Brose-Artikelzeilen (Sonderbehandlung)
  - **DrittgeschaeftAvisLogik:** Entfernt Drittgeschaeft-Aviszeilen
  - **LeergutZustellAvisLogik:** Leergut-Zustellavis-Verarbeitung
- **Ablauf:**
  1. Daten holen: Touren der letzten 22 Tage bis gestern (`Date.Today.AddDays(-22)` bis `-1`)
  2. Brose-Artikelzeilen entfernen
  3. Drittgeschaeft-Aviszeilen entfernen
  4. TourDetailErweitert-Touren entfernen
  5. Pro Tour: Logik anwenden, bei Bedarf Haken setzen
- **Fehlerbehandlung:** Fehler-Mail per SMTP

---

# Zusammenfassung

## Uebersicht Berichtsmodule

| # | Berichtname | Typ | Format | Bemerkung |
|---|---|---|---|---|
| 1 | Erlosuebersicht | Abfertigung | CSV | Abfertigungs-Erloesdaten |
| 2 | Ausfallfrachten | Abfertigung | CSV | Soll/Ist-Vergleich |
| 3 | Ausfallfrachten Tour | Abfertigung | CSV | VG/LG getrennt |
| 4 | Borderodruck | Abfertigung | Print | 6 OEM-Layouts, Gefahrgut |
| 5 | E+V Kosten | Abfertigung | Report+CSV | Kosten je Relation |
| 6 | Frachtbrief | Abfertigung | Report | Pro Sendung |
| 7 | Gefahrgut-Statistik | Abfertigung | CSV | Gefahrgut-Auswertung |
| 8 | Interne Bemerkung | Abfertigung | CSV | Sendungsbemerkungen |
| 9 | Mistralvorschlag | Abfertigung | CSV | Mistral-Tourenvorschlag |
| 10 | Offene DFUE | Abfertigung | Report | PUS + VDA4913 |
| 11 | Packmittelbegleitschein | Abfertigung | Report | Pro Abfahrt |
| 12 | PUS Nachdruck | Abfertigung | Report | Nachdruck + Versand |
| 13 | Reklamation | Abfertigung | CSV | Reklamationsauswertung |
| 14 | Rollkarte | Abfertigung | Report | Pro Rollkarte |
| 15 | Sendungen mit Tauschmittel | Abfertigung | Report+CSV | VG/LG-Filter |
| 16 | Sonderfahrt | Abfertigung | CSV | Mit OEM/Kostentraeger-Filter |
| 17 | Verbringungsnachweis | Abfertigung | Report | Einzel- und Sammelauswertung |
| 18 | Weikunda-Protokoll | Abfertigung | Report | Entladelisten-Protokoll |
| 19 | Ausgangsplan | Dispo | Report+CSV | Tourdaten mit Strecken |
| 20 | Avis | Dispo | Report | Einzel-Avis |
| 21 | DESADV-Avise | Dispo | CSV | DESADV-Export |
| 22 | DESADV zuruecksetzen | Dispo | Operation | Status-Reset |
| 23 | Fahranweisung | Dispo | Report | Mit Arbeitsablauf-Subreport |
| 24 | Fahrzeugliste | Dispo | Report | Multi-Sprache, Auto-Versand |
| 25 | IFCSUM Uebersicht | Dispo | - | IFCSUM-Daten |
| 26 | IFCSUM zuruecksetzen | Dispo | Operation | Status-Reset |
| 27 | Ladeschein | Dispo | Report | Ladestellendaten |
| 28 | Lagerliste | Dispo | Report | Zulauf/Anschluss/Direkt |
| 29 | Lieferantenavis | Dispo | Report | Auto-Versand an Lieferanten |
| 30 | Masterplan | Dispo | Report | Umschlagspunktgruppierung |
| 31 | NVDispoplan | Dispo | Report | Nachlauf-Dispoplan |
| 32 | PUS Daten zuruecksetzen | Dispo | Operation | PUS-Status-Reset |
| 33 | Tagesuebersicht TU | Dispo | Report | Auto-Versand Fax/EMail |
| 34 | Tagesuebersicht TU Voravis | Dispo | Report | Auto-Versand Fax/EMail |
| 35 | Verladeplan | Dispo | Report | Umschlagspunktgruppierung |
| 36 | Ueberhangliste | Dispo | Report | Nach Richtungsart |
| 37 | Anzahl Avise | DB-Anfrage | CSV | Zaehlung pro Zeitraum |
| 38 | DFUE Fehler | DB-Anfrage | CSV+Excel | 7 AbfrageTypen |
| 39 | DFUE Quote | DB-Anfrage | CSV | Pro Herkunft |
| 40 | Lieferantenexport | DB-Anfrage | CSV | Stammdaten-Export |
| 41 | Offene LG-Zustellavise | DB-Anfrage | CSV | Leergut-Uebersicht |
| 42 | Entladeliste | OhneUI | Report | Pro Sendungsliste |
| 43 | Quittungsanforderung | OhneUI | Report+Versand | Automatisch |
| 44 | Transportauftrag | OhneUI | Report+Versand | Mehrsprachig |
| 45 | TU-Abrechnung | OhneUI | CSV | Abrechnungsexport |
| 46 | BMW Versandavis | - | - | Leer/Platzhalter |
| 47 | Weibuda-Uebergabeprotokoll | Unbekannt | Report | Transfer-Protokoll |

## EDI/VDA-Schnittstellen

| Schnittstelle | Typ | Richtung | OEM |
|---|---|---|---|
| VDA 4913 VWT | Flatfile 128 Byte | Import | VW Transport |
| VDA 4913 BMW | Flatfile 128 Byte | Import | BMW |
| VDA 4913 Daimler | Flatfile 128 Byte | Import | Daimler |
| VDA 4913 MAN | Flatfile 128 Byte | Import | MAN |
| VDA 4913 Porsche | Flatfile 128 Byte | Import | Porsche |
| VDA 4913 Export (VA30MOD) | Flatfile .mod | Export | DB Schenker |
| VDA 4927 VWT | Flatfile | Import | VW Transport |
| DESADV VWT | EDIFACT | Import | VW Transport |
| IFCSUM | EDIFACT | Import | Brose |
| IFTSTA | EDIFACT | Import | (generisch) |

## Hintergrunddienste

| Dienst | Technologie | Beschreibung |
|---|---|---|
| AutoFaktura | Quartz + WCF | Automatische Faktura-Erstellung pro Abfahrt |
| Mailempfang | Quartz + POP3 | POP3-Mailempfang, Attachment-Speicherung |
| Mailversand | Quartz + SMTP | Automatischer Mailversand aus DB-Queue |
| Datenimport | Windows Forms | GUI-Tool fuer Stammdaten-Import |
| TourStatus | Quartz | Automatische Tourstatus-Aktualisierung (22 Tage) |
