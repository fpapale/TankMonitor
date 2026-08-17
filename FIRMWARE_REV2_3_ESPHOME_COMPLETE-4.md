# TankMonitor — Firmware ESPHome REV2.3 completo
Bundle sorgente della build **TankMonitor-HA** per XIAO ESP32-C6 + TL-136 + INA228.
> Nota: il progetto è stato scritto contro le API/documentazione ESPHome correnti, ma non è stato compilato in questo ambiente. Eseguire `Validate` nel proprio ESPHome Device Builder prima del flash.

---

## README

# TankMonitor ESPHome REV2.3 — Home Assistant build

Questa cartella contiene il firmware aggiornato per:

- Seeed Studio XIAO ESP32-C6
- TL-136 4–20 mA
- INA228 con shunt 5.10 Ω
- batteria LiPo 1S
- deep sleep
- Home Assistant via ESPHome Native API
- fino a 5 profili Wi-Fi runtime
- captive portal per primo setup/recovery
- IPv4 statico configurabile a runtime
- fallback automatico DHCP se lo statico non torna raggiungibile

## File

```text
tankmonitor-ha.yaml
secrets.example.yaml
components/
  tankmonitor_network/
    __init__.py
    tankmonitor_network.h
```

## Primo avvio

1. Copiare `secrets.example.yaml` come `secrets.yaml`.
2. Inserire API key, password OTA e password dell'AP di setup.
3. Compilare e flashare via USB.
4. Il primo power-on entra in Maintenance Mode.
5. Non essendoci una rete domestica nel firmware, viene reso disponibile:
   `TankMonitor Setup`.
6. Collegarsi all'AP e usare il captive portal per scegliere una delle reti Wi-Fi visibili.
7. Home Assistant scoprirà il dispositivo tramite ESPHome.

## Gestione reti da Home Assistant

Il firmware crea azioni ESPHome:

- `network_add_profile(ssid, password, priority)`
- `network_remove_profile(ssid)`
- `network_use_dhcp(ssid)`
- `network_set_static(ssid, static_ip, gateway, subnet, dns1, dns2)`
- `network_scan()`
- `network_clear_all_profiles()`

Il primo Wi-Fi configurato dal captive portal viene automaticamente "adottato"
nella lista TankMonitor alla prima connessione effettiva di Home Assistant.

## Static IP fault tolerant

`network_set_static` NON interrompe subito la connessione corrente.

La configurazione viene salvata come PENDING e testata al wake successivo:

```text
STATIC
  ↓
HA API raggiungibile?
  ├─ sì → STATIC VALIDATED
  └─ no → stesso SSID con DHCP
                 ↓
             HA API?
              ├─ sì → DHCP_FALLBACK
              └─ no → prova tutte le altre reti note
```

Le coordinate dello static IP fallito restano memorizzate per poterle correggere,
ma non vengono riutilizzate automaticamente fino a una nuova
`network_set_static`.

## Sleep

Configurabile da Home Assistant:

- minimo: 15 minuti
- massimo: 1440 minuti
- step: 15 minuti
- default: 180 minuti

## Recovery fisico

Il reed su GPIO1 viene usato come wake magnetico.

In Maintenance Mode il fallback AP può essere aperto se nessuna rete nota funziona.
Durante un normale wake periodico il fallback AP è disabilitato per non sprecare batteria.

## Nota importante sulla compilazione

La configurazione è stata aggiornata contro la documentazione/API ESPHome 2026.7.4.
In questo ambiente non era disponibile il toolchain ESPHome per eseguire una compilazione
completa; il primo passaggio operativo deve quindi essere un `Validate`/`Install` nel tuo
ESPHome Device Builder. Se emerge un errore di API interna dell'external component,
va corretto prima di collegare la parte 18 V/TL-136.

## Fonti tecniche

- https://esphome.io/components/wifi/
- https://esphome.io/components/captive_portal/
- https://esphome.io/components/deep_sleep/
- https://esphome.io/components/api/
- https://esphome.io/components/sensor/ina2xx/
- https://esphome.io/components/sensor/adc/


---

## Firmware ESPHome

```yaml
substitutions:
  device_name: "tankmonitor-cisterna"
  friendly_name: "TankMonitor Cisterna"
  project_version: "2.3.0"

  # Validità ELETTRICA del loop. La calibrazione idraulica è lato HACS.
  loop_min_valid_ma: "3.0"
  loop_max_valid_ma: "21.5"

esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}
  comment: "TankMonitor REV2.3 HA - XIAO ESP32-C6 + TL-136 + INA228"
  project:
    name: "fpapale.tankmonitor"
    version: "${project_version}"

  on_boot:
    - priority: 800
      then:
        - lambda: |-
            id(boot_millis) = millis();

    - priority: -100
      then:
        - script.execute: boot_coordinator

esp32:
  board: seeed_xiao_esp32c6
  framework:
    type: esp-idf

logger:
  level: INFO

preferences:
  flash_write_interval: 1s

# External component locale: gestisce fino a 5 profili Wi-Fi,
# IPv4 statico runtime e rollback automatico a DHCP.
external_components:
  - source:
      type: local
      path: components
    components:
      - tankmonitor_network

tankmonitor_network:
  id: tank_net
  max_profiles: 5
  static_validation_timeout: 8s
  dhcp_fallback_timeout: 10s
  maintenance_ap_timeout: 15s

# Nessun SSID/password/IP domestico è presente nel firmware.
# Al primo avvio (o in recovery) l'AP di fallback + captive portal
# consente di scegliere una rete visibile.
wifi:
  id: wifi_ctrl
  enable_on_boot: false
  reboot_timeout: 0s

  fast_connect:
    enabled: true
    storage: rtc

  power_save_mode: NONE
  min_auth_mode: WPA2

  ap:
    ssid: "TankMonitor Setup"
    password: !secret tankmonitor_setup_ap_password
    ap_timeout: 0s

captive_portal:

# Utile anche per provisioning via USB.
improv_serial:

api:
  id: api_server
  reboot_timeout: 0s
  batch_delay: 50ms

  encryption:
    key: !secret tankmonitor_api_key

  actions:
    # ----------------------------
    # Parametri operativi
    # ----------------------------
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

    - action: set_sleep_minutes
      variables:
        minutes: int
      then:
        - number.set:
            id: sleep_minutes_control
            value: !lambda |-
              if (minutes < 15) return 15.0f;
              if (minutes > 1440) return 1440.0f;
              return static_cast<float>(minutes);

    - action: set_warmup_seconds
      variables:
        seconds: int
      then:
        - number.set:
            id: warmup_seconds_control
            value: !lambda |-
              if (seconds < 1) return 1.0f;
              if (seconds > 15) return 15.0f;
              return static_cast<float>(seconds);

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
                  format: "Force Measurement disponibile solo in maintenance mode"

    # ----------------------------
    # Network Manager
    # Le password sono parametri API, non entità.
    # ----------------------------
    - action: network_add_profile
      variables:
        ssid: string
        password: string
        priority: int
      then:
        - lambda: |-
            const bool ok = id(tank_net).add_or_update_profile(ssid, password, priority);
            ESP_LOGI("tankmonitor.net", "network_add_profile(%s): %s",
                     ssid.c_str(), ok ? "OK" : "FAILED");

    - action: network_remove_profile
      variables:
        ssid: string
      then:
        - lambda: |-
            const bool ok = id(tank_net).remove_profile(ssid);
            ESP_LOGI("tankmonitor.net", "network_remove_profile(%s): %s",
                     ssid.c_str(), ok ? "OK" : "NOT FOUND");

    - action: network_use_dhcp
      variables:
        ssid: string
      then:
        - lambda: |-
            const bool ok = id(tank_net).set_dhcp(ssid);
            ESP_LOGI("tankmonitor.net", "network_use_dhcp(%s): %s",
                     ssid.c_str(), ok ? "OK" : "NOT FOUND");

    - action: network_set_static
      variables:
        ssid: string
        static_ip: string
        gateway: string
        subnet: string
        dns1: string
        dns2: string
      then:
        - lambda: |-
            const bool ok = id(tank_net).set_static(
              ssid, static_ip, gateway, subnet, dns1, dns2
            );
            ESP_LOGI("tankmonitor.net",
                     "network_set_static(%s): %s - verrà validato al prossimo wake",
                     ssid.c_str(), ok ? "PENDING" : "FAILED");

    - action: network_scan
      then:
        - lambda: |-
            id(tank_net).start_scan();

    - action: network_clear_all_profiles
      then:
        - lambda: |-
            id(tank_net).clear_profiles();

ota:
  - platform: esphome
    password: !secret tankmonitor_ota_password

i2c:
  id: i2c_bus
  sda: GPIO22
  scl: GPIO23
  frequency: 400kHz
  scan: false

deep_sleep:
  id: deep_sleep_ctrl

  esp32_ext1_wakeup:
    pins:
      - GPIO1
    mode: ANY_HIGH

globals:
  - id: boot_millis
    type: uint32_t
    restore_value: no
    initial_value: '0'

  - id: sleep_minutes_value
    type: float
    restore_value: yes
    update_interval: 1s
    initial_value: '180.0'

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

switch:
  # D2 / GPIO2 -> Q2/Q1 -> boost + TL-136
  - platform: gpio
    id: sensor_power
    internal: true
    pin:
      number: GPIO2
      mode:
        output: true
    restore_mode: ALWAYS_OFF

  # D3 / GPIO21 -> abilita il divisore batteria
  - platform: gpio
    id: battery_adc_enable
    internal: true
    pin:
      number: GPIO21
      mode:
        output: true
    restore_mode: ALWAYS_OFF

  - platform: template
    id: measurement_enabled_control
    name: "Measurement Enabled"
    icon: "mdi:gauge"
    lambda: |-
      return id(measurement_enabled_value);

    turn_on_action:
      - lambda: |-
          id(measurement_enabled_value) = true;

    turn_off_action:
      - lambda: |-
          id(measurement_enabled_value) = false;
      - switch.turn_off: sensor_power

number:
  - platform: template
    id: sleep_minutes_control
    name: "Sleep Interval"
    icon: "mdi:sleep"
    unit_of_measurement: "min"
    min_value: 15
    max_value: 1440
    step: 15
    lambda: |-
      return id(sleep_minutes_value);
    set_action:
      - lambda: |-
          id(sleep_minutes_value) = x;
          ESP_LOGI("tankmonitor", "Sleep interval: %.0f min", x);

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
          ESP_LOGI("tankmonitor", "TL-136 warm-up: %.0f s", x);

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

  - platform: template
    id: static_ip_fallback
    name: "Static IP Fallback"
    entity_category: diagnostic
    lambda: |-
      return id(tank_net).is_static_fallback_active();

  - platform: template
    id: static_ip_pending
    name: "Static IP Pending"
    entity_category: diagnostic
    lambda: |-
      return id(tank_net).has_pending_static();

sensor:
  - platform: ina2xx_i2c
    id: ina228
    model: INA228
    i2c_id: i2c_bus
    address: 0x40

    shunt_resistance: 5.10 ohm
    max_current: 0.025 A
    adc_range: 0

    adc_averaging: 16
    adc_time: 1052us
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
  - platform: wifi_info
    ip_address:
      id: network_ip
      name: "Network IP"
      entity_category: diagnostic
    ssid:
      id: network_ssid
      name: "Network SSID"
      entity_category: diagnostic
    scan_results:
      id: network_scan_results
      name: "Available WiFi Networks"
      entity_category: diagnostic

  - platform: template
    id: network_mode
    name: "Network Mode"
    entity_category: diagnostic
    update_interval: never
    lambda: |-
      return id(tank_net).current_mode();

  - platform: template
    id: network_profiles
    name: "Network Profiles"
    entity_category: diagnostic
    update_interval: never
    lambda: |-
      return id(tank_net).profile_summary();

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
                format: "Force Measurement disponibile solo in maintenance mode"

  - platform: template
    id: sleep_now_button
    name: "Sleep Now"
    icon: "mdi:power-sleep"
    on_press:
      - script.execute: go_to_sleep

  - platform: template
    id: wifi_scan_button
    name: "Scan WiFi"
    icon: "mdi:wifi-find"
    on_press:
      - lambda: |-
          id(tank_net).start_scan();

script:
  # INA228 ADC_CONFIG (0x01): modifica solo MODE[15:12].
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

          cfg = (cfg & 0x0FFFu) | 0xF000u;

          const uint8_t out[2] = {
            static_cast<uint8_t>((cfg >> 8) & 0xFF),
            static_cast<uint8_t>(cfg & 0xFF)
          };

          err = id(ina228).write_register(0x01, out, 2);
          id(ina_mode_write_ok) = (err == esphome::i2c::ERROR_OK);

  - id: ina228_shutdown
    mode: restart
    then:
      - lambda: |-
          uint8_t data[2] = {0, 0};
          auto err = id(ina228).read_register(0x01, data, 2);

          if (err != esphome::i2c::ERROR_OK) {
            ESP_LOGW("tankmonitor", "INA228 shutdown read failed");
            return;
          }

          uint16_t cfg =
              (static_cast<uint16_t>(data[0]) << 8) |
              static_cast<uint16_t>(data[1]);

          cfg &= 0x0FFFu;

          const uint8_t out[2] = {
            static_cast<uint8_t>((cfg >> 8) & 0xFF),
            static_cast<uint8_t>(cfg & 0xFF)
          };

          id(ina228).write_register(0x01, out, 2);

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
            id(battery_low).publish_state(std::isfinite(pct) && pct <= 20.0f);
            id(battery_critical).publish_state(std::isfinite(pct) && pct <= 10.0f);
          }
      - switch.turn_off: battery_adc_enable

  - id: measure_tank
    mode: restart
    then:
      - if:
          condition:
            lambda: 'return id(measurement_enabled_value);'
          then:
            - script.execute: ina228_enable
            - script.wait: ina228_enable
            - delay: 100ms

            - switch.turn_on: sensor_power

            - delay: !lambda |-
                return static_cast<uint32_t>(
                  id(warmup_seconds_value) * 1000.0f
                );

            - component.update: ina228
            - delay: 300ms

            - switch.turn_off: sensor_power

            - lambda: |-
                const float ma = id(loop_current).state;

                if (!id(ina_mode_write_ok)) {
                  id(measurement_valid).publish_state(false);
                  id(sensor_fault).publish_state(true);
                  id(last_error).publish_state("INA228_CONFIG_ERROR");
                } else if (!std::isfinite(ma)) {
                  id(measurement_valid).publish_state(false);
                  id(sensor_fault).publish_state(true);
                  id(last_error).publish_state("INA228_NO_DATA");
                } else if (
                  ma < ${loop_min_valid_ma}f ||
                  ma > ${loop_max_valid_ma}f
                ) {
                  id(measurement_valid).publish_state(false);
                  id(sensor_fault).publish_state(true);
                  id(last_error).publish_state("LOOP_CURRENT_OUT_OF_RANGE");
                } else {
                  id(measurement_valid).publish_state(true);
                  id(sensor_fault).publish_state(false);
                  id(last_error).publish_state("OK");
                }

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

  - id: sample_cycle
    mode: restart
    then:
      - script.execute: measure_battery
      - script.wait: measure_battery
      - script.execute: measure_tank
      - script.wait: measure_tank

  - id: network_sync
    mode: restart
    then:
      - lambda: |-
          id(tank_net).prepare_normal();

      - wifi.enable

      # Con una config statica pending, tank_net può fare internamente
      # static -> DHCP fallback; per questo la finestra è un po' più ampia.
      - wait_until:
          condition:
            wifi.connected:
          timeout: 22s

      - if:
          condition:
            wifi.connected:
          then:
            - component.update: wifi_rssi

            - wait_until:
                condition:
                  api.connected:
                    state_subscription_only: true
                timeout: 20s

            - if:
                condition:
                  api.connected:
                    state_subscription_only: true
                then:
                  - lambda: |-
                      id(tank_net).confirm_backend_connected();
                      id(awake_time).publish_state(
                        (millis() - id(boot_millis)) / 1000.0f
                      );
                  - component.update: network_mode
                  - component.update: network_profiles
                  - delay: 3s
                else:
                  - lambda: |-
                      id(last_error).publish_state("HOME_ASSISTANT_API_TIMEOUT");
          else:
            - lambda: |-
                id(last_error).publish_state("WIFI_TIMEOUT");

      - wifi.disable

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

  - id: maintenance_cycle
    mode: restart
    then:
      - binary_sensor.template.publish:
          id: maintenance_mode
          state: ON

      - script.execute: sample_cycle
      - script.wait: sample_cycle

      - lambda: |-
          id(tank_net).prepare_maintenance();

      - wifi.enable

      # Se non esistono reti valide, dopo il timeout viene attivato
      # "TankMonitor Setup" e captive_portal consente di scegliere
      # una delle reti rilevate.
      - delay: 10min

      - if:
          condition:
            api.connected:
              state_subscription_only: true
          then:
            - lambda: |-
                id(tank_net).confirm_backend_connected();

      - binary_sensor.template.publish:
          id: maintenance_mode
          state: OFF
      - wifi.disable
      - script.execute: go_to_sleep

  - id: boot_coordinator
    mode: restart
    then:
      - switch.turn_off: sensor_power
      - switch.turn_off: battery_adc_enable

      - lambda: |-
          id(firmware_version).publish_state("${project_version}");

          const auto cause = esp_sleep_get_wakeup_cause();
          switch (cause) {
            case ESP_SLEEP_WAKEUP_EXT1:
              id(wakeup_reason).publish_state("MAGNET_EXT1");
              break;
            case ESP_SLEEP_WAKEUP_TIMER:
              id(wakeup_reason).publish_state("TIMER");
              break;
            case ESP_SLEEP_WAKEUP_UNDEFINED:
              id(wakeup_reason).publish_state("POWER_ON_OR_RESET");
              break;
            default:
              id(wakeup_reason).publish_state("OTHER");
              break;
          }

      - if:
          condition:
            lambda: |-
              const auto cause = esp_sleep_get_wakeup_cause();
              return cause == ESP_SLEEP_WAKEUP_EXT1 ||
                     cause == ESP_SLEEP_WAKEUP_UNDEFINED;
          then:
            - script.execute: maintenance_cycle
          else:
            - script.execute: normal_cycle

  - id: go_to_sleep
    mode: restart
    then:
      - switch.turn_off: sensor_power
      - switch.turn_off: battery_adc_enable

      - script.execute: ina228_shutdown
      - script.wait: ina228_shutdown

      - lambda: |-
          id(awake_time).publish_state(
            (millis() - id(boot_millis)) / 1000.0f
          );

      # Evita un loop immediato se il magnete è ancora sul reed.
      - if:
          condition:
            binary_sensor.is_on: maintenance_reed
          then:
            - wait_until:
                condition:
                  binary_sensor.is_off: maintenance_reed

      - delay: 100ms

      - deep_sleep.enter:
          id: deep_sleep_ctrl
          sleep_duration: !lambda |-
            if (!id(measurement_enabled_value)) {
              return static_cast<uint32_t>(
                24UL * 60UL * 60UL * 1000UL
              );
            }

            return static_cast<uint32_t>(
              id(sleep_minutes_value) * 60.0f * 1000.0f
            );

```

---

## Secrets template

```yaml
# Credenziali NON legate alla rete domestica.
# Generare valori propri prima del flash.

tankmonitor_api_key: "INSERIRE_CHIAVE_API_BASE64_32_BYTE"
tankmonitor_ota_password: "CAMBIARE_QUESTA_PASSWORD"
tankmonitor_setup_ap_password: "TankSetup123!"

```

---

## External component — __init__.py

```python
import esphome.codegen as cg
import esphome.config_validation as cv
from esphome.const import CONF_ID

DEPENDENCIES = ["wifi"]
CODEOWNERS = []

CONF_MAX_PROFILES = "max_profiles"
CONF_STATIC_VALIDATION_TIMEOUT = "static_validation_timeout"
CONF_DHCP_FALLBACK_TIMEOUT = "dhcp_fallback_timeout"
CONF_MAINTENANCE_AP_TIMEOUT = "maintenance_ap_timeout"

tankmonitor_network_ns = cg.esphome_ns.namespace("tankmonitor_network")
TankMonitorNetworkManager = tankmonitor_network_ns.class_(
    "TankMonitorNetworkManager", cg.Component
)

CONFIG_SCHEMA = cv.Schema(
    {
        cv.GenerateID(): cv.declare_id(TankMonitorNetworkManager),
        cv.Optional(CONF_MAX_PROFILES, default=5): cv.int_range(min=1, max=5),
        cv.Optional(
            CONF_STATIC_VALIDATION_TIMEOUT, default="8s"
        ): cv.positive_time_period_milliseconds,
        cv.Optional(
            CONF_DHCP_FALLBACK_TIMEOUT, default="10s"
        ): cv.positive_time_period_milliseconds,
        cv.Optional(
            CONF_MAINTENANCE_AP_TIMEOUT, default="15s"
        ): cv.positive_time_period_milliseconds,
    }
).extend(cv.COMPONENT_SCHEMA)


async def to_code(config):
    # WiFiAP::set_manual_ip() è compilato condizionalmente nel core.
    cg.add_define("USE_WIFI_MANUAL_IP")

    var = cg.new_Pvariable(config[CONF_ID])
    await cg.register_component(var, config)

    cg.add(var.set_max_profiles(config[CONF_MAX_PROFILES]))
    cg.add(
        var.set_static_validation_timeout(
            config[CONF_STATIC_VALIDATION_TIMEOUT].total_milliseconds
        )
    )
    cg.add(
        var.set_dhcp_fallback_timeout(
            config[CONF_DHCP_FALLBACK_TIMEOUT].total_milliseconds
        )
    )
    cg.add(
        var.set_maintenance_ap_timeout(
            config[CONF_MAINTENANCE_AP_TIMEOUT].total_milliseconds
        )
    )

```

---

## External component — tankmonitor_network.h

```cpp
#pragma once

#include <algorithm>
#include <cstdint>
#include <cstring>
#include <string>

#include "esphome/components/network/ip_address.h"
#include "esphome/components/wifi/wifi_component.h"
#include "esphome/core/component.h"
#include "esphome/core/log.h"
#include "esphome/core/preferences.h"

namespace esphome {
namespace tankmonitor_network {

static const char *const TAG = "tankmonitor_network";

class TankMonitorNetworkManager : public Component {
 public:
  void set_max_profiles(uint8_t value) {
    this->max_profiles_ = std::min<uint8_t>(value, MAX_PROFILES);
  }

  void set_static_validation_timeout(uint32_t value) {
    this->static_validation_timeout_ms_ = value;
  }

  void set_dhcp_fallback_timeout(uint32_t value) {
    this->dhcp_fallback_timeout_ms_ = value;
  }

  void set_maintenance_ap_timeout(uint32_t value) {
    this->maintenance_ap_timeout_ms_ = value;
  }

  float get_setup_priority() const override {
    // WiFi ha enable_on_boot=false: carichiamo i profili dopo il setup
    // del componente WiFi ma prima delle automazioni on_boot.
    return setup_priority::AFTER_WIFI;
  }

  void setup() override {
    this->pref_ = global_preferences->make_preference<StoredConfig>(
        PREF_KEY, true
    );

    if (!this->pref_.load(&this->config_) ||
        this->config_.magic != CONFIG_MAGIC ||
        this->config_.version != CONFIG_VERSION) {
      this->reset_config_();
      ESP_LOGI(TAG, "No TankMonitor network profiles stored");
      // Non tocchiamo le STA del core: in questo modo eventuali
      // credenziali salvate dal captive portal / wifi.configure restano valide.
      return;
    }

    this->sanitize_config_();

    const int pending = this->find_pending_static_();
    if (pending >= 0) {
      this->test_index_ = pending;
      this->test_stage_ = TestStage::STATIC;
      this->rebuild_wifi_profiles_();
      ESP_LOGI(TAG, "Pending static IPv4 test for SSID '%s'",
               this->config_.profiles[pending].ssid);
    } else {
      this->rebuild_wifi_profiles_();
    }
  }

  void loop() override {
    if (this->test_stage_ == TestStage::NONE)
      return;

    auto *wifi = wifi::global_wifi_component;
    if (wifi == nullptr || wifi->is_disabled()) {
      this->attempt_started_ms_ = 0;
      return;
    }

    if (this->attempt_started_ms_ == 0) {
      this->attempt_started_ms_ = millis();
      return;
    }

    const uint32_t elapsed = millis() - this->attempt_started_ms_;

    if (this->test_stage_ == TestStage::STATIC &&
        elapsed >= this->static_validation_timeout_ms_) {
      ESP_LOGW(TAG, "Static IPv4 validation timed out; retrying same SSID with DHCP");
      this->mark_static_failed_before_fallback_();
      this->test_stage_ = TestStage::DHCP;
      this->attempt_started_ms_ = 0;
      this->restart_wifi_for_current_stage_();
      return;
    }

    if (this->test_stage_ == TestStage::DHCP &&
        elapsed >= this->dhcp_fallback_timeout_ms_) {
      ESP_LOGW(TAG, "DHCP fallback test timed out; restoring all known profiles");
      this->test_stage_ = TestStage::NONE;
      this->test_index_ = -1;
      this->attempt_started_ms_ = 0;
      this->restart_wifi_all_profiles_();
    }
  }

  void dump_config() override {
    ESP_LOGCONFIG(TAG, "TankMonitor Network Manager:");
    ESP_LOGCONFIG(TAG, "  Stored profiles: %u", this->profile_count_());
    ESP_LOGCONFIG(TAG, "  Max profiles: %u", this->max_profiles_);
    ESP_LOGCONFIG(TAG, "  Static validation timeout: %u ms",
                  this->static_validation_timeout_ms_);
    ESP_LOGCONFIG(TAG, "  DHCP fallback timeout: %u ms",
                  this->dhcp_fallback_timeout_ms_);
    // Password deliberatamente NON loggate.
  }

  // Chiamato prima di un wake normale: niente AP di setup automatico.
  void prepare_normal() {
    if (wifi::global_wifi_component != nullptr)
      wifi::global_wifi_component->set_ap_timeout(0);
  }

  // Chiamato in maintenance: se le reti note falliscono, il fallback AP
  // parte dopo maintenance_ap_timeout_ms_.
  void prepare_maintenance() {
    if (wifi::global_wifi_component != nullptr)
      wifi::global_wifi_component->set_ap_timeout(
          this->maintenance_ap_timeout_ms_
      );
  }

  bool add_or_update_profile(const std::string &ssid,
                             const std::string &password,
                             int priority) {
    if (ssid.empty() || ssid.size() > 32 || password.size() > 64)
      return false;

    int idx = this->find_profile_(ssid);
    if (idx < 0) {
      idx = this->find_free_slot_();
      if (idx < 0)
        return false;
      std::memset(&this->config_.profiles[idx], 0, sizeof(Profile));
      this->config_.profiles[idx].enabled = 1;
    }

    auto &p = this->config_.profiles[idx];
    copy_string_(p.ssid, sizeof(p.ssid), ssid);
    copy_string_(p.password, sizeof(p.password), password);
    p.priority = static_cast<int8_t>(
        std::max(-100, std::min(100, priority))
    );

    this->save_();
    return true;
  }

  bool remove_profile(const std::string &ssid) {
    const int idx = this->find_profile_(ssid);
    if (idx < 0)
      return false;

    std::memset(&this->config_.profiles[idx], 0, sizeof(Profile));
    this->save_();
    return true;
  }

  bool set_dhcp(const std::string &ssid) {
    const int idx = this->find_profile_(ssid);
    if (idx < 0)
      return false;

    auto &p = this->config_.profiles[idx];
    p.use_static = 0;
    p.static_pending = 0;
    p.static_validated = 0;
    p.static_failed = 0;

    this->save_();
    return true;
  }

  bool set_static(const std::string &ssid,
                  const std::string &static_ip,
                  const std::string &gateway,
                  const std::string &subnet,
                  const std::string &dns1,
                  const std::string &dns2) {
    const int idx = this->find_profile_(ssid);
    if (idx < 0)
      return false;

    if (!valid_required_ip_(static_ip) ||
        !valid_required_ip_(gateway) ||
        !valid_required_ip_(subnet) ||
        !valid_optional_ip_(dns1) ||
        !valid_optional_ip_(dns2)) {
      ESP_LOGW(TAG, "Invalid IPv4 configuration for SSID '%s'", ssid.c_str());
      return false;
    }

    auto &p = this->config_.profiles[idx];
    p.use_static = 1;
    p.static_pending = 1;
    p.static_validated = 0;
    p.static_failed = 0;

    copy_string_(p.static_ip, sizeof(p.static_ip), static_ip);
    copy_string_(p.gateway, sizeof(p.gateway), gateway);
    copy_string_(p.subnet, sizeof(p.subnet), subnet);
    copy_string_(p.dns1, sizeof(p.dns1), dns1);
    copy_string_(p.dns2, sizeof(p.dns2), dns2);

    // Non interrompiamo la sessione corrente.
    // La configurazione verrà testata al prossimo wake/reboot.
    this->save_();
    return true;
  }

  void clear_profiles() {
    this->reset_config_();
    this->save_();

    // Non cancelliamo qui le credenziali interne di ESPHome/captive portal.
    // Il recovery locale resta quindi disponibile.
  }

  void start_scan() {
    auto *wifi = wifi::global_wifi_component;
    if (wifi != nullptr && !wifi->is_disabled())
      wifi->start_scanning();
  }

  // Da chiamare SOLO dopo che Home Assistant è realmente connesso
  // (api.connected state_subscription_only).
  void confirm_backend_connected() {
    auto *wifi = wifi::global_wifi_component;
    if (wifi == nullptr || !wifi->is_connected())
      return;

    this->adopt_current_network_if_needed_();

    if (this->test_stage_ == TestStage::NONE)
      return;

    if (this->test_index_ < 0 || this->test_index_ >= MAX_PROFILES)
      return;

    auto &p = this->config_.profiles[this->test_index_];

    if (this->test_stage_ == TestStage::STATIC) {
      ESP_LOGI(TAG, "Static IPv4 validated for '%s'", p.ssid);
      p.static_pending = 0;
      p.static_validated = 1;
      p.static_failed = 0;
    } else if (this->test_stage_ == TestStage::DHCP) {
      ESP_LOGW(TAG, "DHCP fallback validated for '%s'; static settings retained for editing", p.ssid);
      p.static_pending = 0;
      p.static_validated = 0;
      p.static_failed = 1;
    }

    this->save_();
    this->test_stage_ = TestStage::NONE;
    this->test_index_ = -1;
    this->attempt_started_ms_ = 0;
  }

  bool is_static_fallback_active() const {
    for (uint8_t i = 0; i < this->max_profiles_; i++) {
      const auto &p = this->config_.profiles[i];
      if (p.enabled && p.use_static && p.static_failed)
        return true;
    }
    return false;
  }

  bool has_pending_static() const {
    return this->find_pending_static_() >= 0;
  }

  std::string current_mode() const {
    auto *wifi = wifi::global_wifi_component;
    if (wifi == nullptr || !wifi->is_connected())
      return "OFFLINE";

    const wifi::WiFiAP ap = wifi->get_sta();
    const std::string ssid(ap.get_ssid().c_str());
    const int idx = this->find_profile_(ssid);

    if (idx < 0)
      return "DHCP";

    const auto &p = this->config_.profiles[idx];

    if (p.use_static && p.static_failed)
      return "DHCP_FALLBACK";
    if (p.use_static && (p.static_validated || p.static_pending))
      return "STATIC";

    return "DHCP";
  }

  std::string profile_summary() const {
    std::string out;
    out.reserve(256);

    for (uint8_t i = 0; i < this->max_profiles_; i++) {
      const auto &p = this->config_.profiles[i];
      if (!p.enabled)
        continue;

      if (!out.empty())
        out += " | ";

      out += p.ssid;
      out += " [p=";
      out += std::to_string(static_cast<int>(p.priority));

      if (!p.use_static) {
        out += ",DHCP]";
      } else if (p.static_failed) {
        out += ",DHCP_FALLBACK,static=";
        out += p.static_ip;
        out += "]";
      } else if (p.static_pending) {
        out += ",STATIC_PENDING,";
        out += p.static_ip;
        out += "]";
      } else {
        out += ",STATIC,";
        out += p.static_ip;
        out += "]";
      }
    }

    if (out.empty())
      return "Captive-portal/default WiFi only";

    return out;
  }

 protected:
  static constexpr uint8_t MAX_PROFILES = 5;
  static constexpr uint32_t PREF_KEY = 0x544D4E32;  // "TMN2"
  static constexpr uint32_t CONFIG_MAGIC = 0x544D4E32;
  static constexpr uint16_t CONFIG_VERSION = 1;

  struct Profile {
    uint8_t enabled;
    int8_t priority;

    uint8_t use_static;
    uint8_t static_pending;
    uint8_t static_validated;
    uint8_t static_failed;

    char ssid[33];
    char password[65];

    char static_ip[16];
    char gateway[16];
    char subnet[16];
    char dns1[16];
    char dns2[16];
  };

  struct StoredConfig {
    uint32_t magic;
    uint16_t version;
    uint16_t reserved;
    Profile profiles[MAX_PROFILES];
  };

  enum class TestStage : uint8_t {
    NONE = 0,
    STATIC = 1,
    DHCP = 2,
  };

  ESPPreferenceObject pref_;
  StoredConfig config_{};

  uint8_t max_profiles_{MAX_PROFILES};
  uint32_t static_validation_timeout_ms_{8000};
  uint32_t dhcp_fallback_timeout_ms_{10000};
  uint32_t maintenance_ap_timeout_ms_{15000};

  TestStage test_stage_{TestStage::NONE};
  int8_t test_index_{-1};
  uint32_t attempt_started_ms_{0};

  void reset_config_() {
    std::memset(&this->config_, 0, sizeof(this->config_));
    this->config_.magic = CONFIG_MAGIC;
    this->config_.version = CONFIG_VERSION;
  }

  void sanitize_config_() {
    for (uint8_t i = 0; i < MAX_PROFILES; i++) {
      auto &p = this->config_.profiles[i];
      p.ssid[sizeof(p.ssid) - 1] = '\0';
      p.password[sizeof(p.password) - 1] = '\0';
      p.static_ip[sizeof(p.static_ip) - 1] = '\0';
      p.gateway[sizeof(p.gateway) - 1] = '\0';
      p.subnet[sizeof(p.subnet) - 1] = '\0';
      p.dns1[sizeof(p.dns1) - 1] = '\0';
      p.dns2[sizeof(p.dns2) - 1] = '\0';
    }
  }

  void save_() {
    this->config_.magic = CONFIG_MAGIC;
    this->config_.version = CONFIG_VERSION;

    if (!this->pref_.save(&this->config_))
      ESP_LOGW(TAG, "Failed to save network configuration");
  }

  static void copy_string_(char *dest, size_t size,
                           const std::string &value) {
    if (size == 0)
      return;
    std::strncpy(dest, value.c_str(), size - 1);
    dest[size - 1] = '\0';
  }

  static bool valid_required_ip_(const std::string &value) {
    if (value.empty())
      return false;
    const network::IPAddress ip(value);
    return ip.is_set();
  }

  static bool valid_optional_ip_(const std::string &value) {
    if (value.empty())
      return true;
    const network::IPAddress ip(value);
    return ip.is_set();
  }

  uint8_t profile_count_() const {
    uint8_t count = 0;
    for (uint8_t i = 0; i < this->max_profiles_; i++)
      if (this->config_.profiles[i].enabled)
        count++;
    return count;
  }

  int find_profile_(const std::string &ssid) const {
    for (uint8_t i = 0; i < this->max_profiles_; i++) {
      const auto &p = this->config_.profiles[i];
      if (p.enabled && ssid == p.ssid)
        return i;
    }
    return -1;
  }

  int find_free_slot_() const {
    for (uint8_t i = 0; i < this->max_profiles_; i++)
      if (!this->config_.profiles[i].enabled)
        return i;
    return -1;
  }

  int find_pending_static_() const {
    for (uint8_t i = 0; i < this->max_profiles_; i++) {
      const auto &p = this->config_.profiles[i];
      if (p.enabled && p.use_static &&
          p.static_pending && !p.static_failed)
        return i;
    }
    return -1;
  }

  wifi::WiFiAP make_ap_(const Profile &p, bool force_dhcp) const {
    wifi::WiFiAP ap;
    ap.set_ssid(p.ssid);
    ap.set_password(p.password);
    ap.set_priority(p.priority);

#ifdef USE_WIFI_MANUAL_IP
    if (p.use_static && !p.static_failed && !force_dhcp) {
      wifi::ManualIP manual;
      manual.static_ip = network::IPAddress(p.static_ip);
      manual.gateway = network::IPAddress(p.gateway);
      manual.subnet = network::IPAddress(p.subnet);
      manual.dns1 = p.dns1[0] ? network::IPAddress(p.dns1) : network::IPAddress();
      manual.dns2 = p.dns2[0] ? network::IPAddress(p.dns2) : network::IPAddress();
      ap.set_manual_ip(manual);
    }
#endif

    return ap;
  }

  void rebuild_wifi_profiles_() {
    auto *wifi = wifi::global_wifi_component;
    if (wifi == nullptr)
      return;

    // Durante un test STATIC/DHCP proviamo intenzionalmente solo
    // il profilo interessato, così il risultato non è ambiguo.
    if (this->test_stage_ != TestStage::NONE &&
        this->test_index_ >= 0 &&
        this->test_index_ < this->max_profiles_) {
      const auto &p = this->config_.profiles[this->test_index_];
      if (p.enabled) {
        wifi->clear_sta();
        wifi->init_sta(1);
        wifi->add_sta(this->make_ap_(
            p, this->test_stage_ == TestStage::DHCP
        ));
      }
      return;
    }

    const uint8_t count = this->profile_count_();

    // Se non abbiamo profili nostri NON cancelliamo la STA eventualmente
    // salvata dal captive portal/wifi.configure.
    if (count == 0)
      return;

    wifi->clear_sta();
    wifi->init_sta(count);

    for (uint8_t i = 0; i < this->max_profiles_; i++) {
      const auto &p = this->config_.profiles[i];
      if (!p.enabled)
        continue;
      wifi->add_sta(this->make_ap_(p, false));
    }
  }

  void restart_wifi_for_current_stage_() {
    auto *wifi = wifi::global_wifi_component;
    if (wifi == nullptr)
      return;

    wifi->disable();

    this->set_timeout("tm_net_restart", 250, [this]() {
      this->rebuild_wifi_profiles_();
      if (wifi::global_wifi_component != nullptr)
        wifi::global_wifi_component->enable();
    });
  }

  void restart_wifi_all_profiles_() {
    auto *wifi = wifi::global_wifi_component;
    if (wifi == nullptr)
      return;

    wifi->disable();

    this->set_timeout("tm_net_restart_all", 250, [this]() {
      this->rebuild_wifi_profiles_();
      if (wifi::global_wifi_component != nullptr)
        wifi::global_wifi_component->enable();
    });
  }

  void mark_static_failed_before_fallback_() {
    if (this->test_index_ < 0 ||
        this->test_index_ >= this->max_profiles_)
      return;

    auto &p = this->config_.profiles[this->test_index_];
    p.static_pending = 0;
    p.static_validated = 0;
    p.static_failed = 1;

    // Salviamo PRIMA del tentativo DHCP: anche un power-loss non
    // farà rientrare il nodo in un loop con lo static IP rotto.
    this->save_();
  }

  void adopt_current_network_if_needed_() {
    auto *wifi = wifi::global_wifi_component;
    if (wifi == nullptr || !wifi->is_connected())
      return;

    const wifi::WiFiAP current = wifi->get_sta();
    const std::string ssid(current.get_ssid().c_str());

    if (ssid.empty() || this->find_profile_(ssid) >= 0)
      return;

    const std::string password(current.get_password().c_str());

    if (this->find_free_slot_() < 0)
      return;

    if (this->add_or_update_profile(ssid, password, 0)) {
      ESP_LOGI(TAG,
               "Adopted captive-portal/runtime WiFi '%s' into TankMonitor profiles",
               ssid.c_str());
    }
  }
};

}  // namespace tankmonitor_network
}  // namespace esphome

```

---

## Manifest

```json
{
  "revision": "2.3.0",
  "transport": "ESPHome Native API / Home Assistant",
  "network_hardcoded": false,
  "max_wifi_profiles": 5,
  "sleep_minutes": {
    "min": 15,
    "max": 1440,
    "step": 15,
    "default": 180
  }
}
```
