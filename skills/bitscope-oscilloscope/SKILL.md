---
name: bitscope-oscilloscope
description: Use when debugging electronics, analyzing signals, measuring voltage waveforms, capturing logic patterns, generating test signals, or working with hardware that needs oscilloscope/logic analyzer/waveform generator capabilities
---

# BitScope Micro - Oscilloscope/Logic Analyzer/Waveform Generator

## Overview

BitScope Micro (BS000501) - USB осциллограф, логический анализатор и генератор сигналов.

## Specs

| Параметр | Значение |
|----------|----------|
| Device ID | BS000501 |
| Путь | `/dev/ttyUSB0` |
| Аналоговые каналы | 2 (CH-A, CH-B) |
| Логические каналы | 8 (L0-L7) |
| Размер буфера | 12288 сэмплов |
| Max sample rate | 20 MHz (40 MHz clock) |
| AWG wavetable | 1024 сэмплов |
| AWG max voltage | 3.3V |
| Clock voltage | 3.3V |

## Pin Mapping

| Pin | Function | Notes |
|-----|----------|-------|
| CH-A | Analog input | Shared with L7 |
| CH-B | Analog input | Shared with L6 |
| L0-L3 | Logic inputs | - |
| L4 | AWG output | Waveform generator |
| L5 | CLK output | Clock generator |
| L6 | Logic input | Shared with CH-B |
| L7 | Logic input | Shared with CH-A |

## Libraries

**ВСЕГДА используйте `/home/lacitis/UTILS/bitscope/scopething/`**

Старая библиотека `bitscope` имеет баг с кэшированием данных - возвращает устаревшие/закэшированные значения (например, всегда 0.545V).

| Библиотека | Путь | Использовать |
|------------|------|--------------|
| `scopething` | `/home/lacitis/UTILS/bitscope/scopething/` | ✅ ВСЕГДА |
| `bitscope` | `~/UTILS/bitscope/bitscope/` | ❌ БАГ - не использовать |

```python
import sys
sys.path.insert(0, '/home/lacitis/UTILS/bitscope/scopething')
from scope import Scope
```

## Быстрый Старт: Измерение напряжения (scopething)

```python
#!/usr/bin/env python3
import sys
sys.path.insert(0, '/home/lacitis/UTILS/bitscope/scopething')

import asyncio
from scope import Scope

async def main():
    s = await Scope().connect('file:/dev/ttyUSB0')
    
    # Захват с CH-A
    traces = await s.capture(
        channels=['A'],
        period=1e-3,        # 1ms
        nsamples=1000,
        timeout=0.5,
        low=0,
        high=3.3,
        trigger='A',
        trigger_level=1.65,
        trigger_type='rising'
    )
    
    samples = traces.A.samples
    print(f"Min: {min(samples):.3f}V")
    print(f"Max: {max(samples):.3f}V")
    print(f"Samples: {len(samples)}")
    
    s.close()

asyncio.run(main())
```

## Быстрый Старт: Генерация Clock (L5)

```python
#!/usr/bin/env python3
import sys
sys.path.insert(0, '/home/lacitis/UTILS/bitscope/scopething')

import asyncio
from scope import Scope

async def main():
    s = await Scope().connect('file:/dev/ttyUSB0')
    
    # Запуск clock на L5: 1kHz, 50% duty, 3.3V
    freq, duty = await s.start_clock(frequency=1000, ratio=0.5)
    print(f"Clock: {freq:.1f}Hz, {duty*100:.0f}% duty")
    
    await asyncio.sleep(10)  # Работает 10 сек
    
    await s.stop_clock()
    s.close()

asyncio.run(main())
```

## Быстрый Старт: Генерация Waveform (L4)

```python
#!/usr/bin/env python3
import sys
sys.path.insert(0, '/home/lacitis/UTILS/bitscope/scopething')

import asyncio
from scope import Scope

async def main():
    s = await Scope().connect('file:/dev/ttyUSB0')
    
    # Sine wave: 2kHz, 0-3.3V
    freq = await s.start_waveform(
        frequency=2000,
        waveform='sine',    # 'sine', 'triangle', 'exponential', 'square'
        ratio=0.5,
        low=0,
        high=3.3
    )
    print(f"Waveform: {freq:.1f}Hz")
    
    await asyncio.sleep(5)
    
    await s.stop_waveform()
    s.close()

asyncio.run(main())
```

## Генерация + Измерение одновременно

```python
#!/usr/bin/env python3
import sys
sys.path.insert(0, '/home/lacitis/UTILS/bitscope/scopething')

import asyncio
from scope import Scope

async def main():
    s = await Scope().connect('file:/dev/ttyUSB0')
    
    # Запуск генератора на L5
    await s.start_clock(frequency=1000, ratio=0.5)
    
    # Измерение на CH-A (подключите L5 к CH-A)
    traces = await s.capture(
        channels=['A'],
        period=2e-3,
        nsamples=2000,
        timeout=0.1,
        low=0,
        high=3.3
    )
    
    await s.stop_clock()
    
    samples = traces.A.samples
    print(f"Min: {min(samples):.3f}V")
    print(f"Max: {max(samples):.3f}V")
    print(f"P-P: {max(samples)-min(samples):.3f}V")
    
    s.close()

asyncio.run(main())
```

## Clock Generator API

```python
# Запуск clock на L5
freq, duty = await s.start_clock(frequency=1000, ratio=0.5)
# frequency: Hz (ограничено available clock dividers)
# ratio: duty cycle 0.0-1.0 (fall/ticks, max ratio = (ticks-1)/ticks)

# Остановка
await s.stop_clock()
```

## Waveform Generator API

```python
# Запуск AWG на L4
freq = await s.start_waveform(
    frequency=2000,           # Hz
    waveform='sine',          # 'sine', 'triangle', 'exponential', 'square'
    ratio=0.5,                # for triangle/square
    low=0,                    # min voltage
    high=3.3                  # max voltage (max 3.3V)
)

# Остановка
await s.stop_waveform()
```

## Capture API

```python
traces = await s.capture(
    channels=['A', 'B'],      # или ['A'], ['B'], ['L'], ['L0'], ...
    period=1e-3,              # период захвата в секундах
    nsamples=1000,            # количество сэмплов
    timeout=0.5,              # таймаут триггера
    low=0,                    # минимум напряжения
    high=3.3,                 # максимум напряжения
    trigger='A',              # канал триггера
    trigger_level=1.65,       # уровень триггера
    trigger_type='rising',    # 'rising', 'falling', 'above', 'below'
    hair_trigger=False,       # более чувствительный триггер
    trigger_position=0.25,    # позиция триггера (0-1)
    probes='x1'               # тип пробника
)

# Результат
traces.A.samples      # array.array('f') - напряжения
traces.A.timestamps   # array.array('d') - временные метки
traces.A.sample_rate  # частота дискретизации
traces.A.cause        # 'trigger', 'timeout', 'cancel'
```

## Диапазон напряжений

| Параметр | Значение |
|----------|----------|
| Безопасный диапазон | **-5.5V до 8V** |
| AWG output | 0 - 3.3V |

**Важно:** При измерении напряжения всегда указывайте правильный диапазон `low`/`high`:

```python
# Для сигналов до 3.3V
traces = await s.capture(channels=['A'], low=0, high=3.3, ...)

# Для сигналов до 5V
traces = await s.capture(channels=['A'], low=0, high=5, ...)

# Полный диапазон (например, для неизвестных сигналов)
traces = await s.capture(channels=['A'], low=-5.5, high=8, ...)
```

Если `high` слишком низкий, сигнал будет обрезан и показания неверны.

## Ограничения

1. **Нельзя одновременно**: clock (L5) + waveform (L4)
2. **Shared pins**: CH-A/L7, CH-B/L6
3. **AWG voltage**: max 3.3V
4. **Clock ratio**: max (ticks-1)/ticks (не 100%)
5. **Very low frequencies**: могут не поддерживаться
6. **Диапазон измерения**: -5.5V до 8V (указывайте в low/high)

## Визуализация

```python
import matplotlib.pyplot as plt

samples = traces.A.samples
timestamps = traces.A.timestamps

plt.figure(figsize=(12, 4))
plt.plot(timestamps, samples)
plt.xlabel('Time (s)')
plt.ylabel('Voltage (V)')
plt.grid(True)
plt.show()
```

## Калибровка

```python
# Автокалибровка (требует numpy, scipy)
await s.calibrate(probes='x1', n=32, save=True)
# Сохраняет параметры в ~/.config/scopething/analog.conf
```

## Проверка Доступности

```bash
# Устройство
ls -la /dev/ttyUSB0

# Права
groups | grep dialout

# Тест подключения
python3 -c "
import sys; sys.path.insert(0, '/home/lacitis/UTILS/bitscope/scopething')
import asyncio; from scope import Scope
async def test():
    s = await Scope().connect('file:/dev/ttyUSB0')
    print(f'Connected: {s}')
    print(f'AWG max: {s.awg_maximum_voltage}V')
    s.close()
asyncio.run(test())
"
```

## Типичные Ошибки

| Ошибка | Решение |
|--------|---------|
| Device not found | Проверить `/dev/ttyUSB0` |
| Permission denied | `sudo usermod -aG dialout $USER` |
| No solution to frequency | Изменить frequency/ratio |
| Trigger timeout too long | Уменьшить timeout или trigger_position |
| Cannot start clock while AWG in use | Остановить waveform сначала |
