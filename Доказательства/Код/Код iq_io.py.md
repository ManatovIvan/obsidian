# Код iq_io.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/STKM6PZM).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Чтение записей HackRF (весенние сессии 4 мая 2026).

Обоснования строк кода (источники, цитаты, решения) — в хранилище статьи Obsidian, а не в коде.
"""
```

```python
import hashlib  # ВЫБОР: SHA-256 — тот же алгоритм, что в выводе sha256sum
```
↳ [[Вывод GJI5ZH7N|↗ Вывод GJI5ZH7N]] ^c-e293cc

```python
from pathlib import Path

import numpy as np

```

```python
SAMPLE_RATE = 20_000_000
```
↳ [[Показание AFRD8SPW#^t-afrd8spw-3|↗ Показание AFRD8SPW]] ^c-04b95b

```python
N_SAMPLES = 2_400_000_000
```
↳ [[Показание AFRD8SPW#^t-afrd8spw-3|↗ Показание AFRD8SPW]] ^c-a1e0d7

```python
BYTES_PER_SAMPLE = 2
```
↳ [[9. HackRF One in Python#^eqkq2c4m-3-26u3rvwq|↗ Lichtman, PySDR, с. 3]] ^c-99cf6f

```python

```

```python
DATASET = Path(__file__).resolve().parent.parent / "dataset"
```
↳ [[Вывод GJI5ZH7N#^o-gji5zh7n-0|↗ Вывод GJI5ZH7N]] ^c-b2b23d

```python

```

```python
SESSION_ORDER = ["R1", "E1", "R2", "E2", "R3", "E3"]
```
↳ [[Показание QNVEHN9R#^t-qnvehn9r-5|↗ Показание QNVEHN9R]] ^c-cacaea

```python

SHA256 = {
```

```python
    "E1": "902d78940b554820b5348e2b902923e9449709cfcecf80ade01f9b0a9f2eb585",
```
↳ [[Вывод GJI5ZH7N#^o-gji5zh7n-11|↗ Вывод GJI5ZH7N]] ^c-a5c2bf

```python
    "E2": "21390b857ea21b943d25c067b4d04038e565624d449a4f1da04462544c893ec6",
```
↳ [[Вывод GJI5ZH7N#^o-gji5zh7n-12|↗ Вывод GJI5ZH7N]] ^c-8a7898

```python
    "E3": "af9ec037b7dfe955378540cd15ae9e94b20ae8cd8da68a978daf864711dd5514",
```
↳ [[Вывод GJI5ZH7N#^o-gji5zh7n-13|↗ Вывод GJI5ZH7N]] ^c-e77c87

```python
    "R1": "55392275ee5394e53cfa62fac43f17b3dc73166110f45188cdb52cfb5606fa55",
```
↳ [[Вывод GJI5ZH7N#^o-gji5zh7n-14|↗ Вывод GJI5ZH7N]] ^c-7374e6

```python
    "R2": "666803985dfc268d10818c1b6339ad7853dee8353bfd20f2a9eaf7ed123159ef",
```
↳ [[Вывод GJI5ZH7N#^o-gji5zh7n-15|↗ Вывод GJI5ZH7N]] ^c-881417

```python
    "R3": "3444ef5359ecd51e3b35df5d88f8b0d7d9833f981c94b25a390556428afcb143",
```
↳ [[Вывод GJI5ZH7N#^o-gji5zh7n-16|↗ Вывод GJI5ZH7N]] ^c-3cbf78

```python
}


def session_path(sid):
```

```python
    return DATASET / f"session_{sid}.bin"
```
↳ [[Вывод GJI5ZH7N#^o-gji5zh7n-4|↗ Вывод GJI5ZH7N]] ^c-366511

```python


def open_session(sid):
    """Отображение файла в память без чтения целиком (ВЫБОР: файл 4,8 ГБ, ОЗУ 15 ГБ)."""
    p = session_path(sid)
    size = p.stat().st_size
```

```python
    if size != N_SAMPLES * BYTES_PER_SAMPLE:  # = n × 2 байта
```
↳ [[Вывод GJI5ZH7N#^o-gji5zh7n-4|↗ Вывод GJI5ZH7N]] ^c-4706e7

```python
        raise ValueError(f"{p}: размер {size}, ожидалось {N_SAMPLES * BYTES_PER_SAMPLE}")
```

```python
    return np.memmap(p, dtype=np.int8, mode="r")
```
↳ [[9. HackRF One in Python#^eqkq2c4m-3-cgakr5b9|↗ Lichtman, PySDR, с. 3]] ^c-f8a296

```python


def to_complex(raw_int8):
    """Байты I,Q,I,Q,... -> комплексные отсчёты в единицах АЦП (без масштабирования)."""
    raw = np.asarray(raw_int8, dtype=np.float32)  # ВЫБОР: float32 точно представляет целые −128..127
```

```python
    return raw[0::2] + 1j * raw[1::2]
```
↳ [[9. HackRF One in Python#^eqkq2c4m-3-tabx4xiu|↗ Lichtman, PySDR, с. 3]] ^c-357384

```python


def read_samples(mm, start, stop):
    """Отсчёты [start, stop) сессии как complex64; индексы — в отсчётах, а не в байтах."""
    start = max(int(start), 0)                    # ВЫБОР: окно у начала файла обрезается, а не падает
    stop = min(int(stop), N_SAMPLES)              # ВЫБОР: то же у конца файла
```

```python
    return to_complex(mm[2 * start:2 * stop]).astype(np.complex64)
```
↳ [[9. HackRF One in Python#^eqkq2c4m-3-4fwsbag3|↗ Lichtman, PySDR, с. 3]] ^c-097472

```python


def iter_blocks(sid, block_samples, hasher=None):
    """Последовательно отдаёт непересекающиеся блоки сырых байт; попутно считает SHA-256,
    чтобы проверить целостность за тот же единственный проход по файлу (ВЫБОР: экономия чтения 4,8 ГБ)."""
    p = session_path(sid)
    with open(p, "rb") as f:
        while True:
            buf = f.read(block_samples * BYTES_PER_SAMPLE)
            if not buf:
                break
            if hasher is not None:
                hasher.update(buf)                # ВЫБОР: хэш по всем байтам файла по порядку, как sha256sum
```

```python
            yield np.frombuffer(buf, dtype=np.int8)
```
↳ [[9. HackRF One in Python#^eqkq2c4m-3-ffnaqmjd|↗ Lichtman, PySDR, с. 3]] ^c-c83b3a

```python


def check_sha(sid, hexdigest):
    if hexdigest != SHA256[sid]:
        raise RuntimeError(f"session_{sid}.bin: SHA-256 {hexdigest} не совпадает с GJI5ZH7N")
```
