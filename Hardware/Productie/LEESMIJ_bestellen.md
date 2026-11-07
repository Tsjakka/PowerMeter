# Bestellen bij JLCPCB: PowerMeter v2

Gemaakt op 7 november 2026 uit `PowerMeter.kicad_pcb`.
Stand bij export:
- DRC: 0 onverbonden, 0 elektrische fouten, 0 fouten in de koppeling met het schema.
- ERC: 0 fouten.

## Bestanden

| Bestand | Upload bij JLCPCB als |
|---|---|
| `PowerMeter_v2_Gerbers.zip` | **Gerber-bestand** (koper, soldeermasker, pasta en silkscreen voor- en achterkant, omtrek, boorbestanden PTH + NPTH) |
| `JLCPCB_BOM.csv` | **BOM** bij PCB Assembly (18 regels, 122 SMD-onderdelen) |
| `JLCPCB_CPL.csv` | **CPL / pick-and-place** bij PCB Assembly |
| `Gerbers/` | dezelfde bestanden los, om te bekijken in een Gerber-viewer |

## Instellingen PCB

- **Printplaat:** 2 lagen, FR-4, 1,6 mm dik, 1 oz koper. De afmeting (140 × 98 mm) leest JLC uit de Gerbers.
- **Aantal:** 5 printen.
- **Afwerking:** *ENIG* is het vlakst voor de TQFP-48 met 0,5 mm pitch. *HASL lead-free* kan ook en is goedkoper.
- **Vias:** *tented*.
- **Ordernummer:** JLC drukt standaard een ordernummer op de silkscreen. Kies desgewenst een locatie of laat het verwijderen (dat kost iets extra).

## Instellingen PCB Assembly

- **Assembly:** *Economic*, alleen de **bovenkant** (*Top side*).
- **Aantal bestukt:** minimaal 2. Dan heb je een reserve.
- **Tooling holes:** *Added by JLCPCB*.
- **Plaatsing bevestigen:** zet *Confirm parts placement* aan. Je krijgt dan een foto/preview ter controle vóór productie.

**Controleer in de preview na het uploaden van BOM en CPL** de draaiing en de pin-1-markering. Afwijkende draaiingen kun je in de preview per onderdeel corrigeren.

| Onderdeel | Waar je op let |
|---|---|
| U2, U3, U5 (ATM90E32AS) | De pin-1-punt moet linksonder in het footprint staan (de chips zijn 90° gedraaid; de stroomingangen wijzen naar de CT-klemmen) |
| U4 (AP2112K, SOT-23-5) | Pin 1 en de oriëntatie. SOT-23 wordt bij JLC vaak 180° verkeerd gezet |
| D1 (SS14) | De kathodestreep moet aan de +5V-kant staan: de onderkant van D1, richting C1 |
| D2, D3, D7, D11 (LED's) | De kathode moet aan de GND-kant zitten |
| Y1–Y3 (kristallen) | De pin-1-hoek |
| D4–D14 (TVS) | Bidirectioneel: richting maakt niet uit |

Niet bestukt (DNP): R7 en R8 (I²C pull-ups, alleen nodig als je OLED-module ze zelf niet heeft).

## LCSC-onderdelen (gecontroleerd op lcsc.com, 7 november 2026)

| LCSC | Onderdeel |
|---|---|
| C177153 | ATM90E32AS-AU-R (3×) |
| C51118 | AP2112K-3.3TRG1 |
| C13739 | Kristal 16,384 MHz 3225, 20 pF, ±10 ppm (YXC X322516384MSB4SI) |
| C1518212 | 33 nF C0G 50 V 0805 (Murata GRM2165C1H333JA01D) (20×) |
| C140894 | SMAJ5.0CA bidirectionele TVS (LRC) (9×) |
| C2480 | SS14 |
| C1017 | Ferriet 600 Ω@100 MHz 0805 |
| C2297 | LED groen 0805 |
| C14663 / C15850 / C1779 / C12891 | 100 nF 0603 / 10 µF 0805 / 4,7 µF 0805 / 22 µF 1206 |
| C17561 / C17513 / C17560 | 22 Ω / 1 kΩ / 22 kΩ, 1 %, 0805 |
| C21190 / C25804 / C23140 | 1 kΩ / 10 kΩ / 33 Ω, 1 %, 0603 |

- **Voorraad:** controleer die vlak voor het bestellen; JLC laat het in de BOM-stap zien. Is er een onderdeel op, kies dan in die stap een gelijkwaardig alternatief (zelfde waarde, footprint en diëlektricum; bij de 33 nF beslist **C0G/NP0**).
- **Kosten:** de onderdelen die niet tot JLC's "basic parts" horen (o.a. de ATM90E32AS, AP2112K, het kristal, de TVS en de 33 nF C0G) kosten een eenmalige instelvergoeding per soort.

## Zelf solderen (through-hole, niet in de JLC-BOM)

| Aantal | Onderdeel | Refs |
|---|---|---|
| 11 | Schroefklem 2-polig 5,08 mm, horizontaal (Phoenix MKDS 1,5/2-5,08 of een kloon) | J1–J9 (CT), J10 (5 V), J14 (7 V AC) |
| 1 | DC-barreljack 5,5/2,1 mm (optioneel, parallel aan J14) | J13 |
| 2 | Female header 1×8, 2,54 mm (voor de SuperMini) | U1 |
| 1 | Pin header 1×4, 2,54 mm | J11 (OLED) |
| 1 | Pin header 1×2, 2,54 mm | J12 (knop) |

Daarnaast heb je nodig:
- de ESP32-C3 SuperMini, het OLED (SSD1306 I²C) en een paneeldrukknop;
- 4× M3-schroeven en wartels;
- optioneel 9× een P4KE6.8CA over de draden van elke spoel.
