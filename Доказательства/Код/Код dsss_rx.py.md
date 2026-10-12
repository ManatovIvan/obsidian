# Код dsss_rx.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/VX42ZXA6).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Приёмник 802.11 DSSS (1 и 2 Мбит/с) для записей HackRF 20 MSPS, написанный заново.

Задача модуля — найти в записи кадры DSSS и восстановить их байты, чтобы метку
«кто передал кадр» брать из самого кадра, а не из имени сессии.

Обоснования строк кода (источники, цитаты, решения) — в хранилище статьи Obsidian, а не в коде.

Индексация, используемая во всём модуле:
  c[n] = Σ_k x[n+k]·tmpl[k], k = 0..19 — выход согласованного фильтра для символа, окно которого
  начинается с отсчёта n. «Символ j кадра» — окно, начинающееся с отсчёта a + 20·j, где a — начало
  окна последнего символа SFD. Бит j несёт символ j: он закодирован разностью фаз символов j−1 и j.
"""
import numpy as np                                # ВЫБОР: обработка отдельных кадров на CPU

from iq_io import SAMPLE_RATE, read_samples       # частота дискретизации обоснована в iq_io.py

# --- Параметры PHY ----------------------------------------------------------------------------
```

```python
SYMBOL_RATE = 1_000_000
                                                  # ВЫБОР: DBPSK — один бит на символ, значит 1 Мсимв/с
```
↳ [[IEEE Std 802.11-2020#^ieee-2754-z9z2qhce|↗ IEEE 802.11-2020, с. 2754]] ^c-a2ca74

```python
SPS = SAMPLE_RATE // SYMBOL_RATE                  # 20 отсчётов на символ; целое число, передискретизация не нужна
assert SPS * SYMBOL_RATE == SAMPLE_RATE           # ПРОВЕРКА: 20e6 / 1e6 — целое
```

```python
CHIPS = 11
```
↳ [[IEEE Std 802.11-2020#^ieee-2765-7rnvvtj7|↗ IEEE 802.11-2020, с. 2765]] ^c-25ae19

```python
BARKER = np.array([+1, -1, +1, +1, -1, +1, +1, +1, -1, -1, -1], dtype=np.float32)
```
↳ [[IEEE Std 802.11-2020#^ieee-2765-nshcwn8j|↗ IEEE 802.11-2020, с. 2765]] · [[IEEE Std 802.11-2020#^ieee-2765-a63d3ej6|↗ IEEE 802.11-2020, с. 2765]] ^c-399c24

```python


```

```python
def make_template():
```
↳ [[IEEE Std 802.11-2020#^ieee-2765-wp8je8fb|↗ IEEE 802.11-2020, с. 2765]] · [[IEEE Std 802.11-2020#^ieee-2767-s9k5b7f7|↗ IEEE 802.11-2020, с. 2767]] ^c-61ccf5

```python
    """Отсчёты кода Баркера на сетке 20 MSPS: отсчёт k (момент k/20 мкс) попадает в чип ⌊k·11/20⌋;
    ВЫБОР: прямоугольная форма чипа — стандарт ограничивает спектр маской, а форму импульса
    не задаёт; потери от несовпадения формы видны по доле кадров, прошедших CRC."""
    k = np.arange(SPS)
    return BARKER[(k * CHIPS) // SPS]             # чип, в интервал которого попадает отсчёт k


```

```python
TEMPLATE = make_template()
```
↳ [[Цифровая обработка сигналов#^5frqd424-97-chvmzeqe|↗ Матвеев 2013, с. 97]] · [[Цифровая обработка сигналов#^5frqd424-97-cafv2utp|↗ Матвеев 2013, с. 97]] ^c-c50c0f

```python

# --- Поля PHY-заголовка ----------------------------------------------------------------------
```

```python
SFD_LONG = 0xF3A0
```
↳ [[IEEE Std 802.11-2020#^ieee-2754-qjb4e5nm|↗ IEEE 802.11-2020, с. 2754]] ^c-f41571

```python
SFD_SHORT = 0x05CF
```
↳ [[IEEE Std 802.11-2020#^ieee-2780-3r7zsiqp|↗ IEEE 802.11-2020, с. 2780]] ^c-e566ba

```python
SFD_PATTERNS = {                                  # слово w из 16 дескремблированных бит, первый по времени = бит 0
    SFD_LONG: ("long", False),
    SFD_SHORT: ("short", False),
    SFD_LONG ^ 0xFFFF: ("long", True),            # ПРОВЕРКА (tests): инверсия всех бит до дескремблера инвертирует выход
    SFD_SHORT ^ 0xFFFF: ("short", True),          #   — это случай фазы θ+π (неоднозначность оценки по z²)
}
```

```python
SIGNAL_RATE = {0x0A: 1, 0x14: 2, 0x37: 5.5, 0x6E: 11}
```
↳ [[IEEE Std 802.11-2020#^ieee-2754-vveitadp|↗ IEEE 802.11-2020, с. 2754]] · [[IEEE Std 802.11-2020#^ieee-2754-ubi9g2x5|↗ IEEE 802.11-2020, с. 2754]] · [[IEEE Std 802.11-2020#^ieee-2781-hj8qtn6t|↗ IEEE 802.11-2020, с. 2781]] · [[IEEE Std 802.11-2020#^ieee-2781-rhgz5d2c|↗ IEEE 802.11-2020, с. 2781]] ^c-2f6361

```python
OCTET_PER_US = {1: 1 / 8, 2: 1 / 4}
```
↳ [[IEEE Std 802.11-2020#^ieee-2757-isfpz4bj|↗ IEEE 802.11-2020, с. 2757]] ^c-eece72

```python
PSDU_MAX_OCTETS = 2 ** 12 - 1
```
↳ [[IEEE Std 802.11-2020#^ieee-2764-qsii4cd5|↗ IEEE 802.11-2020, с. 2764]] ^c-860e26

```python
HDR_SYMBOLS = {"long": 48, "short": 24}           # длинный: 48 бит по 1 биту; короткий: 48 бит по 2 бита на символ
```
↳ [[IEEE Std 802.11-2020#^ieee-2753-zi4cbs7a|↗ IEEE 802.11-2020, с. 2753]] · [[IEEE Std 802.11-2020#^ieee-2775-a6xmjeir|↗ IEEE 802.11-2020, с. 2775]] ^c-31ce44

```python
SYNC_SYMBOLS = {"long": 128, "short": 56}
```
↳ [[IEEE Std 802.11-2020#^ieee-2753-hseiqbkd|↗ IEEE 802.11-2020, с. 2753]] · [[IEEE Std 802.11-2020#^ieee-2775-9tvx3iej|↗ IEEE 802.11-2020, с. 2775]] ^c-c2ca5d

```python
CFO_SYMBOLS = 48                                  # ВЫБОР: 48 символов перед концом SFD лежат внутри SYNC+SFD
                                                  #   и для длинной (128+16), и для короткой (56+16) преамбулы
TIMING_SEARCH = 5                                 # ВЫБОР: ±5 отсчётов покрывают разброс фаз, на которых
                                                  #   сработал поиск SFD (кластер ≤ 10 отсчётов, см. cluster)
```

```python
TRACK_BLOCK = 32                                  # ВЫБОР: подстройка момента отсчёта раз в 32 символа (32 мкс).
                                                  #   точность HackRF в источниках нет, поэтому фактический сдвиг
                                                  #   записывается в timing_drift и виден в выводе
```
↳ [[IEEE Std 802.11-2020#^ieee-2767-hj8ej2ht|↗ IEEE 802.11-2020, с. 2767]] ^c-fbe87c

```python


```

```python
def descramble(raw_bits, prev7=None):
```
↳ [[IEEE Std 802.11-2020#^ieee-2756-2maumi5g|↗ IEEE 802.11-2020, с. 2756]] · [[IEEE Std 802.11-2020#^ieee-2756-ig7zcsj6|↗ IEEE 802.11-2020, с. 2756]] ^c-fef0f9

```python
    """Самосинхронизирующийся дескремблер G(z) = z^−7 + z^−4 + 1; начальное состояние скремблера
    передатчика знать не нужно.
    ВЫБОР: y[k] = b[k] ⊕ b[k−4] ⊕ b[k−7] — прямое прочтение G(z) для самосинхронизирующейся (feedthrough)
    схемы; ПРОВЕРКА (tests): обращает скремблер с той же G(z). prev7 — семь предшествующих бит."""
    b = np.asarray(raw_bits, dtype=np.uint8)
    hist = np.zeros(7, np.uint8) if prev7 is None else np.asarray(prev7, np.uint8)
    full = np.concatenate([hist, b])              # ВЫБОР: явная история вместо внутреннего состояния
    return full[7:] ^ full[3:-4] ^ full[0:-7]     # full[7+i]=b[i], full[3+i]=b[i−4], full[i]=b[i−7]


```

```python
def bits_to_int_lsb_first(bits):
```
↳ [[IEEE Std 802.11-2020#^ieee-2754-jcuv8g7p|↗ IEEE 802.11-2020, с. 2754]] · [[IEEE Std 802.11-2020#^ieee-755-dbfsw9h6|↗ IEEE 802.11-2020, с. 755]] ^c-a0e887

```python
    """Первый переданный бит — младший разряд (и в полях PHY, и в октетах MAC)."""
    return int(sum(int(b) << i for i, b in enumerate(bits)))


def bits_to_bytes(bits):
    n = len(bits) // 8
    return bytes(bits_to_int_lsb_first(bits[8 * i:8 * i + 8]) for i in range(n))


```

```python
def crc16_ok(field_bits32, crc_bits16):
```
↳ [[IEEE Std 802.11-2020#^ieee-2755-a2swpbhi|↗ IEEE 802.11-2020, с. 2755]] · [[IEEE Std 802.11-2020#^ieee-2755-vgqes27r|↗ IEEE 802.11-2020, с. 2755]] · [[IEEE Std 802.11-2020#^ieee-2755-v8gwqc6f|↗ IEEE 802.11-2020, с. 2755]] · [[IEEE Std 802.11-2020#^ieee-2756-rfpq56cc|↗ IEEE 802.11-2020, с. 2756]] ^c-b04689

```python
    """CRC-16 заголовка PHY: x^16 + x^12 + x^5 + 1, регистр предустановлен единицами, передаётся
    дополнение остатка, биты в порядке передачи.
    ВЫБОР: первый по времени бит FCS сравнивается со старшим разрядом регистра — так устроен пример
    рис. 15-3 стандарта; ПРОВЕРКА (tests): пример из стандарта проходит."""
    reg = 0xFFFF
    for b in field_bits32:
        fb = ((reg >> 15) & 1) ^ int(b)
        reg = (reg << 1) & 0xFFFF
        if fb:
            reg ^= 0x1021                         # x^12 + x^5 + 1 (x^16 — вытолкнутый разряд)
    crc = reg ^ 0xFFFF                            # дополнение до единиц
    rx = sum(int(b) << (15 - i) for i, b in enumerate(crc_bits16))  # первый по времени — старший разряд
    return crc == rx


```

```python
FCS_POLY = 0x04C11DB7                             # полином CRC-32 без старшего члена x^32
```
↳ [[IEEE Std 802.11-2020#^ieee-790-tvp5hc7r|↗ IEEE 802.11-2020, с. 790]] ^c-605104

```python
FCS_RESIDUE = 0xC704DD7B
```
↳ [[IEEE Std 802.11-2020#^ieee-791-xhactgja|↗ IEEE 802.11-2020, с. 791]] ^c-97fa58

```python


```

```python
def fcs_ok(mpdu_bits):
```
↳ [[IEEE Std 802.11-2020#^ieee-791-98bmew6d|↗ IEEE 802.11-2020, с. 791]] ^c-6cdad0

```python
    """Проверка FCS на приёмнике: регистр предустановлен единицами, через деление проходят биты кадра
    вместе с FCS; без ошибок остаётся фиксированный остаток FCS_RESIDUE.
    mpdu_bits — дескремблированные биты MPDU в порядке поступления."""
    reg = 0xFFFFFFFF
    for b in mpdu_bits:
        fb = ((reg >> 31) & 1) ^ int(b)
        reg = (reg << 1) & 0xFFFFFFFF
        if fb:
            reg ^= FCS_POLY
    return reg == FCS_RESIDUE


# --- Этап 1: поиск кандидатов на GPU ---------------------------------------------------------
def find_sfd_candidates(x_chunk, torch, device, window=32):
    """Возвращает (позиции a в отсчётах блока, слова SFD) для всех 20 фаз символа.
    Шаги: согласованный фильтр -> разность фаз соседних символов -> локальная оценка фазы
    сдвига частоты -> решения DBPSK -> дескремблер -> сравнение 16-битного слова с SFD."""
    xr = torch.from_numpy(np.ascontiguousarray(x_chunk.real)).to(device).view(1, 1, -1)
    xi = torch.from_numpy(np.ascontiguousarray(x_chunk.imag)).to(device).view(1, 1, -1)
    w = torch.from_numpy(TEMPLATE).to(device).view(1, 1, -1)
    cr = torch.nn.functional.conv1d(xr, w).view(-1)  # ПРОВЕРКА (tests): conv1d в torch — корреляция без
    ci = torch.nn.functional.conv1d(xi, w).view(-1)  #   переворота ядра, совпадает с np.correlate
    c = torch.complex(cr, ci)                     # c[n] = Σ x[n+k]·tmpl[k]; шаблон вещественный
    m = c.numel() // SPS
    s = c[:m * SPS].view(m, SPS)                  # s[j, p] = c[20j + p]: символ j на фазе p
```

```python
    z = s[1:] * torch.conj(s[:-1])  # бит кодируется сменой фазы (0 -> 0, 1 -> π)
```
↳ [↗ IEEE 802.11-2020, с. 2765, табл. 15-7 — сверить глазами](zotero://open-pdf/library/items/VI9FAQRH?page=2765) ^c-7312c2

```python
    q = z * z                                     # возведение в квадрат снимает модуляцию 0/π, остаётся 2θ
    cs = torch.cumsum(q, dim=0)
    qs = cs.clone()
    qs[window:] = cs[window:] - cs[:-window]      # скользящая сумма по 32 предыдущим символам (ВЫБОР: окно
                                                  #   короче самой короткой SYNC — 56 символов, см. SYNC_SYMBOLS)
    theta = torch.angle(qs) / 2                   # фаза сдвига частоты за символ, с неоднозначностью π
    zr = z * torch.exp(-1j * theta)
```

```python
    b = (zr.real < 0).to(torch.int32)  # сдвиг фазы π -> бит 1, 0 -> бит 0
```
↳ [↗ IEEE 802.11-2020, с. 2765, табл. 15-7 — сверить глазами](zotero://open-pdf/library/items/VI9FAQRH?page=2765) ^c-69bd90

```python
    y = b[7:] ^ b[3:-4] ^ b[:-7]                  # descramble(): y[k] = b[k]^b[k−4]^b[k−7]
    word = torch.zeros_like(y[15:])
    for i in range(16):
        word |= y[i:y.shape[0] - 15 + i] << i     # бит i слова — i-й по времени (LSB первым, см. SFD_LONG)
    hit = torch.zeros_like(word, dtype=torch.bool)
    for pat in SFD_PATTERNS:
        hit |= word == pat
    jj, pp = torch.nonzero(hit, as_tuple=True)
    # word[k] заканчивается на y[k+15] = бит с индексом k+22 в z, который несёт символ k+23
    sym = jj + 23
    pos = sym * SPS + pp                          # начало окна последнего символа SFD в отсчётах блока
    words = word[jj, pp]
    return pos.cpu().numpy().astype(np.int64), words.cpu().numpy().astype(np.int64)


def cluster(positions, gap=10):
    """Одна посылка срабатывает на соседних фазах; объединяем позиции ближе 10 отсчётов
    (ВЫБОР: половина символа) и берём медиану кластера."""
    if len(positions) == 0:
        return np.zeros(0, np.int64)
    p = np.sort(positions)
    starts = np.concatenate([[0], np.nonzero(np.diff(p) > gap)[0] + 1])
    ends = np.concatenate([starts[1:], [len(p)]])
    return np.array([int(np.median(p[s:e])) for s, e in zip(starts, ends)], np.int64)


# --- Этап 2: разбор одного кандидата на CPU --------------------------------------------------
def _corr(x):
```

```python
    return np.correlate(x, TEMPLATE, mode="valid")  # тот же c[n], что на этапе 1
```
↳ [[Цифровая обработка сигналов#^5frqd424-30-9rnk83v3|↗ Матвеев 2013, с. 30]] ^c-41d322

```python


def _dbpsk_bits(s_prev, s, theta):
    z = s * np.conj(s_prev) * np.exp(-1j * theta)
```

```python
    return (z.real < 0).astype(np.uint8)
```
↳ [↗ IEEE 802.11-2020, с. 2765, табл. 15-7 — сверить глазами](zotero://open-pdf/library/items/VI9FAQRH?page=2765) ^c-449ede

```python


```

```python
def _dqpsk_bits(s_prev, s, theta):
```
↳ [↗ IEEE 802.11-2020, с. 2765, табл. 15-8 — сверить глазами](zotero://open-pdf/library/items/VI9FAQRH?page=2765) · [[IEEE Std 802.11-2020#^ieee-2765-9ufuczkb|↗ IEEE 802.11-2020, с. 2765]] · [[IEEE Std 802.11-2020#^ieee-2765-dbw8p5kz|↗ IEEE 802.11-2020, с. 2765]] ^c-7be28c

```python
    """DQPSK: дибит (d0,d1): 00 -> 0, 01 -> π/2, 11 -> π, 10 -> 3π/2; положительный угол — поворот
    против часовой стрелки; d0 передаётся первым."""
    z = s * np.conj(s_prev) * np.exp(-1j * theta)
    quad = np.round(np.angle(z) / (np.pi / 2)).astype(int) % 4  # ближайшая из четырёх фаз
    table = np.array([[0, 0], [0, 1], [1, 1], [1, 0]], np.uint8)
    return table[quad].reshape(-1)                # d0, d1, d0, d1, ... в порядке передачи


def decode_candidate(mm, a0):
    """Разбирает кадр, у которого окно последнего символа SFD начинается около отсчёта a0.
    Возвращает словарь; поле 'stage' говорит, на каком шаге разбор остановился."""
    rec = {"a0": int(a0), "stage": "start"}
    back = (CFO_SYMBOLS + 1) * SPS + TIMING_SEARCH  # история для оценки фазы и подстройки момента
    x = read_samples(mm, a0 - back, a0 + SPS + SPS + TIMING_SEARCH)
    if len(x) < back + 2 * SPS:
        rec["stage"] = "edge"                     # ВЫБОР: кадры у края файла не разбираем
        return rec
    c = _corr(x)
    base = back                                   # индекс a0 в массиве c
    js = np.arange(-CFO_SYMBOLS, 1)               # символы −48..0
    energy = [np.sum(np.abs(c[base + d + SPS * js]) ** 2) for d in range(-TIMING_SEARCH, TIMING_SEARCH + 1)]
```

```python
    d = int(np.argmax(energy)) - TIMING_SEARCH
```
↳ [[Цифровая обработка сигналов#^5frqd424-97-5q5pr8rc|↗ Матвеев 2013, с. 97]] ^c-41e634

```python
    a = int(a0) + d
    s = c[base + d + SPS * js]                    # s[0] — символ −48, s[-1] — символ 0
    z = s[1:] * np.conj(s[:-1])
    theta0 = np.angle(np.sum(z * z)) / 2          # ВЫБОР: z² снимает модуляцию (±1)² = 1, остаётся удвоенная фаза сдвига частоты; неоднозначность π
    raw = _dbpsk_bits(s[:-1], s[1:], theta0)      # биты символов −47..0
    y = descramble(raw[7:], prev7=raw[:7])        # дескремблированные биты символов −40..0
    word = bits_to_int_lsb_first(y[-16:])         # символы −15..0 = SFD
    if word not in SFD_PATTERNS:
        rec["stage"] = "no_sfd"
        return rec
    preamble, inverted = SFD_PATTERNS[word]
    theta = theta0 + (np.pi if inverted else 0.0)  # неоднозначность π снимается полярностью SFD
    theta = (theta + np.pi) % (2 * np.pi) - np.pi  # приводим к (−π, π]
    raw = _dbpsk_bits(s[:-1], s[1:], theta)
    rec.update(a=a, preamble=preamble, cfo_hz=float(theta / (2 * np.pi) * SYMBOL_RATE))
    hdr_syms = HDR_SYMBOLS[preamble]
    xh = read_samples(mm, a - TIMING_SEARCH, a + SPS * (hdr_syms + 2) + TIMING_SEARCH)
    ch = _corr(xh)
    sh = ch[TIMING_SEARCH + SPS * np.arange(0, hdr_syms + 1)]  # символы 0..hdr_syms
    if preamble == "long":
        hraw = _dbpsk_bits(sh[:-1], sh[1:], theta)  # см. SYMBOL_RATE: длинный заголовок — 1 Мбит/с DBPSK
    else:
```

```python
        hraw = _dqpsk_bits(sh[:-1], sh[1:], theta)
```
↳ [[IEEE Std 802.11-2020#^ieee-2775-8zwmj3ej|↗ IEEE 802.11-2020, с. 2775]] ^c-92127d

```python
    hy = descramble(hraw, prev7=raw[-7:])
```
↳ [[IEEE Std 802.11-2020#^ieee-2754-nbtrxdqt|↗ IEEE 802.11-2020, с. 2754]] ^c-e4555e

```python
    signal = bits_to_int_lsb_first(hy[0:8])       # порядок и длины полей — см. SYNC_SYMBOLS (рис. 15-1)
    service = bits_to_int_lsb_first(hy[8:16])
    length = bits_to_int_lsb_first(hy[16:32])
    rec.update(signal=signal, service=service, length_us=length,
```

```python
               crc16=bool(crc16_ok(hy[0:32], hy[32:48])))
```
↳ [[IEEE Std 802.11-2020#^ieee-2781-fbazsqkj|↗ IEEE 802.11-2020, с. 2781]] ^c-acb7d5

```python
    if not rec["crc16"]:
        rec["stage"] = "crc16_fail"
        return rec
    rate = SIGNAL_RATE.get(signal)
    rec["rate_mbps"] = rate
    if rate not in OCTET_PER_US:                  # ВЫБОР: CCK 5,5/11 Мбит/с не демодулируем — только заголовок
        rec["stage"] = "header_only"
        return rec
```

```python
    n_oct = length * OCTET_PER_US[rate]
```
↳ [[IEEE Std 802.11-2020#^ieee-2754-sbuu78gz|↗ IEEE 802.11-2020, с. 2754]] ^c-605841

```python
    if n_oct != int(n_oct) or not (1 <= n_oct <= PSDU_MAX_OCTETS):
        rec["stage"] = "bad_length"
        return rec
    n_oct = int(n_oct)
    n_sym = length                                # 1 символ = 1 мкс (SYMBOL_RATE), LENGTH — в мкс
```

```python
    p0 = a + SPS * hdr_syms                       # последний символ заголовка; MPDU — со следующего:
```
↳ [[IEEE Std 802.11-2020#^ieee-2757-vxfxue4x|↗ IEEE 802.11-2020, с. 2757]] ^c-ed51f6

```python
    xp = read_samples(mm, p0 - 2 * SPS, p0 + SPS * (n_sym + 2) + 4 * SPS)
    cp = _corr(xp)
    off = 2 * SPS                                 # индекс p0 в cp
    pos = [off]
    for blk in range(0, n_sym, TRACK_BLOCK):      # подстройка момента: ±1 отсчёт за блок, критерий —
        nb = min(TRACK_BLOCK, n_sym - blk)        #   максимум выхода согласованного фильтра (см. выбор d выше)
        idx = pos[-1] + SPS * np.arange(1, nb + 1)
        best = max((-1, 0, 1), key=lambda dd: (np.sum(np.abs(cp[idx + dd]) ** 2), dd == 0))
        pos.extend((idx + best).tolist())
    pos = np.array(pos)
    if pos[-1] >= len(cp):
        rec["stage"] = "edge"
        return rec
    sp = cp[pos]                                  # символ перед MPDU и n_sym символов MPDU
    if rate == 1:
```

```python
        praw = _dbpsk_bits(sp[:-1], sp[1:], theta)
```
↳ [↗ IEEE 802.11-2020, с. 2765, табл. 15-7 — сверить глазами](zotero://open-pdf/library/items/VI9FAQRH?page=2765) ^c-56832f

```python
    else:
```

```python
        praw = _dqpsk_bits(sp[:-1], sp[1:], theta)
```
↳ [↗ IEEE 802.11-2020, с. 2765, табл. 15-8 — сверить глазами](zotero://open-pdf/library/items/VI9FAQRH?page=2765) ^c-6388cf

```python
    py = descramble(praw, prev7=hraw[-7:])
    rec.update(n_octets=n_oct, fcs=bool(fcs_ok(py)), mpdu=bits_to_bytes(py),
               timing_drift=int(pos[-1] - off - SPS * n_sym),
               start=int(a - SPS * (SYNC_SYMBOLS[preamble] + 15)),  # начало SYNC: SFD (16 бит) — символы −15..0
               end=int(p0 + SPS * (n_sym + 1)))
    xa = read_samples(mm, a + SPS, rec["end"])    # заголовок + MPDU: интервал, точно известный из заголовка
    rec["power"] = float(np.mean(np.abs(xa) ** 2))  # средняя мощность в единицах АЦП² (без нормировки)
    rec["stage"] = "fcs_ok" if rec["fcs"] else "fcs_fail"
    return rec

```
