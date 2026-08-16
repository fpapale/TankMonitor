# Water Tank Monitor — Specifica Hardware Ufficiale REV2.0 TL-136

**Stato:** DESIGN BASELINE  
**Revisione:** 2.0  
**Controller:** Seeed Studio XIAO ESP32-C6  
**Sensore:** TL-136 idrostatico 4–20 mA  
**Firmware:** ESPHome  
**Backend:** Home Assistant + integrazione HACS `Water Tank Monitor`  
**Alimentazione:** LiPo/Li-ion 1S 3,7 V ricaricabile  
**Obiettivo energetico:** circa 12 mesi o più tra due ricariche, da validare sul prototipo.

---

## 1. Scopo

Il dispositivo misura il livello dell'acqua di una cisterna mediante un sensore idrostatico immerso **TL-136 con uscita industriale 4–20 mA**.

Il nodo deve:

- effettuare misure periodiche;
- trascorrere la maggior parte del tempo in deep sleep;
- alimentare il TL-136 esclusivamente durante la misura;
- trasmettere a Home Assistant la corrente grezza in mA;
- trasmettere tensione e stato batteria;
- identificare misure non valide o guasti del loop;
- consentire configurazione del tempo di sleep;
- consentire attivazione/disattivazione delle misure;
- consentire risveglio manuale tramite magnete;
- consentire OTA/manutenzione senza aprire il contenitore.

La trasformazione:

**mA → cm acqua → litri → percentuale → riserva**

non viene eseguita dal dispositivo, ma dall'integrazione HACS `Water Tank Monitor`.

---

## 2. Sensore ufficiale

Il sensore di riferimento è:

**TL-136 — 2-wire hydrostatic liquid level transmitter — 4–20 mA**

Sono ammesse le versioni:

- 0–1 m
- 0–2 m
- 0–3 m
- 0–4 m
- 0–5 m

purché siano specificate:

- uscita 4–20 mA;
- alimentazione 12–32 VDC;
- protezione IP68;
- uso per acqua;
- compensazione atmosferica mediante cavo ventilato.

**Il range NON è fissato dal PCB.**

Il PCB funzionerà con qualunque fondo scala TL-136 compatibile.

### Criterio di scelta

Scegliere il fondo scala immediatamente superiore alla massima colonna d'acqua realmente misurabile.

Esempio:

cisterna alta 165 cm → **TL-136 0–2 m**

è preferibile a un 0–5 m perché utilizza una porzione maggiore del range 4–20 mA.

---

## 3. Relazione corrente-livello

Il comportamento nominale è:

```text
4 mA  = livello zero
20 mA = fondo scala TL-136
```

Il dispositivo ESPHome trasmette **la corrente misurata**, senza trasformarla direttamente in litri.

La HACS integration applicherà:

```text
livello_cm =
    (corrente_mA - I_ZERO)
    ---------------------- × FULL_SCALE_CM
       I_FULL - I_ZERO
```

dove `I_ZERO` e `I_FULL` saranno valori **calibrabili**.

Quindi non assumiamo necessariamente:

```text
I_ZERO = 4.000 mA
I_FULL = 20.000 mA
```

ma possiamo calibrare il sensore reale.

---

## 4. Controller

Controller ufficiale:

**Seeed Studio XIAO ESP32-C6**

Caratteristiche utilizzate:

- alimentazione Li-ion/LiPo 3,7 V;
- ricarica tramite USB-C;
- Wi-Fi;
- deep sleep;
- ADC;
- I²C;
- risveglio mediante LP GPIO.

---

## 5. Pinout ufficiale REV2

| Funzione | XIAO | ESP32-C6 |
|---|---|---|
| Battery ADC | D0 / A0 | GPIO0 |
| Reed wake-up | D1 | GPIO1 |
| TL-136 power enable | D2 | GPIO2 |
| Battery ADC enable | D3 | GPIO21 |
| INA228 SDA | D4 | GPIO22 |
| INA228 SCL | D5 | GPIO23 |
| Future / debug | D6 | GPIO16 |
| Future / debug | D7 | GPIO17 |

GPIO1/D1 è riservato al reed magnetico di wake-up.

---

## 6. Alimentazione

Batteria ufficiale:

**LiPo/Li-ion 1S protetta, 3,7 V nominali**

Capacità consigliata:

```text
5000 mAh
```

Range tipico della cella:

```text
~3,0 → 4,2 V
```

Connessione:

```text
LiPo
 │
 ├──────────► BAT XIAO ESP32-C6
 │
 └──────────► SENSOR POWER SWITCH
```

La ricarica viene affidata esclusivamente al circuito onboard del XIAO tramite USB-C.

Non installare:

- TP4056;
- battery shield;
- secondo caricabatterie;
- power bank module;
- step-up permanente.

---

## 7. Alimentazione TL-136

Il TL-136 non viene mai alimentato continuamente.

La catena ufficiale è:

```text
LiPo 3.7 V
    │
    ▼
Q1 high-side switch
    │
    ▼
BOOST_VBAT
    │
    ▼
TPS61040
    │
    ▼
18 V nominali
    │
    ▼
TL-136
```

### 7.1 High-side physical disconnect

Componenti:

```text
Q1 = AO3401A P-MOSFET
Q2 = 2N7002 N-MOSFET
```

Configurazione:

```text
VBAT → Q1 source
Q1 drain → BOOST_VBAT

Q1 gate → 1 MΩ → VBAT
Q1 gate → Q2 drain

Q2 source → GND

D2 / GPIO2
      │
     4.7k
      │
      ▼
Q2 gate

Q2 gate → 1 MΩ → GND
```

Logica:

```text
D2 LOW / floating
        ↓
Q2 OFF
        ↓
Q1 OFF
        ↓
BOOST completamente scollegato
```

```text
D2 HIGH
        ↓
Q2 ON
        ↓
Q1 ON
        ↓
BOOST alimentato
        ↓
TL-136 alimentato
```

Questa è la modalità ufficiale REV2.

Non ci affidiamo solamente al pin EN del convertitore.

---

## 8. Convertitore DC/DC

Convertitore ufficiale:

**Texas Instruments TPS61040**

Input:

```text
LiPo 3,0–4,2 V
```

Output nominale:

```text
18,0 V
```

Corrente richiesta:

```text
≥ 20 mA continui durante la misura
```

La sezione boost deve seguire il **reference design TPS61040 18 V / 20 mA di Texas Instruments**.

I valori definitivi di induttore, diodo Schottky, condensatori e feedback divider devono essere derivati dall'ultima revisione del datasheet/EVM TI prima della produzione e non inventati dall'AI EDA.

Condensatori sul rail 18 V: rating minimo 25 V, preferibile 35 V.

---

## 9. Loop 4–20 mA

```text
           +18 V
             │
          TL-136
             │
             │ LOOP_RETURN
             ▼
          RSHUNT
             │
             ▼
            GND
```

Il TL-136 è trattato come dispositivo **2-wire current loop**.

Connettore J2:

```text
1  LOOP_18V+
2  LOOP_RETURN
3  SHIELD / RESERVED
```

Utilizzare preferibilmente una morsettiera a vite o spring terminal.

---

## 10. Misura 4–20 mA

Componente ufficiale: **Texas Instruments INA228**.

- Protocollo: I²C
- Alimentazione: 3V3 XIAO
- SDA → D4 / GPIO22
- SCL → D5 / GPIO23
- Indirizzo iniziale: `0x40`

ESPHome supporta INA228 tramite il componente `ina2xx_i2c`.

---

## 11. Resistenza di shunt

```text
RSHUNT = 5.10 Ω
tolleranza = 0.1 %
tempco ≤ 50 ppm/°C
```

```text
4 mA  × 5.1 Ω = 20.4 mV
20 mA × 5.1 Ω = 102 mV
```

A 20 mA la dissipazione è circa 2 mW; una resistenza da 0,1 W o superiore è sufficiente.

---

## 12. Gestione INA228 in sleep

INA228 rimane collegato alla 3V3 del XIAO ma deve essere portato via software in modalità shutdown prima del deep sleep.

---

## 13. Misura della batteria

Usare `D0 / A0 / GPIO0` con divisore 1:2, disconnesso quando non utilizzato.

Componenti:

```text
Q3 = AO3401A
Q4 = 2N7002
Rtop    = 200 kΩ 1 %
Rbottom = 200 kΩ 1 %
Cfilter = 100 nF
```

Controllo: `D3 / GPIO21`.

```text
D3 LOW  → battery divider OFF
D3 HIGH → battery divider ON
```

Procedura:

```text
ENABLE divider
→ attesa 50–100 ms
→ 16 campioni ADC
→ media
→ DISABLE divider
```

---

## 14. Reed switch di manutenzione

Pin: `D1 / GPIO1`.

```text
3V3
 │
 REED NO
 │
 ├────► D1
 │
100k
 │
GND
```

Wake timer:

```text
misura → trasmissione → deep sleep
```

Wake magnetico:

```text
magnete
→ GPIO wake
→ Maintenance Mode
→ Wi-Fi permanente temporaneo
→ ESPHome API
→ OTA
```

Durata Maintenance Mode predefinita: **10 minuti**.

---

## 15. Parametri ESPHome

```text
number.cisterna_sleep_hours
switch.cisterna_measurement_enabled
number.cisterna_sensor_warmup_seconds
```

Default:

```text
sleep_hours             = 3 h
measurement_enabled     = ON
sensor_warmup_seconds   = 3 s
```

`sensor_warmup_seconds` deve essere configurabile nel range consigliato 1–15 secondi.

---

## 16. Modalità disabled

Quando `measurement_enabled = OFF`, il TL-136 non viene alimentato e non viene eseguita la misura.

Il nodo continua però a svegliarsi ogni 24 h per collegarsi a Home Assistant e ricevere eventuali modifiche. Il reed magnetico continua sempre a funzionare.

---

## 17. Sequenza operativa normale

```text
DEEP SLEEP
→ TIMER WAKE
→ misura batteria
→ D2 HIGH
→ Q1 ON
→ TPS61040 ON
→ 18 V
→ TL-136 ON
→ warm-up configurabile
→ INA228
→ letture multiple
→ validazione
→ D2 LOW
→ TL-136 + BOOST OFF
→ Wi-Fi
→ ESPHome API
→ Home Assistant
→ attesa conferma / timeout
→ INA228 shutdown
→ DEEP SLEEP
```

---

## 18. Dati prodotti dall'ESP32

```text
sensor.cisterna_loop_current        [mA]
sensor.cisterna_battery_voltage     [V]
sensor.cisterna_battery_percent     [%]
sensor.cisterna_wifi_rssi
sensor.cisterna_awake_time

binary_sensor.cisterna_measurement_valid
binary_sensor.cisterna_sensor_fault
binary_sensor.cisterna_battery_low

text_sensor.cisterna_wakeup_reason
text_sensor.cisterna_last_error
```

---

## 19. Diagnostica del loop

Indicativamente:

```text
0–3 mA       → possibile loop aperto / sensore spento
~4–20 mA     → misura normale
>21 mA       → possibile errore
```

Le soglie precise devono essere configurabili e validate sul TL-136 realmente acquistato. Non assumere che il TL-136 economico implementi formalmente NAMUR NE43.

---

## 20. Responsabilità della HACS integration

L'integrazione `Water Tank Monitor` riceve:

```text
corrente mA
battery voltage
battery percent
measurement valid
timestamp
```

Parametri:

```text
tipo sensore = TL-136
full_scale_cm
calibration_zero_mA
calibration_full_mA
altezza cisterna
larghezza cisterna
profondità cisterna
quota sensore dal fondo
livello minimo pompa
riserva cm
warning %
critical %
```

Produce:

```text
sensor.cisterna_livello_cm
sensor.cisterna_litri_totali
sensor.cisterna_litri_presenti
sensor.cisterna_litri_utilizzabili
sensor.cisterna_percentuale
sensor.cisterna_percentuale_utilizzabile
sensor.cisterna_autonomia_stimata

binary_sensor.cisterna_riserva
binary_sensor.cisterna_vuota
binary_sensor.cisterna_batteria_bassa
binary_sensor.cisterna_sensore_guasto
binary_sensor.cisterna_dati_scaduti
```

---

## 21. Allarme dispositivo mancante

Lo stato `unavailable` di ESPHome durante il deep sleep non è automaticamente un guasto.

```text
deadline = last_measurement
           + sleep_interval
           + tolerance
```

Default:

```text
tolerance = max(30 min, sleep_interval × 0.5)
```

Solo superata tale soglia viene attivato `binary_sensor.cisterna_dati_scaduti`.

---

## 22. Installazione TL-136

Il TL-136 deve essere:

- immerso;
- in posizione stabile;
- sufficientemente vicino al fondo;
- lontano dal punto di aspirazione diretta della pompa;
- protetto da urti e sedimenti eccessivi.

Il cavo deve raggiungere una zona asciutta sopra il livello dell'acqua per consentire la compensazione atmosferica.

**L'estremità di compensazione atmosferica NON deve essere sigillata con silicone, resina o colla.** Deve respirare restando protetta dall'umidità.

---

## 23. PCB

```text
2 layers
FR4
1.6 mm
1 oz copper
```

Dimensione target: **≤ 65 × 50 mm**.  
Montaggio: **4 × M3**.

XIAO preferibilmente removibile mediante header e USB-C accessibile.

---

## 24. RF

Mantenere libera la keep-out area dell'antenna XIAO. Evitare GND plane, piste, batteria, cavo TL-136 e metallo direttamente sotto/davanti all'antenna.

---

## 25. Connettori

### J1 — Battery

```text
JST-PH 2.0
1 BAT+
2 BAT-
```

### J2 — TL-136

```text
1 LOOP_18V+
2 LOOP_RETURN
3 SHIELD / RESERVED
```

### J3 — Reed

```text
1 3V3
2 REED_WAKE
```

---

## 26. Test point obbligatori

```text
TP1   VBAT
TP2   3V3
TP3   GND
TP4   BOOST_VBAT
TP5   18V_LOOP
TP6   LOOP_RETURN
TP7   SHUNT+
TP8   SHUNT-
TP9   SENSOR_ENABLE
TP10  BATTERY_ADC
TP11  REED_WAKE
TP12  SDA
TP13  SCL
```

---

## 27. Componenti principali congelati

| Ref. | Componente | Funzione |
|---|---|---|
| U1 | XIAO ESP32-C6 | MCU / Wi-Fi / charging |
| U2 | TPS61040 | Boost 3,7 → 18 V |
| U3 | INA228 | misura loop |
| Q1 | AO3401A | isolamento fisico boost |
| Q2 | 2N7002 | driver Q1 |
| Q3 | AO3401A | battery ADC switch |
| Q4 | 2N7002 | driver Q3 |
| RSHUNT | 5.10 Ω 0.1% | misura 4–20 mA |
| Sensor | TL-136 | pressione idrostatica |
| Battery | LiPo 1S ~5000 mAh | alimentazione |

---

## 28. Criteri di accettazione prototipo

### Alimentazione
Con D2 LOW: TL-136 non alimentato e boost fisicamente isolato.

### Boost
Con batteria a 3.2 V, 3.7 V e 4.2 V, l'uscita deve restare entro **18.0 V ±5%** con loop a 20 mA.

### Loop
Simulare 4, 8, 12, 16 e 20 mA e verificare la misura INA228.

### Deep sleep
Target iniziale del dispositivo completo: **< 30 µA**, esclusa autoscarica della batteria.

### Sensor start-up
Verificare stabilità del TL-136 dopo 1, 2, 3, 5 e 10 secondi e scegliere il più breve `warmup_seconds` affidabile.

### Wi-Fi
Misurare il tempo `wake → HA connected` per determinare il consumo reale del ciclo.

---

## 29. Filosofia energetica

```text
1. TL-136 completamente spento
2. boost completamente scollegato
3. battery divider spento
4. INA228 shutdown
5. ESP32 deep sleep
6. Wi-Fi attivo solo dopo la misura
7. connessione HA più breve possibile
```

La trasmissione Wi-Fi deve essere l'ultima operazione del ciclo.

---

## 30. Design Freeze REV2.0

Sono considerati **congelati**:

- XIAO ESP32-C6;
- TL-136 4–20 mA;
- alimentazione LiPo 1S;
- loop 18 V;
- TPS61040;
- physical power disconnect;
- INA228;
- shunt 5.10 Ω;
- reed magnetico;
- misura batteria;
- ESPHome;
- deep sleep;
- elaborazione volume lato HACS.

Rimane parametro di installazione, e **non modifica il PCB**, il fondo scala TL-136.

Il range concreto 0–1 / 0–2 / 0–3 / 0–4 / 0–5 m viene scelto in funzione della cisterna e configurato successivamente nella HACS integration.

**Questa REV2.0 sostituisce ufficialmente la precedente REV1 basata su A02YYUW come configurazione principale del progetto.**

---

## Riferimenti tecnici

- Seeed Studio XIAO ESP32-C6: https://wiki.seeedstudio.com/xiao_esp32c6_getting_started/
- Texas Instruments TPS61040: https://www.ti.com/product/TPS61040
- Texas Instruments INA228: https://www.ti.com/product/INA228
- ESPHome INA2xx I²C: https://esphome.io/components/sensor/ina2xx/
- TL-136 manual/specification reference: https://manuals.plus/ae/1005001447225174
