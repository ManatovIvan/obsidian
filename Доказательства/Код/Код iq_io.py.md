# Код iq_io.py

Файл verification/iq_io.py на сервере. Сгенерировано obsidian_export.py — не править вручную.

```python
"""Чтение записей HackRF (весенние сессии 4 мая 2026).

Обоснования: ⟦КЛЮЧ с.N: «цитата»⟧ — дословный фрагмент источника из Zotero (проверяется
check_citations.py); ВЫБОР — решение этого кода с причиной.
"""
import hashlib                                    # ВЫБОР: SHA-256 — тот же алгоритм, что в выводе sha256sum (GJI5ZH7N)
from pathlib import Path

import numpy as np

```

`SAMPLE_RATE = 20_000_000` [↗ Манатов 2026](zotero://select/library/items/AFRD8SPW) ^l11

`N_SAMPLES = 2_400_000_000` [↗ Манатов 2026](zotero://select/library/items/AFRD8SPW) ^l12

`BYTES_PER_SAMPLE = 2` [[9. HackRF One in Python#^eqkq2c4m-3-26u3rvwq|↗ Lichtman, PySDR, с. 3]] ^l13

```python

```

`DATASET = Path(__file__).resolve().parent.parent / "dataset"` [↗ Манатов 2026](zotero://select/library/items/GJI5ZH7N) ^l15

```python

```

`SESSION_ORDER = ["R1", "E1", "R2", "E2", "R3", "E3"]` [↗ Манатов 2026](zotero://select/library/items/QNVEHN9R) ^l17

```python

SHA256 = {
```

`"E1": "902d78940b554820b5348e2b902923e9449709cfcecf80ade01f9b0a9f2eb585",` [↗ Манатов 2026](zotero://select/library/items/GJI5ZH7N) ^l20

`"E2": "21390b857ea21b943d25c067b4d04038e565624d449a4f1da04462544c893ec6",` [↗ Манатов 2026](zotero://select/library/items/GJI5ZH7N) ^l21

`"E3": "af9ec037b7dfe955378540cd15ae9e94b20ae8cd8da68a978daf864711dd5514",` [↗ Манатов 2026](zotero://select/library/items/GJI5ZH7N) ^l22

`"R1": "55392275ee5394e53cfa62fac43f17b3dc73166110f45188cdb52cfb5606fa55",` [↗ Манатов 2026](zotero://select/library/items/GJI5ZH7N) ^l23

`"R2": "666803985dfc268d10818c1b6339ad7853dee8353bfd20f2a9eaf7ed123159ef",` [↗ Манатов 2026](zotero://select/library/items/GJI5ZH7N) ^l24

`"R3": "3444ef5359ecd51e3b35df5d88f8b0d7d9833f981c94b25a390556428afcb143",` [↗ Манатов 2026](zotero://select/library/items/GJI5ZH7N) ^l25

```python
}


def session_path(sid):
```

`return DATASET / f"session_{sid}.bin"` [↗ Манатов 2026](zotero://select/library/items/GJI5ZH7N) ^l30

```python


def open_session(sid):
    """Отображение файла в память без чтения целиком (ВЫБОР: файл 4,8 ГБ, ОЗУ 15 ГБ)."""
    p = session_path(sid)
    size = p.stat().st_size
```

`if size != N_SAMPLES * BYTES_PER_SAMPLE:` [↗ Манатов 2026](zotero://select/library/items/GJI5ZH7N) = n × 2 байта ^l37

```python
        raise ValueError(f"{p}: размер {size}, ожидалось {N_SAMPLES * BYTES_PER_SAMPLE}")
```

`return np.memmap(p, dtype=np.int8, mode="r")` [[9. HackRF One in Python#^eqkq2c4m-3-cgakr5b9|↗ Lichtman, PySDR, с. 3]] ^l39

```python


def to_complex(raw_int8):
    """Байты I,Q,I,Q,... -> комплексные отсчёты в единицах АЦП (без масштабирования)."""
    raw = np.asarray(raw_int8, dtype=np.float32)  # ВЫБОР: float32 точно представляет целые −128..127
```

`return raw[0::2] + 1j * raw[1::2]` [[9. HackRF One in Python#^eqkq2c4m-3-tabx4xiu|↗ Lichtman, PySDR, с. 3]] ^l45

```python


def read_samples(mm, start, stop):
    """Отсчёты [start, stop) сессии как complex64; индексы — в отсчётах, а не в байтах."""
    start = max(int(start), 0)                    # ВЫБОР: окно у начала файла обрезается, а не падает
    stop = min(int(stop), N_SAMPLES)              # ВЫБОР: то же у конца файла
```

`return to_complex(mm[2 * start:2 * stop]).astype(np.complex64)` [[9. HackRF One in Python#^eqkq2c4m-3-4fwsbag3|↗ Lichtman, PySDR, с. 3]] ^l52

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

`yield np.frombuffer(buf, dtype=np.int8)` [[9. HackRF One in Python#^eqkq2c4m-3-ffnaqmjd|↗ Lichtman, PySDR, с. 3]] ^l66

```python


def check_sha(sid, hexdigest):
    if hexdigest != SHA256[sid]:
        raise RuntimeError(f"session_{sid}.bin: SHA-256 {hexdigest} не совпадает с GJI5ZH7N")
```
