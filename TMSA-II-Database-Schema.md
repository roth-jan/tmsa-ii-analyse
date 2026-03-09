# TMSA-II Complete Database Schema Analysis

**Database:** Microsoft SQL Server
**Database Name:** TMSA2DEV / TMSA2TEST (dev/test environments)
**Connection:** `Data Source=NTCLGSDEV001; Initial Catalog=TMSA2TEST; User ID=sa; Password=ntcadmin`
**Local Cache:** SQL Server Compact Edition 3.5 (client-side caching via `cache.sdf`)

## Architecture Overview

TMSA-II uses a **three-tier architecture**:
1. **SQL Server** - Main database with stored procedures for all CRUD operations
2. **WCF Services** (KommunikationsDienste) - Data access layer via typed DataSets
3. **WinForms Client** - Rich client with local SQL CE cache for master data

**Key Patterns:**
- All primary keys are `uniqueidentifier` (GUIDs)
- Every table has audit columns: `BenutzerAenderung`, `DatumAenderung`, `BenutzerErstellung`, `DatumErstellung`
- Optimistic concurrency via `TStamp` (timestamp/rowversion) column
- Client-side caching uses `LesbarerTimestamp` (datetime) for incremental sync
- Deleted records tracked via `view_DEL_Cache_*` views (soft delete pattern)
- Stored procedure naming convention: `TableName_SEL`, `TableName_INS`, `TableName_UPD`, `TableName_DEL`
- Foreign keys suffixed with `FK` (e.g., `OEMFK`, `NiederlassungFK`)
- CargoConnect (CC) integration via `ccID` and `ccStatus` columns

---

## DOMAIN 1: ORGANISATIONSSTRUKTUR (Organizational Structure)

### 1.1 Niederlassung (Branch Office)
**Business Entity:** A company branch/location that manages logistics operations
**Table:** `Niederlassung`

| Column | Type | Description |
|--------|------|-------------|
| NiederlassungID | uniqueidentifier | PK |
| Matchcode | nvarchar(50) | Short name/code |
| Niederlassungskuerzel | nvarchar | Branch abbreviation |
| Niederlassungsnummer | int | Branch number |
| GewichtLdmBerechnung | int | Weight/LDM calculation method |
| GewichtLdmBerechnung2 | int | Secondary calculation method |
| Gesperrt | bit | Locked flag |
| GutschriftsKopf | nvarchar | Credit note header text |
| GutschriftsFuss | nvarchar | Credit note footer text |
| UmsatzsteuerIdent | nvarchar | VAT ID |
| UmsatzsteuerNummer | nvarchar | Tax number |
| Disporelevant | bit | Relevant for dispatch |
| ReklaNrFuehrendeNull | int | Leading zeros in complaint numbers |
| ReklaNrStart | int | Complaint number range start |
| ReklaNrEnde | int | Complaint number range end |
| ReklaDokumentPfad | nvarchar | Complaint document path |
| TourDokumentPfad | nvarchar | Tour document path |
| LeergutZuordnung_LieferantFK | uniqueidentifier | FK -> Lieferanten (empty goods supplier) |
| BorderierendeNiederlassungFK | uniqueidentifier | FK -> Niederlassung (bordering branch) |
| DispositionNotfallKontaktFK | uniqueidentifier | FK -> Kontakte (dispatch emergency contact) |
| AbrechnungNotfallKontaktFK | uniqueidentifier | FK -> Kontakte (billing emergency contact) |
| HauptKontaktFK | uniqueidentifier | FK -> Kontakte (main contact/address) |
| DrittgeschaeftOEMFK | uniqueidentifier | FK -> OEM (third-party business OEM) |
| SofaOEMFK | uniqueidentifier | FK -> OEM (special trip OEM) |
| TA_SpracheFK | uniqueidentifier | FK -> Sprache (transport order language) |
| Waehrungssymbol | nvarchar | Currency symbol |
| VorstandText | nvarchar | Board/management text |
| LesbarerTimestamp | datetime | Sync timestamp |
| TStamp | timestamp | Concurrency token |
| + Audit columns | | |

**Stored Procedures:** `Niederlassung_SEL`, implied CRUD
**Views:** `view_DEL_Cache_Niederlassung` (deleted records tracking)

### 1.2 NiederlassungOptionen (Branch Options)
**Business Entity:** Configuration settings per branch
**Table:** `NiederlassungOptionen`

| Column | Type | Description |
|--------|------|-------------|
| NiederlassungOptionenID | uniqueidentifier | PK |
| NiederlassungFK | uniqueidentifier | FK -> Niederlassung |
| OptionenSchluessel | nvarchar | Option key |
| Wert | nvarchar | Option value |
| + Standard columns | | |

### 1.3 NiederlassungOEM (Branch-OEM Assignment)
**Business Entity:** Which OEMs are handled by which branch
**Table:** `NiederlassungOEM`

| Column | Type | Description |
|--------|------|-------------|
| NiederlassungOEMID | uniqueidentifier | PK |
| NiederlassungFK | uniqueidentifier | FK -> Niederlassung |
| OEMFK | uniqueidentifier | FK -> OEM |
| + Standard columns | | |

**Views:** `view_DEL_Cache_NiederlassungOEM`

### 1.4 NiederlassungRegion (Branch Region)
**Business Entity:** Regional assignment of branches
**Table:** `NiederlassungRegion`

| Column | Type | Description |
|--------|------|-------------|
| NiederlassungRegionID | uniqueidentifier | PK |
| NiederlassungFK | uniqueidentifier | FK -> Niederlassung |
| + Standard columns | | |

### 1.5 NiederlassungUebergabeNiederlassung (Branch Handover)
**Business Entity:** Transfer relationships between branches
**Table:** Implied by stored procedures

| Column | Type | Description |
|--------|------|-------------|
| NiederlassungUebergabeNiederlassungID | uniqueidentifier | PK |
| NiederlassungFK | uniqueidentifier | FK -> Niederlassung (source) |
| Uebergabe_NiederlassungFK | uniqueidentifier | FK -> Niederlassung (target) |

### 1.6 Geschaeftsstelle (Business Office) [TeKoS]
**Business Entity:** Office/location for the TeKoS module
**Table:** `Geschaeftsstelle`

| Column | Type | Description |
|--------|------|-------------|
| GeschaeftsstelleGUID | uniqueidentifier | PK |
| Bezeichnung | nvarchar | Name |
| + Standard columns | | |

**Related Tables:**
- `GeschaeftsstellenOptionen` - Options per office
- `GeschaeftsstellenLager` - Warehouses per office
- `GeschaeftsstellenNummernKreise` - Number ranges per office
- `Geschaeftsjahre` - Fiscal years
- `GSTRDC` - Office RDC mapping

---

## DOMAIN 2: STAMMDATEN - KUNDEN & PARTNER (Master Data - Customers & Partners)

### 2.1 OEM (Original Equipment Manufacturer)
**Business Entity:** Automotive manufacturer (BMW, Daimler, VW, Porsche, MAN, etc.)
**Table:** `OEM`

| Column | Type | Description |
|--------|------|-------------|
| OEMID | uniqueidentifier | PK |
| Matchcode | nvarchar(50) | Short name (e.g., "BMW", "DC") |
| HauptkontaktFK | uniqueidentifier | FK -> Kontakte (main contact/address) |
| TStamp | timestamp | Concurrency |
| + Audit columns | | |

**Stored Procedures:** `OEM_SEL`, `OEM_INS`, `OEM_UPD`, `OEM_DEL`, `OEM_NachNiederlassungFK_SEL`

### 2.2 Werk (Factory/Plant)
**Business Entity:** OEM factory/plant receiving goods
**Table:** `Werk`

| Column | Type | Description |
|--------|------|-------------|
| WerkID | uniqueidentifier | PK |
| OEMFK | uniqueidentifier | FK -> OEM |
| Matchcode | nvarchar(50) | Short name |
| Kundennummer | int | Customer number |
| ExterneNummer | nvarchar(10) | External number |
| VollgutRelation | int | Full goods relation |
| LeergutRelation | int | Empty goods relation |
| IstLeergutWerk | bit | Is empty goods plant |
| IstVollgutWerk | bit | Is full goods plant |
| Gesperrt | bit | Locked |
| KommunikationsParameter | int | Communication parameters |
| HauptkontaktFK | uniqueidentifier | FK -> Kontakte |
| DokumentenKontaktFK | uniqueidentifier | FK -> Kontakte (document address) |
| KCCWerkFK | uniqueidentifier | FK -> Werk (KCC reference plant) |
| Werkskreis | nvarchar(10) | Plant circle/region |
| IstKCC | bit | Is KCC plant |
| ccID | nvarchar(400) | CargoConnect ID |
| ccStatus | int | CargoConnect status |
| LesbarerTimestamp | datetime | |
| TStamp | timestamp | |
| + Audit columns | | |

**Stored Procedures:** `Werk_SEL`, `Werk_INS`, `Werk_UPD`, `Werk_DEL`, `Werk_HauptkontaktFK_SEL_ByWerkID`
**Views:** `view_DEL_Cache_Werk`

### 2.3 WerkNiederlassungDispoOrt (Plant-Branch-DispoLocation)
**Business Entity:** Links plants to branches with dispatch locations
**Table:** `WerkNiederlassungDispoOrt`

| Column | Type | Description |
|--------|------|-------------|
| WerkNiederlassungDispoOrtID | uniqueidentifier | PK |
| NiederlassungFK | uniqueidentifier | FK -> Niederlassung |
| WerkFK | uniqueidentifier | FK -> Werk |
| DispoOrt | nvarchar(50) | Dispatch location |
| Sortierung | int | Sort order |
| GrenzzollAmtFK | uniqueidentifier | FK -> GrenzzollAmt |
| LesbarerTimestamp | datetime | |
| TStamp | timestamp | |
| + Audit columns | | |

**Views:** `view_DEL_Cache_WerkNiederlassungDispoOrt`

### 2.4 WerkSperrzeiten (Plant Closure Periods)
**Table:** `WerkSperrzeiten`

**Views:** `view_DEL_Cache_WerkSperrzeiten`

### 2.5 Werkschliesstage (Plant Closing Days)
**Table:** `Werkschliesstage` - Per-plant holiday/closing calendar

### 2.6 Lieferanten (Suppliers)
**Business Entity:** Parts suppliers delivering to OEM plants
**Table:** `Lieferanten`

| Column | Type | Description |
|--------|------|-------------|
| LieferantenID | uniqueidentifier | PK |
| Matchcode | nvarchar(50) | Short name |
| InterneNummer | nvarchar | Internal number |
| ExterneNummer | nvarchar | External number |
| Gesperrt | bit | Locked |
| KommunikationsParameter | int | Communication parameters |
| HauptkontaktFK | uniqueidentifier | FK -> Kontakte |
| KommunikationsKontaktFK | uniqueidentifier | FK -> Kontakte |
| LesbarerTimestamp | datetime | |
| TStamp | timestamp | |
| + Audit columns | | |

**Stored Procedures:** `Lieferanten_SEL`, `Lieferanten_NachNiederlassungFK_SEL`

### 2.7 LieferantenOEM (Supplier-OEM Assignment)
**Business Entity:** Links suppliers to OEMs with DUNS numbers
**Table:** `LieferantenOEM`

| Column | Type | Description |
|--------|------|-------------|
| LieferantenOEMID | uniqueidentifier | PK |
| LieferantFK | uniqueidentifier | FK -> Lieferanten |
| OEMFK | uniqueidentifier | FK -> OEM |
| DUNS1 | nvarchar | DUNS number 1 |
| DUNS2 | nvarchar | DUNS number 2 |
| Bemerkung | nvarchar | Remark |
| ccStatus | int | CargoConnect status |
| ccID | nvarchar | CargoConnect ID |
| LesbarerTimestamp | datetime | |
| TStamp | timestamp | |
| + Audit columns | | |

**Views:** `view_DEL_Cache_LieferantenOEM`

### 2.8 LieferantenNiederlassung (Supplier-Branch)
**Table:** `LieferantenNiederlassung` - Assigns suppliers to branches

**Views:** `view_DEL_Cache_LieferantenNiederlassung`

### 2.9 LieferantenWerkSperrtage (Supplier-Plant Block Days)
**Table:** `LieferantenWerkSperrtage`

**Views:** `view_DEL_Cache_LieferantenWerkSperrtage`

### 2.10 LieferantenZeitfenster (Supplier Time Windows)
**Table:** `LieferantenZeitfenster` - Delivery time windows per supplier

**Views:** `view_DEL_Cache_LieferantenZeitfenster`

### 2.11 Kontakte (Contacts/Addresses)
**Business Entity:** Central address/contact table used by all entities
**Table:** `Kontakte`

| Column | Type | Description |
|--------|------|-------------|
| KontaktID | uniqueidentifier | PK |
| KontaktReferenzFK | uniqueidentifier | FK -> parent entity |
| Beschreibung | nvarchar | Description/type |
| Name1 | nvarchar | Name line 1 |
| Name2 | nvarchar | Name line 2 |
| Strasse | nvarchar | Street |
| Land | nvarchar | Country |
| PLZ | nvarchar | Postal code |
| Ort | nvarchar | City |
| OrtZusatz | nvarchar | City supplement |
| Telefon | nvarchar | Phone |
| Fax | nvarchar | Fax |
| Mail | nvarchar | Email |
| Ansprechpartner | nvarchar | Contact person |
| MistralNummer | nvarchar | Mistral number |
| Bemerkung | nvarchar | Remark |
| AnzeigeText | nvarchar | Display text |
| LesbarerTimestamp | datetime | |
| TStamp | timestamp | |
| + Audit columns | | |

**Views:** `view_DEL_Cache_Kontakte`
**Note:** This is a polymorphic contact table. `KontaktReferenzFK` points to different parent entities (Werk, Lieferant, OEM, Niederlassung, etc.)

### 2.12 Abladestelle (Unloading Point)
**Business Entity:** Specific unloading location at a plant
**Table:** `Abladestelle`

| Column | Type | Description |
|--------|------|-------------|
| AbladestelleID | uniqueidentifier | PK |
| Matchcode | nvarchar(50) | Short name |
| Nummer | nvarchar(10) | Number |
| Lagernummer | nvarchar(10) | Warehouse number |
| ExterneNummer | nvarchar(10) | External number |
| ExterneWerksNummer | nvarchar(10) | External plant number |
| Gesperrt | bit | Locked |
| IstAbladestelle | bit | Is unloading point |
| IstLadestelle | bit | Is loading point |
| HauptKontaktFK | uniqueidentifier | FK -> Kontakte |
| KontaktFK | uniqueidentifier | FK -> Kontakte (secondary) |
| EntladeZoneFK | uniqueidentifier | FK -> EntladeZone |
| OEMFK | uniqueidentifier | FK -> OEM |
| ccID | nvarchar(400) | CargoConnect ID |
| ccStatus | int | CargoConnect status |
| LesbarerTimestamp | datetime | |
| TStamp | timestamp | |
| + Audit columns | | |

**Views:** `view_DEL_Cache_Abladestelle`

### 2.13 EntladeZone (Unloading Zone)
**Table:** `EntladeZone`

| Column | Type | Description |
|--------|------|-------------|
| EntladeZoneID | uniqueidentifier | PK |
| Bezeichnung | nvarchar | Name |
| WerkFK | uniqueidentifier | FK -> Werk |
| + Standard columns | | |

**Views:** `view_DEL_Cache_EntladeZone`

---

## DOMAIN 3: TRANSPORTMITTEL (Transport Resources)

### 3.1 TU (Transportunternehmen / Carrier)
**Business Entity:** Transport company / carrier
**Table:** `TU`

| Column | Type | Description |
|--------|------|-------------|
| TUID | uniqueidentifier | PK |
| ExterneNummer | nvarchar(50) | External carrier number |
| Steuernummer | nvarchar(50) | Tax number |
| UmsatzsteuerID | nvarchar(50) | VAT ID |
| TASprache | nvarchar(3) | Transport order language |
| HauptKontaktFK | uniqueidentifier | FK -> Kontakte |
| TOSMailKontakt | nvarchar(250) | TOS mail contact |
| gesperrt | bit | Locked |
| TOSTeilnehmer | bit | TOS participant |
| LesbarerTimestamp | datetime | |
| TStamp | timestamp | |
| + Audit columns | | |

**Stored Procedures:** `TU_SEL`, `TU_INS`, `TU_UPD`, `TU_DEL`
**Views:** `view_DEL_Cache_TU`

### 3.2 TUNiederlassung (Carrier-Branch)
**Table:** `TUNiederlassung` - Assigns carriers to branches
**Views:** `view_DEL_Cache_TUNiederlassung`

### 3.3 TULizenz (Carrier License)
**Table:** `TULizenz`

| Column | Type | Description |
|--------|------|-------------|
| TULizenzID | uniqueidentifier | PK |
| TUFK | uniqueidentifier | FK -> TU |
| Lizenzdatum | datetime | License date |
| LizenzNummer | nvarchar(50) | License number |
| LizenzBemerkung | nvarchar(50) | License remark |
| LizenzFK | uniqueidentifier | FK -> Lizenz (license type) |
| + Standard columns | | |

**Views:** `view_DEL_Cache_TULizenz`

### 3.4 TUSAMEinsatzart (Carrier SAM Usage Type)
**Table:** `TUSAMEinsatzart`

### 3.5 TUSAMFahrzeugart (Carrier SAM Vehicle Type)
**Table:** `TUSAMFahrzeugart`

### 3.6 KFZ (Vehicle)
**Business Entity:** Individual truck/vehicle
**Table:** `KFZ`

| Column | Type | Description |
|--------|------|-------------|
| KFZID | uniqueidentifier | PK |
| TUFK | uniqueidentifier | FK -> TU (carrier) |
| Kennzeichen | nvarchar(20) | License plate |
| Telefon | nvarchar(50) | Phone |
| Telematik | nvarchar(50) | Telematics ID |
| LKWTypFK | uniqueidentifier | FK -> LKWTyp |
| SchadstoffKlasseFK | uniqueidentifier | FK -> SchadstoffKlasse |
| FabrikatFK | uniqueidentifier | FK -> Fabrikat |
| gesperrt | bit | Locked |
| LesbarerTimestamp | datetime | |
| TStamp | timestamp | |
| + Audit columns | | |

**Stored Procedures:** `KFZ_SEL`, `KFZ_INS`, `KFZ_UPD`, `KFZ_DEL`
**Views:** `view_DEL_Cache_KFZ`

### 3.7 KFZNiederlassung (Vehicle-Branch Assignment)
**Table:** `KFZNiederlassung`

| Column | Type | Description |
|--------|------|-------------|
| KFZNiederlassungID | uniqueidentifier | PK |
| KFZFK | uniqueidentifier | FK -> KFZ |
| NiederlassungFK | uniqueidentifier | FK -> Niederlassung |
| Bemerkung | nvarchar(500) | Remark |
| NLKennung | nvarchar(10) | Branch identifier |
| NVNr | nvarchar(20) | Short-distance number |
| Cadis | bit | CADIS system flag |
| CadisNR | nvarchar(50) | CADIS number |
| NVFahrzeug | bit | Short-distance vehicle |
| Festfahrend | bit | Fixed-route vehicle |
| + Standard columns | | |

**Views:** `view_DEL_Cache_KFZNiederlassung`

### 3.8 KFZNiederlassungEinsatzart (Vehicle Usage Types)
**Table:** `KFZNiederlassungEinsatzart`
**Views:** `view_DEL_Cache_KFZNiederlassungEinsatzart`

### 3.9 KFZNiederlassungKondition (Vehicle Conditions/Pricing)
**Table:** `KFZNiederlassungKondition`

---

## DOMAIN 4: LOGISTIK-KONFIGURATION (Logistics Configuration)

### 4.1 PackMittel (Packaging Material)
**Business Entity:** Types of containers, pallets, packaging
**Table:** `PackMittel`

| Column | Type | Description |
|--------|------|-------------|
| PackMittelID | uniqueidentifier | PK |
| Nummer | nvarchar | Number/code |
| Bezeichnung | nvarchar | Description |
| TaraGewicht | decimal | Tare weight |
| OEMFK | uniqueidentifier | FK -> OEM |
| LaengeLeer | decimal | Length empty |
| LaengeVoll | decimal | Length full |
| BreiteLeer | decimal | Width empty |
| BreiteVoll | decimal | Width full |
| HoeheLeer | decimal | Height empty |
| HoeheVoll | decimal | Height full |
| HoeheLeer_CC | decimal | Height empty (CC) |
| HoeheVoll_CC | decimal | Height full (CC) |
| LMLeer | decimal | Loading meters empty |
| LMVoll | decimal | Loading meters full |
| VolumenLeer | decimal | Volume empty |
| VolumenVoll | decimal | Volume full |
| StapelFaktorLeer | int | Stacking factor empty |
| StapelFaktorVoll | int | Stacking factor full |
| DispoLaengeLeer/Voll | decimal | Dispatch dimensions |
| DispoBreiteLeer/Voll | decimal | |
| DispoHoeheLeer/Voll | decimal | |
| MaterialTyp | int | Material type |
| KontoTyp | int | Account type |
| Tauschmittel | bit | Is exchange material |
| istDummy | bit | Is dummy |
| IstTemporaer | bit | Is temporary |
| gesperrt | bit | Locked |
| ccID | nvarchar | CargoConnect ID |
| ccStatus | int | CargoConnect status |
| + Standard columns | | |

**Views:** `view_DEL_Cache_PackMittel`

### 4.2 Route (Route)
**Table:** `Route`

| Column | Type | Description |
|--------|------|-------------|
| RouteID | uniqueidentifier | PK |
| Bezeichnung | nvarchar | Description |
| NiederlassungFK | uniqueidentifier | FK -> Niederlassung |
| OEMFK | uniqueidentifier | FK -> OEM |
| WerkFK | uniqueidentifier | FK -> Werk |
| LieferantOEMFK | uniqueidentifier | FK -> LieferantenOEM |
| RoutenPreis | decimal | Route price |
| RoutenPreisRL | decimal | Return load price |
| RoutenPreisWA | decimal | Alternative price |
| Schnittgewicht | decimal | Cut weight |
| Sonderkennzeichen | nvarchar | Special identifier |
| FakturaFlag | int | Invoice flag |
| FuehrendePUSZeichen | nvarchar | Leading PUS characters |
| Memo | nvarchar | Notes |
| deaktiviert | bit | Deactivated |
| + Standard columns | | |

**Views:** `view_DEL_Cache_Route`

### 4.3 RouteSuffix (Route Suffix)
**Table:** `RouteSuffix`

| Column | Type | Description |
|--------|------|-------------|
| RouteSuffixID | uniqueidentifier | PK |
| Suffix1 | nvarchar | Suffix value |
| RouteSuffixArt | int | Suffix type |
| + Standard columns | | |

### 4.4 Kondition (Rate/Pricing Condition)
**Table:** `Kondition`

### 4.5 LagerRouting (Warehouse Routing)
**Table:** `LagerRouting`
**Views:** `view_DEL_Cache_LagerRouting`

### 4.6 Transportbehaelter (Transport Container)
**Table:** `Transportbehaelter`
**Views:** `view_DEL_Cache_Transportbehaelter`

### 4.7 UmschlagPunkt (Transshipment Point)
**Table:** `UmschlagPunkt`
**Views:** `view_DEL_Cache_UmschlagPunkt`

### 4.8 Zugnummer (Train Number)
**Table:** `Zugnummer` - Train identification for rail transport

### 4.9 GrenzzollAmt (Border Customs Office)
**Table:** `GrenzzollAmt`

| Column | Type | Description |
|--------|------|-------------|
| GrenzzollAmtID | uniqueidentifier | PK |
| AbfahrtsLandIsoCode | nvarchar | Departure country ISO |
| EingangsLandIsoCode | nvarchar | Entry country ISO |
| LandIsoCode | nvarchar | Country ISO |
| Postleitzahl | nvarchar | Postal code |
| Ort | nvarchar | City |
| Matchcode | nvarchar | Short name |
| gesperrt | bit | Locked |
| ccStatus | int | |
| ccID | nvarchar | |
| + Standard columns | | |

**Views:** `view_DEL_Cache_GrenzzollAmt`

---

## DOMAIN 5: NACHSCHLAGETABELLEN (Lookup Tables)

### 5.1 EinsatzArt (Usage Type)
**Table:** `EinsatzArt`
**Views:** `view_DEL_Cache_EinsatzArt`

### 5.2 SAMEinsatzart / SAMFahrzeugart
**Tables:** `SAMEinsatzart`, `SAMFahrzeugart` - SAM system usage/vehicle types

### 5.3 LKWTyp (Truck Type)
**Table:** `LKWTyp`

### 5.4 SchadstoffKlasse (Emission Class)
**Table:** `SchadstoffKlasse`

### 5.5 Fabrikat (Manufacturer/Brand)
**Table:** `Fabrikat`

### 5.6 Lizenz (License Type)
**Table:** `Lizenz`

### 5.7 Laenderkennzeichen (Country Codes)
**Table:** `Laenderkennzeichen`
**Views:** `view_DEL_Cache_Laenderkennzeichen`

### 5.8 Sprache (Language)
**Table:** `Sprache`

### 5.9 Waehrung (Currency)
**Table:** `Waehrung`

### 5.10 Masseinheit (Unit of Measure)
**Table:** Implied by `Masseinheit_SEL`

### 5.11 ADR (Dangerous Goods)
**Table:** `ADR` - Hazardous materials classification

### 5.12 ADRPackmittel (ADR Packaging)
**Table:** `ADRPackmittel` - ADR-specific packaging materials

### 5.13 Frankatur (Freight Terms)
**Table:** `Frankatur`

### 5.14 FrankaturOEM (Freight Terms per OEM)
**Table:** Referenced via `FrankaturOEM_SEL`

### 5.15 TransportVorschrift (Transport Regulation)
**Table:** `TransportVorschrift`

### 5.16 UebertragungsNorm (Transmission Standard)
**Table:** `UebertragungsNorm` - EDI transmission standards

### 5.17 Carrier
**Table:** `Carrier` - External carrier definitions

### 5.18 Nummernkreis (Number Range)
**Table:** `Nummernkreis`

### 5.19 AnwendungsOptionen (Application Options)
**Table:** `AnwendungsOptionen` - Global application settings

### 5.20 Optionen (Options)
**Table:** `Optionen`

---

## DOMAIN 6: BENUTZERVERWALTUNG (User Management)

### 6.1 Benutzer (User)
**Table:** `Benutzer`

| Column | Type | Description |
|--------|------|-------------|
| BenutzerGUID | uniqueidentifier | PK |
| Password | nvarchar | Hashed password |
| Login | nvarchar | Login name |
| + Standard columns | | |

**Stored Procedures:** `Benutzer_SEL`, `Benutzer_INS`, `Benutzer_UPD`, `Benutzer_DEL`, `Benutzer_Anmelden_SEL`, `Benutzer_Password_Aendern_UPD`, `Benutzer_LFN_SEL`

### 6.2 BenutzerGruppe (User Group)
**Table:** `BenutzerGruppe`

| Column | Type | Description |
|--------|------|-------------|
| BenutzerGruppeGUID | uniqueidentifier | PK |
| Bezeichnung | nvarchar | Group name |

### 6.3 Berechtigung (Permission)
**Table:** `Berechtigung`

| Column | Type | Description |
|--------|------|-------------|
| BerechtigungGUID | uniqueidentifier | PK |
| Bezeichnung | nvarchar | Permission name |

### 6.4 BenutzerGruppeBerechtigung (Group-Permission)
**Table:** `BenutzerGruppeBerechtigung` - M:N link

### 6.5 BenutzerNiederlassung (User-Branch) [TMSA-II]
**Table:** `BenutzerNiederlassung`

### 6.6 BenutzerNiederlassungBenutzerGruppe (User-Branch-Group)
**Table:** `BenutzerNiederlassungBenutzerGruppe` - Assigns user to groups per branch

### 6.7 BenutzerGeschaeftsstelle (User-Office) [TeKoS]
**Table:** `BenutzerGeschaeftsstelle`

### 6.8 BenutzerGeschaeftsstelleBenutzerGruppe [TeKoS]
**Table:** `BenutzerGeschaeftsstelleBenutzerGruppe`

---

## DOMAIN 7: BEWEGUNGSDATEN - AVIS (Advice/Notification)

### 7.1 Avis (Transport Advice/Notification)
**Business Entity:** Advance shipping notice from supplier to OEM plant
**Table:** `Avis` (in context of Disposition)

Key columns extracted from stored procedures and DataSets:
- AvisGUID / AvisID
- AvisNummer
- AuftragsDatum, Ladedatum
- LieferantFK, WerkFK, AbladestelleFK
- NiederlassungFK
- RouteFK
- Status fields
- + Standard columns

**Stored Procedures:** `Avis_SEL`, `Avis_INS`, `Avis_UPD`, `Avis_DEL`, `Avis_KonfliktPruefung`, `Avis_AvisSuche_SEL`, `Avis_Disposition_SEL_sp`

### 7.2 Artikelzeile (Article Line)
**Business Entity:** Individual line items within an Avis
**Table:** Part of Avis DataSet

**Stored Procedures:** `Avis_Artikelzeile_SEL`, `Avis_Artikelzeile_INS`, `Avis_Artikelzeile_UPD`, `Avis_Artikelzeile_DEL`

### 7.3 ArtikelzeileADR (ADR Data per Article)
**Table:** `ArtikelzeileADR` - Hazardous goods info per article line

### 7.4 ArtikelZeileTransportVorschrift
**Table:** `ArtikelZeileTransportVorschrift` - Transport regulations per article

### 7.5 AvisTransportVorschrift
**Table:** `AvisTransportVorschrift` - Transport regulations per Avis

### 7.6 AvisAbsenderTransportVorschrift
**Table:** `AvisAbsenderTransportVorschrift` - Sender transport instructions

### 7.7 AvisEmpfaengerTransportKette
**Table:** `AvisEmpfaengerTransportKette` - Receiver transport chain

### 7.8 AvisArt (Avis Type)
**Table:** `AvisArt` - Lookup for types of advices

### 7.9 AvisVorlage (Avis Template) [TeKoS]
**Table:** `AvisVorlage` - Templates for recurring advices
**Related:** `AvisPositionVorlage`, `AvisPositionADRVorlage`, `AvisAbsenderTransportVorschriftVorlage`, `AvisEmpfaengerTransportKetteVorlage`

### 7.10 AvisPosition [TeKoS]
**Table:** `AvisPosition`

---

## DOMAIN 8: BEWEGUNGSDATEN - SENDUNG (Shipment)

### 8.1 Sendung (Shipment)
**Business Entity:** A consolidated shipment created from Avis data
**Table:** Referenced via `Sendung_SEL` and related stored procedures

**Sub-tables (all part of dsSendung DataSet):**

| Table | Description |
|-------|-------------|
| Sendung | Main shipment header |
| SendungLieferschein | Delivery notes within shipment |
| SendungADR | Hazardous goods data |
| SendungAdresse | Addresses (sender, receiver) |
| SendungPackmittel | Packaging materials used |
| SendungParameterWert | Parameter values |
| SendungArtikel | Articles in shipment |
| SendungInterneBemerkung | Internal notes |
| SendungMetadaten | Metadata |
| SendungLabel | Labels for shipment |
| SendungStatus | Status tracking |

**Stored Procedures (extensive - over 50):**
- `Sendung_SEL`, `Sendung_DEL_Komplett`
- `Sendung_SucheNummer`, `Sendung_SuchePUS`, `Sendung_SucheREF`, `Sendung_SucheSLB`, `Sendung_SucheLabelnummer`
- `Sendung_Avis2Sendung_SEL`, `Sendung_PusZuSendung_SEL`, `Sendung_IFCSUMZuSendung_SEL`, `Sendung_Vda4913ZuSendung_SEL`
- `Sendung_Hallenliste_*` - Warehouse list operations
- `SendungZuSendungListeZuweisen` / `SendungZuSendungListeZuweisungEntfernen`
- `Sendung_ErstelleSendungAdresseZuAbladestellenWerk_INS`, `Sendung_ErstelleSendungAdresseZuLieferant_INS`
- `Sendung_Stanzkreisprüfung`, `Sendung_AktualisiereAngeladenParameter`

**Views:**
- `view_Sendung_Recherche_Kurz` - Short search view
- `view_Sendung_Recherche_Lang_MitSendungLieferscheinNummer` - Extended search view

### 8.2 SendungListe (Shipment List / Consignment List)
**Business Entity:** Collection of shipments assigned to a tour/route
**Table:** `SendungListe`

**Stored Procedures:**
- `SendungListe_SEL`, `SendungListe_SEL_ByTourID`
- `SendungListe_ErstelleHallenliste_ByTour`
- `SendungListe_Kopieren_BySendungListeID`
- `SendungListe_SuchenNachNummer`
- `SendungListeFreigeben`
- `HoleSendungListeNummerZuSendung`

### 8.3 SendungListe_WeikundaLog
**Table:** `SendungListe_WeikundaLog` - Weikunda export logging

---

## DOMAIN 9: DISPOSITION & TOUR (Dispatch & Tour)

### 9.1 Tour (Tour/Route Plan)
**Business Entity:** A planned transport tour
**Table:** `Tour`

**Stored Procedures:**
- `Tour_SEL`, `Tour_Abschluss`, `Tour_NVFolgetourErstellen`
- `Tour_AlleGesplitteten_SEL_ByTourID`
- `Tour_StoppsAktualisieren`, `Tour_ZugnummerSetzen`
- `TourDetailErweitert_SEL`, `TourHatHallenliste_SEL`

**Views:**
- `view_Tour_AZA`, `view_Tour_AZB`, `view_Tour_AZF`
- `view_Tour_Route`, `view_Tour_RouteInformation`
- `view_SumGewichtLDMnachTour`

### 9.2 TourHistory (Tour History)
**Table:** `TourHistory` - Audit log of tour changes

### 9.3 TourRsn (Tour RSN - Reisenummer)
**Table:** `TourRsn` - Journey/trip number tracking

**Stored Procedures:** `TourRsn_SEL`, `TourRsnNachDatumsbereich_SEL`, `TourRsnNachTourFK_SEL`

### 9.4 Reise (Journey)
**Table:** `Reise`

**Stored Procedures:** `Reise_SEL`, `ErstelleReisenummer`, `ErstelleReisenummer_Abfahrt`

### 9.5 TourArbeitsAblauf (Tour Workflow)
**Table:** `TourArbeitsAblauf`

### 9.6 Arbeitsablauf (Workflow/Work Steps)
**Business Entity:** Detailed work steps within a tour
**Table:** Referenced via extensive stored procedures

**Stored Procedures:**
- `Arbeitsablauf_SEL_byTourFK`, `Arbeitsablauf_INS`, `Arbeitsablauf_UPD`, `Arbeitsablauf_DEL`
- `Arbeitsablauf_AktivstatusSetzten`
- `Arbeitsablauf_SplitterStart_INS/DEL`, `Arbeitsablauf_SplitterEnde_INS/DEL`
- `Arbeitsablauf_GleicheZeitenAnNachSplitten`
- `Arbeitsablauf_ZeitAnpassen_UPD`
- `ArbeitsablaufStreckenabschnitt_SEL_byTourFK`

### 9.7 StreckenabschnittDisponiert (Dispatched Route Segments)
**Table:** `StreckenabschnittDisponiert`

**Stored Procedures:**
- `StreckenabschnittDisponiert_SEL`, `StreckenabschnittDisponiert_Dispo_SEL`
- `StreckenabschnittDisponiert_AlleEntladen`, `StreckenabschnittDisponiert_Entladen`

### 9.8 StreckenabschnittUndisponiert (Undispatched Route Segments)
**Table:** `StreckenabschnittUndisponiert`

**Stored Procedures:**
- `StreckenabschnittUndisponiert_SEL`, `StreckenabschnittUndisponiert_ByArtikelzeileFK_SEL`

### 9.9 DispoOrt (Dispatch Location)
**Table:** Referenced via `DispoOrt_SEL`

### 9.10 Disporegel (Dispatch Rule)
**Table:** Referenced via `Disporegel_SEL`, `Disporegel_SEL_ByNiederlassung`

### 9.11 NVGebiete (Short-Distance Areas)
**Table:** Referenced via `NVGebiete_SEL`, `NVGebiete_NachNiederlassung_SEL`

### 9.12 NVVKosten / NVVKostenDetail (Short-Distance Costs)
**Tables:** `NVVKosten`, `NVVKostenDetail`

### 9.13 Inhalt (Content Type)
**Table:** `Inhalt` - Types of cargo content

---

## DOMAIN 10: ABFERTIGUNG (Dispatch/Loading)

### 10.1 Abfahrt (Departure)
**Business Entity:** A truck departure event
**Table:** Referenced via `Abfahrt_Abfahrt_*` stored procedures

### 10.2 Bordero (Bill of Lading / Shipping Manifest)
**Business Entity:** Loading manifest for a departure
**Table:** `Bordero`

**Stored Procedures:**
- `Abfahrt_Bordero_SEL/INS/UPD/DEL`
- `Abfahrt_BorderoEinzel_SEL`
- `Abfahrt_BorderoUebersicht_SEL`
- `Bordero_Weikunda_SEL_ByAbfahrtID`

### 10.3 BorderoNummerKreis (Bordero Number Range)
**Table:** `BorderoNummerKreis`

### 10.4 BorderoDetailBMW (BMW-specific Bordero Details)
**Table:** Referenced via `Abfahrt_BorderoDetailBMW_*` stored procedures

### 10.5 Reklamation (Complaint/Claim)
**Table:** Referenced via `Reklamation_SEL`, `ReklamationDetail_SEL`, etc.

**Views:**
- `view_Reklamation_CSVExport`
- `view_Reklamation_Uebersicht`

---

## DOMAIN 11: ABRECHNUNG (Billing/Settlement)

### 11.1 TUAbrechnung (Carrier Billing)
**Business Entity:** Invoice/settlement with carrier
**Table:** `TUAbrechnung`

**Stored Procedures:**
- `TUAbrechnung_SEL`, `TUAbrechnung_BelegExistiertBereits_SEL`
- `TUAbrechnung_WHDruck_SEL` (reprint)
- `TUAbrechnung_Storno_Sel` (cancellation)

**Views:** `view_TUAbrechnung_CSVExport`

### 11.2 TUAbrechnungsPosition (Billing Position)
**Table:** `TUAbrechnungsPosition`

### 11.3 TUAbrechnungsPositionsDetail (Billing Position Detail)
**Table:** `TUAbrechnungsPositionsDetail`

### 11.4 TUAbrechnungsPositionsZusatz (Billing Position Supplement)
**Table:** `TUAbrechnungsPositionsZusatz`

### 11.5 BuchungsJournal (Booking Journal)
**Table:** `BuchungsJournal`

**Stored Procedures:** `BuchungsTransfer_Neu_sp`, `BuchungsTransfer_DEL_sp`

### 11.6 Forecast (Cost Forecast)
**Table:** `Forecast`

| Column | Type | Description |
|--------|------|-------------|
| ForecastID | uniqueidentifier | PK |
| + Details | | |

### 11.7 ForecastDetail
**Table:** `ForecastDetail`

### 11.8 ForecastTarif (Forecast Tariff)
**Table:** `ForecastTarif`

### 11.9 FakturaMANRelation (MAN Invoice Relation)
**Table:** `FakturaMANRelation`

---

## DOMAIN 12: EDI/SCHNITTSTELLEN (Interfaces)

### 12.1 VDA_4913_Auftrag (VDA 4913 Order)
**Business Entity:** VDA 4913 EDI dispatch note
**Table:** `VDA_4913_Auftrag`

| Column | Type | Description |
|--------|------|-------------|
| AuftragID | uniqueidentifier | PK |
| Version | nvarchar | |
| EmpfaengerNr | nvarchar | Receiver number |
| SenderNr | nvarchar | Sender number |
| + Many fields | | |

### 12.2 VDA_4913_Transport
**Table:** `VDA_4913_Transport`

| Column | Type | Description |
|--------|------|-------------|
| TransportID | uniqueidentifier | PK |
| AuftragFK | uniqueidentifier | FK -> VDA_4913_Auftrag |
| + Transport details | | |

### 12.3 VDA_4913_Lieferschein (Delivery Note)
**Table:** `VDA_4913_Lieferschein`

| Column | Type | Description |
|--------|------|-------------|
| LieferscheinID | uniqueidentifier | PK |
| TransportFK | uniqueidentifier | FK -> VDA_4913_Transport |
| Version | nvarchar | |
| + Many fields | | |

### 12.4 VDA_4913_LieferscheinPosition
**Table:** `VDA_4913_LieferscheinPosition`

### 12.5 VDA_4913_LieferscheinPackstueck
**Table:** `VDA_4913_LieferscheinPackstueck`

### 12.6 VDA_4913_LieferscheinEinzelPackstueck
**Table:** `VDA_4913_LieferscheinEinzelPackstueck`

### 12.7 VDA_4913_LieferscheinText
**Table:** `VDA_4913_LieferscheinText`

### 12.8 PUSAuftrag (PUS Order)
**Business Entity:** PUS (Pick-Up-Sheet) EDI order
**Table:** `PUSAuftrag`

### 12.9 PUSAuftragPosition
**Table:** `PUSAuftragPosition`

### 12.10 DESADVAuftrag (DESADV Order)
**Table:** `DESADVAuftrag`

### 12.11 DESADVAuftragPosition
**Table:** `DESADVAuftragPosition`

### 12.12 DESADVLieferantenSperrliste
**Table:** `DESADVLieferantenSperrliste` - Blocked suppliers for DESADV

### 12.13 IFCSUM_Auftrag
**Table:** `IFCSUM_Auftrag` - EDIFACT IFCSUM message

### 12.14 IFCSUM_AuftragPosition
**Table:** `IFCSUM_AuftragPosition`

### 12.15 IFCSUM_File / IFCSUM_FileData
**Tables:** `IFCSUM_File`, `IFCSUM_FileData` - File storage for IFCSUM messages

### 12.16 VDA4927 (VDA 4927 Standard)
**Tables:** Referenced via stored procedures:
- `VDA4927_Position`
- `VDA4927_Matching`

---

## DOMAIN 13: DFU (Datenfernuebertragung / Electronic Data Interchange)

### 13.1 DFUAutorouting
**Table:** `DFUAutorouting` - Automatic routing of EDI messages

### 13.2 DFUFreischaltung (EDI Activation)
**Table:** `DFUFreischaltung` - EDI channel activation

### 13.3 DFUUebersetzung (EDI Translation)
**Table:** `DFUUebersetzung` - EDI code translation tables

### 13.4 SchnittstellenBenachrichtigungMaileinstellung
**Table:** `SchnittstellenBenachrichtigungMaileinstellung` - Interface notification email settings

---

## DOMAIN 14: DOKUMENTE & BERICHTE (Documents & Reports)

### 14.1 DocumentAblage (Document Storage)
**Table:** `DocumentAblage`

### 14.2 BibliothekEintrag (Library Entry)
**Table:** `BibliothekEintrag` - File/document library

### 14.3 TextBaustein (Text Building Block)
**Table:** `TextBaustein`

### 14.4 TextBausteinRubrik (Text Block Category)
**Table:** `TextBausteinRubrik`

### 14.5 DokumentVersand (Document Dispatch)
**Table:** Referenced via `DokumentVersand_SEL` and related procedures

### 14.6 NOFA (Sonderfahrt-Formular)
**Table:** `NOFA` - Special trip forms

### 14.7 Menuebaum (Menu Tree)
**Table:** `Menuebaum` - Application menu structure

---

## DOMAIN 15: TEKOS-SPECIFIC TABLES

### 15.1 DispositionsTopf / DispositionsTopfRelation
**Tables:** `DispositionsTopf`, `DispositionsTopfRelation` - Dispatch groupings

### 15.2 Linie / LinienPlan (Route Line)
**Tables:** `Linie`, `LinienPlan` - Fixed route line definitions

### 15.3 LinienverechnungsArt (Line Billing Type)
**Table:** `LinienverechnungsArt`

### 15.4 Kostenstellen (Cost Centers)
**Table:** Referenced in TeKoS

### 15.5 KfzKalender (Vehicle Calendar)
**Table:** Referenced in TeKoS

### 15.6 Adresse (Address) [TeKoS]
**Table:** `Adresse` - TeKoS-specific address table

### 15.7 AdresseTransportVorschrift [TeKoS]
**Table:** `AdresseTransportVorschrift`

---

## DOMAIN 16: SYSTEM TABLES

### 16.1 LetzteAktualisierung (Last Update)
**Table:** `LetzteAktualisierung` - Tracks when cache was last synced per table

### 16.2 Version
**Table:** `Version` - Database schema version tracking

### 16.3 ASAAProtokoll
**Table:** `ASAAProtokoll` - ASAA (Automatische Sendungsanlage Aus Avisen) protocol/log

### 16.4 Relation
**Table:** `Relation` - Generic relation table

### 16.5 VorholkostenMatrix
**Table:** `VorholkostenMatrix` - Pre-haul cost matrix

---

## STORED PROCEDURE CATALOG (Partial - Most Important)

### Naming Convention
- `{Table}_SEL` - Select/Read
- `{Table}_INS` - Insert
- `{Table}_UPD` - Update
- `{Table}_DEL` - Delete
- `{Table}_NachXYFK_SEL` - Select filtered by FK
- `{Table}_SEL_4DEL` - Select for deletion tracking
- `Bericht_*` - Report procedures
- `Abfahrt_*` - Departure/dispatch procedures
- `Sendung_*` - Shipment procedures
- `Avis_*` - Advice/notification procedures
- `IFCSUM_*` - IFCSUM EDI procedures
- `VDA4921*EX_*` - VDA 4921 export procedures
- `Speditionsbuch_*` - Forwarding book procedures
- `TRP_*` - TRP export procedures
- `TourStatusDienst_*` - Tour status service procedures

### Count: ~350+ stored procedures identified

---

## VIEWS CATALOG

### Data Cache Deletion Tracking Views (28 views)
```
view_DEL_Cache_Abladestelle
view_DEL_Cache_EinsatzArt
view_DEL_Cache_EntladeZone
view_DEL_Cache_Gebietsspediteuerschluessel
view_DEL_Cache_GrenzzollAmt
view_DEL_Cache_KFZ
view_DEL_Cache_KFZNiederlassung
view_DEL_Cache_KFZNiederlassungEinsatzart
view_DEL_Cache_Kontakte
view_DEL_Cache_Laenderkennzeichen
view_DEL_Cache_LagerRouting
view_DEL_Cache_LieferantenNiederlassung
view_DEL_Cache_LieferantenOEM
view_DEL_Cache_LieferantenWerkSperrtage
view_DEL_Cache_LieferantenZeitfenster
view_DEL_Cache_NiederlassungOEM
view_DEL_Cache_NiederlassungOptionen
view_DEL_Cache_NiederlassungRegion
view_DEL_Cache_PackMittel
view_DEL_Cache_Route
view_DEL_Cache_RouteSuffix
view_DEL_Cache_Transportbehaelter
view_DEL_Cache_TU
view_DEL_Cache_TULizenz
view_DEL_Cache_TUNiederlassung
view_DEL_Cache_UmschlagPunkt
view_DEL_Cache_Werk
view_DEL_Cache_WerkNiederlassungDispoOrt
view_DEL_Cache_WerkSperrzeiten
```

### Business Views
```
view_Bericht_Sonderfahrt          -- Special trip report
view_Lagerliste                    -- Warehouse list
view_Reklamation_CSVExport         -- Complaint CSV export
view_Reklamation_Uebersicht        -- Complaint overview
view_Sendung_Recherche_Kurz        -- Shipment search (short)
view_Sendung_Recherche_Lang_MitSendungLieferscheinNummer  -- Extended shipment search
view_SumGewichtLDMnachTour         -- Weight/LDM sum by tour
view_Tour_AZA / AZB / AZF         -- Tour work time views
view_Tour_Route                    -- Tour route view
view_Tour_RouteInformation         -- Tour route details
view_TUAbrechnung_CSVExport        -- Carrier billing CSV export
```

---

## RELATIONSHIP DIAGRAM (Key Relationships)

```
Niederlassung ──1:N── NiederlassungOEM ──N:1── OEM
Niederlassung ──1:N── NiederlassungOptionen
Niederlassung ──1:N── NiederlassungRegion

OEM ──1:N── Werk ──1:N── WerkNiederlassungDispoOrt ──N:1── Niederlassung
OEM ──1:N── Werk ──1:N── Abladestelle
OEM ──1:N── Werk ──1:N── EntladeZone

Lieferanten ──1:N── LieferantenOEM ──N:1── OEM
Lieferanten ──1:N── LieferantenNiederlassung ──N:1── Niederlassung
Lieferanten ──1:N── LieferantenWerkSperrtage
Lieferanten ──1:N── LieferantenZeitfenster

TU ──1:N── KFZ ──1:N── KFZNiederlassung ──N:1── Niederlassung
TU ──1:N── TUNiederlassung ──N:1── Niederlassung
TU ──1:N── TULizenz ──N:1── Lizenz

Benutzer ──1:N── BenutzerNiederlassung ──N:1── Niederlassung
BenutzerNiederlassung ──1:N── BenutzerNiederlassungBenutzerGruppe ──N:1── BenutzerGruppe
BenutzerGruppe ──1:N── BenutzerGruppeBerechtigung ──N:1── Berechtigung

Kontakte ──N:1── (Werk | Lieferant | OEM | Niederlassung | Abladestelle)  [via KontaktReferenzFK]

Route ──N:1── Niederlassung
Route ──N:1── OEM
Route ──N:1── Werk
Route ──1:N── RouteSuffix

Avis ──N:1── Lieferant
Avis ──N:1── Werk
Avis ──N:1── Route
Avis ──1:N── Artikelzeile ──1:N── ArtikelzeileADR
Avis ──1:N── AvisTransportVorschrift

Sendung ──1:N── SendungLieferschein
Sendung ──1:N── SendungPackmittel
Sendung ──1:N── SendungAdresse
Sendung ──1:N── SendungADR
Sendung ──1:N── SendungArtikel
Sendung ──1:N── SendungLabel
Sendung ──N:M── SendungListe  (via SendungZuSendungListeZuweisen)

SendungListe ──N:1── Tour
Tour ──1:N── Arbeitsablauf
Tour ──1:N── Streckenabschnitt (Disponiert/Undisponiert)
Tour ──1:1── TourRsn
Tour ──1:N── TourHistory

Tour ──1:N── Abfahrt ──1:N── Bordero
Bordero ──N:N── Sendung

TUAbrechnung ──1:N── TUAbrechnungsPosition ──1:N── TUAbrechnungsPositionsDetail

VDA_4913_Auftrag ──1:N── VDA_4913_Transport ──1:N── VDA_4913_Lieferschein
VDA_4913_Lieferschein ──1:N── VDA_4913_LieferscheinPosition
VDA_4913_Lieferschein ──1:N── VDA_4913_LieferscheinPackstueck
VDA_4913_Lieferschein ──1:N── VDA_4913_LieferscheinEinzelPackstueck
VDA_4913_Lieferschein ──1:N── VDA_4913_LieferscheinText
```

---

## TOTAL TABLE COUNT

| Domain | Table Count |
|--------|------------|
| Organization Structure | ~10 |
| Customers & Partners | ~15 |
| Transport Resources | ~12 |
| Logistics Configuration | ~12 |
| Lookup Tables | ~20 |
| User Management | ~8 |
| Avis (Notifications) | ~12 |
| Sendung (Shipments) | ~15 |
| Disposition & Tour | ~15 |
| Dispatch/Loading | ~6 |
| Billing/Settlement | ~10 |
| EDI Interfaces | ~15 |
| DFU | ~4 |
| Documents | ~7 |
| TeKoS-specific | ~10 |
| System Tables | ~5 |
| **TOTAL** | **~175+ tables** |

---

## NOTES FOR REWRITE

1. **GUID-based PKs everywhere** - Consider whether to keep GUIDs or switch to auto-increment integers for performance
2. **Polymorphic Kontakte table** - The contact/address table uses `KontaktReferenzFK` as a polymorphic FK pointing to many parent entities. Consider splitting or using a proper pattern.
3. **Client-side caching pattern** with `LesbarerTimestamp` + `view_DEL_Cache_*` views is a complex sync mechanism. A modern approach would use real-time sync (SignalR/WebSockets) or REST API with ETag/Last-Modified headers.
4. **Stored procedure CRUD pattern** (~350+ SPs) - All data access goes through stored procedures. For a rewrite, consider ORM (Entity Framework / Prisma) with repository pattern.
5. **Timestamp-based optimistic concurrency** via SQL Server `timestamp` column is well-established but needs equivalent in new system.
6. **Multi-OEM architecture** - The system is designed to handle multiple automotive OEMs (BMW, Daimler, VW, Porsche, MAN) with OEM-specific logic in VDA export procedures.
7. **TeKoS module** has its own parallel table structure (Geschaeftsstelle vs Niederlassung, different user management) that would benefit from unification.
8. **EDI/VDA tables** (VDA 4913, VDA 4921, VDA 4927, IFCSUM, PUS, DESADV) are highly OEM-specific and may need to remain as specialized import/export staging tables.
