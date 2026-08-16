# TankMonitor — Firmware REV2.3
## Due build ufficiali: Home Assistant / MQTT

**Hardware:** TankMonitor REV2.0 TL-136  
**Firmware:** REV2.3  
**MCU:** Seeed Studio XIAO ESP32-C6  
**Sensore:** TL-136 4–20 mA  
**Current monitor:** INA228  
**Alimentazione:** LiPo 1S 3.7 V  
**Obiettivo:** ultra-low-power, configurabile, fault-tolerant, senza credenziali di rete scolpite nel firmware.

---

# 1. Stato ufficiale

La REV2.3 definisce due firmware ufficiali alternativi:

```text
TankMonitor-HA
```

e:

```text
TankMonitor-MQTT
```

Non viene utilizzato ESP-NOW.

Le due versioni condividono:

- hardware;
- misura TL-136;
- INA228;
- misura batteria;
- deep sleep;
- wake magnetico;
- configurazione sleep;
- Network Manager;
- captive setup;
- multi-Wi-Fi;
- DHCP;
- IP statico runtime;
- rollback automatico a DHCP;
- configurazione persistente;
- diagnostica;
- OTA/manutenzione.

Cambia esclusivamente il layer di trasporto:

```text
TankMonitor-HA
    ↓
ESPHome Native API
    ↓
Home Assistant
```

oppure:

```text
TankMonitor-MQTT
    ↓
MQTT
    ↓
Broker
```

Le due modalità **non devono funzionare contemporaneamente nello stesso build**.

---

# 2. Perché due build separate

Non vogliamo:

```text
Native API + MQTT + Web Server
```

tutti attivi nello stesso ciclo.

Questo introdurrebbe:

- più RAM;
- più socket;
- più codice attivo;
- più handshake;
- maggiore tempo radio;
- maggiore complessità;
- più possibilità di failure.

Il consumo energetico è dominato principalmente da:

```text
tempo Wi-Fi acceso
+
associazione
+
DHCP/static IP
+
handshake API/MQTT
```

non dalla sola dimensione del firmware in Flash.

Quindi la strategia ufficiale è:

```text
un solo transport per build
```

---

# 3. Componenti comuni

Entrambi i build implementano:

```text
TankMonitor Core
TankNetworkManager
TankSetupPortal
```

Architettura:

```text
                 TankMonitor Core
        ┌────────────────────────────┐
        │ TL-136                     │
        │ INA228                     │
        │ Batteria                   │
        │ Deep sleep                 │
        │ Configurazione             │
        │ Diagnostica                │
        └──────────────┬─────────────┘
                       │
                       ▼
             TankNetworkManager
                       │
        ┌──────────────┼──────────────┐
        │              │              │
     Wi-Fi         DHCP/static      Setup Portal
    profiles        fallback          on demand
        │
        ▼
     Transport
        │
   ┌────┴─────┐
   │          │
HA build   MQTT build
```

---

# 4. Configurazione rete

Nessun build contiene hard-coded:

- SSID;
- password Wi-Fi;
- IP statico;
- gateway;
- subnet;
- DNS;
- broker MQTT;
- username MQTT;
- password MQTT.

Il provisioning avviene tramite:

```text
TankSetupPortal
```

attivo soltanto:

- al primo setup;
- in maintenance mode;
- in recovery.

---

# 5. Profili Wi-Fi

TankNetworkManager salva fino a:

```text
5 profili
```

Ogni profilo:

```text
profile_id
ssid
password
priority

ipv4_mode:
    DHCP
    STATIC

static_ip
gateway
subnet
dns1
dns2

last_success
last_rssi
validated
```

Le password non vengono mai esposte come entità.

---

# 6. Selezione Wi-Fi

Ordine:

```text
1. Last Known Good
2. profili visibili per priorità
3. a parità di priorità: RSSI migliore
```

Se Last Known Good funziona, non viene effettuata una scansione completa.

Questo riduce:

- tempo radio;
- consumo;
- latenza.

---

# 7. IP statico fault-tolerant

Una nuova configurazione statica viene considerata:

```text
PENDING
```

fino a quando non è stata verificata anche la connettività al backend.

## TankMonitor-HA

Validazione:

```text
Wi-Fi
+
IP
+
Home Assistant Native API
```

## TankMonitor-MQTT

Validazione:

```text
Wi-Fi
+
IP
+
connessione MQTT broker
```

Se fallisce:

```text
STATIC FAILED
↓
stesso SSID
↓
DHCP
↓
backend reconnect
```

Se DHCP funziona:

```text
network_mode = DHCP_FALLBACK
```

Il dispositivo resta gestibile.

---

# 8. Last Known Good

Viene memorizzato:

```text
last_good_profile
last_good_ipv4_mode
```

Una configurazione diventa `GOOD` soltanto dopo backend handshake riuscito.

---

# 9. Setup Portal

Non viene utilizzato il `web_server` completo ESPHome.

Viene implementato un piccolo:

```text
TankSetupPortal
```

attivo esclusivamente durante setup/manutenzione.

Normalmente:

```text
HTTP OFF
AP OFF
Wi-Fi OFF
deep sleep
```

Setup:

```text
magnete
↓
maintenance
↓
TankMonitor Setup AP
↓
mini HTTP portal
```

Timeout default:

```text
10 minuti
```

---

# 10. Setup Portal — sezione Network

UI indicativa:

```text
TankMonitor Setup

NETWORK

Available Wi-Fi:
[ Casa Sardegna ▼ ]

Password:
[ *************** ]

Priority:
[ 100 ]

IPv4:
(*) DHCP
( ) Static

Static IP:
[               ]

Gateway:
[               ]

Subnet:
[               ]

DNS1:
[               ]

[ TEST & SAVE ]
```

---

# 11. Build ufficiale A — TankMonitor-HA

Firmware ottimizzato per:

```text
Home Assistant + ESPHome Native API
```

Non contiene MQTT.

Sequenza:

```text
wake
↓
battery
↓
TL-136 measure
↓
TL-136 OFF
↓
Wi-Fi
↓
Native API
↓
Home Assistant
↓
sync config
↓
send states
↓
Wi-Fi OFF
↓
sleep
```

---

# 12. Configurazione HA

Lato Home Assistant sono configurabili:

```text
Sleep Interval
TL-136 Warm-up
Measurement Enabled
```

Sleep:

```text
range: 15–1440 min
step: 15 min
default: 180 min
```

Warm-up:

```text
1–15 s
default: 3 s
```

---

# 13. HA API actions

La build HA implementa almeno:

```text
set_sleep_minutes(minutes)
set_warmup_seconds(seconds)
set_measurement_enabled(enabled)
force_measurement()
```

La futura HACS mantiene:

```text
desired config
reported config
```

perché il nodo può dormire quando l'utente modifica un parametro.

---

# 14. Entità TankMonitor-HA

Sensori:

```text
Loop Current
Battery Voltage
Battery Level
WiFi RSSI
Awake Time
```

Binary:

```text
Measurement Valid
Sensor Fault
Battery Low
Battery Critical
Maintenance Mode
Static IP Fallback
```

Text:

```text
Wakeup Reason
Last Error
Network SSID
Network IP
Network Mode
Network Status
Firmware Version
```

---

# 15. Build ufficiale B — TankMonitor-MQTT

La build MQTT non contiene Native API.

Sequenza:

```text
wake
↓
battery
↓
TL-136 measure
↓
TL-136 OFF
↓
Wi-Fi
↓
MQTT connect
↓
receive desired config
↓
publish telemetry
↓
publish reported config
↓
publish sleeping
↓
disconnect
↓
Wi-Fi OFF
↓
sleep
```

---

# 16. Setup MQTT

La pagina Setup aggiunge:

```text
TRANSPORT: MQTT

Broker:
[ mqtt.local ]

Port:
[ 1883 ]

Username:
[ tankmonitor ]

Password:
[ *************** ]

TLS:
[ OFF ]

Topic prefix:
[ tankmonitor ]

Device ID:
[ cisterna-sardegna ]

[ TEST MQTT ]
```

Questi parametri sono salvati in NVS.

Non vengono scolpiti nello YAML.

---

# 17. MQTT topic root

Default:

```text
tankmonitor/<device_id>
```

Esempio:

```text
tankmonitor/cisterna-sardegna
```

---

# 18. MQTT telemetry

Topic:

```text
tankmonitor/cisterna-sardegna/telemetry
```

Payload:

```json
{
  "timestamp": 0,
  "loop_ma": 11.842,
  "battery_v": 3.91,
  "battery_pct": 74,
  "measurement_valid": true,
  "sensor_fault": false,
  "rssi": -61,
  "awake_time_s": 4.3
}
```

Configurazione:

```text
QoS = 1
retain = true
```

Il timestamp può essere:

- epoch, se disponibile;
- 0/null se il nodo non ha ancora sincronizzato il tempo.

---

# 19. MQTT desired configuration

Topic:

```text
tankmonitor/<device_id>/config/desired
```

Retained JSON:

```json
{
  "revision": 17,
  "sleep_minutes": 180,
  "warmup_seconds": 3,
  "measurement_enabled": true
}
```

Il broker conserva la configurazione anche mentre TankMonitor dorme.

---

# 20. MQTT reported configuration

Topic:

```text
tankmonitor/<device_id>/config/reported
```

Payload:

```json
{
  "revision": 17,
  "sleep_minutes": 180,
  "warmup_seconds": 3,
  "measurement_enabled": true,
  "status": "applied"
}
```

Configurazione:

```text
retain = true
QoS = 1
```

---

# 21. Config revision

Il nodo memorizza:

```text
last_applied_revision
```

Quando riceve:

```text
revision <= last_applied_revision
```

ignora il comando.

Quando riceve:

```text
revision > last_applied_revision
```

applica la configurazione e pubblica `reported`.

---

# 22. Stato MQTT

Topic:

```text
tankmonitor/<device_id>/status
```

Stati:

```text
online
sleeping
lost
setup
```

Normal flow:

```text
connect
↓
online
↓
telemetry
↓
sleeping
↓
disconnect
```

Last Will:

```text
lost
```

---

# 23. MQTT events

Topic:

```text
tankmonitor/<device_id>/event
```

Esempio:

```json
{
  "event": "sensor_fault",
  "loop_ma": 0.12
}
```

oppure:

```json
{
  "event": "battery_low",
  "battery_v": 3.48
}
```

Gli eventi non sostituiscono la telemetry retained.

---

# 24. MQTT broker fault tolerance

Timeout:

```text
Wi-Fi connect:   10 s
MQTT connect:    10 s
```

Se il broker non risponde:

```text
MQTT FAIL
↓
Wi-Fi OFF
↓
deep sleep
```

Non deve esistere un retry infinito.

---

# 25. Static IP validation MQTT

Con IP statico pending:

```text
STATIC IP
↓
Wi-Fi association
↓
MQTT broker connect
```

Se MQTT connect riesce:

```text
STATIC VALIDATED
```

Se fallisce:

```text
disconnect
↓
DHCP
↓
same Wi-Fi
↓
MQTT connect
```

Se DHCP riesce:

```text
DHCP_FALLBACK
```

---

# 26. Cambi broker

Il broker MQTT NON viene modificato tramite MQTT.

Per cambiare:

- broker;
- porta;
- username;
- password;
- TLS;
- topic root;

si usa:

```text
Setup Portal
```

Questo evita loop di recovery molto complessi.

---

# 27. Parametri modificabili via MQTT

Sono modificabili via `config/desired`:

```text
sleep_minutes
warmup_seconds
measurement_enabled
```

Possibili parametri futuri:

```text
sample_count
diagnostic_level
measurement_policy
```

Non sono modificabili via MQTT:

```text
Wi-Fi credentials
IPv4
MQTT broker credentials
```

Questi restano infrastrutturali e richiedono Setup Portal.

---

# 28. MQTT e Home Assistant

La build MQTT può comunque essere integrata in Home Assistant tramite:

```text
MQTT integration
```

oppure tramite futura HACS `Water Tank Monitor`.

La HACS può ascoltare:

```text
telemetry
config/reported
status
event
```

e pubblicare:

```text
config/desired
```

Quindi:

```text
TankMonitor-MQTT
↓
Mosquitto
↓
Home Assistant
```

rimane perfettamente supportato.

---

# 29. Configurazione senza Home Assistant

La build MQTT può funzionare anche con:

```text
Node-RED
OpenHAB
Domoticz
custom backend
Python service
Grafana/Telegraf pipeline
```

purché esista un broker MQTT.

---

# 30. Deep sleep

Entrambe le build condividono:

```text
Sleep Interval:
15–1440 min

Default:
180 min

Step:
15 min
```

Se:

```text
measurement_enabled = false
```

wake di servizio:

```text
ogni 24 ore
```

---

# 31. Maintenance mode

Reed magnetico:

```text
MAGNET
↓
wake
↓
maintenance mode
↓
network
↓
setup / OTA / diagnostics
```

Timeout:

```text
10 min
```

Se Wi-Fi non disponibile:

```text
TankMonitor Setup AP
```

---

# 32. Recovery mode

Magnete tenuto presente:

```text
> 10 s
```

può attivare:

```text
NETWORK RECOVERY
```

Cancella:

```text
Wi-Fi profiles
network settings
```

NON cancella:

```text
TL-136 calibration
sleep
sensor parameters
tank parameters
```

---

# 33. Factory reset

Separato da Network Recovery.

Factory reset cancella:

```text
network
transport config
runtime settings
calibration
```

Deve richiedere:

- azione esplicita;
- conferma;
- oppure sequenza fisica dedicata.

---

# 34. Persistenza

Configurazioni in NVS:

```text
DeviceConfig
NetworkProfiles[5]
NetworkState
```

Build MQTT aggiunge:

```text
MQTTConfig
```

Scritture Flash soltanto in caso di variazione reale.

---

# 35. Config structure comune

Esempio logico:

```cpp
struct DeviceConfig {
  uint16_t version;
  uint16_t sleep_minutes;
  uint8_t warmup_seconds;
  bool measurement_enabled;
};
```

---

# 36. Network profile structure

```cpp
struct NetworkProfile {
  bool enabled;
  char ssid[33];
  char password[65];
  int priority;

  bool use_static_ip;

  uint32_t static_ip;
  uint32_t gateway;
  uint32_t subnet;
  uint32_t dns1;
  uint32_t dns2;

  bool validated;
};
```

Le dimensioni reali dovranno essere definite con attenzione per NVS e sicurezza.

---

# 37. MQTT config structure

```cpp
struct MQTTConfig {
  char host[128];
  uint16_t port;

  char username[64];
  char password[128];

  bool tls;

  char topic_prefix[64];
  char device_id[64];
};
```

---

# 38. Sicurezza credenziali

Password:

```text
NON loggate
NON pubblicate
NON esposte come sensor/text_sensor
```

Setup portal:

```text
solo maintenance/setup
timeout 10 min
```

Dove possibile:

- API encryption per HA;
- MQTT TLS per deployment remoti/non trusted;
- broker LAN senza TLS ammesso per semplicità se la rete è controllata.

---

# 39. Consumi

La presenza del codice MQTT o HA in Flash non è il principale problema energetico.

Conta:

```text
T_active =
T_measure
+
T_wifi
+
T_transport
```

Il consumo medio è approssimativamente:

```text
Iavg =
(Isleep × Tsleep + Iactive × Tactive)
/
(Tsleep + Tactive)
```

Quindi bisogna minimizzare:

```text
T_wifi
T_transport
```

---

# 40. Ottimizzazioni rete

Per entrambe:

```text
Last Known Good
fast reconnection
RTC BSSID/channel cache
static IP se validato
timeout brevi
no AP durante wake normale
```

Una configurazione static IP corretta può ridurre la latenza eliminando DHCP.

Se è errata:

```text
automatic DHCP fallback
```

---

# 41. Scelta consigliata

Per una casa con Home Assistant:

```text
TankMonitor-HA
```

è la scelta predefinita.

Vantaggi:

- integrazione nativa;
- meno configurazione;
- gestione diretta entità;
- API dedicate;
- OTA ESPHome;
- HACS più semplice.

Per impianti senza Home Assistant:

```text
TankMonitor-MQTT
```

è la scelta consigliata.

Vantaggi:

- backend indipendente;
- protocollo standard;
- retained config;
- desired/reported;
- facile integrazione con software differenti.

---

# 42. Repository structure

```text
TankMonitor/
│
├── firmware/
│   ├── common/
│   │   ├── tank_core.*
│   │   ├── tank_network_manager.*
│   │   ├── tank_setup_portal.*
│   │   └── tank_config.*
│   │
│   ├── ha/
│   │   └── tankmonitor-ha.yaml
│   │
│   └── mqtt/
│       ├── tankmonitor-mqtt.yaml
│       └── tank_mqtt_transport.*
│
├── components/
│   ├── tankmonitor_network/
│   └── tankmonitor_mqtt/
│
└── docs/
    └── FIRMWARE_REV2_3_HA_MQTT.md
```

---

# 43. Build matrix

| Feature | TankMonitor-HA | TankMonitor-MQTT |
|---|---:|---:|
| TL-136 | ✓ | ✓ |
| INA228 | ✓ | ✓ |
| Battery monitor | ✓ | ✓ |
| Deep sleep | ✓ | ✓ |
| Reed wake | ✓ | ✓ |
| Multi-Wi-Fi | ✓ | ✓ |
| DHCP | ✓ | ✓ |
| Static IPv4 | ✓ | ✓ |
| DHCP fallback | ✓ | ✓ |
| Setup portal | ✓ | ✓ |
| Native API | ✓ | ✗ |
| MQTT | ✗ | ✓ |
| HA direct entities | ✓ | ✗ |
| MQTT retained telemetry | ✗ | ✓ |
| MQTT desired/reported | ✗ | ✓ |
| ESP-NOW | ✗ | ✗ |

---

# 44. Design rule

La regola ufficiale REV2.3 è:

```text
ONE DEVICE
ONE TRANSPORT
ONE WAKE SESSION
```

Mai:

```text
HA API + MQTT simultaneamente
```

---

# 45. Versionamento

```text
Hardware REV2.0
Firmware REV2.3
Network Manager v1
Setup Portal v1
HA Transport v1
MQTT Transport v1
```

Questa specifica sostituisce la REV2.2 Network Manager come riferimento firmware ufficiale.

---

# 46. Decisione finale

Le due build ufficiali sono:

## TankMonitor-HA

```text
ESPHome
+
Native API
+
Home Assistant
```

## TankMonitor-MQTT

```text
ESPHome Core
+
MQTT
+
Broker
```

Non viene adottato ESP-NOW nella REV2.x.

La parte hardware e di misura rimane identica per entrambe.
