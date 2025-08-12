# doel

bouw een midi-controller die een fatar-klavier (velocity met 2 contacten per toets) en 9 potmeters uitleest en vertaalt naar midi (din en/of usb).

---

# uitgangspunten & eisen

* **platform**: ch32v003 (risc‑v, 3.3 v, beperkte i/o en ram/flash).
* **invoer**:

  * fatar-toetsenbord, 61 toetsen als voorbeeld (schaalbaar naar 49/76/88), **2 schakelaars per toets** → 122 contacten totaal.
  * **9 potmeters** (lineair, 10 kΩ aangeraden).
* **uitvoer**: midi note on/off met velocity, cc voor potmeters.
* **latency**: waarneembaar < 5 ms, doelgemiddelde < 2 ms.
* **jitter**: zo laag mogelijk; vaste scansnelheid.
* **voeding**: 5 v in, 3.3 v mcu-rail, galvanische scheiding voor midi in (optioneel) conform midi-spec.

---

# architectuuroverzicht

1. **keyboard-scan subsystem**

   * matrix- of shiftregister-gebaseerd scannen van 2× zoveel contacten als toetsen.
   * per toets een kleine state-machine om velocity te bepalen uit *tijd tussen contact a en b*.
2. **analog read subsystem**

   * 9 potmeters via 74hc4051 analoge multiplexer(s) naar de adc van de ch32v003.
   * signaalconditioning: rc-filter + eventueel buffer opamp.
3. **midi i/o**

   * **din-midi out** via uart @ 31.250 bps.
   * **usb-midi**: niet native op ch32v003 → alternatief mcu met usb (zie opties) of extern usb‑uart‑midi bridge.
4. **firmware**

   * vaste tijdslot-planner (tick), key-state machines, debouncing, velocity mapping, adc-scheduler, midi-queue met interruptgedreven uart.

---

# keuze: i/o-uitbreiding voor het klavier

ch32v003 heeft te weinig directe i/o voor 122 contacten. twee robuuste opties:

## optie a — shiftregisters (aanrader voor snelheid/prijs)

* **uit** (rij-aansturing): 74hc595 (serieel→parallel), ketenbaar. 3 lijnen vanaf mcu (spi: sck, mosi, latch).
* **in** (kolommen lezen): 74hc165 (parallel→serieel). 3 lijnen (zelfde clk + mosi/miso en /load). vaak kun je klok/latch delen → totaal 3–4 mcu‑pinnen.
* schaalbaar naar honderden inputs met minimale mcu-pinnen.
* snelheid: zeer hoog; volledige scan < 1 ms mogelijk met paar ketens.

## optie b — i2c-gpio-expanders

* **mcp23017** (16 i/o per chip) of **pcf8574** (8 i/o).
* voordeel: simpele bedrading (i2c), adressering; nadeel: lagere snelheid en i2c‑latency → meer jitter.
* mcu‑pinnen: 2 (sda, scl) + evt. int-lijnen.

**advies**: optie a met 595/165 voor strakke timing en lage kosten.

---

# matrixindeling (voorbeeld 61 toetsen, 122 contacten)

* maak **2 gescheiden matrices**: een voor contact **A** en een voor **B** (velocity).
* voorbeeld: 8 rijen × 8 kolommen per matrix → 64 posities; voor 61 toetsen past dit; je gebruikt 2 identieke ketens (a en b). totaal 16 "rijselects" en 16 "kolomsense" equivalenten, maar via shiftregisters teruggebracht naar \~4 mcu‑lijnen.
* **diodes** per schakelaar vermijden ghosting (1n4148).
* opmerking: sommige fatar-velden gebruiken “make‑make” (beide sluiten bij indrukken) met offset; release is omgekeerde volgorde. de firmware moet beide richtingen correct afhandelen.

---

# velocitymeting

* principe: tijd Δt tussen **A closes** en **B closes**.
* hogere aanslagsnelheid ⇒ kleinere Δt ⇒ hogere velocity (1–127).
* implementatie:

  1. bij detectie **A=closed**: `t_start[note] = systick_us(); state = waiting_B`.
  2. bij **B=closed** vóór timeout: `dt = systick_us() - t_start`; map naar velocity via **curvetabel** (log, exp, lineair, of gebruikersprofielen).
  3. **debounce**: 300–1000 µs digitaal (geen mechanische condensatoren nodig in matrix) + vereenvoudigde edge-detectie.
  4. **release**: detecteer **B opens** dan **A opens**; stuur note off (met last velocity of fixed 64) na threshold of bij echte open‑detectie.
* **timerresolutie**: 1–10 µs ideaal. gebruik een hardwaretimer vrijlopend op bijv. 1 mhz.

---

# analoge potmeters (9x)

* **mux**: 74hc4051 (8:1) + een tweede kanaal voor de 9e pot (bijv. direct op adc of tweede 4051, of 2×4051 → 16 kanalen “future proof”).
* **selectlijnen**: s0–s2 delen beide 4051’s; per 4051 een **enable** lijn (twee enable-lijnen totaal) of via analoge oring.
* **impedantie**: adc van ch32v003 verwacht lage bronimpedantie; zet **serieweerstand \~100 Ω** + **c\_samp 100 nF** aan adc-pin per kanaal (sample/hold buffer) of een **opamp buffer** (rail-to-rail, bijv. mcp6002) tussen mux en adc.
* **ratiometrisch**: voed potmeters tussen 3.3 v en gnd; meet adc referentie = vdda = 3.3 v; zo vallen supplyvariaties weg.
* **filtering & dither**: simpele iir (exponentiële moving average) en **deadband** ±1–2 lsb om midi-ruis te voorkomen.

---

# midi i/o opties

* **din midi out** (aanrader op ch32v003):

  * uart @ 31.250 bps, 8n1. uitgangsdriver volgens midi-spec: `uart_tx → 220 Ω → pin 5`, `+3.3/5 v via 220 Ω → pin 4`, pin 2 = gnd. (controleer actuele spec; 5 v is gebruikelijk, 3.3 v werkt vaak maar 5 v-niveau via transistor kan robuuster zijn.)
* **midi in** (optioneel):

  * 5‑pin din via **6n138/6n137 optocoupler**, pull‑up op uitgang, schmitt-trigger of uart rx direct (met passende weerstanden).
* **usb‑midi**:

  * ch32v003 heeft geen native usb device. opties:

    1. kies mcu met usb device (ch32v203, stm32g0/g4, rp2040, atmega32u4, nrf52) → direct usb‑midi class compliant.
    2. externe usb‑uart bridge die **midi class** emuleert is ongebruikelijk; eenvoudiger is mcu‑swap als usb‑midi gewenst is.

**praktisch advies**: wil je usb‑midi naar linux 22.04, overweeg **rp2040** (ruim i/o, 2×core, pico-sdk/tinyusb) of **stm32g0**. wil je bij ch32v003 blijven: gebruik **din-midi** en evt. een externe usb‑midi interface naar je pc.

---

# pinbudget (ch32v003, indicatief)

* spi/shiftregisters voor keyboard: **3–4 pins** (sck, mosi, miso, latch/\~load gedeeld).
* mux select s0–s2: **3 pins**.
* mux enable(s): **1–2 pins**.
* adc input: **1 pin**.
* uart tx (din out): **1 pin** (rx optioneel +1, midi in opto vereist nog 1 pin).
* status-led: **1 pin**.
* totale mcu‑pins nodig: **\~10–13** → **haalbaar** op ch32v003.

---

# firmwareontwerp

## taken & timing

* **systick**: 1 khz (1 ms) voor housekeeping; **high‑res timer** 1 mhz voor velocity timestamps.
* **scanloop** (loopt in vaste cadence, bv. 1–2 khz):

  1. shift nieuwe rijpatroon naar 74hc595.
  2. kleine settle‑tijd (0.5–2 µs), dan 74hc165 parallel‑load, daarna bitshift in.
  3. vergelijk met vorige scan → detecteer edges per contact.
  4. update key state machines; bereken velocity bij A→B.
  5. queue midi events.
* **adc-scheduler**: elke tick volgende mux‑kanaal, lees n samples, low‑pass, stuur **cc#** (bijv. cc#1 mod, cc#11 expression, etc.) met **rate limiting** (alleen bij verandering > threshold of elke 10–20 ms).
* **midi-queue**: ringbuffer; uart tx irq‑gedreven om blocking te vermijden.
* **config/profiles**: velocity‑curves, key ranges, cc mapping; opslaan in flash/eeprom‑emulatie.

## key state machine (per toets)

* states: `idle → a_closed → waiting_b → note_on → waiting_release → idle`.
* timeouts: max Δt (bijv. 20–30 ms); bij timeout fallback velocity (zachte aanslag) of negeren.
* release: detecteer b\_open dan a\_open → stuur note off.

## debouncing

* digital: vereis **n** opeenvolgende gelijke scans (n=2–4 bij 1–2 khz) voor een stabiele edge.
* hardware: houd matrixlijnen strak en kort; pull‑ups/down op sense-lijnen (10–47 kΩ), serieweerstanden (100–220 Ω) om ringing te dempen.

---

# hardwareblokken

1. **voeding**

   * 5 v in (usb of dc‑jack) → **buck** naar 3.3 v (mp1584 of ldo als thermiek oké).
   * aparte 5 v rail voor midi/din en evt. led/backlight; verbind gnd’s op één sterpunt.
2. **keyboard-matrix**

   * 74hc595 keten (rij drive) + diodes naar rijen; 74hc165 keten (kolom sense) met pull‑ups.
3. **potmeter-mux**

   * 2× 74hc4051, s0–s2 gedeeld, enable per chip, buffer naar adc.
4. **midi din**

   * out: transistor/optodriver-schematiek conform spec.
   * in (optioneel): 6n138 + uart rx.
5. **debug/programmeren**

   * swd/jtag of wch‑link (risc‑v) headers; uart log op 115200 voor debug.

---

# alternatieve mcu’s (als usb‑midi of meer marge gewenst)

* **rp2040 (raspberry pi pico)**: 2× core, veel gpio, adc ok; **tinyusb** usb‑midi werkt uitstekend.
* **stm32g0/g4/f103**: ruime timers, adc, usb fs (op sommige varianten).
* **nrf52840**: ble midi + usb‑midi; meer complex.
* **atmega32u4**: bewezen in midi-controllers; beperkter maar usb‑midi class‑compliant.

**kansrijke route** als je bij wch wilt blijven: **ch32v203** (usb device fs) + zelfde 74hc595/165/4051 architectuur.

---

# midi-berichtenmapping (voorstel)

* **notekanalen**: kanaal 1.
* **note on/off**: standaard; velocity uit Δt.
* **potmeters**: map naar cc’s (bijv. p1→cc1, p2→cc11, p3→cc74, p4→cc71, p5→cc73, p6→cc10, p7→cc7, p8→cc91, p9→cc93). schaal 0–1023 → 0–127 met hysteresis.
* **startup**: all notes off, reset controllers (cc121), set omni off/poly mode indien gewenst.

---

# linux 22.04 workflow

* toolchain: **riscv-none-elf-gcc** voor ch32v003 + **wch‑link** tools (open‑source varianten bestaan) of **openocd** indien ondersteund.
* midi‑testen: gebruik **aconnect/aseqdump** (als usb‑midi via alternatief mcu) of externe usb‑midi interface voor din.
* logic analyzer (saleae/clone + sigrok/pulseview) om scan‑timing en contactedges te valideren.

---

# test- & kalibratieplan

1. **scan-timing**: meet volledige matrixscan; streef < 1 ms.
2. **debounce-validatie**: herhaal toetsaanslagen, controleer dubbele triggers.
3. **velocity‑curves**: neem sample van Δt bij verschillende speelstijlen; bouw 3–5 presets + user curve.
4. **adc‑ruis**: meet lsb‑jitter met stilstaande pot; pas rc/ema/oversampling aan.
5. **midi‑throughput**: worst‑case clusters (glissando + modwheel sweep) → geen buffer overruns.

---

# bom (indicatief)

* mcu: ch32v003 (tssop20 of qfn20) + header.
* 74hc595 × 2–4 (afhankelijk van rijen).
* 74hc165 × 2–4 (afhankelijk van kolommen).
* 1n4148 diodes per toetscontact.
* 74hc4051 × 2.
* opamp mcp6002 (buffer adc) + r/c netwerk.
* optocoupler 6n138 (midi in, optioneel), transistor + weerstanden (midi out).
* passives, headers naar fatar flatcables, 3.3 v regulator, 5 v buck, connectors.

---

# risico’s & mitigatie

* **usb‑midi ontbreekt** op ch32v003 → kies din of andere mcu.
* **contactruis & ghosting** → diodes, degelijke debounce, nette routing.
* **adc‑lek via mux** → buffer na mux, sample‑hold c nabij adc‑pin.
* **scan‑jitter** → timer‑gedreven scan en korte isr’s; geen blocking.

---

# next steps

1. kies **i/o‑strategie** (aanrader: 74hc595/165) en maak een **pinout‑schets**.
2. teken **blokdiagram + schema** (kicad) met één octaaf als proefopstelling.
3. schrijf **firmware skeleton**: timers, spi‑shift, matrix‑scan, midi uart, adc‑scheduler.
4. bouw **velocity‑curve** en **pot‑filtering**; maak config via sysex of compile‑time.
5. breid uit naar volledige toetsbed + 9 potten; validatie & kalibratie.

---

# uitbreidingen (later)

* aftertouch (individueel vereist druksensoren) of channel aftertouch (fader/ffs sensor).
* pedalen (sustain, expression) op extra adc/gpio.
* oled/lcd + encoders voor live-config.
* ble‑midi (andere mcu).

Je hebt gelijk: de CH32V003 (TSSOP-20) heeft **geen SWCLK**. Programmeren/debuggen gaat via **één draad: SWIO** (propriëtair, niet ARM-SWD).

## Wat werkt (en wat niet)

* **Aanrader**: **WCH-LinkE** (let op de **E**). Die spreekt de 1-wire **SWIO/SDI** interface van de CH32V003. Aansluiten: **3V3**, **GND**, **SWIO (PD1)**, en optioneel **NRST**. ([GitHub][1], [Zephyr Project Documentation][2], [element14 Community][3])
* **Raspberry Pi Debug Probe**: **nee** — dat is CMSIS-DAP voor ARM-**SWD** (2-wire), **niet** compatibel met WCH-SWIO. ([Raspberry Pi][4])
* **Tigard (FT2232H)**: ook **nee** voor SWIO. Tigard kan JTAG/SWD/I²C/SPI e.d., maar WCH’s **single-wire** protocol niet. (Handig wél voor UART/logic-analyzer tijdens debug.) ([Crowd Supply][5])

## OpenOCD + GDB met CH32V003

1. **Gebruik de WCH-fork van OpenOCD** (of een fork die **wlink** ondersteunt). Installeer/bouw bijv.:

   * `openwch/openocd_wch` (officiële WCH-fork), of
   * `kprasadvnsi/riscv-openocd-wch` (mirror). ([GitHub][6])
2. **Zet WCH-LinkE in RISC-V-modus** (lang ModeS indrukken tijdens inpluggen of met WCH-LinkUtility). ([Olimex][7])
3. **Start OpenOCD** met de meegeleverde cfg’s (namen verschillen per fork, maar typisch zoiets als):

   ```bash
   openocd -f interface/wlink.cfg \
           -f target/wch-riscv.cfg \
           -f target/ch32v003.cfg
   ```

   (Als er geen aparte `ch32v003.cfg` is, volstaat vaak alleen `wch-riscv.cfg` van die fork — check de repo’s scripts-map.) ([GitHub][6])
4. **Koppel GDB** vanaf Ubuntu 22.04:

   ```bash
   riscv-none-elf-gdb build/firmware.elf
   (gdb) target remote :3333
   (gdb) monitor reset halt
   (gdb) load
   (gdb) continue
   ```

   (Werkt met de WCH-OpenOCD backend via WCH-LinkE.) ([Hackaday][8])

## Alternatieven (zonder WCH-LinkE)

* **PicoRVD**: RP2040-based SWIO-debugger die als **GDB remote** fungeert. Werkt verrassend goed, maar is nog **alpha**. ([GitHub][9])
* **CLI-flashers** zoals **wlink** of **minichlink** (flash/erase/reset), zonder GDB-stap-voor-stap debug. ([GitHub][10])

## Bonus: seriële bootloader

De CH32V003 heeft een **factory bootloader** die je (na eenmalige optie-byte setup via SWIO) ook over **UART** kunt gebruiken voor firmware-updates. Handig voor “in-field” updates, maar eerste keer heb je toch SWIO nodig. ([GitHub][11], [Pallav Aggarwal][12])

---

### Kort antwoord op je vraag

* **TSSOP-20 CH32V003** programmeer je via **SWIO (PD1)**.
* **Tigard** en **RPi Debug Probe** zijn **niet geschikt** voor SWIO.
* **OpenOCD + GDB** werken prima met **WCH-LinkE** en de **WCH-OpenOCD fork**.

Wil je dat ik een mini “quick-start” pinout/bedradingsschets + exacte OpenOCD-cmd’s maak voor jouw board (met udev-rules voor Ubuntu 22.04)?

[1]: https://github.com/cjacker/opensource-toolchain-ch32v?utm_source=chatgpt.com "Opensource toolchain for WCH ch32v RISC-V 32bit MCU"
[2]: https://docs.zephyrproject.org/latest/boards/wch/ch32v003f4p6_dev_board/doc/index.html?utm_source=chatgpt.com "WCH CH32V003F4P6 Development Board"
[3]: https://community.element14.com/technologies/embedded/b/blog/posts/low-cost-microcontrollers-using-a-ch32v003-risc-v-device?utm_source=chatgpt.com "Low-Cost Microcontrollers: Using a CH32V003 RISC-V ..."
[4]: https://www.raspberrypi.com/documentation/microcontrollers/debug-probe.html?utm_source=chatgpt.com "Raspberry Pi Debug Probe - Raspberry Pi Documentation"
[5]: https://www.crowdsupply.com/securinghw/tigard/updates/i2c-and-swd-mode?utm_source=chatgpt.com "Tigard - I2C & SWD mode"
[6]: https://github.com/openwch/openocd_wch "GitHub - openwch/openocd_wch: OpenOCD for WCH CH32"
[7]: https://www.olimex.com/Products/RISC-V/WCH/WCH-LinkE/resources/WCH-LinkUserManual.PDF?utm_source=chatgpt.com "[PDF] WCH-LinkUserManual - Olimex"
[8]: https://hackaday.io/page/394994?utm_source=chatgpt.com "Getting blinky working on the WCH CH32V003F4 development board"
[9]: https://github.com/aappleby/picorvd?utm_source=chatgpt.com "aappleby/picorvd: GDB-compatible RISC-V Debugger for ..."
[10]: https://github.com/ch32-rs/wlink?utm_source=chatgpt.com "ch32-rs/wlink: An open source WCH-Link library/command line tool ..."
[11]: https://github.com/basilhussain/ch32v003-bootloader-docs?utm_source=chatgpt.com "WCH CH32V003 Factory Bootloader - The Missing Manual"
[12]: https://pallavaggarwal.in/2023/11/22/ch32v003-how-to-flash-program-using-serial-port/?utm_source=chatgpt.com "CH32V003: How To Flash Program Using Serial Port »"

