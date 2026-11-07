# PowerMeter v2: ESP32-C3 SuperMini + 3× ATM90E32AS

> Projectnaam sinds 7 november 2026: **PowerMeter** (voorheen CurrentMeasurement / Groepenmeter). De KiCad-bestanden heten nu `PowerMeter.kicad_*`; de eigen bibliotheken heten `PowerMeter.kicad_sym` en `PowerMeter.pretty`.

Ontwerp van november 2026. Het vervangt het ontwerp met RPi, ADS1115 en LM358.

**Uitgangspunten** (afgestemd op 6 november 2026):

- **7 groepen**, éénfase. Op één groep zit de **omvormer van de zonnepanelen**.
- ESP32-C3 **SuperMini**, gevoed met een **USB-lader**.
- Een **webpagina op de ESP** zelf, zonder Home Assistant. Daarnaast een **OLED-scherm** (SSD1306 128×64, I²C) en **één drukknop**.
- **Schroefklemmen**, met de burden op het bord. **Alle draden komen van onderen.**
- Behuizing: **Kradex Z74JPH** (176 × 126 × 57 mm, IP65, grijs met doorzichtig deksel), liggend gemonteerd.
- De SMD-kant wordt **bestukt door JLCPCB**. De through-hole-onderdelen soldeer je zelf.

---

## 1. Samenvatting

Er komen geen losse ADC's met opamps meer. Het bord gebruikt **drie energiemeet-IC's, de Microchip ATM90E32AS**.

- Elke chip meet 3 stromen en 3 spanningen **tegelijk** (8 kHz per kanaal) en berekent zelf het werkelijk vermogen (met teken), Irms, Vrms, de cos φ en de energie, apart voor afname en teruglevering.
- 3 chips geven **9 kanalen: 7 groepen + 2 reserve.**
- De ESP32-C3 leest de chips via SPI uit. Hij verzorgt ook de webpagina, het OLED, de knop en de opslag.

| | Oud ontwerp | Nieuw ontwerp |
|---|---|---|
| ADC | 2–3× ADS1115, max. 860 samples/s **gedeeld** over 4 ingangen | 3× ATM90E32AS, elk kanaal **tegelijk** op 8 kHz |
| Werkelijk vermogen | Niet betrouwbaar te berekenen | In hardware, ook voor teruglevering |
| Opamps / bias / AD584 | Nodig | **Niet nodig** |
| Solderen | Alles zelf | SMD door JLCPCB; zelf alleen klemmen en headers |

**Wat blijft:** de **DL-CT08CL5**-spoelen en de **7 V AC-adapter** (alleen nog als meetsignaal voor de spanning).
**Wat niet meer nodig is:** ADS1115, AD584 en LM358.

---

## 2. Waarom het oude PCB niet werkte

Dit blijkt uit `Photos/Back.png` (EAGLE-schema) en `Photos/Front.png` (layout):

1. **Twee ADS1115's op hetzelfde I²C-adres.**
   - Bij de tweede module is ADDR (pin 6) niet aangesloten. Met de pull-down op de module komt die dus ook op 0x48 terecht, hetzelfde adres als de eerste.
   - Twee chips die tegelijk antwoorden is al genoeg om niets zinnigs uit te lezen.

2. **LM358 is niet rail-to-rail.**
   - Op 3,3 V wordt de sinus afgekapt.
   - Op 5 V komen de I²C-lijnen van de Pi op 5 V te staan.

3. **De ADS1115 is te traag en meet niet gelijktijdig.** Daardoor kun je P = gemiddelde(u·i) niet bepalen.

4. **Geen ontkoppelcondensatoren en geen burden-waarden.** De AD584 geeft zonder strapping 10 V in plaats van 2,5 V.

---

## 3. Architectuur

```
            ┌──────────────────────────── PCB 140 × 98 mm, 2 lagen ──────────────────────────────┐
            │  [ESP32-C3 SuperMini] ← antenne naar boven, geen koper eronder                     │
            │        │ SPI + CS1/CS2/CS3        J11 OLED (I²C)   J12 knop                        │
            │        ▼                                                                           │
            │  [ATM90E32 U2]         [ATM90E32 U3]         [ATM90E32 U5]                         │
            │   CT1  CT2  CT3         CT4  CT5  CT6         CT7  CT8* CT9*      (* = reserve)    │
            │   TVS + burden 2×22Ω + RC 1k/33nF per kanaal                                       │
            │  deler 22k/1k + RC → alle V-ingangen         LDO 3V3_A ← +5V ← SS14                │
            │══ J1 … J9 (CT's) ══════════════════ J14+J13 (7V AC) ════════ J10 (5V) ═════════════│
            └────────────────────────────────── alle draden van onderen ─────────────────────────┘
```

- **De 7 V AC-adapter is alleen meetsignaal.** Hij is te zwak om de ESP te voeden, en belasting vervormt de sinus.
- **De voeding komt binnen via schroefklem J10:** een USB-kabel waarvan je de kleine stekker afknipt, met de lader in het stopcontact.
  Dat gaat makkelijk door een kabelwartel. De USB-C van de SuperMini gebruik je alleen om te programmeren (zie 4.1).
- **De meetchips krijgen een eigen, schone 3,3 V** (3V3_A). De ESP gebruikt de LDO op zijn module.
- **Eén ononderbroken massavlak.**

---

## 4. Schema per blok

### 4.1 Voeding

| Ref | Onderdeel | Aansluiting |
|---|---|---|
| J10 | Schroefklem 2p 5,08 mm | `+5V_IN` (rode ader van de USB-kabel), `GND` (zwarte ader) |
| D1 | SS14 (Schottky, SMA) | `+5V_IN` → `+5V`. Beschermt tegen ompolen en tegen terugvoeden |
| C1 | 22 µF 10 V (1206) | `+5V` – GND |
| U1 | ESP32-C3 SuperMini | **5V-pin aan `+5V`**, GND aan GND |
| U4 | AP2112K-3.3 (SOT-23-5) | VIN = EN = `+5V`, VOUT → `3V3_A_RAW` |
| C2, C3 | 10 µF X5R 0805 | op VIN en VOUT van U4 |
| FB1 | Ferriet 600 Ω @ 100 MHz (0805) | `3V3_A_RAW` → `3V3_A` |
| C4, C5 | 10 µF + 100 nF | `3V3_A` – GND |
| LED1 + R | groen + 1 kΩ | `3V3_A` – GND |

- **Verbruik:** 3 × 13 mA voor de ATM's, ~20 mA voor het OLED, plus de ESP (gemiddeld ~80 mA, pieken tot ~350 mA). Een merklader van 5 V/1 A is ruim genoeg.
- **Programmeren:** de SuperMini zit in female headers. De **eerste keer flash je hem los van het bord via USB**, daarna gaat updaten via Wi-Fi (OTA).
  Steek **niet tegelijk** USB in de SuperMini en spanning op J10, want de 5V-pin van de module zit op USB-VBUS.

### 4.2 ESP32-C3 SuperMini en pintoewijzing

Plaats de module **bovenaan het bord, met de antenne naar de bovenrand**. Leg onder de antenne op beide lagen **geen koper** (een keep-out van ~15 × 12 mm), of laat hem over de rand steken.

| Signaal | GPIO | Opmerking |
|---|---|---|
| SCLK | GPIO4 | 33 Ω in serie |
| MISO (← SDO van de ATM's) | GPIO5 | |
| MOSI (→ SDI van de ATM's) | GPIO6 | 33 Ω in serie |
| CS1 (U2: CT1–3) | GPIO7 | 10 kΩ pull-up naar 3V3 |
| CS2 (U3: CT4–6) | GPIO10 | 10 kΩ pull-up naar 3V3 |
| CS3 (U5: CT7–9) | GPIO3 | 10 kΩ pull-up naar 3V3 |
| I²C SDA (OLED) | GPIO20 | footprint voor een 4,7 kΩ pull-up, DNP (de OLED-module heeft er meestal zelf een) |
| I²C SCL (OLED) | GPIO21 | idem |
| Knop | GPIO1 | 10 kΩ pull-up, 1 kΩ in serie, 100 nF naar GND (ontdendering en ESD) |
| reserve (testpad) | GPIO0 | bijv. later een DS18B20-temperatuursensor |
| **niet gebruiken** | GPIO2, GPIO8, GPIO9 | strapping-pinnen |

- GPIO20/21 zijn ook de UART-pinnen. Dat geeft niet, want de SuperMini logt via de ingebouwde USB.
- De ESP-kant (de pull-ups, het OLED en de knop) gebruikt de **3V3 van de module**, niet 3V3_A.

### 4.3 ATM90E32AS (U2, U3, U5): pin voor pin

Gebaseerd op de pinbeschrijving (Table-1) in datasheet Atmel-46003. Controleer dit bij het maken van het KiCad-symbool nog een keer.

| Pin | Naam | Aansluiting |
|---|---|---|
| 1 | AVDD | `3V3_A` + 100 nF |
| 2, 12 | AGND | GND-vlak |
| 3/4, 5/6, 7/8 | I1P/N, I2P/N, I3P/N | CT-kanalen (zie 4.4) |
| 9, 10, 36 | IC | GND |
| 11 | Vref | 4,7 µF + 100 nF naar GND |
| 13, 15, 17 | V1P, V2P, V3P | `VSENSE_P` |
| 14, 16, 18 | V1N, V2N, V3N | `VSENSE_N` |
| 19, 44, 47 | DGND | GND-vlak |
| 20 / 21 | OSCI / OSCO | kristal **16,384 MHz** ±20 ppm, CL 10–20 pF, **zonder** externe load-C's |
| 22–24 | ZX0–ZX2 | open (testpad op ZX0) |
| 25 | CF1 | LED + 1 kΩ naar GND (knippert met het verbruik: handig bij het testen) |
| 26–31 | CF2–CF4, WarnOut, IRQ0, IRQ1 | open (testpad op WarnOut) |
| 32 | TEST | GND |
| 33, 34 | PM0, PM1 | beide aan `3V3_A` (Normal mode) |
| 35, 45, 46 | NC | open |
| 37 | CS | CS1 / CS2 / CS3 |
| 38 / 39 / 40 | SCLK / SDO / SDI | SCLK / MISO / MOSI |
| 41 | RESET | 10 kΩ naar `3V3_A` + 100 nF naar GND |
| 42, 43 | VDD18 | samen, 10 µF naar GND (**niet** aan 3,3 V!) |
| 48 | DVDD | `3V3_A` + 10 µF + 100 nF |

Gebruik voor de SPI 200 kHz – 1 MHz. Het maximum van de chip is ~2 MHz.

### 4.4 Stroomingang (9×) met schroefklemmen

```
                          op de PCB
   Jn.1 ──┬──────┬── Rb1 22Ω ──┬── Rb2 22Ω ──┬────── Jn.2
          │      │             │             │
          │    Dn TVS 5V      GND            │
          │    (bidir., SMA) ──── Jn.1–Jn.2  │
          Ra 1k                              Ra 1k
          ├────► IxP                         ├────► IxN
          Ca 33nF C0G                        Ca 33nF C0G
          │                                  │
         GND                                GND
```

**Burden: 2 × 22 Ω = 44 Ω**, met het middelpunt aan GND.

| Primaire stroom | Over de burden | Opmerking |
|---|---|---|
| 5,5 mA (~1,3 W) | 120 µVrms | ondergrens van de chip |
| 16 A | 352 mVrms (498 mV piek) | ruimte voor crestfactor ~2 |
| 32,7 A | 720 mVrms | maximaal ingangsbereik |

- **Anti-aliasing:** 1 kΩ + 33 nF **C0G** geeft f_c = 4,8 kHz. C0G houdt de faseverschuiving stabiel.

**Veiligheid:**

- **De burden zit direct achter de klem.** Zolang de draden vastzitten, is de CT belast.
- **Een SMD-TVS per kanaal** beschermt de chip.
- **Optioneel: een P4KE6.8CA over de CT-draden** bij de CT, onder krimpkous. Dan is de CT ook veilig als de draad los is.
- **Sluit CT's alleen aan of los met de groep uit**, en gebruik adereindhulzen.

**De reservekanalen CT8 en CT9** worden gewoon volledig bestukt. Je kunt ze later gebruiken voor een extra groep.
Wil je ooit de **hoofdkabel** meten (bijv. 1×35 A of 1×40 A), dan heb je een split-core CT nodig die om de dikkere kabel past. Bij een verhouding van 2000:1 vervang je de burden op dat kanaal door 2 × 15 Ω, en dan reikt het bereik tot ~48 A.

### 4.5 Spanningsingang (één keer)

```
 7V AC-adapter
   ader 1 ── J14.1 ─ Rd1 22k ──┬──────────── VSENSE_P → V1P/V2P/V3P van U2, U3, U5
                               │       │
                             Rd2 1k  Cv 33nF C0G
                               │       │
   ader 2 ── J14.2 ─ GND ──────┴───────┴── GND

   VSENSE_N → V1N/V2N/V3N van U2, U3, U5 ── Rn 1k ── GND,  Cn 33nF C0G naar GND
```

- **Deler 1/23.** Een 7 V-adapter geeft ~8,5–10,5 V, en dat wordt maximaal ~460 mVrms op de ingang. Dat past ruim binnen 720 mVrms.
- **Belasting door de chips:** 9 V-ingangen van 120 kΩ parallel is 13,3 kΩ. Dat geeft ~7 % lagere amplitude, en die kalibreer je weg.
  Het faseverschil met de stroomkanalen is 0,08° bij 50 Hz, en dat is verwaarloosbaar.
- **Zet op VSENSE_P geen extra serieweerstand.**
- **J14 + J13:** een 2-polige schroefklem (J14) met parallel een DC-barreljack 5,5/2,1 mm (J13). Gebruik wat bij je adapter past.

### 4.6 OLED en knop

- **J11, OLED:** een 1×4 header (3V3, GND, SCL, SDA), aangesloten met losse dupont-draadjes.
  De pinvolgorde verschilt per OLED-module (soms staat VCC eerst, soms GND). Met losse draadjes maakt dat niet uit.
  Let op: als je module **7 pinnen** heeft, staat hij standaard op SPI. De 4-pins versie is I²C.
- **J12, knop:** een 2-polige header of schroefklem naar een **paneelmontage-drukknop** in de behuizing (bijv. 12 of 16 mm, IP65/IP67, maakcontact). Het ene contact gaat naar GPIO1, het andere naar GND.

**Wat de knop doet** (dit regelt de firmware):

| Actie | Functie |
|---|---|
| Kort drukken | Het scherm gaat aan, of naar de volgende pagina: totaal → groep 1 … 7 → zonnepanelen → Wi-Fi/IP-adres |
| 5 s ingedrukt | **Wi-Fi-instelmodus.** De ESP wordt een eigen access point ("PowerMeter-setup"). Met je telefoon stel je dan een nieuw Wi-Fi-netwerk in, zonder opnieuw te flashen |
| Ingedrukt tijdens opstarten (10 s) | Fabrieksinstellingen terug (kalibratie blijft bewaard). Dit is optioneel |

- **Het scherm gaat na 60 s uit**, om inbranden van het OLED te voorkomen. De knop maakt het weer wakker.
- **De behuizing is grijs en niet doorzichtig.** Om het OLED te kunnen zien, zaag je een venster in het deksel en lijm je er een stukje plexiglas achter.
  Je kunt ook de **Z74JPH** nemen: dezelfde doos met een doorzichtig deksel. Monteer het OLED tegen het deksel en sluit het met een kabeltje van ~15 cm aan.

---

## 5. Zonnepanelen op één groep

De ATM90E32 meet het vermogen **met teken**. Daarom moeten alle CT's **op dezelfde manier** gemonteerd zijn: dezelfde kant (bijv. de kant met de opdruk) naar de automaat, en de rode draad in de linker klem. Heeft je spoel een pijl (of een P1/K-markering), laat die dan bij alle spoelen dezelfde kant op wijzen.

- **Bij alle groepen is verbruik positief.**
- **Op de omvormergroep is het vermogen negatief zolang de panelen opwekken.** 's Nachts zie je daar een paar watt positief: het stand-by-verbruik van de omvormer.
- **De chip telt afname ("forward") en teruglevering ("reverse") apart.** De firmware bewaart dus per groep twee tellers. Voor de zonnegroep heb je daarmee direct de opgewekte kWh.
- **De omvormer is ~2 kW**, dus ~8,7 A bij 230 V. Dat valt ruim binnen het meetbereik van 32 A, met een goede resolutie.

Wat je op de webpagina kunt tonen:

| Grootheid | Berekening |
|---|---|
| Opwek zon | −P van de omvormergroep (wanneer die negatief is) |
| Verbruik huis | som van de overige 6 groepen |
| Netto van/naar het net | som van alle 7 groepen. Die hoort ongeveer gelijk te zijn aan wat je slimme meter aangeeft: een mooie controle van je kalibratie |
| Eigen verbruik van de zonnestroom | min(opwek, verbruik huis) |

---

## 6. Mechanisch: behuizing Kradex Z74JPH (doorzichtig deksel)

De maten komen uit de tekening van de bodem van de Z74. Coördinaten zijn gemeten vanaf de buitenhoek. De doos ligt **liggend**: de lange zijde (176 mm) is horizontaal, en de kabels komen door de onderste lange zijde.

| Element | Maat / positie |
|---|---|
| Binnenmaat bodem | 164,55 × 119,7 mm (bij de rand 170 × 119,7) |
| Schroeftorens van het deksel, halverwege de lange zijden | lengte 82,1–94,45 mm. Vrije breedte daartussen: **101,9 mm** |
| Grote bussen R7 (diagonaal) | Ø14 mm, hartlijn (13,65 ; 169,6) en (111,45 ; 6,1) |
| **4 binnenste montagebussen** | hart-op-hart **92,0 × 80,1 mm**, gecentreerd in de doos |
| Bussen langs de wand | 2 × 4 stuks, vlak langs de lange wanden. De print blijft daar binnen |

**Printomtrek: 140 × 98 mm.**

- **Breedte 98 mm:** blijft met ~2 mm per kant vrij van de schroeftorens in het midden (101,9 mm vrij).
- **Lengte 140 mm:** blijft ~2 mm vrij van de twee grote R7-bussen.
- **Montage:** 4 gaten van Ø3,2 mm op de binnenste bussen. Ten opzichte van de linkeronderhoek van de print liggen ze op (24,0 ; 8,95), (116,0 ; 8,95), (24,0 ; 89,05) en (116,0 ; 89,05) mm. De ZMB74-montageplaat is dan niet nodig.
- **De onderrand van 140 mm** biedt plaats aan 11 schroefklemmen van 5,08 mm (~112 mm) plus de DC-jack.

**Overig:**

- **Hoogte:** de SuperMini in de female headers komt ~13 mm boven de print uit, en de schroefklemmen ~15 mm. Dat past ruim.
- **OLED:** tegen de binnenkant van het doorzichtige deksel, met een kabeltje van ~15 cm naar J11.
- **Kabels van onderen:**
  - **CT-draden:** bundel ze per 4 in een **UTP-kabel** (4 getwiste paren). Twee UTP-kabels geven 8 paren, en de getwiste paren zijn goed tegen storing.
  - **Wartels:** 4× M16 in de onderkant, voor 2× UTP, de 7 V AC-kabel en de 5 V USB-kabel.
  - IP65 is in een meterkast eigenlijk niet nodig, maar de wartels geven trekontlasting.
- **Wi-Fi:** de ABS-doos laat Wi-Fi goed door. Een metalen meterkastdeur of -kast niet. Bij zwakke ontvangst zet je in de firmware het zendvermogen lager (bij sommige SuperMini-klonen helpt dat juist) of gebruik je een andere plek.

---

## 7. Bestellen bij JLCPCB

> **De bestelbestanden staan klaar in `Productie/`** (7 november 2026): `PowerMeter_v2_Gerbers.zip`, `JLCPCB_BOM.csv` en `JLCPCB_CPL.csv`. De instellingen en controlepunten staan in `Productie/LEESMIJ_bestellen.md`. Alle SMD-onderdelen hebben nu een gecontroleerd LCSC-nummer, ook in het schema en de print.

- **Wie wat doet:** JLCPCB plaatst alle SMD-onderdelen. Jij soldeert de schroefklemmen, de DC-jack, de headers en de female headers voor de SuperMini.
- **Bestanden:** Gerbers, een BOM en een CPL uit KiCad. Elk onderdeel krijgt een veld `LCSC`.
- **De ATM90E32AS-AU-R heeft LCSC-nummer [C177153](https://www.lcsc.com/product-detail/C177153.html)** (~$3,20 per stuk). Er zijn er 3 per bord nodig. Controleer de voorraad vlak voor het bestellen.
- **Controleer in de JLC-preview de rotatie** van de ATM's, de LDO, de TVS'en en de diode.
- **Prijs:** een print groter dan 100 × 100 mm is iets duurder, maar blijft bij 5 stuks goedkoop. JLC bestukt er minimaal 2.

### Stuklijst

| Aantal | Waarde | Footprint | Wie | Functie |
|---|---|---|---|---|
| 3 | ATM90E32AS-AU-R | TQFP-48 7×7 | JLC | U2, U3, U5 |
| 3 | Kristal 16,384 MHz ±20 ppm, CL 10–20 pF | SMD 3225 | JLC | Y1–Y3 |
| 1 | AP2112K-3.3 | SOT-23-5 | JLC | U4 |
| 1 | SS14 | SMA | JLC | D1 |
| 1 | Ferriet 600 Ω@100 MHz | 0805 | JLC | FB1 |
| 9 | TVS bidir. 5 V (SMAJ5.0CA) | SMA | JLC | D4–D6, D8–D10, D12–D14 |
| 18 | 22 Ω 1 % | 0805 | JLC | burden |
| 20 | 1 kΩ 1 % | 0805 | JLC | 18× RC stroom, Rd2, Rn |
| 1 | 22 kΩ 1 % | 0805 | JLC | Rd1 |
| 20 | 33 nF **C0G/NP0** 50 V | 0805 | JLC | 18× stroom, Cv, Cn |
| 9 | 10 µF X5R 10 V | 0805 | JLC | DVDD, VDD18 (2 per chip), LDO in/uit, 3V3_A |
| 3 | 4,7 µF X5R | 0805 | JLC | Vref |
| 14 | 100 nF X7R | 0603 | JLC | AVDD, DVDD, Vref, RESET (4 per chip), 3V3_A, knop |
| 1 | 22 µF 10 V | 1206 | JLC | +5V |
| 7 | 10 kΩ | 0603 | JLC | RESET (3), CS (3), knop (1) |
| 2 | 4,7 kΩ (DNP) | 0603 | — | I²C pull-ups, alleen indien nodig |
| 2 | 33 Ω | 0603 | JLC | SCLK, MOSI |
| 5 | 1 kΩ | 0603 | JLC | power-LED, 3× CF1-LED, knop-serie |
| 4 | LED groen | 0805 | JLC | power + 3× CF1 |
| 2 | Female header 1×8, 2,54 mm | THT | zelf | SuperMini |
| 11 | Schroefklem 2p 5,08 mm | THT | zelf | J1–J9 (CT), J14 (7V AC), J10 (5V) |
| 1 | DC-jack 5,5/2,1 | THT | zelf | J13 (parallel aan J14) |
| 1 | Header 1×4 + 1×2, 2,54 mm | THT | zelf | J11 OLED, J12 knop |
| 1 | Paneeldrukknop, maakcontact, IP65 | — | zelf | knop |
| 4 | Wartel M16 | — | zelf | kabeldoorvoer |
| (9) | P4KE6.8CA (optioneel) | DO-41 | zelf | over de CT-draden |
| — | Testpunten | | | +5V, 3V3_A, GND, VDD18 + Vref per chip, SCLK, MISO, MOSI, CS1–3 |

---

## 8. Layout-regels

1. **2 lagen. De onderlaag is een ononderbroken GND-vlak.** Leg sporen bovenop.
2. **Indeling** (zie de schets in 3):
   - **Onderrand:** de CT-klemmen J1–J9, rond de twee montagegaten heen. J10 (5 V), J14 (7 V AC) en J13 (jack) zitten op de linkerrand, omdat de montagegaten geen ruimte lieten voor 11 klemmen op één rij.
   - **Daarboven:** per kanaal de TVS, de burden en de RC, met **per chip een eigen "strook" van 3 kanalen**.
   - **Midden:** U2, U3 en U5, elk boven zijn eigen 3 kanalen.
   - **Boven:** de LDO, de SuperMini (met de antenne naar de bovenrand en de keep-out), J11, J12 en de testpunten.
3. **Houd IxP/IxN als paar naast elkaar** en even lang. Houd de route van de klem naar het filter kort.
4. **Ontkoppel-C's** binnen 2 mm van de pin, met een via direct naar het vlak.
5. **Kristal tegen pin 20/21**, met GND eromheen en geen signalen eronder.
6. **SPI langs de bovenkant van de chips** (pinnen 37–40), weg van de analoge ingangen.
7. **4 montagegaten M3**, ~4 mm uit de hoeken.
8. **Silkscreen:**
   - de namen bij de klemmen (CT1 … CT9, 7V AC, 5V +/−);
   - de montageregel voor de CT's (zelfde kant naar de automaat, rode draad links);
   - welke klem de zonnegroep is, leeg te laten om met de hand in te vullen.

---

## 9. Vóór het bestellen: de controlelijst

- [ ] **ERC zonder fouten.** Alle open pinnen hebben een no-connect-vlag.
- [ ] **Elke chip-pin nagelopen tegen tabel 4.3** (VDD18 ≠ 3,3 V, PM0/PM1 hoog, TEST/IC aan GND).
- [ ] **De drie CS-lijnen** gaan elk naar één chip.
- [x] **Footprint van de SuperMini gecontroleerd tegen de echte module** (bovenaanzicht = gespiegeld ten opzichte van de opdruk op de onderkant van de module).
- [ ] **Layout 1:1 geprint**, met de klemmen en de SuperMini erop gelegd. Gecontroleerd tegen de **gemeten binnenmaat** van de Z74.
- [ ] **DRC zonder fouten** met de regels van JLCPCB.
- [ ] **Gerbers bekeken en de BOM/CPL-preview van JLC gecontroleerd** (rotatie en pin 1).

### KiCad-bestanden (stand 7 november 2026)

| Bestand | Inhoud |
|---|---|
| `PowerMeter.kicad_sch` | Het complete schema (A2). ERC: 0 fouten, 0 waarschuwingen |
| `PowerMeter.kicad_sym` | Eigen symbolen: **ATM90E32AS** (pinnen volgens datasheet-tabel 1) en **ESP32-C3_SuperMini** |
| `PowerMeter.pretty/ESP32-C3_SuperMini_Socket` | Footprint voor de SuperMini in 2× 1×8 female headers (rijen 15,24 mm uit elkaar), met de pinnamen op de silkscreen en een antenne-keep-out |
| `sym-lib-table`, `fp-lib-table` | Koppelen de eigen bibliotheken aan het project |
| `PowerMeter.kicad_pcb` | De print met alle footprints geplaatst en alle netten toegekend (zie hieronder). **Gerouteerd**: kritieke analoge sporen vergrendeld, de rest met Freerouting |
| `PowerMeter_v2_schema.pdf`, `PowerMeter_v2_BOM.csv`, `PowerMeter_v2_PCB_render.png` | Een export van het schema, de stuklijst en een 3D-bovenaanzicht van de print |

**Nog te doen:**

1. ~~Pinvolgorde van de SuperMini controleren~~ **Gecontroleerd op 7 november 2026.**
   - De pinnamen staan op de **onderkant** van de module.
   - KiCad tekent footprints in **bovenaanzicht**: de module zit erin met de chip en antenne naar boven.
   - Het footprint is daarom het spiegelbeeld van wat je op de module leest. Met de USB-C naar boven staat de rij `5V, GND, 3V3, IO4 … IO0` **rechts** en de rij `IO5 … IO21` **links**.
2. **LCSC-nummers invullen** (veld `LCSC`). Alleen de ATM90E32AS (C177153) staat er al in. De rest kun je bij het uploaden van de BOM door JLC laten koppelen.
3. **PCB-layout:** de plaatsing staat (zie hieronder). De sporen moeten nog getrokken worden.

### PCB: wat er al staat

- **Omtrek en montage:** de omtrek is 140 × 98 mm met hoeken van r = 2 mm. De 4 M3-gaten zitten op de binnenste bussen van de Z74.
- **Onderrand** (draadopening naar beneden):
  - J1 = CT1 links van het linker gat;
  - J2–J8 = CT2–CT8 tussen de twee onderste montagegaten;
  - J9 = CT9 rechts van het rechter gat.
  - CT1 en CT9 staan symmetrisch (allebei ~8 mm van de zijkant).
- **Linkerrand** (draadopening naar links): J10 = 5 V, J14 = 7 V AC, J13 = DC-jack.
  - De jack zit ~5 mm boven het CT1-paar.
  - De binnenkant van de doos laat ~12 mm vrij naast de print. Gebruik voor de jack dus een haakse plug, of sluit de adapter via J14 aan.
- **Silkscreen (Engels):**
  - boven de CT-rij *"WARNING: switch OFF the circuit breaker before connecting or disconnecting a CT"*;
  - en *"Mount all CTs the same way round (same side towards the breaker), red wire in the LEFT terminal"*.
  - Waarom: de chip meet het vermogen met teken. Een omgekeerde spoel (of omgewisselde draden) geeft een negatief vermogen. Wissel dan de twee draden van die spoel om, of draai het teken om in de firmware.
  - Verder: *5V IN (+ this side)* boven J10, *7V AC IN* direct boven J14, *OLED*, *BUTTON*.
  - De pinnamen van de SuperMini staan gecentreerd tussen de pinrijen, dus niet over de gaten.
- **Per CT-kanaal**, recht boven de klem: de TVS, de burden 2×22 Ω (met de GND-pads naast elkaar, zodat daar één via naar het GND-vlak kan), daarboven 2× 1 kΩ en 2× 33 nF.
- **Midden:** de drie ATM90E32AS, **90° gedraaid**, zodat de stroomingangen (pinnen 3–8) naar beneden wijzen, naar de CT-kanalen.
  - De spanningsingangen en het kristal zitten rechts, de SPI, de reset en VDD18 links, en CF/IRQ/PM boven.
  - Rondom zitten de ontkoppel-C's, het kristal, de reset-RC, de CF1-LED en 4 testpunten.
- **Boven:**
  - links de voeding (D1, U4, FB1, C's);
  - de OLED- en knop-header;
  - de SuperMini (180° gedraaid: de antenne steekt ~0,4 mm over de bovenrand en de USB-C wijst naar binnen);
  - de systeemtestpunten.
- **GND-vlak op beide lagen.** SMD-pads zijn massief verbonden en THT-pads met thermal-spaken.
- **Regels (JLCPCB):**
  - spoor en afstand minimaal 0,15 mm;
  - via 0,6/0,3 mm;
  - koper minimaal 0,3 mm van de rand;
  - netklasse **Power** (GND, +5V, +5V_IN, +3V3, +3.3VA, 3V3A_RAW) met sporen van 0,5 mm; standaard 0,25 mm.
- **DRC:** 0 fouten in de koppeling met het schema. Er zijn alleen nog "niet verbonden"-meldingen (de sporen ontbreken) en cosmetische silkscreen-overlappingen.

### Vooraf gerouteerd en vergrendeld (7 november 2026)

De eerste Freerouting-run (7 november 2026) verbond bijna alles, maar met twee problemen:
- **CT-paren:** de CT-kanalen liepen niet als paar.
- **GND-pinnen:** 28 GND-pinnen van de meetchips bleven los, omdat het GND-vlak niet tussen de fijne TQFP-pinnen komt.

Daarom zijn de kritieke sporen nu met een script gelegd en **vergrendeld** (locked), zodat Freerouting ze laat staan:

- **Per CT-kanaal:**
  - A en B van de klem via de TVS naar de burden en de 1 kΩ (0,4 mm);
  - een via van het burden-middelpunt naar GND, en een GND-via bij elke 33 nF;
  - **CTn_P/CTn_N als paar**: 0,25 mm sporen, 0,5 mm hart-op-hart (precies de pinafstand). Het paar loopt van de filter-C's via een eigen "baan" (y = 108,5 of 112,0 mm; 3,5 mm tussen de banen) naar de pinnen 3–8. De paren kruisen elkaar nergens.
    - Waar twee paren een stukje parallel lopen, is dat bij 50 Hz en een bron van 22 Ω elektrisch geen probleem; de extra afstand is alleen marge.
- **Per chip:**
  - alle GND-pinnen (2, 9, 10, 12, 19, 32, 36, 44, 47) via korte sporen naar **6 GND-vias onder de chip**;
  - de spanningsingangen: V1P/V2P/V3P aan de buitenkant aan elkaar met een via, en V1N/V2N/V3N onder de chip aan elkaar met een via.
- **GND-vias** bij elke ontkoppel-C, het kristal en de CF1-LED. De C's zijn zo gedraaid dat de voedingspad naar de chip wijst en de GND-pad naar buiten.
- **VSENSE_P en VSENSE_N** lopen als twee sporen naast elkaar op de onderlaag, van chip 1 via chip 2 naar chip 3, **onder** alle GND-vias van de chips door.
  - In een eerdere versie liepen ze boven en onder die vias. Daardoor sneden ze een strook uit het GND-vlak, en hingen enkele GND-pinnen los.

DRC na deze stap: 0 clearance-fouten, 0 fouten in de koppeling met het schema, alle CT- en GND-netten volledig verbonden. Zie `PowerMeter_v2_PCB_koper.png`.

**Stand na Freerouting (7 november 2026):** alle netten zijn gerouteerd.
- Daarna nog toegevoegd: een GND-via bij GND-pin 2 van de SuperMini, 3 GND-vias in losse eilandjes van het GND-vlak op de bovenlaag, en bij twee vias het gat verkleind van 0,4 naar 0,3 mm (anders was de koperring te smal).
- DRC: **0 onverbonden, 0 elektrische fouten, 0 fouten in de koppeling met het schema.** Er resteren alleen silkscreen-waarschuwingen (referentieletters van kleine R's en C's over pads; JLCPCB knipt die weg).

### Routeren: wat Freerouting nog moet doen

1. **Draai Freerouting opnieuw.** De vergrendelde sporen blijven staan; hij routeert alleen de rest:
   - de voeding (+5V, +3V3, +3.3VA, 3V3A_RAW);
   - de SPI en CS1–3;
   - de ontkoppel-C's, de reset, de kristallen, de LED's, de testpunten, de OLED en de knop;
   - het korte stukje VSENSE van de deler naar chip 1.
2. **Controleer daarna in KiCad:**
   - **Kristal:** de sporen moeten kort zijn;
   - **Ontkoppeling:** de C's moeten direct aan hun pin zitten;
   - **Antenne:** onder de SuperMini-antenne mag geen koper liggen.
   Herroute met de hand wat er rommelig uitziet.
3. **Ontkoppeling:** van elke C een kort spoor naar zijn pin, met een via naar GND direct bij de C.
4. **Kristal:** sporen zo kort mogelijk, met GND eromheen.
5. **SPI (SCLK/MOSI/MISO/CS1–3)** van de SuperMini naar de bovenkant van de chips (pinnen 37–40), bij voorkeur langs de bovenlaag.
6. **Voeding:**
   - +3.3VA als een stervormige of brede baan naar de drie chips;
   - +5V en +3V3 naar de SuperMini en de headers.
7. **Tot slot:** zones opnieuw vullen (B), DRC draaien, en de silkscreen-referenties opschuiven waar ze over pads vallen.

## 10. Opstartplan

1. **Zonder ESP, 5 V op J10.**
   - Meet 3V3_A ≈ 3,3 V.
   - Meet per chip **VDD18 ≈ 1,8 V** en **Vref ≈ 1,2 V**.
2. **Flash de SuperMini los via USB** (Wi-Fi-gegevens erin), en plaats hem dan op het bord.
3. **Controleer in de log of de webpagina dat alle 3 de chips antwoorden.**
4. **AC-adapter erbij.** Kalibreer de spanning met een multimeter.
5. **Eén CT om een bekende belasting** (waterkoker). Is het vermogen negatief? Draai de CT om. Kalibreer daarna de stroom.
6. **De overige CT's.** Controleer tot slot overdag of de som van alle groepen klopt met de slimme meter.

---

## 11. Firmware

### Fase 1: hardware testen met ESPHome (zonder Home Assistant)

Dit is bewezen code met een ingebouwde webpagina. Werkt het niet, dan ligt het aan de hardware.

```yaml
esphome:
  name: powermeter
esp32:
  board: esp32-c3-devkitm-1
  framework: { type: esp-idf }
logger:
ota:
  - platform: esphome
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  output_power: 8.5dB   # sommige SuperMini-klonen hebben slechte Wi-Fi op vol vermogen
web_server:
  port: 80
  version: 3            # klik op een sensor voor een grafiek van het verloop

spi:
  clk_pin: GPIO4
  miso_pin: GPIO5
  mosi_pin: GPIO6

sensor:
  - platform: atm90e32
    id: chip1
    cs_pin: GPIO7
    line_frequency: 50Hz
    gain_pga: 1X
    update_interval: 5s
    phase_a: { voltage: { name: "Netspanning" }, current: { name: "Groep 1 stroom" }, power: { name: "Groep 1 vermogen" }, gain_voltage: 7305, gain_ct: 27961 }
    phase_b: { current: { name: "Groep 2 stroom" }, power: { name: "Groep 2 vermogen" }, gain_voltage: 7305, gain_ct: 27961 }
    phase_c: { current: { name: "Groep 3 stroom" }, power: { name: "Groep 3 vermogen" }, gain_voltage: 7305, gain_ct: 27961 }
    frequency: { name: "Netfrequentie" }
  - platform: atm90e32
    id: chip2
    cs_pin: GPIO10
    line_frequency: 50Hz
    gain_pga: 1X
    update_interval: 5s
    phase_a: { current: { name: "Groep 4 stroom" }, power: { name: "Groep 4 vermogen" }, gain_voltage: 7305, gain_ct: 27961 }
    phase_b: { current: { name: "Groep 5 stroom" }, power: { name: "Groep 5 vermogen" }, gain_voltage: 7305, gain_ct: 27961 }
    phase_c: { current: { name: "Groep 6 stroom" }, power: { name: "Groep 6 vermogen" }, gain_voltage: 7305, gain_ct: 27961 }
  - platform: atm90e32
    id: chip3
    cs_pin: GPIO3
    line_frequency: 50Hz
    gain_pga: 1X
    update_interval: 5s
    phase_a: { current: { name: "Groep 7 stroom" }, power: { name: "Groep 7 vermogen" }, gain_voltage: 7305, gain_ct: 27961 }
```

Kalibreren gaat zo: nieuw = (referentiewaarde / gemeten waarde) × oud.

### Fase 2: eigen firmware (PlatformIO, Arduino-framework)

- **Chipdriver:** de [CircuitSetup ATM90E32-library](https://github.com/CircuitSetup/ATM90E32), voor 3 instanties (CS1–CS3).
- **Webpagina** uit LittleFS:
  - live vermogens per groep;
  - opwek, verbruik en netto (zie 5);
  - grafieken per dag, week en maand (Chart.js);
  - een kalibratiepagina.
- **JSON-API:** `/api/now`, `/api/history`.
- **Opslag:** per groep de afname- en teruglevertellers per kwartier, in LittleFS. Dat past ruim in de 4 MB flash.
- **OLED en knop:** de pagina's, de time-out en de Wi-Fi-instelmodus zoals in 4.6. De tijd komt via NTP.

---

## 12. Veiligheid in de meterkast

- Het bord werkt helemaal op veilige laagspanning (7 V AC en 5 V DC). Er komt geen netspanning op de PCB.
- De DL-CT08CL5 is een **gesloten** ringkern. Om hem te plaatsen moet de groepsdraad los van de automaat, dus de **hoofdschakelaar moet uit**.
  Laat het doen door een installateur als je twijfelt.
- Meet per groep **alleen de fasedraad (L)**.
- **Zet de USB-lader en de 7 V-adapter in een gewoon stopcontact**, niet los in de groepenkast.
