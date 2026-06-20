# Инструкция по реализации ESPHome climate для Midea

## Лицензия

Этот проект опубликован как open source по лицензии `MIT`.

Полный текст лицензии находится в корневом файле `LICENSE`.

## Что это за проект

Этот проект посвящён управлению кондиционером Midea через ESPHome и Home Assistant.

Текущая рабочая реализация построена на `platform: midea`, а не на `platform: thermostat`.

Это означает следующее:

- ESPHome не строит термостат с нуля
- ESPHome подключается к реальному кондиционеру Midea
- управление идёт по UART-протоколу устройства

## Где применяется

Этот проект подходит, если:

- используется кондиционер Midea или совместимая модель
- есть доступ к UART-интерфейсу кондиционера
- управление нужно перенести в Home Assistant через ESPHome

## Источники информации

- Базовый climate: https://esphome.io/components/climate/
- Midea climate: https://esphome.io/components/climate/midea/
- Сравнение climate и thermostat: см. `docs/climate-vs-thermostat.md`

## Что нужно по железу

Минимальный набор:

- ESP8266 или ESP32
- совместимый Midea dongle или собственный UART-адаптер
- питание для платы
- общий GND
- доступ к TX/RX кондиционера

Важные замечания из документации:

- UART для Midea должен работать на `9600`
- многие Midea-устройства ожидают `5V` логические уровни
- при необходимости нужен преобразователь уровней `3.3V -> 5V`

## Общая схема реализации

1. Подготовить плату ESP и питание.
2. Подключить UART к кондиционеру.
3. Отключить конфликтующий UART-логгер.
4. Настроить ESPHome `platform: midea`.
5. Добавить climate-сущность в Home Assistant.
6. Проверить режимы, температуру, swing и preset-ы.

Для практической проверки подключения и самых частых ошибок см. также `docs/wiring-and-troubleshooting.md`.

## Полный разбор конфига

### Блок `esphome`

```yaml
esphome:
  name: climat-komnata
  friendly_name: climat_komnata
```

Назначение:

- `name` — системное имя устройства
- `friendly_name` — удобное отображаемое имя в Home Assistant

### Блок платы

```yaml
esp8266:
  board: d1_mini_lite
```

Назначение:

- задаёт модель платы
- влияет на доступные GPIO и сборку прошивки

Если используется ESP32, блок должен быть другим.

### Блок `logger`

```yaml
logger:
  baud_rate: 0
```

Это критически важный момент.

Причина:

- аппаратный UART ESP8266 использует `GPIO1` и `GPIO3`
- эти же линии задействованы под связь с кондиционером
- обычный serial logger будет мешать Midea UART

Поэтому для такого подключения отключение UART-логгера — правильное решение.

Если `TX/RX` не используют конфликтующий аппаратный UART, этот блок не всегда нужно менять. Но если связь с Midea идёт именно по аппаратным UART-пинам, конфликт с логгером нужно проверить в первую очередь.

### Блок `api`

```yaml
api:
  encryption:
    key: !secret esphome_api_key
```

Назначение:

- подключение устройства к Home Assistant через native API
- шифрование соединения

### Блок `ota`

```yaml
ota:
  - platform: esphome
    password: !secret esphome_ota_password
```

Назначение:

- обновление устройства по сети

### Блок `wifi`

```yaml
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "Climat-Komnata Fallback Hotspot"
    password: !secret climat_fallback_password
```

Назначение:

- подключение к основной Wi‑Fi сети
- резервная точка доступа при потере основной сети

### Блок `captive_portal`

```yaml
captive_portal:
```

Назначение:

- упрощённый вход в fallback hotspot

### Блок `uart`

```yaml
uart:
  tx_pin: GPIO1
  rx_pin: GPIO3
  baud_rate: 9600
```

Назначение:

- физический обмен данными с кондиционером

Что важно проверить:

- не перепутаны `TX` и `RX`
- есть общий `GND`
- логические уровни совместимы
- скорость именно `9600`

Правило подключения простое:

- `ESP TX -> Midea RX`
- `ESP RX -> Midea TX`
- `ESP GND -> Midea GND`

### Блок `climate`

```yaml
climate:
  - platform: midea
    name: climat_komnata
    period: 1s
    timeout: 2s
    num_attempts: 3
    autoconf: true
    beeper: true
```

Назначение полей:

- `platform: midea` — включает поддержку кондиционера Midea
- `name` — имя climate-сущности
- `period` — минимальный интервал между запросами
- `timeout` — время ожидания ответа
- `num_attempts` — число повторов запроса
- `autoconf: true` — автодетект возможностей
- `beeper: true` — кондиционер подаёт звуковой отклик на команды

### Блок `visual`

```yaml
visual:
  min_temperature: 17 °C
  max_temperature: 30 °C
  temperature_step: 1 °C
```

Назначение:

- задаёт диапазон и шаг температуры в интерфейсе Home Assistant

Важно:

- это влияет прежде всего на UI
- шаг можно оставить `1 °C`, если так удобнее по факту

### Блок поддерживаемых режимов

```yaml
supported_modes:
  - FAN_ONLY
  - HEAT_COOL
  - COOL
  - HEAT
  - DRY
```

Означает:

- `FAN_ONLY` — только вентилятор
- `HEAT_COOL` — авто-режим нагрев/охлаждение
- `COOL` — охлаждение
- `HEAT` — нагрев
- `DRY` — осушение

### Блок custom fan modes

```yaml
custom_fan_modes:
  - SILENT
  - TURBO
```

Означает:

- специальные режимы вентилятора, если их поддерживает кондиционер

### Блок preset-ов

```yaml
supported_presets:
  - ECO
  - BOOST
  - SLEEP

custom_presets:
  - FREEZE_PROTECTION
```

Означает:

- стандартные и дополнительные preset-режимы

### Блок swing modes

```yaml
supported_swing_modes:
  - VERTICAL
  - HORIZONTAL
  - BOTH
```

Означает:

- управление жалюзи кондиционера

## Пошаговая реализация

### Шаг 1. Подготовить секреты

Создать `secrets.yaml` по примеру `examples/secrets.example.yaml`.

### Шаг 2. Проверить плату

Сверить:

- модель платы
- реальные UART-пины
- уровень логики

### Шаг 3. Собрать базовый конфиг

Сначала достаточно такого минимума:

```yaml
climate:
  - platform: midea
    name: climat_komnata
    period: 1s
    timeout: 2s
    num_attempts: 3
    autoconf: true
```

### Шаг 4. Проверить UART

Если климатическая сущность появляется, но кондиционер не отвечает, проверить:

- `TX/RX`
- общий `GND`
- уровень сигнала
- отключение `logger`

### Шаг 5. Добавить визуальные и функциональные режимы

После появления устойчивой связи добавить:

- `visual`
- `supported_modes`
- `custom_fan_modes`
- `supported_presets`
- `custom_presets`
- `supported_swing_modes`

### Шаг 6. Проверить в Home Assistant

Нужно убедиться, что:

- доступны режимы кондиционера
- меняется целевая температура
- работают swing mode и preset-ы
- команды реально доходят до кондиционера

## Типовые ошибки

### Конфликт UART и логгера

Если оставить обычный UART-логгер, обмен с кондиционером может не работать.

### Перепутаны TX и RX

Это одна из самых частых причин отсутствия связи.

### Нет общего GND

Даже при правильных пинах связь может не работать стабильно.

### Неправильный уровень сигнала

Если кондиционер ожидает `5V`, а ESP работает на `3.3V`, может потребоваться level shifter.

### Указаны режимы, которых нет у конкретной модели

Home Assistant покажет лишние функции, но железо их не выполнит.

## Что можно добавить дальше

Если конкретная модель поддерживает, можно расширить конфиг:

- `outdoor_temperature`
- `power_usage`
- `humidity_setpoint`
- `midea_ac.follow_me`
- `midea_ac.display_toggle`
- `midea_ac.swing_step`

## Практический вывод

Этот проект — отдельный climate-репозиторий под кондиционер Midea и ESPHome.

Его логика строится вокруг:

- `platform: midea`
- UART-подключения
- Home Assistant
- корректной настройки возможностей кондиционера
