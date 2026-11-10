# Antena tracker — UTFPR Rocket

Estação de solo que **aponta uma antena para o foguete durante o voo**. O foguete transmite GPS, barômetro e bateria por LoRa a 5 Hz. A estação recebe os pacotes, prevê onde o foguete está e move uma base pan/tilt com dois servos de 180°, cobrindo os 360° com uma manobra de *flip*.

```
FOGUETE (foguete_tx)                                   BASE (tracker_rx)
ESP32-C3 + GPS u-blox + BMP280 ── LoRa 917 MHz ──>  ESP32 + RFM95W ──> servos pan/tilt ──> antena Yagi
   pacote de 26 B a 5 Hz                              predição 50 Hz       OLED, botões, buzzer
```

| Pasta | Conteúdo |
|---|---|
| [`firmware/foguete_tx/`](firmware/foguete_tx) | Transmissor embarcado (ESP32-C3 SuperMini) |
| [`firmware/tracker_rx/`](firmware/tracker_rx) | Receptor da base + controle da antena (ESP32 DevKit V1) |
| [`docs/Avionica_F-500_v2_e_Antena_Tracker.pdf`](docs/Avionica_F-500_v2_e_Antena_Tracker.pdf) | Integração com a aviônica F-500 v2: enlace, pacote byte a byte, geometria az/el, flip, calibração, falhas e testes (9 p.) |

A aviônica [F-500 v2](https://github.com/stansca/f500-avionica) já transmite o mesmo pacote, nos mesmos pinos de rádio, então funciona com este tracker sem mudanças. O PDF é gerado pelo script `scripts/gen_doc_tracker_pdf.py` daquele repositório.

## Hardware e pinagem

| Sketch | Placa | Periférico | Pinos |
|---|---|---|---|
| foguete_tx | ESP32-C3 SuperMini | LoRa RFM95W (SX1276) ou SX1262 | SCK 4, MISO 5, MOSI 6, NSS 7, DIO0 3, RST 10 |
| foguete_tx | ESP32-C3 SuperMini | GPS u-blox M8/M10 (UART1) | RX 20 (← TX do GPS), TX 21 (→ RX do GPS) |
| foguete_tx | ESP32-C3 SuperMini | BMP280 (I2C, 0x76 ou 0x77) | SDA 0, SCL 2 |
| foguete_tx | ESP32-C3 SuperMini | Bateria (divisor 100k/100k) | ADC 1 |
| tracker_rx | ESP32 DevKit V1 | LoRa RFM95W (SX1276) ou SX1262 | SCK 18, MISO 19, MOSI 23, NSS 5, RST 14, DIO0 26 |
| tracker_rx | ESP32 DevKit V1 | Servos pan/tilt (50 Hz, 500–2500 µs) | PAN 32, TILT 33 |
| tracker_rx | ESP32 DevKit V1 | OLED SSD1306 128×64 (I2C, 0x3C) | SDA 21, SCL 22 |
| tracker_rx | ESP32 DevKit V1 | Botões e buzzer | ALINHAR 0 (BOOT), BASE 27, BUZZER 25 |

## Rádio e pacote

LoRa **917,0 MHz**, BW 250 kHz, SF7, CR 4/5, preâmbulo 8, CRC ligado, 14 dBm. Um pacote a cada 200 ms (5 Hz), ~33 ms no ar. GPS a 5 Hz, modelo dinâmico AIRBORNE4g, protocolo UBX.

Estrutura `packed` de **26 bytes**, little-endian:

| Offset | Campo | Tipo | Unidade |
|---|---|---|---|
| 0 | `magic` | uint8 | sempre `0xA5` |
| 1 | `flags` | uint8 | bit 0 = fix 3D; bit 1 = barômetro OK |
| 2 | `seq` | uint16 | contador de pacotes |
| 4 | `lat` | int32 | graus × 1e7 |
| 8 | `lon` | int32 | graus × 1e7 |
| 12 | `altMsl` | int32 | cm (GPS, acima do nível do mar) |
| 16 | `baroRel` | int16 | dm, relativa ao boot do transmissor |
| 18 | `vN` | int16 | cm/s (norte) |
| 20 | `vE` | int16 | cm/s (leste) |
| 22 | `vD` | int16 | cm/s (para baixo; positivo = descendo) |
| 24 | `sats` | uint8 | satélites usados |
| 25 | `vbat` | uint8 | × 20 mV |

## Como o tracker aponta

**Calibração em campo** (gravada na NVS, namespace `tracker`):
- **BASE** (GPIO 27 por 2 s), com o foguete em cima da base: média de 20 pacotes com fix (mínimo 10, timeout 15 s). Grava `homeLat`, `homeLon` e `homeAlt`.
- **ALINHAR** (GPIO 0 por 2 s), com o foguete na rampa e a antena apontada à mão (pan 90°, tilt 0°): média de 10 pacotes (mínimo 5, timeout 10 s, exige rampa a ≥ 30 m). Grava o azimute de referência `azFront`, a altitude GPS da rampa, a leitura do barômetro na rampa e a distância.

**Laço de controle a 50 Hz:**
1. Converte latitude e longitude para coordenadas locais (leste, norte, cima) em relação à base.
2. Extrapola a posição com a velocidade: `dt = idade do pacote + LEAD_S` (0,15 s), limitado a `MAX_PREDICT_S` (1,5 s).
3. Sem fix de GPS, mas com barômetro: mantém a última posição horizontal (ou a da rampa), calcula a altura pelo barômetro relativo à rampa e não extrapola.
4. Ignora atualizações a menos de 3 m da base, para não oscilar.
5. Ângulos: `az = atan2(e, n)`, `el = atan2(u, √(n² + e²))` limitado a 0–90°, `rel = az − azFront`.
6. **Flip** (servos de 180° cobrindo 360°): liga quando |rel| > 95° e desliga quando |rel| < 85° (histerese). Normal: `pan = 90 + rel`, `tilt = el`. Invertido: `pan = 90 + (rel ± 180)`, `tilt = 180 − el`.
7. Limita a velocidade dos servos a `SLEW_DEG_S` (200°/s).

## Ajustes no topo do `tracker_rx.ino`

| Parâmetro | Padrão | Para quê |
|---|---|---|
| `SIMULATE` | 0 | 1 = move os servos com um voo simulado, sem rádio (teste da mecânica) |
| `USE_OLED` | 1 | 0 se não houver display |
| `RADIO_SX1262` | 0 | 1 para placas com SX1262 (ex.: Heltec V3) |
| `PAN_DIR` / `TILT_DIR` | 1 | −1 se o servo girar ao contrário |
| `PAN_TRIM_US` / `TILT_TRIM_US` | 0 | ajuste fino do centro e do horizonte |
| `LINK_LOST_MS` | 2000 | tempo sem pacote para considerar o enlace perdido |
| distância padrão da rampa | 500 m | usada antes do ALINHAR |

## Compilar
Arduino IDE com o pacote **esp32** (Espressif). Bibliotecas: **RadioLib** e **SparkFun u-blox GNSS v3** (foguete_tx); **RadioLib**, **ESP32Servo**, **Adafruit SSD1306** e **Adafruit GFX** (tracker_rx).

## Cuidados
- **Nunca grave o firmware do tracker na placa do foguete:** na F-500 v2, IO32/IO33 são os pyros, e aqui são os servos.
- Com bateria 2S, mande `vbat` **por célula**: o campo vai no máximo a 5,1 V.
- `baroRel` deve ser sempre relativo ao boot (não zerar no arme), senão a altura pelo barômetro erra.
- Se a placa do foguete reiniciar na rampa, refaça o ALINHAR.

## Orçamento de enlace
~31 ms no ar por pacote; margem acima de 40 dB a 3 km com 14 dBm e uma Yagi de 8 dBi na base (detalhes no PDF).

## Pendências
- [ ] Projetar e documentar a mecânica pan/tilt (ainda não há CAD).
- [ ] Teste de campo: alcance a 100 m, 500 m e 1 km, com a calibração BASE/ALINHAR.
- [ ] Teste de rastreamento real num lançamento.
