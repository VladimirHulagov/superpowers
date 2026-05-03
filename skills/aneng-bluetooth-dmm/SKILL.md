---
name: aneng-bluetooth-dmm
description: Use when reading measurements from Bluetooth multimeters (Aneng, Zotec, BSIDE, OYI) via BLE, automating DMM data collection, logging measurements to MQTT, or writing test scripts that involve multimeter readings
---

# Aneng Bluetooth DMM - BLE Multimeter Reader

## Overview

Библиотека для чтения данных с Bluetooth мультиметров (Aneng, Zotec, BSIDE, OYI) через BLE.
Установлена как пакет `aneng_dmm` из `/home/lacitis/UTILS/Bluetooth-DMM-Aneng/`.

## Supported Devices

| Device | Protocol |
|--------|----------|
| Aneng AN9002 | 11 byte |
| Zotec ZT-5566 | 11 byte |
| BSIDE ZT-300AB | 11 byte |
| OYI-360 | 11 byte |
| Aneng V05B, BSIDE ZT-5B | 10 byte |
| Aneng ST207, BSIDE ZT-5BQ | 10 byte |

## BLE Connection

| Параметр | Значение |
|----------|----------|
| Device name | `Bluetooth DMM` |
| MAC (наш) | `FC:58:FA:BB:C1:CD` |
| Notify UUID | `0000fff4-0000-1000-8000-00805f9b34fb` |
| Зависимость | `bleak>=0.14.0` |

## Быстрый Старт: Чтение показаний

```python
import asyncio
from aneng_dmm import DMMClient

def handle_data(data):
    print(f"{data['display']} {data['icons']}")

async def main():
    async with DMMClient() as client:
        client.register_callback(handle_data)
        await client.start_listening()
        await asyncio.sleep(60)

asyncio.run(main())
```

## Подключение по MAC (быстрее, без сканирования)

```python
async with DMMClient(address="FC:58:FA:BB:C1:CD") as client:
    client.register_callback(handle_data)
    await client.start_listening()
    await asyncio.sleep(10)
```

## Декодирование сырых данных (без BLE)

```python
from aneng_dmm import decode

# Raw BLE packet as bytearray, bytes, list[int], or hex string
result = decode(bytearray.fromhex("hex data here"))
# result = {"typeID": "11", "display": 220.5, "value_type": "float", "icons": ["AC", "V", "AUTO"]}
```

## Data Structure

Коллбэк получает dict:

```python
{
    "typeID": "11",           # "11" = Big-DMM, "10" = Small-DMM, "01" = Clamp-DMM
    "display": 220.5,         # float или str — значение на дисплее
    "value_type": "float",    # тип Python
    "icons": ["AC", "V", "AUTO"]  # активные иконки
}
```

## Icons Reference (typeID "11")

| Icon | Значение |
|------|----------|
| AC, DC | тип тока |
| V, m(V) | вольты |
| A, m(A), u(A) | амперы |
| ohm, K(ohm), M(ohm) | сопротивление |
| F, u(F), m(F), n(F) | ёмкость |
| Hz | частота |
| ºC, ºF | температура |
| % | процент |
| DIODE | режим диода |
| HOLD | удержание |
| AUTO | авто-диапазон |
| LowBattery | батарея разряжена |
| MAX, MIN | пиковые значения |
| BT | Bluetooth подключен |

## DMMClient API

```python
client = DMMClient(address=None, name="Bluetooth DMM")

await client.scan(timeout=20.0)          # сканирование по имени
await client.connect(timeout=20.0)       # подключение (сканирует если нет address)
await client.start_listening()           # запуск нотификаций
await client.stop_listening()            # остановка нотификаций
await client.disconnect()                # отключение

client.register_callback(func)           # добавить коллбэк
client.remove_callback(func)             # убрать коллбэк
```

Поддерживает `async with` — автоматически подключается/отключается.

## MQTT Logger

Готовый скрипт для логирования в MQTT: `/home/lacitis/UTILS/Bluetooth-DMM-Aneng/examples/aneng_dmm_mqtt_logger.py`

```bash
python /home/lacitis/UTILS/Bluetooth-DMM-Aneng/examples/aneng_dmm_mqtt_logger.py
# Конфиг: examples/config.yaml
```

MQTT топики:
- `<prefix>/json` — Telegraf-совместимый flat JSON (display, unit, mode, ac_dc, typeID)
- `<prefix>/display` — числовое значение
- `<prefix>/icons` — список иконок
- `<prefix>/status` — online/offline (LWT)

Дополнительные зависимости: `pip install aneng_dmm[mqtt]` (paho-mqtt, pyyaml)

## Проверка Bluetooth

```bash
# Проверить BLE адаптер
bluetoothctl show

# Сканировать BLE устройства
bluetoothctl scan on

# Проверить установку библиотеки
python3 -c "from aneng_dmm import DMMClient; print('OK')"

# Тест подключения
python3 -c "
import asyncio
from aneng_dmm import DMMClient
async def test():
    async with DMMClient(address='FC:58:FA:BB:C1:CD') as c:
        print('Connected!')
asyncio.run(test())
"
```

## Типичные Проблемы

| Проблема | Решение |
|----------|---------|
| Device not found | Проверить питание мультиметра, включить Bluetooth |
| Scan timeout | Увеличить `scan_timeout` или задать MAC напрямую |
| Bleak error | `pip install --upgrade bleak` |
| decode возвращает {} | Неверная длина пакета (ожидается 10 или 11 байт) |
| Connection drops | Мультиметр отключился по таймауту — перезапустить |
