# Разница между climate и thermostat в ESPHome

## Короткий ответ

`climate` — это базовый слой и общая документация для всех климатических устройств в ESPHome.

`thermostat` — это конкретная платформа `platform: thermostat`, которая использует базовый `climate`, но добавляет логику термостата: датчик, уставки, hysteresis, действия нагрева и охлаждения, задержки и preset-ы.

## Главная разница

| Статья | Что описывает | Для чего нужна |
|---|---|---|
| `climate/` | Общий интерфейс климатических устройств в ESPHome | Понять общие поля, отображение в Home Assistant, MQTT, automation и управление |
| `climate/thermostat/` | Отдельный контроллер-термостат | Настроить устройство, которое само решает, когда включать нагрев, охлаждение или idle |

## Что есть в статье climate

- общие поля `id`, `name`, `icon`
- блок `visual`
- диапазоны `min_temperature`, `max_temperature`
- шаг температуры `temperature_step`
- диапазоны влажности `min_humidity`, `max_humidity`
- MQTT topic-ы для climate-сущности
- автоматизации `climate.control`
- триггеры `on_state` и `on_control`
- lambda-доступ к состояниям climate

Эта статья отвечает на вопрос:

`Как вообще устроены climate-сущности в ESPHome?`

## Что есть в статье thermostat

- `platform: thermostat`
- обязательный `sensor`
- single-point и dual-point режимы
- `heat_action`, `cool_action`, `idle_action`
- hysteresis и тайминги
- preset-ы и восстановление состояния
- fan mode / swing mode / humidity control actions

Эта статья отвечает на вопрос:

`Как собрать в ESPHome термостат, который сам управляет нагревом и/или охлаждением?`

## Как это связано с этим репозиторием

Текущий проект использует не `thermostat`, а `platform: midea`.

То есть:

- базовая статья `climate` относится к проекту напрямую
- статья `thermostat` полезна для понимания, но не является основной документацией именно для этого конфига
- для текущего рабочего решения ключевая специализированная документация — `midea`
