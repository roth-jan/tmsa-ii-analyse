# TMSA-II Gebrochene Verkehre — Multi-Leg Transport System

## Überblick

"Gebrochene Verkehre" sind Transporte, die nicht direkt vom Lieferanten zum Empfänger gehen, sondern über einen oder mehrere **Umschlagspunkte** (USP) gebrochen werden. Dies ist ein Kern-Feature von TMSA-II und ein wesentliches Differenzierungsmerkmal.

### Beispiel

```
Lieferant A ──VL──→ Umschlagspunkt X ──HL──→ Umschlagspunkt Y ──NL──→ BMW Werk München
               ↑                          ↑                          ↑
          Vorlauf (VL)              Hauptlauf (HL)             Nachlauf (NL)
```

### Leg-Klassifikation

| Kürzel | Deutsch | Englisch | Beschreibung |
|--------|---------|----------|--------------|
| **VL** | Vorlauf | Pre-haul | Lieferant → Umschlagspunkt |
| **HL** | Hauptlauf | Main haul | Umschlagspunkt → Umschlagspunkt |
| **HLD** | Hauptlauf Direkt | Main haul direct | Direkttransport ohne Umschlag |
| **HLSG** | Hauptlauf Sammelgut | Main haul consolidated | Konsolidierter Hauptlauf |
| **NL** | Nachlauf | Post-haul | Umschlagspunkt → Empfänger (Werk) |
| **ZWV** | Zwischenverkehr | Intermediate | Sonderverkehr zwischen Standorten |

---

## 1. Datenmodell: Doubly-Linked List

Jeder Transport wird als **verkettete Liste von Streckenabschnitten** abgebildet. Zwei Tabellen:

- **StreckenabschnittUndisponiert** — Noch nicht einer Tour zugewiesene Abschnitte
- **StreckenabschnittDisponiert** — Bereits einer Tour zugewiesene Abschnitte

### Felder

```
StreckenabschnittID              GUID (PK)
StreckenabschnittArtikelzeileFK  GUID → Artikelzeile
StreckenabschnittNiederlassungFK GUID → Niederlassung
StreckenabschnittStartkontaktFK  GUID → Kontakt (Von)
StreckenabschnittZielkontaktFK   GUID → Kontakt (Nach)

── Verkettung ──
StreckenabschnittVorgaengerFK    GUID → vorheriger Abschnitt (NULL = erster)
StreckenabschnittNachfolgerFK    GUID → nächster Abschnitt (NULL = letzter)

── Klassifikation ──
VonNachBeschreibung              TINYINT (Bitmask, s.u.)
DispoEinschränkung               TINYINT (Restriktionslevel)
```

### Verkettungs-Prinzip

```
Abschnitt 1 (VL)           Abschnitt 2 (HL)           Abschnitt 3 (NL)
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│ VorgängerFK=NULL │   │ VorgängerFK=#1   │   │ VorgängerFK=#2   │
│ NachfolgerFK=#2  │──→│ NachfolgerFK=#3  │──→│ NachfolgerFK=NULL│
│ Start=Lieferant  │   │ Start=USP-X      │   │ Start=USP-Y      │
│ Ziel=USP-X       │   │ Ziel=USP-Y       │   │ Ziel=Werk        │
│ VonNach=33       │   │ VonNach=34       │   │ VonNach=18       │
└──────────────────┘   └──────────────────┘   └──────────────────┘
```

---

## 2. VonNachBeschreibung — Bitmask Enum

Klassifiziert jeden Streckenabschnitt nach Start- und Zieltyp.

### Definition

```vb
<Flags()>
Public Enum VonNachBeschreibung
  Nichts = 0
  StartIstStart = 1              ' Start ist Original-Versender
  StartIstUmschlagspunkt = 2     ' Start ist Umschlagspunkt
  ' Bit 2-3: reserviert
  ZielIstZiel = 16               ' Ziel ist End-Empfänger
  ZielIstUmschlagsPunkt = 32     ' Ziel ist Umschlagspunkt
  ' Bit 6-7: reserviert
End Enum
```

**Quellen:** `tmsa-ii-tourstatus/.../VonNachBeschreibung.vb`, `WcfLib_Disposition/.../partMengenplan.vb`

### Kombinationen

| Wert | Binär | Start | Ziel | Leg-Typ |
|------|-------|-------|------|---------|
| **17** | 00010001 | Lieferant (1) | Empfänger (16) | Direktverkehr (kein Umschlag) |
| **33** | 00100001 | Lieferant (1) | USP (32) | Vorlauf (VL) |
| **18** | 00010010 | USP (2) | Empfänger (16) | Nachlauf (NL) |
| **34** | 00100010 | USP (2) | USP (32) | Hauptlauf (HL) — USP zu USP |

### Bit-Operationen

```vb
' Bit setzen
Private Sub Bit_Setzen(ByRef Feld As Object, ByVal e As enuStreckenabschnittVonNachBeschreibung)
  Feld = Feld Or e
End Sub

' Bit entfernen
Private Sub Bit_Entfernen(ByRef Feld As Object, ByVal e As enuStreckenabschnittVonNachBeschreibung)
  Feld = Feld And (Not e)
End Sub

' Bit prüfen
Private Function Bit_Prüfen(ByVal Feld As Object, ByVal e As enuStreckenabschnittVonNachBeschreibung) As Boolean
  Return (Feld And e) > 0
End Function
```

---

## 3. DispoEinschränkung — Dispositionsbeschränkung

Steuert, ob/wie ein Streckenabschnitt disponiert werden darf.

| Wert | Bedeutung |
|------|-----------|
| **0** | Keine Einschränkung — frei disponierbar |
| **1** | Vorlauf eingeschränkt |
| **2** | Split-Tour gesperrt |

**Regel:** Beim Zusammenfassen wird die Einschränkung des Nachfolgers übernommen. Wird ein Abschnitt wieder zum Komplettverkehr (Start=Lieferant + Ziel=Empfänger), wird die Einschränkung auf 0 zurückgesetzt.

---

## 4. Übergangsarten (Disposition Modes)

Definiert, was am Ziel eines Streckenabschnitts passiert.

### Enum

```vb
Public Enum enuUebergangsart
  AnEmpfaenger = 0              ' Direkt an Endempfänger liefern
  UebergabeAnNiederlassung = 1  ' Übergabe an andere Niederlassung
  KeineWeitereStrecke = 2       ' Transport endet hier
  AnUmschlagspunkt = 3          ' Weiter über Umschlagspunkt
End Enum
```

### Dispositions-Flow

```
Artikelzeile anlegen
  ├─ Übergangsart = AnEmpfaenger (0)
  │   → Ein Streckenabschnitt: Lieferant → Empfänger (VonNach=17, Direkt)
  │
  ├─ Übergangsart = AnUmschlagspunkt (3)
  │   → Zwei+ Streckenabschnitte:
  │     1. Lieferant → USP (VonNach=33, VL)
  │     2. USP → Empfänger (VonNach=18, NL)
  │     oder: USP → USP → Empfänger (VonNach=34+18, HL+NL)
  │
  ├─ Übergangsart = UebergabeAnNiederlassung (1)
  │   → NiederlassungFK wechselt auf FolgeNiederlassungFK
  │   → Nächste NL übernimmt die weitere Disposition
  │
  └─ Übergangsart = KeineWeitereStrecke (2)
      → Transport endet, keine Folgelegs
```

---

## 5. LagerRouting — Automatische Routenbrechung

Definiert **welche Umschlagspunkte** für welche Lieferant-OEM-Kombination verwendet werden.

### Konfiguration

```
LagerRoutingID        GUID (PK)
NiederlassungFK       GUID → Niederlassung
LieferantenKontaktFK  GUID → Lieferant
OEMFK                 GUID → OEM
UmschlagPunktFK       GUID → Umschlagspunkt
```

**Quelle:** `WcfLib_Stammdaten/LagerRouting/dsLagerRouting.xsd`

### Logik

Wenn für eine Lieferant+OEM-Kombination ein LagerRouting-Eintrag existiert:
1. System erstellt automatisch **zwei Streckenabschnitte** statt einem
2. Erster: Lieferant → Umschlagspunkt (VL)
3. Zweiter: Umschlagspunkt → Empfänger (NL)
4. Verkettung via VorgaengerFK/NachfolgerFK

---

## 6. Zusammenfassen — Legs konsolidieren

Vereinigt mehrere Streckenabschnitte zu einem, wenn der Umschlag nicht mehr nötig ist.

### Algorithmus

```
1. Finde alle Streckenabschnitte einer Artikelzeile (via ArtikelzeileFK)
2. Traversiere die verkettete Liste vorwärts (NachfolgerFK)
3. Für jeden Abschnitt + seinen Nachfolger:
   a. Vorgänger.ZielKontaktFK = Nachfolger.ZielKontaktFK
   b. Vorgänger.DispoEinschränkung = Nachfolger.DispoEinschränkung
   c. Entferne Ziel-Bits vom Vorgänger (ZielIstUSP, ZielIstZiel)
   d. Übernimm Ziel-Bits vom Nachfolger
   e. Lösche den Nachfolger-Abschnitt
4. Wenn Ergebnis = StartIstStart + ZielIstZiel → DispoEinschränkung = 0
```

**Quelle:** `WcfLib_Disposition/.../partMengenplan.vb` (Zeilen 311-342)

### Einstiegspunkte

- `Mengenplan_Zusammenfassen_Alle()` — Alle Abschnitte konsolidieren
- `Mengenplan_Zusammenfassen_Auswahl()` — Nur ausgewählte
- `StarteZusammenfassen()` — WCF Service-Methode

---

## 7. Tour Splitten — Begegnungsverkehre

Splittet eine Tour an einem Umschlagspunkt in separate Teiltouren.

### Anwendungsfall

Eine Tour fährt: Lieferant A → USP → Lieferant B → USP → Werk.
Durch Splitten am USP entstehen:
- Tour 1 (SplitIndex=0): Lieferant A + B → USP (Vorlauf)
- Tour 2 (SplitIndex=1): USP → Werk (Nachlauf)

### Implementation

```vb
Public Sub Tour_Splitten(ByVal fzlRow As FahrzeuglisteEintrag,
                         ByVal Umschlagspunkt As DispoUmschlagspunktAuswahlListeEintrag)
  ' 1. Tour muss abgeschlossen sein (IstAbgeschlossen = True)
  ' 2. WCF Service: svc.TourSplitten(kob, Umschlagspunkt.ID)
  ' 3. Neuer Split-Eintrag mit SplitIndex > 0
  ' 4. SplitVorgängerID verweist auf Original-Tour
  ' 5. Neuberechnung der Kosten pro Teiltour
End Sub
```

**Quelle:** `tmsa.ui.mengenplan/daMengenplan_TourSplitten.vb`

### Reverse: Ent-Splitten

```vb
Private Sub Tour_EntSplitten(ByVal fzlRow As FahrzeuglisteEintrag)
  ' Voraussetzung: TourNummerIndex > 0 (ist ein Split)
  ' Löscht Split-Tour und zugehörige Strecken
  ' Stellt Original-Tour wieder her
End Sub
```

### Tour-Felder für Splits

| Feld | Typ | Beschreibung |
|------|-----|--------------|
| SplitIndex | Int | 0=Original, 1..N=Split-Teiltour |
| SplitVorgängerID | GUID | FK zur Original-Tour vor dem Split |
| Stopps | Int | Anzahl Haltepunkte in dieser Teiltour |
| IstTourletzterSplit | Bool | Letzter Split → Forecast-Erlös berechnen |

---

## 8. NL-Übergabe — Niederlassungswechsel

Wenn Übergangsart = `UebergabeAnNiederlassung`:

1. `StreckenabschnittNiederlassungFK` wechselt auf `FolgeNiederlassungFK`
2. Die Folge-Niederlassung sieht den Abschnitt in ihrem Mengenplan
3. Disposition erfolgt durch die übernehmende NL
4. Separate Tourzuweisung und Abrechnung

### Auswirkung

- **Dispo-Sicht:** Jede NL sieht nur ihre eigenen Streckenabschnitte
- **Abrechnung:** Getrennte TU-Abrechnung pro NL
- **Tracking:** Gesamtkette über VorgaengerFK/NachfolgerFK nachvollziehbar

---

## 9. Kostenberechnung pro Strecke

Jeder Split/Streckenabschnitt wird **separat bewertet**.

### Konditionsformel

```
Kosten = StoppFaktor × Stopps                    (wenn RechneMitStopps)
       + TourFaktor                               (wenn RechneMitTouren)
       + TagFaktor / AnzahlTouren                 (wenn RechneMitTagen)
       + LastKilometer × LastKmFaktor             (wenn RechneMitLastkilometer)
       + LeerKilometer × LeerKmFaktor             (wenn RechneMitLeerkilometer)
       + MautKilometer × MautKmFaktor             (wenn RechneMitMautkilometer)

Caps:
  Max pro Tour:  if Kosten > MaximalFrachtProTour → cap
  Max pro Tag:   if GesamtKosten > MaximalfrachtProTag → cap / AnzahlTouren
```

**Quelle:** `tmsa.ui.mengenplan/daMengenplan_Kondition.vb`

### Vorholkosten (NVV)

Separate Berechnung für Vorlauf:
```
NVV-Kosten = Fixum + SatzPro100kg × Gewicht / 100
```
Unterscheidung zwischen E+V (Empfang/Versand) und Standard-NVV.

---

## 10. UI: Streckenabschnitt-Editor

**Form:** `frmStreckenabschnitt` (in `tmsa.ui.nacharbeit`)

### Funktionen
- Datumsauswahl → alle Streckenabschnitte des Tages laden
- Infragistics UltraGrid mit Filter
- Spalten: Avisnummer, Start/Ziel (PLZ+Ort), LDM, Gewicht, Avisart
- Nur-Lesen (AllowAddNew=No, AllowDelete=False)
- Multi-Selektion möglich

### Zugriff
- Erreichbar aus dem Nacharbeit-Modul
- Zeigt sowohl disponierte als auch undisponierte Abschnitte

---

## 11. Multi-User Konfliktschutz

### Mechanismus

```vb
' Tabelle: StreckenabschnittUndisponiertMultiUser
' Service: IMengenplan.PrüfeAufÄnderung(...)
```

1. Client lädt Streckenabschnitte
2. Vor Änderung: `PrüfeAufÄnderung()` prüft ob ein anderer User geändert hat
3. Bei Konflikt: Änderung abgelehnt, Daten neu laden
4. Paging: Max 100 Abschnitte gleichzeitig, Rest im Zwischenspeicher

---

## 12. Datenfluss-Beispiel

**Szenario:** Lieferant A → Umschlag Gersthofen → BMW Werk München

```
1. AVIS ERFASSEN
   → Artikelzeile mit Lieferant A, Empfänger BMW München
   → LagerRouting findet: A + BMW → USP Gersthofen

2. STRECKENABSCHNITTE ERZEUGEN
   → STA #1: A → Gersthofen     (VonNach=33, VL)
   → STA #2: Gersthofen → BMW   (VonNach=18, NL)
   → #1.NachfolgerFK = #2, #2.VorgängerFK = #1

3. DISPOSITION (Mengenplan)
   → STA #1 erscheint bei NL Gersthofen
   → Disponent weist STA #1 → Tour 101 (VL-Tour)
   → STA #2 erscheint nach Umschlag
   → Disponent weist STA #2 → Tour 202 (NL-Tour)

4. ABFAHRT
   → Tour 101: TU Meyer, KFZ M-AB-123, fährt VL
   → Tour 202: TU Schmidt, KFZ A-CD-456, fährt NL

5. ABRECHNUNG
   → Tour 101: Kondition VL berechnen → Bewerten → Freigeben → Erzeugen
   → Tour 202: Kondition NL berechnen → Bewerten → Freigeben → Erzeugen

6. OPTIONAL: ZUSAMMENFASSEN
   → Wenn Umschlag nicht nötig: Zusammenfassen()
   → STA #1 + #2 → Ein STA: A → BMW (VonNach=17, Direkt)
   → Nur noch eine Tour nötig
```

---

## Zusammenfassung

Das Gebrochene-Verkehre-System ist das **komplexeste Subsystem** in TMSA-II und besteht aus:

| Komponente | Funktion |
|------------|----------|
| **Doubly-Linked List** | Flexible Verkettung beliebig vieler Legs |
| **VonNachBeschreibung Bitmask** | Kompakte Leg-Typ-Klassifikation |
| **DispoEinschränkung** | Geschäftsregel-gesteuerte Dispositionskontrolle |
| **4 Übergangsarten** | Steuern was am Ziel eines Legs passiert |
| **LagerRouting** | Automatische Routenbrechung pro Lieferant+OEM |
| **Zusammenfassen** | Konsolidierung nicht benötigter Umschläge |
| **Tour Splitten** | Aufteilen für Begegnungsverkehre |
| **NL-Übergabe** | Niederlassungswechsel für übergreifende Disposition |
| **Per-Leg Costing** | Separate Kostenberechnung pro Abschnitt |
| **Multi-User Lock** | Konflikterkennung bei gleichzeitiger Disposition |

Für den Web-App-Rewrite muss dieses System **vollständig** abgebildet werden — es ist der Kern des Wettbewerbsvorteils von TMSA-II gegenüber einfacheren TMS-Systemen.
