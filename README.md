# pond-high-availability-2-iot

Firmware embarcado para **Raspberry Pi Pico W** (simulado no Wokwi) que coleta dados de sensores físicos e envia telemetria ao backend de alta disponibilidade desenvolvido na [Atividade 1](https://github.com/mcouto0xml/pond-high-availability).

---

## Sumário

- [Contexto](#contexto)
- [Framework e Toolchain](#framework-e-toolchain)
- [Sensores Integrados](#sensores-integrados)
- [Diagrama de Conexão](#diagrama-de-conexão)
- [Formato de Telemetria](#formato-de-telemetria)
- [Conectividade Wi-Fi](#conectividade-wi-fi)
- [Compilação e Gravação](#compilação-e-gravação)
- [Configuração de Rede](#configuração-de-rede)
- [Simulação com Wokwi](#simulação-com-wokwi)
- [Evidências de Funcionamento](#evidências-de-funcionamento)

---

## Contexto

Esta atividade expande o sistema de monitoramento industrial da Atividade 1, adicionando a camada de dispositivos embarcados. O firmware roda no Raspberry Pi Pico W, lê sensores analógicos e digitais via GPIO, estabelece conexão Wi-Fi e envia pacotes de telemetria ao endpoint `POST /telemetry` do backend Go, garantindo reconexão automática em caso de falha de rede.

O fluxo completo do sistema pode ser visto no vídeo de demonstração: https://youtube.com/shorts/8hdlshB-LRg

---

## Framework e Toolchain

**Caminho 1 — Arduino Framework (até 8 pontos)**

| Item | Detalhe |
|---|---|
| Framework | Arduino (via Wokwi + extensão Pico W) |
| Linguagem | C++ (Arduino sketch) |
| Arquivo principal | `src/iot/sketch.ini` |
| Simulador | [Wokwi](https://wokwi.com) |
| Bibliotecas | `WiFi`, `HTTPClient`, `DHT sensor library` |

As bibliotecas utilizadas estão declaradas em `src/iot/libraries.txt`:

```
WiFi
HttpClient
DHT sensor library
```

---

## Sensores Integrados

| Sensor | Tipo | Pino GPIO | Variável | Range / Unidade |
|---|---|---|---|---|
| DHT22 | Digital (protocolo 1-wire) | GP2 | `temperature`, `humidity` | -40–80 °C / 0–100 % RH |
| PIR (Motion Sensor) | Digital (HIGH/LOW) | GP3 | `presence` | `true` / `false` |
| LDR (Photoresistor) | Analógico (ADC) | GP27 (ADC1) | `luminosity` | 0–10.000 lx (mapeado de 0–4095) |
| Potenciômetro | Analógico (ADC) | GP26 (ADC0) | `tank_level` | 0–100 % |
| Vibração (simulado) | Gerado por firmware | — | `vibration` | 0.000–0.099 m/s² |

### Detalhes de leitura

**Temperatura e Umidade (DHT22 — GP2)**
Leitura via biblioteca `DHT`. Em caso de falha (`isnan`), o firmware retorna valores de fallback (`25.0 °C` / `60.0 %`) para não interromper o ciclo de envio.

**Presença (PIR — GP3)**
Leitura digital simples: `digitalRead(PIR_PIN) == HIGH`. Configurado como `INPUT` no `setup()`.

**Luminosidade (LDR — GP27)**
ADC de 12 bits (0–4095), mapeado linearmente para 0–10.000 lx:
```cpp
float readLuminosity() {
    int raw = analogRead(LDR_PIN);
    return (raw / 4095.0) * 10000.0;
}
```

**Nível de Tanque (Potenciômetro — GP26)**
ADC de 12 bits mapeado para 0–100 %. No firmware atual, retorna `20.0` como valor fixo de demonstração (pode ser substituído por `(analogRead(POT_PIN) / 4095.0) * 100.0`).

**Vibração (simulado)**
Sensor de vibração não disponível no Wokwi — o firmware gera valores pseudoaleatórios no intervalo 0.000–0.099 m/s² usando `random()` com seed no pino flutuante GP28.

---

## Diagrama de Conexão

```
                         ┌─────────────────────────────────────┐
                         │         Raspberry Pi Pico W          │
                         │                                      │
  DHT22                  │  GP2  ◄── SDA (amarelo)              │
  ├─ VCC ──────────────► │  3V3                                  │
  ├─ GND ──────────────► │  GND                                  │
  └─ SDA ──────────────► │  GP2                                  │
                         │                                      │
  PIR Motion             │  GP3  ◄── OUT (roxo)                  │
  ├─ VCC ──────────────► │  3V3                                  │
  ├─ GND ──────────────► │  GND                                  │
  └─ OUT ──────────────► │  GP3                                  │
                         │                                      │
  LDR Photoresistor      │  GP27 ◄── AO (laranja) [ADC1]        │
  ├─ VCC ──────────────► │  3V3                                  │
  ├─ GND ──────────────► │  GND                                  │
  └─ AO  ──────────────► │  GP27                                 │
                         │                                      │
  Potenciômetro          │  GP26 ◄── SIG (laranja) [ADC0]       │
  ├─ VCC ──────────────► │  3V3                                  │
  ├─ GND ──────────────► │  GND                                  │
  └─ SIG ──────────────► │  GP26                                 │
                         │                                      │
  Resistor 10kΩ          │  GP15 ◄── terminal 1 (verde)         │
  ├─ terminal 1 ───────► │  GP15                                 │
  └─ terminal 2 ───────► │  GND                                  │
                         │                                      │
  Terminal Serial        │  GP0 / GP1 (UART)                    │
  ├─ RX ───────────────► │  GP0                                  │
  └─ TX ───────────────► │  GP1                                  │
                         └─────────────────────────────────────┘
```

> No Wokwi, o diagrama completo está definido em `src/iot/diagram.json`.

---

## Formato de Telemetria

O firmware serializa manualmente um JSON compatível com o schema do backend da Atividade 1 e o envia via `HTTP POST`:

```json
{
  "iot_name": "sensor-01",
  "temperature": 23.45,
  "humidity": 61.20,
  "presence": false,
  "vibration": 0.042,
  "luminosity": 3821.0,
  "tank_level": 20.0
}
```

| Campo | Tipo | Fonte |
|---|---|---|
| `iot_name` | string | Hardcoded `"sensor-01"` (deve existir na tabela `devices` do backend) |
| `temperature` | float | DHT22 — GP2 |
| `humidity` | float | DHT22 — GP2 |
| `presence` | bool | PIR — GP3 |
| `vibration` | float | Gerado por firmware (simulado) |
| `luminosity` | float | LDR — GP27 (ADC1) |
| `tank_level` | float | Potenciômetro — GP26 (ADC0) |

O endpoint alvo é `POST /telemetry` — o mesmo documentado na Atividade 1. A resposta esperada é `HTTP 202 Accepted`.

---

## Conectividade Wi-Fi

O firmware gerencia a conexão Wi-Fi com reconexão automática:

```
setup()
  └─► connectWiFi()
        ├─ WiFi.begin(SSID, PASSWORD)
        ├─ Aguarda até 30 tentativas (15 segundos)
        └─ Se falhar: rp2040.reboot()

loop() — a cada SEND_INTERVAL (5000 ms)
  └─► sendTelemetry()
        ├─ Se WiFi desconectado: chama connectWiFi() novamente
        └─ Envia POST HTTP com payload JSON
```

Em caso de falha na requisição HTTP, o erro é logado na serial mas o firmware continua tentando no próximo ciclo (sem travamento).

---

## Compilação e Gravação

### Usando o Wokwi (simulação — recomendado)

1. Acesse [wokwi.com](https://wokwi.com) e crie um novo projeto para **Raspberry Pi Pico W**.
2. Copie o conteúdo de `src/iot/sketch.ini` para o editor de código.
3. Copie o conteúdo de `src/iot/diagram.json` para o editor de diagrama.
4. Copie o conteúdo de `src/iot/libraries.txt` para o arquivo de bibliotecas.
5. Clique em **Play** para iniciar a simulação.

### Usando hardware real (Arduino IDE)

**Pré-requisitos:**
- Arduino IDE 2.x
- Suporte ao Raspberry Pi Pico W: adicione a URL abaixo em *File → Preferences → Additional boards manager URLs*:
  ```
  https://github.com/earlephilhower/arduino-pico/releases/download/global/package_rp2040_index.json
  ```
- Instale as bibliotecas via *Sketch → Include Library → Manage Libraries*:
  - `WiFi` (built-in para Pico W)
  - `HTTPClient` (built-in para Pico W)
  - `DHT sensor library` by Adafruit

**Passos:**
1. Abra `src/iot/sketch.ini` no Arduino IDE (renomeie para `.ino` se necessário).
2. Selecione a placa: *Tools → Board → Raspberry Pi Pico W*.
3. Ajuste `WIFI_SSID`, `WIFI_PASSWORD` e `SERVER_URL` no cabeçalho do sketch.
4. Conecte o Pico W via USB enquanto mantém o botão **BOOTSEL** pressionado.
5. Clique em **Upload**.

---

## Configuração de Rede

As configurações de rede estão no cabeçalho do `sketch.ini`:

```cpp
// Wi-Fi
const char* WIFI_SSID     = "Wokwi-GUEST";   // SSID da rede
const char* WIFI_PASSWORD = "";               // Senha (vazio = rede aberta)

// Backend (Atividade 1)
const char* SERVER_URL    = "http://35.192.65.120:8080/telemetry";

// Intervalo de envio
const unsigned long SEND_INTERVAL = 5000;     // 5 segundos
```

Para uso em hardware real ou outra rede:

| Variável | Valor esperado |
|---|---|
| `WIFI_SSID` | SSID da rede corporativa |
| `WIFI_PASSWORD` | Senha da rede (ou `""` para aberta) |
| `SERVER_URL` | URL completa do endpoint `POST /telemetry` do backend |
| `SEND_INTERVAL` | Intervalo em milissegundos entre envios (padrão: 5000) |

> O `SERVER_URL` deve apontar para o IP/domínio onde o Producer Go da Atividade 1 está rodando na porta `8080`.

---

## Simulação com Wokwi

Por não ter um Pico W físico disponível, a simulação foi feita integralmente no **Wokwi**, que suporta o Arduino Framework para Raspberry Pi Pico W e emula corretamente os periféricos utilizados (DHT22, PIR, LDR, potenciômetro, ADC 12-bit).

O diagrama completo de conexões está em `src/iot/diagram.json` e pode ser carregado diretamente no Wokwi.

---

## Evidências de Funcionamento

### Vídeo demonstrativo

Demonstração completa do firmware em execução no Wokwi, com leitura de sensores e telemetria chegando no backend da Atividade 1:

**[▶ Assistir no YouTube Shorts](https://youtube.com/shorts/8hdlshB-LRg)**

### Logs seriais esperados

```
=== Pico W - Telemetria IoT ===
[WiFi] Conectando a: Wokwi-GUEST
....
[WiFi] Conectado!
[WiFi] IP: 10.0.0.2

[HTTP] Enviando: {"iot_name":"sensor-01","temperature":24.30,"humidity":58.40,"presence":false,"vibration":0.042,"luminosity":4120.5,"tank_level":20.0}
[HTTP] Resposta: 202
[HTTP] Body: {"message":"Mensagem adicionada a fila!"}

[HTTP] Enviando: {"iot_name":"sensor-01","temperature":24.50,"humidity":58.10,"presence":true,"vibration":0.017,"luminosity":3990.2,"tank_level":20.0}
[HTTP] Resposta: 202
[HTTP] Body: {"message":"Mensagem adicionada a fila!"}
```

### Fluxo de integração

```
Pico W (Wokwi)
    │
    │  POST /telemetry  (HTTP, a cada 5s)
    ▼
Producer Go (:8080)
    │  202 Accepted — enfileira assincronamente
    ▼
Google Cloud Tasks
    │  Retry automático, backoff exponencial
    ▼
Cloud Function (Consumer Go)
    │  INSERT
    ▼
PostgreSQL (Supabase)
    tabela: telemetry
```

Para detalhes completos sobre o backend (Producer, Consumer, infraestrutura GCP e banco de dados), consulte o README da [Atividade 1](https://github.com/mcouto0xml/pond-high-availability).