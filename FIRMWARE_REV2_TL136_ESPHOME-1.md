# TankMonitor — Firmware ESPHome REV2.0 TL-136

**Stato:** specifica firmware ufficiale / baseline  
**Hardware:** TankMonitor REV2.0 TL-136  
**MCU:** Seeed Studio XIAO ESP32-C6  
**Sensore livello:** TL-136, trasmettitore idrostatico 4–20 mA  
**Monitor corrente:** Texas Instruments INA228, I²C, shunt 5.10 Ω  
**Firmware:** ESPHome 2026.7.x o successivo compatibile  
**Backend:** Home Assistant + futura integrazione HACS `Water Tank Monitor`  
**Obiettivo:** misura affidabile con consumo estremamente ridotto e circa un anno o più tra le ricariche, da verificare sul prototipo reale.

---

## 1. Scopo del firmware

Il firmware ha volutamente responsabilità limitate.

### Esegue sullo XIAO ESP32-C6

- gestione deep sleep;
- risveglio periodico tramite timer;
- risveglio di manutenzione tramite reed magnetico;
- power gating fisico del boost + TL-136;
- warm-up configurabile del TL-136;
- misura della corrente 4–20 mA tramite INA228;
- validazione elettrica di base della misura;
- misura della batteria LiPo;
- segnalazione batteria bassa/critica;
- attivazione/disattivazione delle misure;
- intervallo di sleep configurabile;
- collegamento Wi-Fi solo dopo aver completato le misure;
- trasmissione a Home Assistant tramite ESPHome Native API;
- OTA durante la modalità manutenzione;
- diagnostica di base.

### NON esegue sullo ESP32

Restano responsabilità dell'integrazione HACS `Water Tank Monitor`:

- fondo scala effettivo del TL-136;
- calibrazione reale `I_ZERO` e `I_FULL`;
- distanza/quota del sensore dal fondo;
- altezza della cisterna;
- larghezza e profondità;
- conversione mA → cm;
- conversione cm → litri;
- percentuale riempimento;
- volume utilizzabile;
- livello minimo di pescaggio pompa;
- soglie riserva/vuoto;
- storico e consumo giornaliero;
- autonomia stimata;
- rilevamento `dati scaduti` tenendo conto del deep sleep.

La separazione è intenzionale: la geometria o la taratura possono cambiare senza modificare né riflashare il firmware.

---

## 2. Pinout ufficiale REV2

| Funzione | XIAO | GPIO ESP32-C6 |
|---|---|---:|
| Battery ADC | D0 / A0 | GPIO0 |
| Reed / wake manutenzione | D1 | GPIO1 |
| Power enable boost + TL-136 | D2 | GPIO2 |
| Battery divider enable | D3 | GPIO21 |
| INA228 SDA | D4 | GPIO22 |
| INA228 SCL | D5 | GPIO23 |
| Riservato | D6 | GPIO16 |
| Riservato | D7 | GPIO17 |

Il reed è collegato a GPIO1 e viene usato tramite `EXT1 ANY_HIGH`. GPIO0–GPIO7 sono utilizzabili come sorgenti EXT1 su ESP32-C6.

---

## 3. Stati operativi

### 3.1 Wake periodico normale

```text
DEEP SLEEP
    ↓ timer
BOOT
    ↓
misura batteria
    ↓
abilita INA228
    ↓
accende fisicamente BOOST + TL-136
    ↓
warm-up configurabile
    ↓
misura 4–20 mA
    ↓
spegne BOOST + TL-136
    ↓
INA228 shutdown
    ↓
Wi-Fi ON
    ↓
connessione Home Assistant
    ↓
trasmissione stati
    ↓
Wi-Fi OFF
    ↓
DEEP SLEEP
```

Il Wi-Fi viene deliberatamente attivato **dopo** la misura, così non rimane acceso durante il warm-up del TL-136.

### 3.2 Wake magnetico / manutenzione

Un magnete sul reed genera un wake EXT1.

```text
MAGNETE
   ↓
BOOT
   ↓
misura batteria
   ↓
eventuale misura TL-136
   ↓
Wi-Fi + API
   ↓
modalità manutenzione 10 minuti
   ↓
OTA / diagnostica / modifica parametri
   ↓
sleep
```

Anche un normale power-on/reset viene trattato come modalità manutenzione. Questo facilita primo avvio, test e OTA dopo un reboot.

### 3.3 Misurazione disabilitata

Con `Measurement Enabled = OFF`:

- boost sempre OFF;
- TL-136 sempre OFF;
- nessuna misura del loop;
- batteria comunque controllata;
- dispositivo si sveglia ogni 24 ore per una breve sincronizzazione;
- il reed resta sempre disponibile per manutenzione immediata.

Per una riattivazione remota mentre il nodo dorme, la futura integrazione HACS dovrà memorizzare il comando desiderato e applicarlo al successivo collegamento del dispositivo. Senza HACS è sempre possibile usare il magnete.

---

## 4. Parametri modificabili da Home Assistant

### `Sleep interval`

Default:

```text
3 ore
```

Range:

```text
0.5 – 24 ore
```

Step:

```text
0.5 ore
```

### `TL-136 Warm-up`

Default iniziale:

```text
3 secondi
```

Range:

```text
1 – 15 secondi
```

Il valore definitivo deve essere ricavato sperimentalmente provando il TL-136 reale.

### `Measurement Enabled`

```text
ON / OFF
```

I parametri vengono conservati in memoria persistente. Le modifiche sono rare; la modalità manutenzione è il momento consigliato per effettuarle.

---

## 5. Entità prodotte

### Sensori

- `Loop Current` — mA
- `Battery Voltage` — V
- `Battery Level` — %
- `WiFi RSSI` — dBm
- `Awake Time` — s

### Binary sensor

- `Measurement Valid`
- `Sensor Fault`
- `Battery Low`
- `Battery Critical`
- `Maintenance Mode`
- `Maintenance Reed`

### Text sensor

- `Wakeup Reason`
- `Last Error`
- `Firmware Version`

### Controlli

- `Sleep Interval`
- `TL-136 Warm-up`
- `Measurement Enabled`
- `Force Measurement`
- `Sleep Now`

---

# 6. Firmware ESPHome completo

Salvare, per esempio, come:

```text
tankmonitor-cisterna.yaml
```

```yaml
substitutions:
  device_name: "tankmonitor-cisterna"
  friendly_name: "TankMonitor Cisterna"
  project_version: "2.0.0"

  # Limiti elettrici di plausibilità del loop.
  # La calibrazione idraulica 4-20 mA viene fatta lato HACS.
  loop_min_valid_ma: "3.0"
  loop_max_valid_ma: "21.5"

esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}
  comment: "TankMonitor REV2 TL-136 - ultra low power"
  project:
    name: "fpapale.tankmonitor"
    version: "${project_version}"

  on_boot:
    # Memorizza il riferimento per l'awake time il prima possibile.
    - priority: 800
      then:
        - lambda: |-
            id(boot_millis) = millis();

    # Tutti i componenti sono ormai inizializzati.
    - priority: -100
      then:
        - script.execute: boot_coordinator

esp32:
  board: seeed_xiao_esp32c6
  framework:
    type: esp-idf

logger:
  level: INFO

# I valori persistenti vengono modificati molto raramente.
# 1s permette di consolidare rapidamente una modifica effettuata
# durante la finestra di manutenzione.
preferences:
  flash_write_interval: 1s

api:
  id: api_server
  encryption:
    key: !secret tankmonitor_api_key
  reboot_timeout: 0s

  actions:
    # API dedicata utile alla futura integrazione HACS:
    # può accodare lato HA un valore mentre il nodo dorme
    # e applicarlo quando il dispositivo ritorna online.
    - action: set_measurement_enabled
      variables:
        enabled: bool
      then:
        - if:
            condition:
              lambda: 'return enabled;'
            then:
              - switch.turn_on: measurement_enabled_control
            else:
              - switch.turn_off: measurement_enabled_control

    - action: set_sleep_hours
      variables:
        hours: float
      then:
        - number.set:
            id: sleep_hours_control
            value: !lambda |-
              if (hours < 0.5f) return 0.5f;
              if (hours > 24.0f) return 24.0f;
              return hours;

    - action: set_warmup_seconds
      variables:
        seconds: float
      then:
        - number.set:
            id: warmup_seconds_control
            value: !lambda |-
              if (seconds < 1.0f) return 1.0f;
              if (seconds > 15.0f) return 15.0f;
              return seconds;

    - action: force_measurement
      then:
        - if:
            condition:
              binary_sensor.is_on: maintenance_mode
            then:
              - script.execute: sample_cycle
            else:
              - logger.log:
                  level: WARN
                  format: "Force Measurement ignored outside maintenance mode"

ota:
  - platform: esphome
    password: !secret tankmonitor_ota_password

wifi:
  id: wifi_ctrl
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Fondamentale per il nostro ciclo:
  enable_on_boot: false

  # Non deve riavviare il nodo mentre volutamente teniamo il Wi-Fi spento.
  reboot_timeout: 0s

  # Conserva BSSID/canale in RTC memory:
  # sopravvive al deep sleep senza scrivere la flash a ogni wake.
  fast_connect:
    enabled: true
    storage: rtc

  # Poiché il Wi-Fi resta acceso solo per pochi secondi,
  # privilegiamo velocità/stabilità della connessione.
  power_save_mode: NONE

  # CONSIGLIATO DOPO IL PRIMO COLLAUDO:
  # una static IP può ridurre il tempo di connessione evitando DHCP.
  #
  # manual_ip:
  #   static_ip: 192.168.0.XXX
  #   gateway: 192.168.0.1
  #   subnet: 255.255.255.0
  #   dns1: 192.168.0.1

i2c:
  id: i2c_bus
  sda: GPIO22
  scl: GPIO23
  frequency: 400kHz
  scan: false

deep_sleep:
  id: deep_sleep_ctrl

  # GPIO1 = D1 del XIAO.
  # Reed normalmente aperto: chiudendosi porta GPIO1 a 3V3.
  esp32_ext1_wakeup:
    pins:
      - GPIO1
    mode: ANY_HIGH

globals:
  - id: boot_millis
    type: uint32_t
    restore_value: no
    initial_value: '0'

  - id: sleep_hours_value
    type: float
    restore_value: yes
    update_interval: 1s
    initial_value: '3.0'

  - id: warmup_seconds_value
    type: float
    restore_value: yes
    update_interval: 1s
    initial_value: '3.0'

  - id: measurement_enabled_value
    type: bool
    restore_value: yes
    update_interval: 1s
    initial_value: 'true'

  - id: ina_mode_write_ok
    type: bool
    restore_value: no
    initial_value: 'true'

# ------------------------------------------------------------
# GPIO DI POTENZA
# ------------------------------------------------------------

switch:
  # D2 / GPIO2:
  # pilota Q2 -> Q1 e alimenta fisicamente boost + TL-136.
  - platform: gpio
    id: sensor_power
    internal: true
    pin:
      number: GPIO2
      mode:
        output: true
    restore_mode: ALWAYS_OFF

  # D3 / GPIO21:
  # abilita il divisore resistivo per la misura batteria.
  - platform: gpio
    id: battery_adc_enable
    internal: true
    pin:
      number: GPIO21
      mode:
        output: true
    restore_mode: ALWAYS_OFF

  # Controllo logico esposto in Home Assistant.
  - platform: template
    id: measurement_enabled_control
    name: "Measurement Enabled"
    icon: "mdi:gauge"
    lambda: |-
      return id(measurement_enabled_value);

    turn_on_action:
      - lambda: |-
          id(measurement_enabled_value) = true;
      - logger.log: "Tank measurements ENABLED"

    turn_off_action:
      - lambda: |-
          id(measurement_enabled_value) = false;
      - switch.turn_off: sensor_power
      - logger.log: "Tank measurements DISABLED"

# ------------------------------------------------------------
# PARAMETRI CONFIGURABILI
# ------------------------------------------------------------

number:
  - platform: template
    id: sleep_hours_control
    name: "Sleep Interval"
    icon: "mdi:sleep"
    unit_of_measurement: "h"
    min_value: 0.5
    max_value: 24
    step: 0.5
    lambda: |-
      return id(sleep_hours_value);
    set_action:
      - lambda: |-
          id(sleep_hours_value) = x;
          ESP_LOGI("tankmonitor", "Sleep interval set to %.1f h", x);

  - platform: template
    id: warmup_seconds_control
    name: "TL-136 Warm-up"
    icon: "mdi:timer-sand"
    unit_of_measurement: "s"
    min_value: 1
    max_value: 15
    step: 1
    lambda: |-
      return id(warmup_seconds_value);
    set_action:
      - lambda: |-
          id(warmup_seconds_value) = x;
          ESP_LOGI("tankmonitor", "TL-136 warm-up set to %.0f s", x);

# ------------------------------------------------------------
# REED
# ------------------------------------------------------------

binary_sensor:
  - platform: gpio
    id: maintenance_reed
    name: "Maintenance Reed"
    entity_category: diagnostic
    pin:
      number: GPIO1
      mode:
        input: true

  - platform: template
    id: measurement_valid
    name: "Measurement Valid"

  - platform: template
    id: sensor_fault
    name: "Sensor Fault"
    device_class: problem

  - platform: template
    id: battery_low
    name: "Battery Low"
    device_class: battery

  - platform: template
    id: battery_critical
    name: "Battery Critical"
    device_class: battery

  - platform: template
    id: maintenance_mode
    name: "Maintenance Mode"
    entity_category: diagnostic

# ------------------------------------------------------------
# INA228 - LOOP 4-20 mA
# ------------------------------------------------------------

sensor:
  - platform: ina2xx_i2c
    id: ina228
    model: INA228
    i2c_id: i2c_bus
    address: 0x40

    shunt_resistance: 5.10 ohm
    max_current: 0.025 A

    # 5.10 ohm @ 20mA = 102mV.
    # Range 0 = ±163.84mV, quindi adeguato.
    adc_range: 0

    # 16 campioni: buon compromesso precisione/tempo.
    adc_averaging: 16
    adc_time: 1052us

    # La lettura viene comandata esplicitamente dagli script.
    update_interval: never

    current:
      id: loop_current
      name: "Loop Current"
      device_class: current
      state_class: measurement
      unit_of_measurement: "mA"
      accuracy_decimals: 3
      filters:
        - multiply: 1000.0

    shunt_voltage:
      id: loop_shunt_voltage
      name: "Loop Shunt Voltage"
      entity_category: diagnostic
      unit_of_measurement: "mV"
      accuracy_decimals: 3

  # Battery ADC.
  # Hardware REV2: divisore commutato 200k/200k = 1:2.
  - platform: adc
    id: battery_voltage
    name: "Battery Voltage"
    pin: GPIO0
    attenuation: 12db
    samples: 16
    sampling_mode: avg
    update_interval: never
    device_class: voltage
    state_class: measurement
    unit_of_measurement: "V"
    accuracy_decimals: 3
    filters:
      - multiply: 2.0

  # Stima SOC indicativa.
  # Non è un fuel gauge coulomb-counter: la curva andrà raffinata
  # dopo aver misurato la batteria reale.
  - platform: template
    id: battery_percent
    name: "Battery Level"
    device_class: battery
    state_class: measurement
    unit_of_measurement: "%"
    accuracy_decimals: 0
    update_interval: never
    lambda: |-
      const float v = id(battery_voltage).state;

      if (!std::isfinite(v)) return NAN;
      if (v >= 4.15f) return 100.0f;
      if (v >= 4.00f) return 90.0f + (v - 4.00f) / 0.15f * 10.0f;
      if (v >= 3.85f) return 70.0f + (v - 3.85f) / 0.15f * 20.0f;
      if (v >= 3.75f) return 50.0f + (v - 3.75f) / 0.10f * 20.0f;
      if (v >= 3.65f) return 25.0f + (v - 3.65f) / 0.10f * 25.0f;
      if (v >= 3.50f) return 10.0f + (v - 3.50f) / 0.15f * 15.0f;
      if (v >= 3.30f) return (v - 3.30f) / 0.20f * 10.0f;
      return 0.0f;

  - platform: wifi_signal
    id: wifi_rssi
    name: "WiFi RSSI"
    entity_category: diagnostic
    update_interval: never

  - platform: template
    id: awake_time
    name: "Awake Time"
    entity_category: diagnostic
    unit_of_measurement: "s"
    accuracy_decimals: 1
    update_interval: never

text_sensor:
  - platform: template
    id: wakeup_reason
    name: "Wakeup Reason"
    entity_category: diagnostic
    update_interval: never

  - platform: template
    id: last_error
    name: "Last Error"
    entity_category: diagnostic
    update_interval: never

  - platform: template
    id: firmware_version
    name: "Firmware Version"
    entity_category: diagnostic
    update_interval: never

# ------------------------------------------------------------
# COMANDI MANUALI
# ------------------------------------------------------------

button:
  - platform: template
    id: force_measurement_button
    name: "Force Measurement"
    icon: "mdi:water-sync"
    on_press:
      - if:
          condition:
            binary_sensor.is_on: maintenance_mode
          then:
            - script.execute: sample_cycle
          else:
            - logger.log:
                level: WARN
                format: "Force Measurement is available only in maintenance mode"

  - platform: template
    id: sleep_now_button
    name: "Sleep Now"
    icon: "mdi:power-sleep"
    on_press:
      - script.execute: go_to_sleep

# ------------------------------------------------------------
# SCRIPT
# ------------------------------------------------------------

script:

  # ----------------------------------------------------------
  # INA228: CONTINUOUS MODE
  #
  # ESPHome configura normalmente MODE=0xF.
  # Dopo ogni misura noi lo poniamo manualmente in shutdown.
  # Prima di una nuova misura riabilitiamo solo i bit MODE.
  #
  # ADC_CONFIG = register 0x01
  # MODE = bit 15..12
  # ----------------------------------------------------------

  - id: ina228_enable
    mode: restart
    then:
      - lambda: |-
          id(ina_mode_write_ok) = false;

          uint8_t data[2] = {0, 0};
          auto err = id(ina228).read_register(0x01, data, 2);

          if (err != esphome::i2c::ERROR_OK) {
            ESP_LOGW("tankmonitor", "INA228 ADC_CONFIG read failed");
            return;
          }

          uint16_t cfg =
              (static_cast<uint16_t>(data[0]) << 8) |
              static_cast<uint16_t>(data[1]);

          // MODE = 0xF: continuous bus + shunt + temperature.
          cfg = (cfg & 0x0FFFu) | 0xF000u;

          uint8_t out[2] = {
            static_cast<uint8_t>((cfg >> 8) & 0xFF),
            static_cast<uint8_t>(cfg & 0xFF)
          };

          err = id(ina228).write_register(0x01, out, 2);

          if (err == esphome::i2c::ERROR_OK) {
            id(ina_mode_write_ok) = true;
          } else {
            ESP_LOGW("tankmonitor", "INA228 enable failed");
          }

  - id: ina228_shutdown
    mode: restart
    then:
      - lambda: |-
          uint8_t data[2] = {0, 0};
          auto err = id(ina228).read_register(0x01, data, 2);

          if (err != esphome::i2c::ERROR_OK) {
            ESP_LOGW("tankmonitor", "INA228 shutdown: ADC_CONFIG read failed");
            return;
          }

          uint16_t cfg =
              (static_cast<uint16_t>(data[0]) << 8) |
              static_cast<uint16_t>(data[1]);

          // MODE = 0x0: shutdown.
          cfg &= 0x0FFFu;

          uint8_t out[2] = {
            static_cast<uint8_t>((cfg >> 8) & 0xFF),
            static_cast<uint8_t>(cfg & 0xFF)
          };

          err = id(ina228).write_register(0x01, out, 2);

          if (err != esphome::i2c::ERROR_OK) {
            ESP_LOGW("tankmonitor", "INA228 shutdown write failed");
          }

  # ----------------------------------------------------------
  # BATTERIA
  # ----------------------------------------------------------

  - id: measure_battery
    mode: restart
    then:
      - switch.turn_on: battery_adc_enable
      - delay: 100ms

      - component.update: battery_voltage
      - delay: 30ms
      - component.update: battery_percent
      - delay: 20ms

      - lambda: |-
          const float v = id(battery_voltage).state;
          const float pct = id(battery_percent).state;

          if (!std::isfinite(v) || v < 2.5f || v > 4.5f) {
            id(battery_low).publish_state(true);
            id(battery_critical).publish_state(true);
            id(last_error).publish_state("BATTERY_ADC_INVALID");
          } else {
            id(battery_low).publish_state(
              std::isfinite(pct) && pct <= 20.0f
            );
            id(battery_critical).publish_state(
              std::isfinite(pct) && pct <= 10.0f
            );
          }

      - switch.turn_off: battery_adc_enable

  # ----------------------------------------------------------
  # MISURA TL-136
  # ----------------------------------------------------------

  - id: measure_tank
    mode: restart
    then:
      - if:
          condition:
            lambda: 'return id(measurement_enabled_value);'
          then:
            # Riporta l'INA228 in continuous conversion.
            - script.execute: ina228_enable
            - script.wait: ina228_enable
            - delay: 100ms

            # Alimentazione FISICA del boost + TL-136.
            - switch.turn_on: sensor_power

            # Warm-up configurabile.
            - delay: !lambda |-
                return static_cast<uint32_t>(
                  id(warmup_seconds_value) * 1000.0f
                );

            # Chiede al componente INA228 di pubblicare una nuova lettura.
            - component.update: ina228

            # Margine per la sequenza di lettura del componente.
            - delay: 300ms

            # Spegni subito la parte che consuma di più.
            - switch.turn_off: sensor_power

            # Valida la corrente grezza.
            - lambda: |-
                const float ma = id(loop_current).state;

                if (!id(ina_mode_write_ok)) {
                  id(measurement_valid).publish_state(false);
                  id(sensor_fault).publish_state(true);
                  id(last_error).publish_state("INA228_CONFIG_ERROR");
                }
                else if (!std::isfinite(ma)) {
                  id(measurement_valid).publish_state(false);
                  id(sensor_fault).publish_state(true);
                  id(last_error).publish_state("INA228_NO_DATA");
                }
                else if (
                  ma < ${loop_min_valid_ma}f ||
                  ma > ${loop_max_valid_ma}f
                ) {
                  id(measurement_valid).publish_state(false);
                  id(sensor_fault).publish_state(true);
                  id(last_error).publish_state("LOOP_CURRENT_OUT_OF_RANGE");

                  ESP_LOGW(
                    "tankmonitor",
                    "Invalid TL-136 loop current: %.3f mA",
                    ma
                  );
                }
                else {
                  id(measurement_valid).publish_state(true);
                  id(sensor_fault).publish_state(false);
                  id(last_error).publish_state("OK");

                  ESP_LOGI(
                    "tankmonitor",
                    "TL-136 loop current: %.3f mA",
                    ma
                  );
                }

            # Porta l'INA228 a pochi µA mentre l'ESP dorme.
            - script.execute: ina228_shutdown
            - script.wait: ina228_shutdown

          else:
            - switch.turn_off: sensor_power

            - lambda: |-
                id(loop_current).publish_state(NAN);
                id(measurement_valid).publish_state(false);
                id(sensor_fault).publish_state(false);
                id(last_error).publish_state("MEASUREMENT_DISABLED");

            - script.execute: ina228_shutdown
            - script.wait: ina228_shutdown

  # ----------------------------------------------------------
  # UN CICLO DI CAMPIONAMENTO
  # ----------------------------------------------------------

  - id: sample_cycle
    mode: restart
    then:
      - script.execute: measure_battery
      - script.wait: measure_battery

      - script.execute: measure_tank
      - script.wait: measure_tank

  # ----------------------------------------------------------
  # RETE
  # ----------------------------------------------------------

  - id: network_sync
    mode: restart
    then:
      - wifi.enable

      # Non vogliamo che un AP assente mantenga l'ESP acceso per minuti.
      - wait_until:
          condition:
            wifi.connected:
          timeout: 10s

      - if:
          condition:
            wifi.connected:
          then:
            - component.update: wifi_rssi

            # Aspetta specificamente un client che sottoscrive gli stati:
            # tipicamente Home Assistant, non un semplice logger.
            - wait_until:
                condition:
                  api.connected:
                    state_subscription_only: true
                timeout: 10s

            - if:
                condition:
                  api.connected:
                    state_subscription_only: true
                then:
                  - lambda: |-
                      id(awake_time).publish_state(
                        (millis() - id(boot_millis)) / 1000.0f
                      );

                  # Lascia a HA il tempo di ricevere gli ultimi stati e,
                  # se necessario, inviare parametri accodati dalla HACS.
                  - delay: 3s

                else:
                  - lambda: |-
                      id(last_error).publish_state("HOME_ASSISTANT_API_TIMEOUT");
                  - logger.log:
                      level: WARN
                      format: "Home Assistant API timeout"

          else:
            - lambda: |-
                id(last_error).publish_state("WIFI_TIMEOUT");
            - logger.log:
                level: WARN
                format: "Wi-Fi connection timeout"

      - wifi.disable

  # ----------------------------------------------------------
  # NORMAL CYCLE
  # ----------------------------------------------------------

  - id: normal_cycle
    mode: restart
    then:
      - binary_sensor.template.publish:
          id: maintenance_mode
          state: OFF

      - script.execute: sample_cycle
      - script.wait: sample_cycle

      - script.execute: network_sync
      - script.wait: network_sync

      - script.execute: go_to_sleep

  # ----------------------------------------------------------
  # MAINTENANCE CYCLE
  # ----------------------------------------------------------

  - id: maintenance_cycle
    mode: restart
    then:
      - binary_sensor.template.publish:
          id: maintenance_mode
          state: ON

      # Facciamo comunque una misura subito:
      # è utile quando si apre la diagnostica.
      - script.execute: sample_cycle
      - script.wait: sample_cycle

      - wifi.enable

      - wait_until:
          condition:
            wifi.connected:
          timeout: 15s

      - if:
          condition:
            wifi.connected:
          then:
            - component.update: wifi_rssi

            - wait_until:
                condition:
                  api.connected:
                    state_subscription_only: true
                timeout: 15s

            - lambda: |-
                id(awake_time).publish_state(
                  (millis() - id(boot_millis)) / 1000.0f
                );

            # 10 minuti disponibili per:
            # - OTA
            # - modifica parametri
            # - Force Measurement
            # - diagnostica
            - delay: 10min

          else:
            # Se la rete non c'è, non ha senso bruciare batteria 10 min.
            - lambda: |-
                id(last_error).publish_state("MAINTENANCE_WIFI_TIMEOUT");
            - delay: 2s

      - binary_sensor.template.publish:
          id: maintenance_mode
          state: OFF

      - wifi.disable
      - script.execute: go_to_sleep

  # ----------------------------------------------------------
  # BOOT COORDINATOR
  # ----------------------------------------------------------

  - id: boot_coordinator
    mode: restart
    then:
      # Safety first: i carichi devono partire OFF.
      - switch.turn_off: sensor_power
      - switch.turn_off: battery_adc_enable

      - lambda: |-
          id(firmware_version).publish_state("${project_version}");

          const auto cause = esp_sleep_get_wakeup_cause();

          switch (cause) {
            case ESP_SLEEP_WAKEUP_EXT1:
              id(wakeup_reason).publish_state("MAGNET_EXT1");
              ESP_LOGI("tankmonitor", "Wake reason: magnetic EXT1");
              break;

            case ESP_SLEEP_WAKEUP_TIMER:
              id(wakeup_reason).publish_state("TIMER");
              ESP_LOGI("tankmonitor", "Wake reason: timer");
              break;

            case ESP_SLEEP_WAKEUP_UNDEFINED:
              id(wakeup_reason).publish_state("POWER_ON_OR_RESET");
              ESP_LOGI("tankmonitor", "Wake reason: power-on/reset");
              break;

            default:
              id(wakeup_reason).publish_state("OTHER");
              ESP_LOGI(
                "tankmonitor",
                "Wake reason code: %d",
                static_cast<int>(cause)
              );
              break;
          }

      # Power-on/reset e magnete = manutenzione.
      - if:
          condition:
            lambda: |-
              const auto cause = esp_sleep_get_wakeup_cause();
              return
                cause == ESP_SLEEP_WAKEUP_EXT1 ||
                cause == ESP_SLEEP_WAKEUP_UNDEFINED;
          then:
            - script.execute: maintenance_cycle
          else:
            - script.execute: normal_cycle

  # ----------------------------------------------------------
  # DEEP SLEEP
  # ----------------------------------------------------------

  - id: go_to_sleep
    mode: restart
    then:
      # Stato hardware sicuro.
      - switch.turn_off: sensor_power
      - switch.turn_off: battery_adc_enable

      - script.execute: ina228_shutdown
      - script.wait: ina228_shutdown

      - lambda: |-
          id(awake_time).publish_state(
            (millis() - id(boot_millis)) / 1000.0f
          );

      # Evita un loop wake/sleep se il magnete è ancora appoggiato.
      - if:
          condition:
            binary_sensor.is_on: maintenance_reed
          then:
            - logger.log:
                level: INFO
                format: "Waiting for maintenance magnet release"
            - wait_until:
                condition:
                  binary_sensor.is_off: maintenance_reed

      - delay: 100ms

      - deep_sleep.enter:
          id: deep_sleep_ctrl
          sleep_duration: !lambda |-
            // Se le misure sono disabilitate,
            // wake di servizio ogni 24 ore.
            if (!id(measurement_enabled_value)) {
              return static_cast<uint32_t>(24UL * 60UL * 60UL * 1000UL);
            }

            const float h = id(sleep_hours_value);
            return static_cast<uint32_t>(
              h * 60.0f * 60.0f * 1000.0f
            );
```

---

# 7. `secrets.yaml`

Aggiungere al `secrets.yaml` di ESPHome:

```yaml
wifi_ssid: "NOME_WIFI"
wifi_password: "PASSWORD_WIFI"

tankmonitor_api_key: "CHIAVE_API_BASE64"
tankmonitor_ota_password: "PASSWORD_OTA"
```

La chiave API può essere generata automaticamente da ESPHome quando si crea inizialmente il dispositivo.

---

# 8. Ottimizzazione IP statico

Dopo che il dispositivo funziona correttamente, è consigliabile assegnargli un IP fisso.

Due possibilità:

1. reservation DHCP sul router;
2. `manual_ip` direttamente in ESPHome.

Per ridurre davvero il tempo di wake, la seconda elimina la negoziazione DHCP.

Esempio:

```yaml
wifi:
  # ...
  manual_ip:
    static_ip: 192.168.0.90
    gateway: 192.168.0.1
    subnet: 255.255.255.0
    dns1: 192.168.0.1
```

Usare naturalmente indirizzi coerenti con la LAN reale.

---

# 9. Perché `fast_connect.storage: rtc`

ESPHome può conservare BSSID e canale dell'ultimo AP nella RTC memory.

Vantaggi:

- sopravvive al deep sleep;
- non richiede una scrittura Flash ad ogni wake;
- evita normalmente una scansione completa Wi-Fi;
- riduce il tempo con radio accesa.

Dopo una perdita completa di alimentazione il dato RTC viene perso e verrà ricostruito al primo collegamento riuscito.

---

# 10. INA228 e modalità shutdown

ESPHome supporta nativamente INA228 tramite `ina2xx_i2c`.

Normalmente il driver configura:

```text
ADC_CONFIG.MODE = 0xF
```

ovvero conversione continua di:

- bus voltage;
- shunt voltage;
- temperature.

La REV2 aggiunge due piccoli script low-level:

```text
ina228_enable
ina228_shutdown
```

che modificano **solo** i quattro bit `MODE` del registro:

```text
ADC_CONFIG = 0x01
```

senza alterare:

- averaging;
- tempi ADC;
- calibrazione shunt.

Prima della misura:

```text
MODE = 0xF
```

Prima del deep sleep:

```text
MODE = 0x0
```

L'INA228 ha un assorbimento molto più basso in shutdown rispetto al funzionamento normale.

---

# 11. Perché `adc_range: 0`

Hardware REV2:

```text
RSHUNT = 5.10 Ω
```

A 20 mA:

```text
VSHUNT = 0.020 × 5.10
       = 0.102 V
       = 102 mV
```

INA228:

```text
adc_range = 0
→ ±163.84 mV
```

quindi 102 mV rientrano nel range.

Il range ±40.96 mV non sarebbe sufficiente.

---

# 12. Validazione del loop

Il firmware **non** interpreta 4 mA come cisterna vuota o 20 mA come piena.

Controlla soltanto che il segnale sia elettricamente plausibile.

Default:

```text
3.0 mA ≤ I ≤ 21.5 mA
```

Fuori da questo intervallo:

```text
Measurement Valid = OFF
Sensor Fault      = ON
Last Error        = LOOP_CURRENT_OUT_OF_RANGE
```

Il margine è intenzionale: sensori economici possono non essere perfettamente calibrati a 4.000 e 20.000 mA.

La futura HACS userà:

```text
I_ZERO
I_FULL
FULL_SCALE_CM
```

misurati durante la calibrazione reale.

Non assumere che un TL-136 generico implementi formalmente NAMUR NE43.

---

# 13. Batteria

Il firmware legge la LiPo **prima** di accendere Wi-Fi e boost.

Questo è utile perché la tensione misurata è meno influenzata dai picchi di corrente.

Hardware:

```text
VBAT
 ↓
switch elettronico
 ↓
200k
 ↓
A0
 ↓
200k
 ↓
GND
```

Il divisore viene acceso solo per circa 150 ms per ciclo.

### Percentuale

`Battery Level` è una stima basata sulla tensione a vuoto.

Non è un vero fuel gauge.

Per una misura SOC molto accurata una futura revisione hardware potrebbe usare:

- MAX17048/MAX17043;
- LC709203F;
- equivalente fuel gauge a bassissimo consumo.

Per la REV2 la tensione batteria rimane il dato diagnostico principale.

---

# 14. Soglie batteria iniziali

Nel firmware:

```text
Battery Low      ≤ 20 %
Battery Critical ≤ 10 %
```

La HACS potrà inoltre creare allarmi propri basandosi direttamente sui volt.

Durante il collaudo è importante annotare la tensione reale a cui:

- il boost smette di mantenere correttamente 18 V;
- il Wi-Fi comincia a diventare instabile;
- la protezione della cella interviene.

La soglia realmente utile sarà stabilita da questi test.

---

# 15. Timeout rete

Per evitare che una rete non disponibile scarichi la batteria:

```text
Wi-Fi timeout: 10 s
HA API timeout: 10 s
```

In modalità manutenzione:

```text
Wi-Fi timeout: 15 s
HA API timeout: 15 s
```

Se il Wi-Fi fallisce durante il normale ciclo:

1. la misura è già stata eseguita;
2. boost e TL-136 sono già spenti;
3. il nodo torna comunque in deep sleep;
4. Home Assistant rileverà successivamente un dato scaduto.

Questo è preferibile a mantenere la radio accesa indefinitamente.

---

# 16. Stato `unavailable` in Home Assistant

Il dispositivo sarà `unavailable` quasi sempre.

È il comportamento previsto.

Non deve essere interpretato direttamente come fault.

La futura HACS userà:

```text
last_measurement
+
sleep_interval
+
tolerance
```

per generare:

```text
binary_sensor.cisterna_dati_scaduti
```

Esempio con sleep 3 h:

```text
ultimo dato 10:00
sleep       3 h
tolleranza  1.5 h

fault soltanto dopo ~14:30
```

---

# 17. Prima installazione

## Fase 1 — Solo XIAO

Non collegare ancora boost/TL-136.

Flash USB del firmware e verificare:

- boot;
- Wi-Fi;
- Home Assistant API;
- `Sleep Interval`;
- `Warm-up`;
- misura ADC batteria;
- reed;
- deep sleep;
- wake da magnete.

Durante lo sviluppo conviene temporaneamente impostare:

```text
Sleep Interval = 0.5 h
```

oppure commentare temporaneamente l'ingresso in deep sleep.

## Fase 2 — INA228

Collegare INA228 e verificare da log:

- manufacturer ID TI;
- device INA228 corretto;
- I²C 0x40;
- corrente prossima a zero con loop spento.

## Fase 3 — Boost senza TL-136

Misurare con multimetro:

```text
D2 OFF → boost realmente disalimentato
D2 ON  → 18 V ± tolleranza
```

Verificare:

- VBAT 3.2 V;
- VBAT 3.7 V;
- VBAT 4.2 V.

## Fase 4 — Simulazione loop

Prima del sensore reale, se possibile usare un calibratore 4–20 mA o un circuito di test.

Punti:

```text
4 mA
8 mA
12 mA
16 mA
20 mA
```

Con shunt 5.10 Ω, tensioni teoriche:

| Corrente | Shunt |
|---:|---:|
| 4 mA | 20.4 mV |
| 8 mA | 40.8 mV |
| 12 mA | 61.2 mV |
| 16 mA | 81.6 mV |
| 20 mA | 102.0 mV |

## Fase 5 — TL-136 reale

Provare diversi warm-up:

```text
1 s
2 s
3 s
5 s
10 s
```

Per ciascuno registrare almeno 20 cicli.

Scegliere il valore più breve per il quale:

- non compaiono outlier;
- la corrente è stabile;
- il risultato non dipende dal tempo di accensione.

---

# 18. Collaudo deep sleep

Il consumo va misurato sul dispositivo completo, non dedotto soltanto dai datasheet.

Durante deep sleep verificare:

```text
BOOST_VBAT     ≈ 0 V
18V_LOOP       ≈ 0 V
TL-136         OFF
battery divider OFF
INA228         shutdown
Wi-Fi          OFF
ESP32-C6       deep sleep
```

Target di progetto preliminare:

```text
< 30 µA complessivi
```

da confermare sul PCB reale.

Se il consumo è sensibilmente superiore, verificare nell'ordine:

1. boost non realmente isolato;
2. INA228 rimasto in continuous mode;
3. divisore batteria rimasto inserito;
4. LED/regolatori presenti sulla board;
5. GPIO che alimentano periferiche attraverso segnali;
6. resistenze di pull troppo basse;
7. leakage dei MOSFET;
8. batteria/protection board.

---

# 19. Misura del tempo di wake reale

`Awake Time` permette di verificare quanto dura un ciclo.

Separare durante i test:

```text
T_measure
T_wifi
T_api
T_total
```

L'obiettivo è che il tempo radio sia il più breve possibile.

L'uso di:

```text
fast_connect + RTC
static IP
buon RSSI
```

può avere un impatto sull'autonomia maggiore di molti piccoli risparmi circuitali.

---

# 20. RSSI e antenna

Il XIAO ESP32-C6 dispone di antenna integrata e possibilità di antenna esterna.

La cisterna può essere un ambiente RF difficile a causa di:

- cemento;
- botola metallica;
- masse d'acqua;
- posizione bassa rispetto all'access point.

Criterio pratico:

- collaudare inizialmente con antenna integrata;
- memorizzare `WiFi RSSI`;
- se connessione lenta/instabile, valutare antenna esterna prima di allungare i timeout.

Un segnale Wi-Fi debole non provoca soltanto perdita di pacchetti: aumenta anche il tempo con radio accesa.

---

# 21. Force Measurement

In modalità manutenzione è disponibile:

```text
button.force_measurement
```

Sequenza:

```text
battery
→ INA enable
→ boost ON
→ warm-up
→ INA read
→ boost OFF
→ INA shutdown
```

Non modifica il timer di sleep.

È utile per:

- taratura;
- simulazione livelli;
- verifica del loop;
- confronto multimetro/INA228.

---

# 22. OTA

La procedura raccomandata è:

1. avvicinare il magnete;
2. XIAO si sveglia tramite EXT1;
3. il dispositivo resta online circa 10 minuti;
4. eseguire OTA;
5. dopo il reboot, `ESP_SLEEP_WAKEUP_UNDEFINED` viene trattato ancora come power-on/reset e quindi entra nuovamente in manutenzione;
6. verificare il firmware;
7. premere `Sleep Now` oppure attendere il timeout.

Questo evita di dover "catturare" la breve finestra del wake periodico.

---

# 23. Ruolo della futura integrazione HACS

La HACS `Water Tank Monitor` dovrebbe ricevere dal nodo:

```text
Loop Current
Battery Voltage
Battery Level
Measurement Valid
Sensor Fault
Sleep Interval
Wakeup Reason
```

e aggiungere almeno:

```text
sensor.cisterna_livello_cm
sensor.cisterna_litri_presenti
sensor.cisterna_litri_utilizzabili
sensor.cisterna_percentuale
sensor.cisterna_percentuale_utilizzabile
sensor.cisterna_autonomia_stimata

binary_sensor.cisterna_riserva
binary_sensor.cisterna_vuota
binary_sensor.cisterna_dati_scaduti
```

Configurazione HACS:

```text
sensor type = TL-136
full scale = es. 200 cm

I_ZERO = valore reale misurato a livello 0
I_FULL = valore reale misurato a livello noto/fondo scala

tank height
tank width
tank depth

sensor offset
pump minimum level
reserve threshold
```

---

# 24. Comandi accodati HACS

Poiché un nodo in deep sleep non può ricevere comandi, la HACS dovrà distinguere:

```text
desired configuration
```

da:

```text
last configuration confirmed by device
```

Esempio:

```text
15:00 utente imposta sleep = 6 h
15:00 XIAO dorme

18:00 XIAO si sveglia
18:00 HACS rileva il collegamento
18:00 HACS chiama set_sleep_hours(6)
18:00 XIAO salva 6 h
18:00 HACS marca configurazione sincronizzata
18:00 XIAO dorme 6 h
```

Le `api.actions` incluse nel firmware sono state previste appositamente per questa futura sincronizzazione.

---

# 25. Eventi/allarmi

Il firmware genera stati diagnostici locali.

La HACS dovrà trasformare le transizioni significative in eventi Home Assistant, ad esempio:

```text
tank_monitor.measurement
tank_monitor.reserve
tank_monitor.empty
tank_monitor.sensor_fault
tank_monitor.battery_low
tank_monitor.battery_critical
tank_monitor.device_stale
```

Il firmware non deve generare ripetutamente allarmi riserva/vuoto, perché non conosce la geometria né la taratura finale.

---

# 26. Evoluzione futura ESP-NOW

La REV2 usa:

```text
XIAO → Wi-Fi → ESPHome Native API → Home Assistant
```

Se il test di autonomia dimostrasse che il tempo di associazione Wi-Fi è il principale consumo, la futura REV2.1 potrà usare:

```text
TankMonitor XIAO
       ↓
    ESP-NOW
       ↓
ESP32 gateway alimentato
       ↓
Wi-Fi / ESPHome
       ↓
Home Assistant
```

La sezione di misura TL-136, batteria e HACS rimarrebbe sostanzialmente invariata.

Questa ottimizzazione va presa in considerazione soltanto dopo aver misurato il consumo reale della REV2 Wi-Fi.

---

# 27. Punti da non modificare senza nuova revisione hardware

Sono congelati:

- XIAO ESP32-C6;
- GPIO0 battery ADC;
- GPIO1 reed EXT1;
- GPIO2 sensor/boost power;
- GPIO21 battery divider enable;
- GPIO22 SDA;
- GPIO23 SCL;
- INA228 0x40;
- shunt 5.10 Ω;
- `adc_range: 0`;
- power gating fisico boost;
- TL-136 4–20 mA;
- deep sleep come modalità normale;
- calcoli idraulici lato HACS.

Una modifica a uno di questi elementi richiede una nuova revisione della specifica.

---

# 28. Migliorie firmware già previste

Dopo il primo prototipo potranno essere aggiunte senza cambiare PCB:

1. media/mediana di più letture 4–20 mA;
2. rilevamento stabilità durante warm-up;
3. warm-up adattivo;
4. conteggio wake falliti;
5. contatore errori Wi-Fi;
6. battery curve calibrata;
7. wake più frequente quando la cisterna è quasi vuota, comandato dalla HACS;
8. sleep più lungo quando la cisterna è piena;
9. sincronizzazione bidirezionale configurazione HACS;
10. ESP-NOW gateway.

La prima release deve però restare semplice per permettere di misurare con precisione consumi e comportamento del sensore reale.

---

# 29. Note di compatibilità

Questa baseline è stata progettata sulla documentazione ESPHome corrente ad agosto 2026.

Funzioni utilizzate ufficialmente da ESPHome:

- ESP32-C6;
- deep sleep;
- EXT1 wake;
- `deep_sleep.enter` con durata templatable;
- `wifi.enable` / `wifi.disable`;
- `wifi.enable_on_boot: false`;
- `fast_connect.storage: rtc`;
- Native API;
- `api.connected` con `state_subscription_only`;
- INA228 tramite `ina2xx_i2c`;
- ADC ESP32-C6;
- OTA ESPHome.

L'unica parte low-level specifica è la modifica del registro `ADC_CONFIG` dell'INA228 per portarlo esplicitamente in shutdown fra una misura e l'altra.

---

# 30. Fonti tecniche ufficiali

- ESPHome — Deep Sleep Component  
  https://esphome.io/components/deep_sleep/

- ESPHome — WiFi Component  
  https://esphome.io/components/wifi/

- ESPHome — Native API  
  https://esphome.io/components/api/

- ESPHome — INA2xx digital power monitors  
  https://esphome.io/components/sensor/ina2xx/

- ESPHome — ADC Sensor  
  https://esphome.io/components/sensor/adc/

- ESPHome — Core / Preferences  
  https://esphome.io/components/esphome/

- ESPHome — OTA  
  https://esphome.io/components/ota/esphome/

- Seeed Studio — XIAO ESP32-C6 Getting Started  
  https://wiki.seeedstudio.com/xiao_esp32c6_getting_started/

- PlatformIO — Seeed Studio XIAO ESP32-C6  
  https://docs.platformio.org/en/latest/boards/espressif32/seeed_xiao_esp32c6.html

- Texas Instruments — INA228  
  https://www.ti.com/product/INA228

---

# 31. Versionamento

```text
Hardware:  REV2.0 TL-136
Firmware:  REV2.0 / 2.0.0
```

Questa specifica sostituisce le precedenti bozze firmware basate sul sensore ultrasuoni.

Qualsiasi futura modifica sostanziale dovrà incrementare almeno:

```text
Firmware REV2.1
```

oppure, se cambia l'hardware:

```text
Hardware REV3
```
