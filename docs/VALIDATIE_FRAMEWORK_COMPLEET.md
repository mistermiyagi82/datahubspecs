# DataPraatFormaat — Compleet Validatie Framework

## Totaaloverzicht

| Fase | Naam | Type | Totaal checks | MVP checks | Gedrag bij fout |
|------|------|------|:---:|:---:|------|
| 1 | Bestandscontrole | Verplicht | 10 | 10 | Stopt bij GEFAALD |
| 2 | Structuurherkenning | Verplicht | 13 | 13 | Stopt bij GEFAALD |
| 3 | Beschrijvende Statistiek | Standaard | 7 | 7 | Waarschuwingen in rapport |
| 4 | Inhoudsvalidatie | Standaard | 19 | 12 | FOUT per cel/rij, WAARSCHUWING per kolom |
| 5 | Consistentiecontroles | Standaard | 15 | 2 | FOUT per rij per geschonden regel |
| 6 | Statistische Analyse | Optioneel | 6 | 2 | WAARSCHUWING per bevinding |
| 7 | Intelligente Analyse (AI/ML/LLM) | Optioneel | 6 | 2 | Verklarend rapport met score en suggesties |
| | **TOTAAL** | | **76** | **48** | |

**Legenda:**
- `[MVP]` — mee in eerste versie (te bouwen)
- `[ ]` — niet in MVP, wel op de lijst (later te bouwen)
- **NIEUW** = check bestaat nog niet in het originele framework, is nieuw toegevoegd op basis van interviews met Thom en Harm Nico

Niets is op dit moment gebouwd. Alles moet nog worden geïmplementeerd.

---

## Fase 1 — Bestandscontrole

> *"Kunnen we het bestand überhaupt openen en lezen?"*
> Uitkomst: GESLAAGD of GEFAALD — applicatie stopt bij GEFAALD

| Status | ID | Check | Omschrijving | Voorbeeld output | Formaat |
|:---:|------|-------|-------------|-----------------|---------|
| [MVP] | 1.1 | Bestand bestaat | Is het opgegeven pad aanwezig en toegankelijk? | GESLAAGD: pad gevonden | Alle |
| [MVP] | 1.2 | Formaat herkenning | Is het bestandstype CSV, Excel, XML of JSON? Controleer extensie én magic bytes. | GESLAAGD: CSV herkend | Alle |
| [MVP] | 1.3 | Niet leeg | Heeft het bestand meer dan 0 bytes? Zijn er rijen aanwezig? | GEFAALD: 0 bytes | Alle |
| [MVP] | 1.4 | Encoding detectie | Welke tekenset (UTF-8, Latin-1)? Zijn er onleesbare tekens? | WAARSCHUWING: Latin-1, 3 fouten op rij 44, 201, 889 | Alle |
| [MVP] | 1.5 | Delimiter detectie | Welk scheidingsteken (, ; tab)? Is het consistent? | GESLAAGD: ";" consistent in 4.312 rijen | CSV |
| [MVP] | 1.6 | Structurele integriteit | Is het bestand niet beschadigd (corrupt Excel, ongeldige JSON, misvormde XML)? | GEFAALD: JSON-syntaxfout op positie 4.821 | Alle |
| [MVP] | 1.7 | Bestandsgrootte | Valt de grootte binnen verwachte grenzen? | GESLAAGD: 1,2 MB (max 50 MB) | Alle |
| [MVP] | 1.8 | Kolombreedtes gedefinieerd | Zijn de kolombreedtes (start- en eindposities) bekend via configuratie? (Fixed-Width) | GEFAALD: kolombreedtes niet gedefinieerd | Fixed-Width |
| [MVP] | 1.9 | Herhalend element bepaald | Welk element fungeert als 'rij'? Is dit element consistent aanwezig? | GEFAALD: herhalend element kon niet worden bepaald, specificeer het XPath-pad | XML/YAML |
| [MVP] | 1.10 | Root-type herkend | Is het root-element een object {} of een array []? | GEFAALD: JSON begint niet met een object of array | JSON |

**Fase 1 totaal: 10 checks — alle MVP**

---

## Fase 2 — Structuurherkenning

> *"Begrijpen we hoe het bestand is opgebouwd?"*
> Uitkomst: GESLAAGD of GEFAALD — applicatie stopt bij GEFAALD

| Status | ID | Check | Omschrijving | Voorbeeld output | Formaat |
|:---:|------|-------|-------------|-----------------|---------|
| [MVP] | 2.1 | Kolomtitels valide | Heeft de eerste rij kolomnamen? Zijn die gevuld (geen lege headers) en uniek? Bij XML/YAML: zijn de kind-elementen benoemd? | GEFAALD: kolom 4 heeft geen naam / "datum" komt 2x voor | Alle |
| [MVP] | 2.2 | Consistent kolomaantal | Heeft elke rij evenveel kolommen als de header? Afwijkende rijen met rijnummer. | WAARSCHUWING: rij 88 (16 ipv 18), rij 204 (19 ipv 18) | CSV |
| [MVP] | 2.3 | Lege rijen & kolommen | Zijn er volledig lege rijen of kolommen? Zo ja, hoeveel en waar? | WAARSCHUWING: 2 lege rijen, 1 lege kolom | Alle |
| [MVP] | 2.4 | Afmetingen bestand | Totaal aantal rijen (exclusief header) en kolommen als basisinformatie. | INFO: 4.312 rijen, 18 kolommen | Alle |
| [MVP] | 2.5 | Tabbladen gevalideerd | Hoeveel tabbladen zijn aanwezig? Alle niet-lege tabbladen worden gevalideerd. | INFO: 3 tabbladen; "Medewerkers" en "Historisch" gevalideerd, "Leeg" overgeslagen | Excel/ODS |
| [MVP] | 2.6 | Verborgen rijen & kolommen | Zijn er verborgen rijen of kolommen? Inhoud wordt meegenomen maar kan voor onverwachte resultaten zorgen. | WAARSCHUWING: tabblad "Resultaten" bevat 5 verborgen kolommen (F-J) | Excel/ODS |
| [MVP] | 2.7 | Consistente arraystructuur | Hebben alle objecten in een array dezelfde sleutels? Ontbrekende of extra sleutels per positie. | WAARSCHUWING: item 14 mist sleutel "geboortedatum" | JSON/JSONL |
| [MVP] | 2.8 | Geen dubbele sleutelnamen | Bevatten objecten geen dubbele sleutelnamen? | WAARSCHUWING: object op positie 5 bevat sleutel "datum" twee keer | JSON/YAML |
| [MVP] | 2.9 | Nesting-diepte acceptabel | Hoe diep is de structuur maximaal genest? Te diep duidt op exportfout. | WAARSCHUWING: maximale nesting-diepte is 9, verwacht max 3 | JSON/XML/YAML |
| [MVP] | 2.10 | Samengevoegde cellen | Zijn er samengevoegde (merged) cellen? Welke cellen en bereiken? Merged cells breken parsing. **NIEUW** | WAARSCHUWING: 14 merged cellen gevonden (A1:C1, D5:D10, ...) | Excel/ODS |
| [MVP] | 2.11 | Formules in cellen | Bevat het bestand cellen met formules in plaats van waarden? Duidt op handmatige bewerking, niet schone export. **NIEUW** | WAARSCHUWING: 23 cellen bevatten formules (B2: =SOM(C2:F2), ...) | Excel/ODS |
| [MVP] | 2.12 | Herhaalde headerrijen | Komt de headerrij meerdere keren voor in het bestand? Veelvoorkomend bij gepagineerde Excel-exports. **NIEUW** | WAARSCHUWING: headerrij herhaald op rij 51, 101, 151 (paginering) | CSV/Excel |
| [MVP] | 2.13 | Signalen van handmatige bewerking | Meta-check: combinatie van formules, verborgen rijen/kolommen, samengevoegde cellen, inconsistente opmaak. Elk signaal apart is mild, samen duiden ze op handwerk. **NIEUW** | WAARSCHUWING: 3 signalen van handmatige bewerking (formules, verborgen kolommen, merged cells) | Excel/ODS |

**Fase 2 totaal: 13 checks — alle MVP**

---

## Fase 3 — Beschrijvende Statistiek

> *"Wat zit er globaal in het bestand — zonder te oordelen?"*
> Uitkomst: Informatief, waarschuwingen bij opvallende waarden

| Status | ID | Check | Omschrijving | Voorbeeld output | Formaat |
|:---:|------|-------|-------------|-----------------|---------|
| [MVP] | 3.1 | Vulgraad per kolom | Percentage gevulde waarden per kolom. Drempel bijv. < 50%. | WAARSCHUWING: "opmerkingen" 32% gevuld | Alle |
| [MVP] | 3.2 | Datatype detectie | Per kolom: overheersend type (getal, datum, tekst, boolean, gemengd). | INFO: "leeftijd" getal 98%, "naam" tekst 100% | Alle |
| [MVP] | 3.3 | Uniekheid per kolom | Percentage unieke waarden. Hoog = mogelijk ID, laag = categorisch. | INFO: "medewerker_id" 100% uniek | Alle |
| [MVP] | 3.4 | Meest voorkomende waarden | Top-5 per kolom. Helpt bij herkennen van waardelijsten of fouten. | INFO: "status" → OPEN 62%, GESLOTEN 31%, ... | Alle |
| [MVP] | 3.5 | Numerieke statistieken | Min, max, gemiddelde, mediaan, stddev voor getalkolommen. | INFO: "salaris" min 1.200, max 12.400, gem 3.847 | Alle |
| [MVP] | 3.6 | Datumbereik | Vroegste en laatste datum, span in dagen/maanden/jaren. | INFO: "geboortedatum" 1942-03-01 t/m 2005-11-30 | Alle |
| [MVP] | 3.7 | Tekstlengte | Kortste, langste en gemiddelde lengte voor tekstkolommen. | WAARSCHUWING: "omschrijving" max 2.048 tekens | Alle |

**Fase 3 totaal: 7 checks — alle MVP**

---

## Fase 4 — Inhoudsvalidatie

> *"Klopt de inhoud van elke cel inhoudelijk?"*
> Uitkomst: FOUT per cel/rij, WAARSCHUWING per kolom

| Status | ID | Check | Categorie | Omschrijving | Voorbeeld output | Formaat |
|:---:|------|-------|-----------|-------------|-----------------|---------|
| [MVP] | 4.1 | Lege en schijnbaar gevulde cellen | Verplichte velden | Welke kolommen bevatten lege, NULL of "nvt"-achtige waarden? Cellen met alleen spaties? | INFO: "geboortedatum" 3,6% leeg / WAARSCHUWING: 8 cellen in "naam" bevatten alleen spaties | Alle |
| [MVP] | 4.2 | Verplichte kolommen altijd gevuld | Verplichte velden | Zijn aangewezen sleutelkolommen altijd gevuld? | FOUT: "medewerker_id" leeg op rij 88, 204 | Alle |
| [MVP] | 4.3 | Juist soort waarde in elke kolom | Typecontroles | Zijn er cellen in een "getal"-kolom die tekst bevatten? Of andersom? | FOUT: "leeftijd" bevat "onbekend" op rij 14, 87 | Alle |
| [MVP] | 4.4 | Geldige en consistente datums | Typecontroles | Is elke waarde een geldige datum? Consistent formaat? Toekomstige of onrealistische datums? | FOUT: 23 ongeldige datums; 120 rijen DD/MM/YYYY, rest DD-MM-YYYY | Alle |
| [MVP] | 4.5 | Getallen zonder opmaakvuil | Typecontroles | Geen valuta-tekens, komma/punt-verwarring of andere opmaak in getalkolommen? | FOUT: "bedrag" bevat euro-teken op rij 5, 6, 7 | Alle |
| [ ] | 4.6 | Geldig e-mailadres | Formaatcontroles | Voldoet aan patroon x@y.z? | FOUT: 5 ongeldige adressen (rij 3, 77, 210) | Alle |
| [ ] | 4.7 | Geldig telefoonnummer | Formaatcontroles | Voldoet aan herkend nummerpatroon (NL, internationaal)? | WAARSCHUWING: 8 nummers matchen geen patroon | Alle |
| [ ] | 4.8 | Geldig postcodeformaat | Formaatcontroles | Voldoet aan verwacht postcode-formaat (bijv. 1234 AB)? | FOUT: 3 ongeldige postcodes (rij 6, 199, 400) | Alle |
| [ ] | 4.9 | Geldig identificatienummer | Formaatcontroles | Formaat- en checksumvalidatie voor IBAN, BSN, KvK en vergelijkbare nummers. | FOUT: 2 BSN-nummers falen elfproef (rij 14, 892) | Alle |
| [ ] | 4.10 | Eigen patrooncheck | Formaatcontroles | Configureerbaar: eigen regex-patroon per kolom (bijv. productnummer PRD-XXXXX). | FOUT: 4 waarden matchen niet op "productnummer" | Alle |
| [MVP] | 4.11 | Waarde staat op de toegestane lijst | Domein & bereik | Vallen waarden binnen een verwachte vaste lijst? | WAARSCHUWING: 3 onbekende statuswaarden ("ONBEKEND", "nvt", "—") | Alle |
| [MVP] | 4.12 | Waarde buiten toegestaan bereik | Domein & bereik | Vallen getallen of datums binnen opgegeven min–max? Negatieve waarden waar dat onlogisch is? | FOUT: leeftijd 150 en -3 (rij 402, 1188) | Alle |
| [MVP] | 4.13 | Dubbele rijen en sleutels | Duplicaten | Volledig identieke rijen? Waarden in ID-kolommen die meerdere keren voorkomen? | WAARSCHUWING: 6 identieke rijen / FOUT: "medewerker_id" M00412 komt 2x voor | Alle |
| [ ] | 4.14 | Null vs ontbrekend veld consistent | JSON-specifiek | Is het verschil tussen null en een ontbrekend veld consequent toegepast? | WAARSCHUWING: "einddatum" in 34 objecten afwezig, in 12 objecten null | JSON/JSONL |
| [ ] | 4.15 | Getal als tekst | JSON-specifiek | Bevatten velden die getallen zouden moeten zijn een string-type? | FOUT: "bedrag" in 67 objecten tekst ("1200" i.p.v. 1200) | JSON/JSONL |
| [MVP] | 4.16 | Bekende placeholder- en standaardwaarden | Placeholders & defaults | Systeemwaarden voor "onbekend"? Datums (1900-01-01, 9999-12-31), getallen (999, -1), tekst (unknown, nvt, test, xxx). | WAARSCHUWING: "geboortedatum" 142x "1900-01-01" | Alle |
| [MVP] | 4.17 | Getalnotatie-ambiguïteit | NL-specifiek | Is `1.234` Nederlands (duizend-tweehonderdvierendertig) of internationaal (één komma twee-drie-vier)? Detecteer komma/punt-verwarring op basis van patroonanalyse. **NIEUW** | WAARSCHUWING: "bedrag" bevat ambigue notatie — 1.234 kan NL (1234) of EN (1.234) zijn. Patroonanalyse: 94% NL-formaat | Alle |
| [MVP] | 4.18 | Voorloopnullen verloren | NL-specifiek | Zijn numeriek opgeslagen kolommen die tekst hadden moeten zijn (postcode, BSN, gemeentecode)? Controle op verwachte lengte. **NIEUW** | FOUT: "postcode_cijfers" bevat waarde 162 (verwacht 4 cijfers) — vermoedelijk 0162 | Excel |
| [MVP] | 4.19 | Inconsistente schrijfwijze en witruimte | Datahygiëne | Bevat een kolom dezelfde waarde in verschillende schrijfwijzen? Trailing/leading spaties, hoofdlettergebruik, accenten. **NIEUW** | WAARSCHUWING: "plaatsnaam" bevat "Amsterdam", "amsterdam", "AMSTERDAM", "Amsterdam " (4 varianten, vermoedelijk 1 bedoeld) | Alle |

**Fase 4 totaal: 19 checks — 12 MVP, 7 later**

---

## Fase 5 — Consistentiecontroles

> *"Kloppen de verbanden tussen velden onderling en met externe bronnen?"*
> Uitkomst: FOUT per rij per geschonden regel

| Status | ID | Check | Omschrijving | Voorbeeld output | Formaat |
|:---:|------|-------|-------------|-----------------|---------|
| [MVP] | 5.1 | Datums in logische volgorde | Is begindatum altijd eerder dan einddatum? | FOUT: einddatum voor begindatum op 14 rijen | Alle |
| [MVP] | 5.2 | Bedragen kloppen rekenkundig | Is totaalbedrag gelijk aan de som van de deelbedragen? P × Q = Totaal? | FOUT: totaal wijkt af op rij 88, 204 | Alle |
| [ ] | 5.3 | Conditionele veldregels | Als type=rechtspersoon, dan kvk_nummer verplicht. Configureerbare if-then regels. | FOUT: kvk_nummer leeg terwijl type=rechtspersoon (rij 18, 77) | Alle |
| [ ] | 5.4 | Waarden staan in de referentielijst | Komen waarden voor in een opgegeven referentielijst (bijv. CBS-gemeentecodes)? | FOUT: gemeentecode 9999 niet in referentielijst (rij 5, 88, 301) | Alle |
| [ ] | 5.5 | Postcode en plaatsnaam kloppen | Komt de combinatie postcode + plaatsnaam overeen met bekende koppeltabellen? | FOUT: 1011 AB hoort bij Amsterdam, niet Rotterdam (rij 22, 109) | Alle |
| [ ] | 5.6 | Consistentie tussen tabbladen | Komen sleutelwaarden in het ene tabblad voor in een ander? Consistent gespeld? | WAARSCHUWING: "Facturen" bevat 34 klant-IDs niet in "Klanten" | Excel/ODS |
| [ ] | 5.7 | Consistentie tussen bestanden | Komen sleutelwaarden voor in een opgegeven referentiebestand? Ontbrekende records? | WAARSCHUWING: medewerker-bestand bevat 12 IDs niet in salarisbestand | Alle |
| [ ] | 5.8 | Significant verschil in aantal rijen | Meer dan drempel (standaard 20%) toe-/afname t.o.v. vorige upload? | WAARSCHUWING: 412 rijen, vorige upload had 4.200 — afname 90% | Alle |
| [ ] | 5.9 | Kolomstructuur ongewijzigd | Kolommen bijgekomen of verdwenen t.o.v. vorige upload? Hernoemde kolommen? | WAARSCHUWING: "afdeling_code" ontbreekt; "dept_id" is nieuw — mogelijk hernoemd | Alle |
| [ ] | 5.10 | Verdeling van waarden stabiel | Verdeling sterk afwijkend van vorige upload? Plotselinge verschuiving? | WAARSCHUWING: gemiddelde "salaris" daalde van €3.840 naar €890 (-77%) | Alle |
| [ ] | 5.11 | Nieuwe of verdwenen categoriewaarden | Waarden bijgekomen of verdwenen in vaste waardelijst-kolommen? | WAARSCHUWING: "status" bevat nieuwe waarde "GEBLOKKEERD" | Alle |
| [ ] | 5.12 | Verwacht aantal rijen | Valt het rijaantal binnen een door de gebruiker opgegeven verwachting? **NIEUW** | WAARSCHUWING: 412 rijen, verwachting was 4.000–5.000 (opgegeven bij upload) | Alle |
| [ ] | 5.13 | Verwachte kolomnamen | Komen de verwachte kolommen voor? Zijn er onverwachte kolommen? **NIEUW** | FOUT: verwachte kolom "factuurbedrag" ontbreekt; onverwachte kolom "test_kolom" aanwezig | Alle |
| [ ] | 5.14 | Verwachte totalen per kolom | Klopt het kolomtotaal met een door de gebruiker opgegeven verwachting? **NIEUW** | WAARSCHUWING: som "bedrag" = €13,5M, verwachting was ~€5,2M (bron: begroting) | Alle |
| [ ] | 5.15 | Verwachte periode | Valt het datumbereik binnen de verwachte rapportageperiode? **NIEUW** | WAARSCHUWING: data loopt t/m oktober, verwachte periode was heel 2024 | Alle |

**Fase 5 totaal: 15 checks — 2 MVP, 13 later**

---

## Fase 6 — Statistische Analyse

> *"Zijn er verborgen afwijkingen die moeilijk te definiëren zijn?"*
> Uitkomst: WAARSCHUWING per bevinding

| Status | ID | Check | Omschrijving | Voorbeeld output | Formaat |
|:---:|------|-------|-------------|-----------------|---------|
| [MVP] | 6.1 | Statistische verdeling en uitbijters | Numerieke waarden ver buiten verwachte spreiding (IQR of stddev)? Ongebruikelijke verdeling? | WAARSCHUWING: "salaris" 3 extreem hoge waarden (€980k, €1,2M, €2,1M) | Alle |
| [ ] | 6.2 | Onverwachte gaten of pieken in de tijd | Gaten of pieken in tijdreeksdata? | WAARSCHUWING: geen data in "registratiedatum" van 14 maart t/m 2 april | Alle |
| [ ] | 6.3 | Verwachte samenhang aanwezig | Kolomcombinaties die normaal samenhangen maar hier niet? | WAARSCHUWING: "datum_in_dienst" meer dan 60 jaar na "geboortedatum" | Alle |
| [ ] | 6.4 | Cijferverdeling (Benford / fraudesignaal) | Voldoet verdeling eerste cijfers aan Benford? Significante afwijking kan duiden op gefabriceerde waarden. | WAARSCHUWING: verdeling eerste cijfers in "factuurbedrag" wijkt af van Benford | Alle |
| [ ] | 6.5 | Bijna-dubbele rijen | Rijen die sterk op elkaar lijken maar niet exact gelijk zijn? | WAARSCHUWING: "Jan de Vries" vs "Jan de Vreis" (geboortedatum identiek) | Alle |
| [MVP] | 6.6 | Homogeniteitsdetectie | Is één waarde dominant (>80%) in een kolom waar variatie verwacht wordt? | WAARSCHUWING: "geboortedatum" 94% de waarde "1990-01-01" — mogelijk standaardwaarde | Alle |

**Fase 6 totaal: 6 checks — 2 MVP, 4 later**

---

## Fase 7 — Intelligente Analyse (AI/ML/LLM)

> *"Wat kunnen we ontdekken dat niet in expliciete regels te vangen is?"*
> Uitkomst: Verklarend rapport met score en suggesties

| Status | ID | Check | Omschrijving | Voorbeeld output | Formaat |
|:---:|------|-------|-------------|-----------------|---------|
| [MVP] | 7.1 | Semantische kolominterpretatie | LLM benoemt wat de kolom waarschijnlijk bevat op basis van naam + waarden. | INFO: "ref_nr" lijkt een intern ordernummer | Alle |
| [ ] | 7.2 | Automatische regelherkenning | LLM herkent bedrijfsregels op basis van patronen in de data. | INFO: kolom X altijd gevuld als kolom Y waarde Z heeft | Alle |
| [ ] | 7.3 | ML anomalie detectie | Isolation Forest markeert uitschieters op rijniveau zonder vooraf regels. | WAARSCHUWING: rij 1.204 afwijkend op 6 van 18 kenmerken | Alle |
| [ ] | 7.4 | Tekst kwaliteitscheck | LLM beoordeelt vrije tekstvelden op testdata of inconsistente stijl. | WAARSCHUWING: "omschrijving" bevat "aaaaaa" op rij 88 | Alle |
| [MVP] | 7.5 | Datakwaliteitsscore | AI genereert overall kwaliteitsscore 0-100 met samenvatting. | Score: 74/100 — 3 kolommen vulprobleem, 12 datumfouten | Alle |
| [ ] | 7.6 | Reparatiesuggesties | LLM stelt concrete verbeteringen voor per kolom of rij. | "geboortedatum" lijkt DD/MM/YYYY, rest is ISO 8601 | Alle |

**Fase 7 totaal: 6 checks — 2 MVP, 4 later**

---

## Rapportage — Statuswaarden

| Status | Veld | Mogelijke waarden |
|:---:|------|-------------------|
| [MVP] | rapport.status | GOEDGEKEURD · GOEDGEKEURD_MET_WAARSCHUWINGEN · AFGEKEURD |
| [MVP] | fase.status | GESLAAGD · WAARSCHUWINGEN · GEFAALD · OVERGESLAGEN |
| [MVP] | bevinding.ernst | FOUT · WAARSCHUWING · INFO |

---

## Rapportage — JSON-structuur

### Rapport-object

| Status | ID | Veld | Type | Omschrijving |
|:---:|------|------|------|-------------|
| [MVP] | R.1 | rapport.bestand | string | Bestandsnaam van het gevalideerde bestand |
| [MVP] | R.2 | rapport.gegenereerd_op | ISO 8601 | Tijdstip van validatie |
| [MVP] | R.3 | rapport.status | enum | Eindoordeel over het gehele bestand |
| [MVP] | R.4 | rapport.samenvatting | string | Mensvriendelijke uitleg van het eindoordeel |
| [MVP] | R.5 | rapport.totalen.fouten | integer | Totaal aantal FOUT-bevindingen |
| [MVP] | R.6 | rapport.totalen.waarschuwingen | integer | Totaal aantal WAARSCHUWING-bevindingen |
| [MVP] | R.7 | rapport.totalen.info | integer | Totaal aantal INFO-bevindingen |
| [MVP] | R.8 | rapport.fases | array | Lijst van fase-objecten |

### Fase-object

| Status | ID | Veld | Type | Omschrijving |
|:---:|------|------|------|-------------|
| [MVP] | F.1 | fase | integer | Fasenummer (1-7) |
| [MVP] | F.2 | naam | string | Naam van de fase |
| [MVP] | F.3 | status | enum | Resultaat van de fase |
| [MVP] | F.4 | samenvatting | string | Mensvriendelijke uitleg van het fase-resultaat |
| [MVP] | F.5 | statistieken | object | Fase-specifieke statistieken (vrije structuur) |
| [MVP] | F.6 | bevindingen | array | Lijst van bevinding-objecten |

### Bevinding-object

| Status | ID | Veld | Type | Omschrijving |
|:---:|------|------|------|-------------|
| [MVP] | B.1 | ernst | enum | FOUT · WAARSCHUWING · INFO |
| [MVP] | B.2 | check_id | string | ID van de uitgevoerde check (bijv. 4.5) |
| [MVP] | B.3 | kolom | string of null | Betreffende kolom, of null bij rij-niveau bevindingen |
| [MVP] | B.4 | omschrijving | string | Mensvriendelijke beschrijving van de bevinding |
| [MVP] | B.5 | details | object | Aanvullende technische details (vrije structuur per check) |
| [MVP] | B.6 | rijen | array | Lijst van rijnummers waar de bevinding optreedt |

---

## Samenvatting

| Fase | MVP | Later | Totaal |
|------|:---:|:---:|:---:|
| 1 — Bestandscontrole | 10 | 0 | 10 |
| 2 — Structuurherkenning | 13 | 0 | 13 |
| 3 — Beschrijvende Statistiek | 7 | 0 | 7 |
| 4 — Inhoudsvalidatie | 12 | 7 | 19 |
| 5 — Consistentiecontroles | 2 | 13 | 15 |
| 6 — Statistische Analyse | 2 | 4 | 6 |
| 7 — Intelligente Analyse | 2 | 4 | 6 |
| **Totaal** | **48** | **28** | **76** |

Alles moet nog gebouwd worden. Niets is geïmplementeerd.
