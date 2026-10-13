# Код paradis.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/XD7AU6CW).

Сгенерировано obsidian_export.py — не править вручную.

Модуль в целом: [[Предрегистрация v4|↗ предрегистрация v4 §4 R-A]] · [[Предрегистрация v4|↗ Предрегистрация v4]]

```python
"""R-A: признаки PARADIS и расстояние до сигнатуры роутера (PARADIS-kNN), PREREG_v4_approved.md §4.

Вход — окна dataset.npz (последние 64 символа SYNC+SFD, 1280 отсчётов) после C1/C2 и снятия CFO,
сырые биты DBPSK этих символов и оценка CFO кадра.

Обоснования строк кода (источники, цитаты, решения) — в хранилище статьи Obsidian, а не в коде.
"""
import numpy as np

import dsss_rx as rx

```

```python
N_SYM = 64                                        # символов в окне (dataset_build.WIN_SYMBOLS)
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §1]] ^c-176416

```python
NAMES = ("frequency_error", "sync_correlation", "iq_offset", "magnitude_error", "phase_error")
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-empdfxs2|↗ Brik 2008, с. 6]] · [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-t5qfqb2x|↗ Brik 2008, с. 6]] ^c-077628

```python


```

```python
def ideal_signs(bits):
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-msebf9ix|↗ Brik 2008, с. 6]] · [↗ IEEE 802.11-2020, с. 2765, табл. 15-7 (DBPSK: бит 1 — смена фазы на π) — сверить глазами](zotero://open-pdf/library/items/VI9FAQRH?page=2765) ^c-147e2b

```python
    """Идеальные символы окна (±1) по сырым битам DBPSK: бит 1 — смена фазы на π, бит 0 — без смены.
    Бит j несёт разность фаз символов j−1 и j, поэтому бит первого символа окна (связь с символом
    вне окна) не используется. ВЫБОР: первый символ окна принят за +1 — общая фаза снимается в phasors_ref."""
    b = np.array([[int(c) for c in s] for s in bits], np.int8)       # (N, 64)
    flips = np.cumsum(b[:, 1:], axis=1) % 2                           # число смен фазы от первого символа
    return np.concatenate([np.ones((len(b), 1)), 1 - 2 * flips], axis=1)


```

```python
def phasors(iq):
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-5-6hscih2a|↗ Brik 2008, с. 5]] · [[Wireless device identification with radiometric signatures#^uxhxnmqg-5-3jtgii5g|↗ Brik 2008, с. 5]] ^c-3b747b

```python
    """Символы окна — выход согласованного фильтра dsss_rx в момент символа: c_j = Σ_k x[20j+k]·tmpl[k]."""
    return iq.reshape(len(iq), N_SYM, rx.SPS) @ rx.TEMPLATE


```

```python
def features(iq, bits, cfo_hz):
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-8tvb7fip|↗ Brik 2008, с. 6]] · [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-k8iga64i|↗ Brik 2008, с. 6]] ^c-8732be

```python
    """Пять признаков на кадр (порядок — NAMES). iq — окна после снятия CFO."""
    sgn = ideal_signs(bits)
    s = phasors(iq)
    ref = np.mean(s * sgn, axis=1, keepdims=True)  # ВЫБОР: общая комплексная амплитуда — МНК для модели s = ref·sgn
    m = s / ref                                   # измеренные символы в масштабе идеальных (идеал — ±1)
```

```python
    offset = np.mean(m - sgn, axis=1)             # ВЫБОР: смещение начала координат — среднее отклонение от идеала
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-xh72gvr2|↗ Brik 2008, с. 6]] ^c-2acea8

```python
    mag_err = np.mean(np.abs(np.abs(m) - 1), axis=1)            # ВЫБОР: среднее модуля ошибки по символам
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-5-jx4775is|↗ Brik 2008, с. 5]] · [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-k8iga64i|↗ Brik 2008, с. 6]] ^c-ca5c6f

```python
    ph_err = np.mean(np.abs(np.angle(m * sgn)), axis=1)         # угол между измеренным и идеальным (sgn вещественный)
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-5-avt2r8i5|↗ Brik 2008, с. 5]] · [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-k8iga64i|↗ Brik 2008, с. 6]] ^c-2a4df2

```python
    wave = (sgn[:, :, None] * rx.TEMPLATE[None, None, :]).reshape(len(iq), -1)   # идеальный SYNC в отсчётах
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-msebf9ix|↗ Brik 2008, с. 6]] ^c-48d316

```python
    corr = np.abs(np.sum(iq * wave, axis=1)) / (np.linalg.norm(iq, axis=1) * np.linalg.norm(wave, axis=1))
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-msebf9ix|↗ Brik 2008, с. 6]] ^c-3a1490

```python
    return np.stack([cfo_hz, corr, np.abs(offset), mag_err, ph_err], axis=1)
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-t5ddtmcm|↗ Brik 2008, с. 6]] · [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-empdfxs2|↗ Brik 2008, с. 6]] ^c-2e5577

```python


```

```python
def fit_signature(f):
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-shx9i5w6|↗ Brik 2008, с. 6]] · [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-sfhwde7c|↗ Brik 2008, с. 6]] ^c-a725fc

```python
    """Сигнатура роутера по обучающим кадрам: нормировка признаков, отбрасывание половины кадров как
    выбросов, покомпонентное среднее остальных."""
    mu, sd = f.mean(axis=0), f.std(axis=0)        # ВЫБОР: нормировка по среднему и с. к. о. обучающих кадров
    z = (f - mu) / sd
    d = np.abs(z - np.median(z, axis=0)).sum(axis=1)   # ВЫБОР: выбросы — дальше всех (L1) от покомпонентной медианы
```

```python
    keep = np.argsort(d, kind="mergesort")[:len(d) // 2]
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-shx9i5w6|↗ Brik 2008, с. 6]] ^c-35ce03

```python
    return mu, sd, z[keep].mean(axis=0)
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-sfhwde7c|↗ Brik 2008, с. 6]] ^c-bbea3d

```python


```

```python
def score(f, signature):
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-sa2biw6e|↗ Brik 2008, с. 6]] ^c-08881f

```python
    """Оценка аномальности — сумма модулей покомпонентных разностей с сигнатурой (L1)."""
    mu, sd, sig = signature
```

```python
    return np.abs((f - mu) / sd - sig).sum(axis=1)
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-sa2biw6e|↗ Brik 2008, с. 6]] ^c-a5571d

```python

```
