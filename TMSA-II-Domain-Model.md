# TMSA-II Domain Model -- Comprehensive Analysis

## 1. System Overview

**TMSA-II** (Transport Management System Application, Version II) is a **VB.NET WinForms + WCF** enterprise application for **automotive logistics and freight management**. It manages the entire lifecycle of goods transport between OEM (Original Equipment Manufacturer) plants, suppliers, and transshipment points in the German automotive supply chain.

The system is built as a rich client (WinForms with Infragistics controls) communicating via WCF (Windows Communication Foundation) services over NetTCP to a SQL Server backend.

---

## 2. Architecture

### 2.1 Client-Server with WCF Services

```
[WinForms Client (frmMain)]
    |
    |-- ServiceFactory (Singleton, manages WCF proxies)
    |       |
    |       |-- StammdatenService      (Master Data)
    |       |-- DispositionService      (Dispatch/Planning)
    |       |-- AbfertigungService      (Processing/Clearance)
    |       |-- BerichtswesenService    (Reporting)
    |       |-- DokumenteService        (Document Dispatch)
    |       |-- DispositionStreaming     (Streamed Disposition Data)
    |
    v
[WCF Service Host (NetTCP)]
    |
    v
[SQL Server Database]
```

### 2.2 Module System (Plugin Architecture)

The main form (`frmMain`) uses a **reflection-based plugin system**:
- Menu items carry assembly/class references as `Tag` strings (e.g., `"tmsa.ui.sendungsbildung.dll|Formularname|default"`)
- Clicking a menu item dynamically loads the assembly via `Assembly.LoadFrom()`
- Creates an instance of `UISteuerung` (which extends `UIModul`)
- Opens the form as an MDI child window

### 2.3 Authentication

- Login form (`frmLogin`) with username, password, and branch/office selection
- Password hashed via MD5 before sending to server
- User credentials passed to all WCF service proxies
- Branch context (Niederlassung) determines visible data
- Rights-based menu visibility (`hatRecht()`)

---

## 3. Core Domain Entities

### 3.1 Niederlassung (Branch/Office)

The organizational unit representing a logistics branch.

| Property | Type | Description |
|---|---|---|
| NiederlassungID | Guid | Primary key |
| Matchcode | String | Short lookup name |
| Niederlassungskuerzel | String | Branch abbreviation |
| Niederlassungsnummer | String | Branch number |
| GewichtLdmBerechnung | Double | Weight per loading meter calculation factor |
| GewichtLdmBerechnung2 | Double | Secondary LDM weight factor |
| Gesperrt | Boolean | Blocked/locked flag |
| GutschriftsKopf / GutschriftsFuss | String | Credit note header/footer |
| UmsatzsteuerIdent | String | VAT identification number |
| UmsatzsteuerNummer | String | VAT number |
| Disporelevant | Boolean | Whether this branch is disposition-relevant |
| ReklaNrFuehrendeNull | Boolean | Leading zeros in complaint numbers |
| ReklaNrStart / ReklaNrEnde | Integer | Complaint number range |
| ReklaDokumentPfad | String | Path for complaint documents |
| LeergutZuordnung_LieferantFK | Guid | FK: Supplier for empty packaging assignment |
| BorderierendeNiederlassungFK | Guid | FK: Branch handling bordero creation |
| DispositionNotfallKontaktFK | Guid | FK: Emergency contact for disposition |
| AbrechnungNotfallKontaktFK | Guid | FK: Emergency contact for billing |
| DrittgeschaeftOEMFK | Guid | FK: OEM for third-party business |
| SofaOEMFK | Guid | FK: OEM for SoFa (Sonderfahrt) |
| Waehrungssymbol | String | Currency symbol |
| TASprache | String | Transport order language |
| TagesEnde | Date | End of business day |
| TourDokumentPfad | String | Path for tour documents |
| Innenhoehetempe / InnenhoeheMegatemp | Decimal | Vehicle inner height settings |
| VorstandText | String | Board of directors text |

**Relationships**: NiederlassungOptionen (key-value options), UebergabeNiederlassung (transfer branches), Waehrung (currency)

### 3.2 Lieferant (Supplier)

Suppliers providing goods to the OEM plants.

| Property | Type | Description |
|---|---|---|
| LieferantenID | Guid | Primary key |
| Matchcode | String | Short lookup name |
| InterneNummer | Integer | Internal supplier number |
| ExterneNummer | String | External supplier number |
| Gesperrt | Boolean | Blocked flag |
| KommunikationsParameter | Integer | Communication settings (EDI etc.) |
| HauptkontaktFK | Guid | FK: Main contact |
| KommunikationsKontaktFK | Guid | FK: Communication contact |

**Sub-entities**:
- **LieferantenLadestellen** (Loading/Unloading Points): LadestellenID, Matchcode, Nummer, Lagernummer, IstAbladestelle, IstLadestelle
- **LieferantenOEM**: Mapping supplier to OEM with DUNS numbers (DUNS1, DUNS2), ccStatus, ccID
- **LieferantenZeitfenster** (Time Windows): Delivery time slots per supplier
- **LieferantenWerkSperrtage** (Supplier Plant Closed Days): Days when supplier plants are closed

### 3.3 OEM (Original Equipment Manufacturer)

Automotive manufacturers (e.g., BMW, Daimler, Porsche, VW, MAN).

| Property | Type | Description |
|---|---|---|
| OEMID | Guid | Primary key |
| Name1 | String | Name |
| Strasse | String | Street |
| Land | String | Country |
| PLZ | String | Postal code |
| Ort | String | City |
| Ansprechpartner | String | Contact person |
| MistralNummer | String | Mistral system number |
| AnzeigeText | String | Display text |

**Related**: NiederlassungOEM (branches assigned to OEMs), FrankaturOEM (freight cost responsibilities per OEM)

### 3.4 Werk (Plant/Factory)

OEM manufacturing plants where goods are delivered.

Related to Werkschliesstage (plant closing days) and WerkSperrzeiten (plant blocked periods).

### 3.5 KFZ (Vehicle)

| Relevant Properties | Description |
|---|---|
| KFZID | Primary key |
| Kennzeichen | License plate |
| TUFK | FK: Transport company |
| Various technical specs | Vehicle specifications |

Linked to: KFZNiederlassung (vehicle-branch assignments), KFZNiederlassungEinsatzart (vehicle usage types per branch), KFZNiederlassungKondition (vehicle pricing conditions per branch)

### 3.6 TU (Transportunternehmen / Transport Company)

The carrier/trucking company that performs the actual transport.

Managed via `ITU` interface with operations: `TU_Aktualisieren`, `TU_Holen`.
Checks for duplicate Mistral numbers (`DoppelteMistralnummerFaultException`).
Linked to TUNiederlassung (transport company branch assignments).

### 3.7 Tour (Tour / Trip)

**The central operational entity** -- a single trip by a vehicle on a specific day.

| Property | Type | Description |
|---|---|---|
| TourID | Guid | Primary key |
| NiederlassungFK | Guid | FK: Branch |
| ReiseFK | Guid | FK: Journey (parent grouping) |
| Nummer | Integer | Tour number (TRNR) |
| NummerDatum | Date | Tour number date |
| Datum | Date | Tour date |
| KonditionFK | Guid | FK: Pricing condition |
| KostenKondition | Decimal | Calculated condition costs |
| KostenManuell | Decimal | Manual costs |
| KostenManuellErfasst | Boolean | Whether manual costs were entered |
| KostenForecast | Decimal | Forecast costs |
| TransportbehaelterFK | Guid | FK: Transport container type |
| KFZFK | Guid | FK: Vehicle |
| EinsatzArtFK | Guid | FK: Usage type |
| Marker | Byte | Color/status marker |
| ZugnummerFK | Guid | FK: Train number (for rail transport) |
| ZugnummerWert | Integer | Train number value |
| IstAbgeschlossen | Boolean | Tour completed/closed |
| AbrechnungsStatus | Byte | Billing status |
| Abrechnungsstopp | Boolean | Billing stop flag |
| BemerkungIntern | String | Internal remarks |
| BemerkungExtern | String | External remarks |
| BemerkungAbrechnung | String | Billing remarks |
| BemerkungAbfertigung | String | Processing remarks |
| BemerkungArbeitsablauf | String | Workflow remarks |
| BemerkungDisposition | String | Disposition remarks |
| LeerKilometer | Integer | Empty kilometers |
| LastKilometer | Integer | Loaded kilometers |
| MautKilometer | Integer | Toll kilometers |
| Quittung | Boolean | Receipt/POD received |
| Quittungdatum | Date | Receipt date |
| QuittungStatus | Integer | Receipt status (0=none, >0=received) |
| SplitIndex | Integer | Split tour index |

**Tour Lifecycle**:
1. Created from Mengenplan (quantity plan) disposition
2. Articles/shipments assigned to route sections
3. Tour can be split (`TourSplitten`) at transshipment points
4. Tour completed (`TourAbschliessen`)
5. Departure created from tour (`erzeugeAbfahrtAusTour`)
6. TU billing (`TUAbrechnung`)

### 3.8 Reise (Journey)

Groups multiple tours into a single journey. A Reise has a number and date, and contains multiple Tour records.

| Property | Type | Description |
|---|---|---|
| ReiseID | Guid | Primary key |
| Nummer | Integer | Journey number |
| NummerDatum | Date | Journey number date |
| NiederlassungFK | Guid | FK: Branch |

### 3.9 Strecke (Route Section) and Streckenabschnitt (Route Segment)

Routes are divided into sections. Each section has disposed (assigned) and undisposed route segments.

**StreckenabschnittDisponiert** (Disposed Route Segment) links articles to tours:
- StreckenabschnittID
- ArtikelzeileFK (FK: Article line item)
- TourFK (FK: Tour)
- StreckenabschnittNachfolgerFK (FK: Next segment -- forms a linked list)
- Sortierung (Sort order)

### 3.10 Sendung (Shipment)

**The core processing entity** -- represents a shipment of goods.

| Property | Type | Description |
|---|---|---|
| SendungID | Guid | Primary key |
| NiederlassungFK | Guid | FK: Branch |
| Status | Byte | Shipment status (see enum below) |
| RichtungsartFK | Integer | Direction type (see enum below) |
| Datum | Date | Shipment date |
| Nummer | Integer | Shipment number |
| OemFK | Guid | FK: OEM |
| OemAnzeigetext | String | OEM display text |
| FahrtypFK | Guid | FK: Trip type |
| FahrtypAnzeigetext | String | Trip type display text |
| EntladedatumVon / EntladedatumBis | Date | Unloading date range |
| LadedatumVon / LadedatumBis | Date | Loading date range |
| AbladestelleFK | Guid | FK: Unloading point |
| AbladestelleNummer | String | Unloading point number |
| EntladezoneFK | Guid | FK: Unloading zone |
| BemerkungExtern | String | External remarks |
| SendungAdresseVersenderFK | Guid | FK: Sender address |
| SendungAdresseEmpfaengerFK | Guid | FK: Recipient address |
| FrankaturFK | Guid | FK: Franking/freight responsibility |
| FrankaturBezeichnung | String | Franking description |
| FrankaturIndex | Integer | Franking sort index |
| Herkunft | String | Origin system |
| HerkunftSchluessel | String | Origin key |
| BorderoFK | Guid | FK: Bordero (bill of lading) |
| Borderierungsdatum | Date | Date of bordero assignment |
| Systemnummernkreis | Boolean | System-generated number range |
| IstSonderFahrt | Boolean | Special trip flag |
| PODStatus | Integer | Proof of delivery status |
| AvisSteuerung | Byte | Avis control flags (bitfield) |

**Sub-entities**:
- **SendungLieferschein** (Delivery Notes): Linked delivery notes
- **SendungADR** (Dangerous Goods): ADR hazmat classifications
- **SendungPackmittel** (Packaging): Packaging used
- **SendungArtikel** (Articles): Article items in shipment
- **SendungAdresse** (Addresses): Sender/receiver addresses
- **SendungParameterWert** (Parameters): Key-value parameters
- **SendungInterneBemerkung** (Internal Notes)
- **SendungLabel** (Labels): Barcode/tracking labels

### 3.11 SendungListe (Shipment List)

Groups shipments for operational handling.

| Type | Code | Description |
|---|---|---|
| Entladeliste | 0 | Unloading list |
| Rollkarte | 1 | Roll card (loading list for a tour) |
| Hallenliste | 2 | Hall list (warehouse allocation) |

Operations: Create, assign shipments, free list, copy list, delete shipments, create tours from lists.

### 3.12 Bordero (Bill of Lading / Manifest)

Groups shipments for a departure. Contains shipment assignments with bordero number, date, relation, and status.

Filter criteria: Date range, relation, bordero number, departure number, OEM, plant, TU, CBill status, bordero status.

### 3.13 Abfahrt (Departure)

A physical departure of a vehicle. Created from tours.

Operations: Search by number/date/ID, save, assign bordero numbers, VDA4921 export (for MAN, Daimler/DC, VWT), manage bordero relations (RELA).

### 3.14 Avis (Notification/Advice)

Advance shipping notifications between parties.

| Key Concepts | Description |
|---|---|
| AvisArt | Type of notification |
| LadeDatum | Loading date |
| Lieferdatum | Delivery date |
| AvisBlöckchen | Avis blocks (groupings) |
| Konfliktprüfung | Conflict check for overlapping avis |
| SoFa-Avis | Special trip (Sonderfahrt) notification |
| LeergutZustellAvis | Empty packaging delivery notification |
| DESADV | Dispatch advice (EDI standard) |

### 3.15 Mengenplan (Quantity Plan)

The core disposition planning view showing all articles to be transported.

**FilterKriterien_Mengenplan** provides extensive filtering:
- By date range (loading/delivery dates)
- By OEM, branch, supplier
- By route, vehicle
- By goods type (Vollgut/Leergut)
- By disposition status

The Mengenplan supports:
- Paging (returns data in batches of 100 rows)
- Zusammenfassen (consolidation of route segments)
- ArtikelKomplettEntladen (complete unloading of articles)
- Multi-user conflict detection

### 3.16 Arbeitsablauf (Workflow / Work Schedule)

Defines the sequence of stops and route sections for a tour.

| Key Properties | Description |
|---|---|
| ArbeitsablaufID | Primary key |
| Streckenabschnitt | Route sections (linked list) |
| Start/End points | Origin and destination |

### 3.17 Kondition (Pricing Condition)

Defines the pricing model for transport. Linked to vehicles via KFZNiederlassungKondition.

### 3.18 TransportAuftrag (Transport Order)

A formal transport order document generated from tour data.
- TransportAuftrag_Holen (load by tour ID)
- TransportAuftrag_Nummer_Holen (get next number per branch)

### 3.19 Reklamation (Complaint)

Complaint management for transport issues.
- Search by shipment ID, date range, status
- ReklamationGrund (complaint reasons) as lookup table
- Document path for complaint files

### 3.20 Kontakte (Contacts)

Contact management attached to various entities (suppliers, branches, etc.).
- Referenced by PkName (primary key name of parent) and TableName
- Used for Hauptkontakt (main contact), KommunikationsKontakt, NotfallKontakt

---

## 4. Key Domain Enumerations

### 4.1 SendungStatus (Shipment Status)
```
offen          = 0   -- Open, not yet processed
borderiert     = 1   -- Assigned to a bordero
storniert      = 2   -- Cancelled
Vorlage        = 3   -- Template
zurPrüfung     = 4   -- Under review
unbestimmt     = 255 -- Undetermined
```

### 4.2 RichtungsArt (Direction Type)
```
WerksEingang   = 0   -- Inbound to plant (supplier -> plant)
WerksAusgang   = 1   -- Outbound from plant (plant -> supplier, empty packaging return)
Drittgeschäft  = 2   -- Third-party business (between external parties)
unbestimmt     = 255 -- Undetermined
```

### 4.3 Besitzer (Owner)
```
Empfänger      = 0   -- Receiver owns the goods
Versender      = 1   -- Sender owns the goods
unbestimmt     = 255
```

### 4.4 SendungListeTyp (Shipment List Type)
```
Entladeliste   = 0   -- Unloading list
Rollkarte      = 1   -- Roll card (loading manifest)
Hallenliste    = 2   -- Hall/warehouse list
unbestimmt     = 255
```

### 4.5 Gutart (Goods Type)
```
Vollgut        = 1   -- Full goods (loaded containers/parts)
Leergut        = 2   -- Empty goods (empty packaging returning)
```

### 4.6 Zeitraumbezug (Time Reference)
```
Abholung       = 1   -- Pickup
Zustellung     = 2   -- Delivery
```

### 4.7 AvisSteuerung (Avis Control -- Bitfield)
```
leer               = 0  -- No special handling
Sofa               = 1  -- SoFa (Sonderfahrt) avis
LeergutZustellung  = 2  -- Empty packaging delivery avis
AvisVerbinden      = 3  -- Connect/merge avis
```

### 4.8 Verbindungsweg (Connection Mode)
```
SIGN = 0  -- Internal/direct
WWW  = 1  -- Web-based
```

---

## 5. Business Logic and Calculations

### 5.1 Rechenwerk (Calculation Engine)

The calculation engine provides distance and cost computations:

**Distance Calculations (EWS - Entfernungswerk/Strecke)**:
- `getKilometer(orte())`: Calculates total kilometers between a series of postal code/country pairs
- `getMautKilometer(orte())`: Calculates toll-liable kilometers
- `existiertOrt()` / `existiertPLZ()`: Validates places and postal codes
- `EWSValidierung()`: Full location validation
- `EWSSuche()`: Location search

**Cost Calculations**:

#### Konditionskosten (Condition-based Transport Costs)
The `KonditionsBerechnung` class implements a flexible cost model:

```
Total = StoppFaktor * Stopps           (if RechneMitStopps)
      + TourFaktor                     (if RechneMitTouren)
      + TagFaktor / AnzahlTouren       (if RechneMitTagen)
      + LastKilometer * LastKmFaktor   (if RechneMitLastkilometer)
      + LeerKilometer * LeerKmFaktor  (if RechneMitLeerkilometer)
      + MautKilometer * MautKmFaktor  (if RechneMitMautkilometer)

If Total > MaximalFrachtProTour -> cap at MaximalFrachtProTour
If GesamtKosten + Total > MaximalfrachtProTag -> cap at MaximalfrachtProTag / AnzahlTouren
```

#### Forecast (Revenue Forecast)
```
Forecast = RoutenPreis + RelationsPreis + 3GAvisUmsatz
```
Only calculated for the last split of a tour (`IstTourletzterSplit`).

#### Routenpreis (Route Price)
Calculated per route with different rates:
- RichtungsTyp=2 (Drittgeschäft/third-party): RL rate
- RichtungsTyp=1 (WerksAusgang/outbound): WA rate
- RichtungsTyp=0 (WerksEingang/inbound): WE rate

#### Relationspreis (Relation Price)
Weight-based pricing per relation:
```
Price = Fixum + SatzPro100kg * Weight / 100
```
Capped at MaxFrachtLager (from warehouse) or MaxFrachtDirekt (direct delivery).

#### Vorholkosten (Pre-haulage Costs)
NVV (Nahverkehr/local transport) costs based on weight and region:
```
NVV Cost = Fixum + SatzPro100kg * Weight / 100
```
Calculated separately for E+V (pickup & delivery) and standard.

### 5.2 TU-Abrechnung (Transport Company Billing)

Complete billing workflow for transport companies:

1. **TUAbrechnung_Laden**: Load billing data for a branch
2. **TourenSpeichernUnddsTUAbrechnungErstellen**: Save tours and create billing documents
   - Checks for existing documents (`BelegeExistierenBereitsFault`)
   - SQL-level validation (`TuAbrechnungSqlFault`)
3. **WEIBUDA_Ubergabe**: Transfer to WEIBUDA accounting system
4. **TUAbrechnung_Storno**: Cancellation processing
5. **TUAbrechnung_Laden_WHDruck**: Reprint billing documents (by year, TU, number range, date range, transfer number)

### 5.3 Sendungsbildung (Shipment Creation)

Core shipment processing workflow:

1. **Sendung_Holen**: Load existing shipments with filters
2. **Sendung_Aktualisieren**: Save shipments with:
   - SoFa-Avis creation (if bit flag set)
   - LeergutZustellAvis creation/deletion
   - Label generation
   - IFTSTA (transport status) message creation for new Vollgut (full goods) shipments
   - Stanzkreisprüfung (number range validation)
3. **Sendung_Suchen**: Search shipments
4. **Sendung_WertBereichsPruefung**: Value range validation (delivery note numbers, SLB numbers)

### 5.4 Halbautomatische Abfahrtserstellung (Semi-automatic Departure Creation)

Automated creation of departures from tours with grouping logic.

### 5.5 VDA Exports

Industry-standard data exchange formats:
- **VDA 4913**: Delivery notification (Lieferavis)
- **VDA 4921**: Transport order confirmation (for MAN, Daimler/DC, VWT variants)
- **VDA 4927**: Freight invoice data
- **IFTSTA**: Transport status message (EDIFACT)
- **IFCSUM**: Consolidated cargo message
- **DESADV**: Dispatch advice (EDI)
- **PUS**: Pickup sheet

---

## 6. Lookup Tables (Nachschlagetabellen)

The system caches extensive lookup data loaded at startup and refreshed periodically:

| Table | German | English |
|---|---|---|
| Niederlassung | Niederlassung | Branch/Office |
| GrenzzollAmt | Grenzzollamt | Border customs office |
| OEM | OEM | Car manufacturer |
| Laenderkennzeichen | Länderkennzeichen | Country codes |
| Werk | Werk | Factory/Plant |
| Niederlassungslieferanten | Niederlassungslieferanten | Branch suppliers |
| Route | Route | Transport route |
| RouteSuffix | Routensuffix | Route suffix |
| Lieferanten | Lieferanten | Suppliers |
| Abladestelle | Abladestelle | Unloading point |
| NiederlassungOptionen | Niederlassungsoptionen | Branch options |
| AvisArt | Avisart | Notification type |
| PackMittel | Packmittel | Packaging materials |
| LieferantenOEM | Lieferanten-OEM | Supplier-OEM mapping |
| NiederlassungOEM | Niederlassung-OEM | Branch-OEM mapping |
| WerkSperrzeiten | Werksperrzeiten | Plant blocked periods |
| TU | TU | Transport company |
| TUNiederlassung | TU-Niederlassung | TU-branch mapping |
| KFZ | KFZ | Vehicles |
| KFZNiederlassung | KFZ-Niederlassung | Vehicle-branch mapping |
| KFZNiederlassungEinsatzart | KFZ-NL-Einsatzart | Vehicle usage types per branch |
| EinsatzArt | Einsatzart | Usage/deployment type |
| Transportbehaelter | Transportbehälter | Transport container |
| UmschlagPunkt | Umschlagpunkt | Transshipment point |
| LagerRouting | Lagerrouting | Warehouse routing |
| SchadstoffKlasse | Schadstoffklasse | Emission class |
| LKWTyp | LKW-Typ | Truck type |
| Fabrikat | Fabrikat | Vehicle make |
| NiederlassungRegion | Niederlassungsregion | Branch regions |
| Kondition | Kondition | Pricing conditions |
| EntladeZone | Entladezone | Unloading zone |
| Forecast | Forecast | Revenue forecast |
| ForecastDetail | Forecast-Detail | Forecast details |
| LieferantenZeitfenster | Lieferantenzeitfenster | Supplier time windows |
| LieferantenWerkSperrtage | Lieferanten-Werksperrtage | Supplier plant closed days |
| Lizenz | Lizenz | License |
| Sprache | Sprache | Language |
| Zugnummer | Zugnummer | Train number |
| Werkschliesstage | Werkschließtage | Plant closing days |
| KFZNiederlassungKondition | KFZ-NL-Kondition | Vehicle pricing per branch |
| ADR | ADR | Dangerous goods (ADR) |
| ADRPackmittel | ADR-Packmittel | ADR packaging |
| UebergabeNiederlassung | Übergabeniederlassung | Transfer branches |
| OEM_Alle | Alle OEM | All OEMs (cross-branch) |
| Masseinheit | Maßeinheit | Unit of measure |
| MistralSondernummer | Mistral-Sondernummer | Mistral special numbers |
| SLBNummerLaenge | SLB-Nummernlänge | SLB number length |
| SpeditionsGebiet | Speditionsgebiet | Forwarding area |
| LieferscheinNummernLaenge | Lieferschein-Nummernlänge | Delivery note number length |
| SendungParameter | Sendungsparameter | Shipment parameters |
| Prozess | Prozess | Process |
| SoFaVerursacher | SoFa-Verursacher | Special trip cause |
| Carrier | Carrier | Carrier |
| UebertragungsNorm | Übertragungsnorm | Transmission standard |
| NummernkreisTyp | Nummernkreistyp | Number range type |
| OFA | OFA | Open freight account |
| Frankatur | Frankatur | Franking/freight terms |
| FrankaturOEM | Frankatur-OEM | Franking per OEM |
| ReklamationGrund | Reklamationsgrund | Complaint reason |
| NOFA | NOFA | Number range management |

---

## 7. Key Workflows

### 7.1 Disposition Workflow (Mengenplan -> Tour -> Abfahrt)

```
1. Avis eingang (Notification received)
   |
2. Mengenplan laden (Load quantity plan)
   |-- Filter by date, OEM, branch, supplier, etc.
   |-- Shows all articles to be transported
   |
3. Artikel disponieren (Dispose articles)
   |-- Assign route segments to tours
   |-- Zusammenfassen (consolidate blocks)
   |-- ArtikelStatusAktualisieren (update article status)
   |
4. Tour erstellen/bearbeiten (Create/edit tour)
   |-- Assign KFZ (vehicle)
   |-- Set Einsatzart (usage type)
   |-- Set Kondition (pricing)
   |-- Calculate kilometers (EWS)
   |-- Optionally split tour at UmschlagPunkt
   |
5. Arbeitsablauf festlegen (Define workflow/route)
   |-- Sequence of stops
   |-- Route sections
   |
6. Tour abschließen (Complete tour)
   |-- Sets IstAbgeschlossen = True
   |
7. Abfahrt erstellen (Create departure)
   |-- erzeugeAbfahrtAusTour or halbautomatische Abfahrtserstellung
   |-- VDA4921 export
```

### 7.2 Sendungsbildung (Shipment Creation Workflow)

```
1. Sendung erstellen (Create shipment)
   |-- From Avis data or manual entry
   |-- Set OEM, direction, dates, addresses
   |-- Assign packaging (Packmittel)
   |-- Add delivery notes (Lieferscheine)
   |-- Add ADR dangerous goods info
   |
2. Sendung zu SendungListe zuweisen (Assign to list)
   |-- Rollkarte (loading card for tour)
   |-- Entladeliste (unloading list)
   |-- Hallenliste (hall allocation)
   |
3. Borderierung (Bordero assignment)
   |-- Sendung gets Status = borderiert
   |-- Assign Bordero number and date
   |
4. Abfertigung (Clearance/dispatch)
   |-- Create labels
   |-- Generate IFTSTA messages
   |-- SoFa/LeergutZustell avis handling
```

### 7.3 Billing Workflow (TU-Abrechnung)

```
1. TUAbrechnung_Laden (Load billing data)
   |-- By branch, with lookups for TU, vehicles
   |
2. Tour-Freigabe (Tour release for billing)
   |-- TuAbrechung_Freigabe with status lists
   |-- Status 0 = open, Status 2 = released, Stopp = blocked
   |
3. Belege erstellen (Create billing documents)
   |-- TourenSpeichernUnddsTUAbrechnungErstellen
   |-- Duplication check (TUAbrechnungDopplungsPruefer)
   |-- Buchungsjahr, Belegnummer generation
   |
4. WEIBUDA-Übergabe (Transfer to accounting)
   |-- Export to WEIBUDA system
   |-- Protocol file management
   |
5. Optional: Storno (Cancellation)
   |-- TUAbrechnung_Storno_Laden/Speichern
```

### 7.4 Reklamation (Complaint Workflow)

```
1. ReklamationSendungSuche (Search shipment for complaint)
2. ReklamationLaden (Load existing complaints)
3. ReklamationSpeichern (Save complaint)
   |-- ReklamationGrund (complaint reason)
   |-- ReklaNr (complaint number from NL range)
   |-- Document attachment path
4. ReklamationUebersicht (Overview/dashboard)
```

---

## 8. Integration Points

### 8.1 EDI / Data Exchange Formats

| Format | Standard | Description |
|---|---|---|
| VDA 4913 | VDA | Delivery notification (Lieferabruf) |
| VDA 4921 | VDA | Transport data (variants: MAN, DC, VWT) |
| VDA 4927 | VDA | Freight/invoice data |
| IFTSTA | EDIFACT | International transport status |
| IFCSUM | EDIFACT | Consolidated cargo summary |
| DESADV | EDIFACT | Dispatch advice |
| PUS | Custom | Pickup sheet |
| 3G-Avis | Custom | Third-party business notifications |

### 8.2 External Systems

- **WEIBUDA**: Accounting/billing system (Weiß-Buch-Datenaustausch)
- **WeiKunDa**: Customer data exchange system
- **Mistral**: Logistics reference number system
- **EWS**: Distance calculation service (Entfernungswerk Straße)
- **Speditionsbuch**: Forwarding book export
- **TRP Export**: Transport export format
- **CTTS Status**: Container tracking status

### 8.3 DFUe (Datenfernuebertragung / Remote Data Transfer)

The system manages EDI data exchange:
- **DFÜAutorouting**: Automatic routing of EDI messages
- **DFÜFreischaltung**: Activation/release of EDI channels
- **DFÜÜbersetzung**: Translation of EDI codes/formats
- **DFÜQuote**: EDI transmission quota tracking
- **DFÜFehler**: EDI error reporting

---

## 9. Reporting (Berichtswesen)

The system generates extensive reports via the `BerichtswesenService`:

| Report | German | English |
|---|---|---|
| Beladeliste | Beladeliste | Loading list |
| Entladeliste | Entladeliste | Unloading list |
| Rollkarte | Rollkarte | Roll card |
| Fahranweisung | Fahranweisung | Driving instructions |
| Frachtbrief | Frachtbrief | Waybill/CMR |
| Ladeschein | Ladeschein | Bill of lading |
| Masterplan | Masterplan | Master plan |
| Verladeplan | Verladeplan | Loading plan |
| Lagerliste | Lagerliste | Warehouse list |
| Hallenliste | Hallenliste | Hall allocation list |
| Ausgangsliste | Ausgangsliste | Outbound list |
| TransportAuftrag | Transportauftrag | Transport order |
| TUAbrechnung | TU-Abrechnung | Transport company billing |
| Borderodruck | Borderodruck | Bordero print |
| Fahrzeugliste | Fahrzeugliste | Vehicle list |
| SendungLabel | Sendungslabel | Shipment label (barcode) |
| Sonderfahrt | Sonderfahrt | Special trip |
| NVDispoplan | NV-Dispoplan | Local transport disposition plan |
| Verbringungsnachweis | Verbringungsnachweis | Transfer certificate |
| Gefahrgut Statistik | Gefahrgut-Statistik | Dangerous goods statistics |
| Packmittelbegleitschein | Packmittelbegleitschein | Packaging companion document |
| Quittungsanforderung | Quittungsanforderung | Receipt request |
| Überhangliste | Überhangliste | Overhang/surplus list |
| Ausfallfrachten | Ausfallfrachten | Lost freight/penalties |
| EVKosten | EV-Kosten | Pick-up and delivery costs |
| MistralVorschlag | Mistralvorschlag | Mistral proposal |
| Tagesübersicht TU | Tagesübersicht TU | Daily TU overview |
| InterneBemerkung | Interne Bemerkung | Internal notes report |
| DESADVAvise | DESADV-Avise | DESADV notifications |
| LieferantenAvis | Lieferantenavis | Supplier notifications |
| BMWVersandAvis | BMW-Versandavis | BMW dispatch advice |
| Reklamation | Reklamation | Complaints |
| OffeneDFÜ | Offene DFÜ | Open EDI transmissions |
| PUS Dateien/Nachdruck | PUS-Dateien | PUS file management |
| DFÜQuote | DFÜ-Quote | EDI quota |
| DFÜFehler | DFÜ-Fehler | EDI errors |
| Statistik | Statistik | Statistics |
| TU Export | TU-Export | TU data export |
| Übergabedatum | Übergabedatum | Transfer date |
| Verlademeldung | Verlademeldung | Loading notification |
| SendungenMitTauschmittel | Sendungen mit Tauschmittel | Shipments with exchange materials |
| AnzahlAvise | Anzahl Avise | Number of notifications |
| OffeneLGZustellavise | Offene LG-Zustellavise | Open empty packaging delivery notifications |
| WeibudaÜbergabeProtokoll | Weibuda-Übergabeprotokoll | WEIBUDA transfer protocol |
| WeikundaProtokollEntladelisten | Weikunda-Protokoll Entladelisten | WeiKunDa protocol unloading lists |
| AbfertigungErlösübersicht | Abfertigung Erlösübersicht | Processing revenue overview |
| IFCSUMUebersicht / IFCSUMReset | IFCSUM-Übersicht | IFCSUM overview/reset |

---

## 10. German-English Domain Glossary

| German Term | English | Context |
|---|---|---|
| Abfahrt | Departure | Physical vehicle departure |
| Abfertigung | Clearance/Processing | Shipment processing and dispatch |
| Abladestelle | Unloading point | Physical location for unloading |
| Abrechnung | Billing/Settlement | Financial settlement |
| Abrechnungsstopp | Billing stop | Block billing |
| ADR | ADR (dangerous goods) | European dangerous goods regulations |
| Arbeitsablauf | Workflow | Sequence of operations/stops |
| Artikel / Artikelzeile | Article / Article line | Line item of goods |
| Avis | Notification/Advice | Advance shipping notification |
| Benutzer | User | System user |
| Bericht | Report | Business report |
| Bordero | Bordero/Manifest | Shipping manifest grouping shipments |
| Borderierung | Bordero assignment | Assigning shipments to a bordero |
| DUNS | DUNS number | Supplier identification (Dun & Bradstreet) |
| Einsatzart | Usage type | Vehicle deployment type |
| Entladeliste | Unloading list | List of items to unload |
| Entladezone | Unloading zone | Zone within a plant for unloading |
| Faktura | Invoice | Invoice/billing interface |
| Fahrtyp | Trip type | Type of journey |
| Forecast | Forecast | Revenue/cost forecast |
| Frankatur | Franking | Freight cost responsibility |
| Grenzzollamt | Border customs office | For international shipments |
| Gutschrift | Credit note | Financial credit |
| Hallenliste | Hall list | Warehouse hall allocation |
| KFZ | Vehicle (Kraftfahrzeug) | Truck/vehicle |
| Kondition | Condition/Pricing | Transport pricing model |
| Ladeplan | Loading plan | Plan for loading vehicles |
| Ladestelle | Loading point | Physical loading location |
| Leergut | Empty goods | Empty containers/packaging returning |
| Lieferant | Supplier | Parts supplier |
| Lieferschein | Delivery note | Delivery document |
| Mengenplan | Quantity plan | Disposition planning view |
| Nacharbeit | Rework/Post-processing | After-the-fact corrections |
| Nachschlagetabellen | Lookup tables | Reference/master data tables |
| Niederlassung | Branch/Office | Organizational unit |
| NOFA | Number range | Number sequence management |
| NVV Kosten | Local transport costs | Nahverkehrsverbund costs |
| Packmittel | Packaging | Packaging materials |
| Quittung | Receipt/POD | Proof of delivery |
| Recherche | Research/Search | Data lookup/search |
| Rechenwerk | Calculation engine | Distance/cost calculations |
| Reise | Journey | Groups multiple tours |
| Reklamation | Complaint | Quality/delivery complaint |
| Rollkarte | Roll card | Loading manifest for a tour |
| Route | Route | Transport route definition |
| Sendung | Shipment | Individual shipment |
| Sendungsbildung | Shipment creation | Process of creating shipments |
| SoFa | Special trip (Sonderfahrt) | Non-standard transport |
| Speditionsbuch | Forwarding book | Forwarding company ledger |
| Strecke / Streckenabschnitt | Route / Route segment | Parts of a transport route |
| Tauschmittel | Exchange materials | Returnable packaging |
| Tour | Tour/Trip | Single vehicle trip on a day |
| Transportauftrag | Transport order | Formal transport instruction |
| Transportbehälter | Transport container | Container/trailer type |
| TU | Transport company | Carrier/trucking company |
| Überhangliste | Surplus list | Goods not yet dispatched |
| Umschlagpunkt | Transshipment point | Transfer location between routes |
| Verbringungsnachweis | Transfer certificate | Proof of cross-border transfer |
| Vollgut | Full goods | Loaded containers/parts |
| Vorhol | Pre-haulage | Pick-up transport to terminal |
| WEIBUDA | Accounting system | Weiß-Buch-Datenaustausch |
| WeiKunDa | Customer data system | Weiße Kundendaten |
| Werkschließtage | Plant closing days | Days the plant is closed |
| Zugnummer | Train number | For intermodal rail transport |

---

## 11. Data Access Patterns

The codebase consistently uses:

1. **Typed DataSets** (`dsXxx`) for all data transfer -- no ORM, direct ADO.NET
2. **TableAdapters** generated from SQL queries/stored procedures
3. **Kommunikationsobjekte (kob_Xxx)** as WCF data contracts wrapping datasets
4. **PartService pattern**: WCF service class is split via `Partial Class` across multiple files, each implementing one interface
5. **Singleton ServiceFactory** with retry logic (3 attempts) and connection state management
6. **NachschlageTabellen caching**: Client-side cache with timestamp-based refresh
7. **Multi-tenant**: Data filtered by NiederlassungFK (branch foreign key)
8. **Optimistic concurrency**: `TStamp` (timestamp) columns and `LesbarerTimestamp` for conflict detection
9. **Audit trail**: All entities carry `BenutzerAenderung`, `DatumAenderung`, `BenutzerErstellung`, `DatumErstellung`
