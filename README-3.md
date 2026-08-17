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
