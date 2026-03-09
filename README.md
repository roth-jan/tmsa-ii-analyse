# TMSA-II Spec Book — Grundlage für Web-App Rewrite

**Stand:** 09.03.2026
**Quelle:** 41 Repositories von git.asp-central.de (Bonobo Git Server)
**Umfang:** ~2,7 Mio Zeilen Code (VB.NET + C#)

---

## Was ist TMSA-II?

**Transport-Management-System-Automotiv II** — ein vollständiges TMS für die Automobillogistik. Eingesetzt bei Speditionen wie Andreas Schmid für den Transport von Automobilteilen zwischen OEMs (BMW, Daimler, VW, Porsche, MAN), Werken, Lieferanten und Umschlagpunkten.

### Aktueller Tech-Stack (Legacy)

| Komponente | Technologie | Version |
|---|---|---|
| Sprache | VB.NET + C# | .NET Framework 3.5 |
| UI | WinForms + DevExpress | v11.2 (2012) |
| Backend | WCF Services | .NET 3.5 |
| Datenbank | SQL Server + SQL CE (Cache) | |
| Berichte | DevExpress Reports | v11.2 |
| EDI | Custom VDA4913/EDIFACT Parser | |

### Letzter Commit
- Hauptprojekt: April 2017
- Neueste Module: September 2021
- System wird nicht mehr aktiv weiterentwickelt

---

## Dokumentation

| Dokument | Inhalt | Zeilen |
|---|---|---|
| [TMSA-II-Domain-Model.md](TMSA-II-Domain-Model.md) | Entities, Enums, Business Rules, Berechnungen, Glossar | 863 |
| [TMSA-II-UI-Workflows.md](TMSA-II-UI-Workflows.md) | Alle 16 UI-Module, Screens, Workflows, Berechtigungen | ~350 |
| [TMSA-II-Database-Schema.md](TMSA-II-Database-Schema.md) | 175+ Tabellen, 350+ Stored Procedures, 16 Domänen | 1249 |
| [TMSA-II-Berichte-EDI-Services-Dokumentation.md](TMSA-II-Berichte-EDI-Services-Dokumentation.md) | 47 Berichte, 10 EDI-Schnittstellen, 5 Hintergrunddienste | 740 |
| [TMSA-II-Gebrochene-Verkehre.md](TMSA-II-Gebrochene-Verkehre.md) | Multi-Leg Transporte, Streckenabschnitte, Tour-Splitting, LagerRouting | ~350 |

---

## Kern-Business-Flow

```
Avis-Erfassung (Lieferant meldet Sendung an)
  → Disposition/Mengenplan (Disponent weist Avise Touren zu)
    → Abfahrtsverwaltung (Borderos erstellen, KFZ zuweisen)
      → Nachbearbeitung (Korrekturen, Quittungen)
        → TU-Abrechnung (Bewerten → Freigeben → Erzeugen)
          → Berichtswesen (50+ Berichte und Dokumente)
```

## Kern-Entities

| Entity | Beschreibung |
|---|---|
| **Niederlassung** | Geschäftsstelle/Standort der Spedition |
| **OEM** | Automobilhersteller (BMW, Daimler, VW, Porsche, MAN) |
| **Werk** | Produktionsstandort eines OEM |
| **Lieferant** | Zulieferer der OEMs |
| **Abladestelle** | Entladeort an einem Werk |
| **TU** | Transport-Unternehmer (Carrier) |
| **KFZ** | Fahrzeug eines TU |
| **Route** | Transportweg zwischen Standorten |
| **Avis** | Voranmeldung einer Sendung |
| **Artikelzeile** | Position innerhalb eines Avis |
| **Tour** | Zusammenfassung von Avisen zu einer Fahrt |
| **Abfahrt** | Tatsächliche Abfahrt eines KFZ |
| **Bordero** | Lademanifest/Ladeschein einer Abfahrt |
| **Sendung** | Einzelne Sendung auf einem Bordero |
| **Kondition** | Preisberechnung (Tour-/Last-/Leerkm-Faktoren) |
| **Forecast** | Kapazitätsplanung pro OEM/NL/Relation |

## Datenbank

- **175+ Tabellen** mit GUID-PKs und Audit-Spalten
- **350+ Stored Procedures** (alle CRUD-Operationen)
- **Optimistic Concurrency** via TStamp/Rowversion
- **Client-Cache-Sync** via LesbarerTimestamp + DEL-Views
- **Multi-Mandanten** via NiederlassungFK-Filter

## EDI-Schnittstellen

| Standard | OEMs | Format |
|---|---|---|
| VDA 4913 | BMW, Daimler, VW, Porsche, MAN | 128-Byte Fixed-Length |
| VDA 4927 | VWT | Packmittelkonto |
| DESADV | VWT | EDIFACT (Altova MapForce) |
| IFCSUM | Brose | EDIFACT → Stored Procedures |
| IFTSTA | Generisch | EDIFACT Status Messages |

## Berechtigungssystem

Feingranulares Rechtesystem via `TmsaInfo.hatRecht("modul|aktion")`:
- Pro Modul: bearbeiten, löschen, speichern
- Spezialrechte: "Avis Freigabe", "Mengenplan nur sehen", "Faktura Status zurücksetzen"
- Niederlassungs-bezogene Benutzergruppen

---

## Nächste Schritte (Web-App Rewrite)

1. **Technologie-Entscheidung**: Next.js / Blazor / anderes Framework
2. **Prioritäten setzen**: Welche Module zuerst?
3. **Datenmodell modernisieren**: GUID→ID, Polymorphe Kontakte aufsplitten, SP→ORM
4. **API-Design**: REST/GraphQL auf Basis der WCF-Interfaces
5. **Modul für Modul umsetzen**: Beginnend mit dem Kern-Flow (Avis → Dispo → Abfahrt)
