# Код dsss_rx.py

Файл verification/dsss_rx.py на сервере. Сгенерировано obsidian_export.py — не править вручную.

```python
"""Приёмник 802.11 DSSS (1 и 2 Мбит/с) для записей HackRF 20 MSPS, написанный заново.

Задача модуля — найти в записи кадры DSSS и восстановить их байты, чтобы метку
«кто передал кадр» брать из самого кадра, а не из имени сессии.

Обоснования:
  ⟦КЛЮЧ с.N: «цитата»⟧  дословный фрагмент источника из Zotero, страница PDF N; проверяется
                        check_citations.py. JHCW99MH — IEEE Std 802.11-2020 (PDF VI9FAQRH),
                        5FRQD424 — Матвеев «Цифровая обработка сигналов»
  ВИЗУАЛЬНО             фрагмент, который pdftotext извлекает с потерей символов (π в таблицах);
                        проверяется глазами по указанной странице
  ВЫБОР                 решение этого кода; причина указана рядом
  ПРОВЕРКА              утверждение, проверяемое тестом или самими данными (CRC), а не цитатой

Индексация, используемая во всём модуле:
  c[n] = Σ_k x[n+k]·tmpl[k], k = 0..19 — выход согласованного фильтра для символа, окно которого
  начинается с отсчёта n. «Символ j кадра» — окно, начинающееся с отсчёта a + 20·j, где a — начало
  окна последнего символа SFD. Бит j несёт символ j: он закодирован разностью фаз символов j−1 и j.
"""
import numpy as np                                # ВЫБОР: обработка отдельных кадров на CPU

from iq_io import SAMPLE_RATE, read_samples       # частота дискретизации обоснована в iq_io.py

# --- Параметры PHY ----------------------------------------------------------------------------
```

`SYMBOL_RATE = 1_000_000` [[IEEE Std 802.11-2020#^ieee-2754-z9z2qhce|↗ IEEE 802.11-2020, с. 2754]] ^l25

```python
                                                  # ВЫБОР: DBPSK — один бит на символ, значит 1 Мсимв/с
SPS = SAMPLE_RATE // SYMBOL_RATE                  # 20 отсчётов на символ; целое число, передискретизация не нужна
assert SPS * SYMBOL_RATE == SAMPLE_RATE           # ПРОВЕРКА: 20e6 / 1e6 — целое
```

`CHIPS = 11` [[IEEE Std 802.11-2020#^ieee-2765-7rnvvtj7|↗ IEEE 802.11-2020, с. 2765]] ^l29

```python
BARKER = np.array([+1, -1, +1, +1, -1, +1, +1, +1, -1, -1, -1], dtype=np.float32)
```

↳ [[IEEE Std 802.11-2020#^ieee-2765-nshcwn8j|↗ IEEE 802.11-2020, с. 2765]] ^l31

↳ [[IEEE Std 802.11-2020#^ieee-2765-a63d3ej6|↗ IEEE 802.11-2020, с. 2765]] ^l32

```python


def make_template():
    """Отсчёты кода Баркера на сетке 20 MSPS: отсчёт k (момент k/20 мкс) попадает в чип ⌊k·11/20⌋;
```

↳ [[IEEE Std 802.11-2020#^ieee-2765-wp8je8fb|↗ IEEE 802.11-2020, с. 2765]] ^l37

```python
    ВЫБОР: прямоугольная форма чипа — стандарт ограничивает спектр маской
```

↳ [[IEEE Std 802.11-2020#^ieee-2767-s9k5b7f7|↗ IEEE 802.11-2020, с. 2767]] , а форму ^l39

```python
    импульса не задаёт; потери от несовпадения формы видны по доле кадров, прошедших CRC."""
    k = np.arange(SPS)
    return BARKER[(k * CHIPS) // SPS]             # чип, в интервал которого попадает отсчёт k


```

`TEMPLATE = make_template()` [[Цифровая обработка сигналов#^5frqd424-97-chvmzeqe|↗ Матвеев 2013, с. 97]] ^l45

↳ [[Цифровая обработка сигналов#^5frqd424-97-cafv2utp|↗ Матвеев 2013, с. 97]] ^l46

```python

# --- Поля PHY-заголовка ----------------------------------------------------------------------
```

`SFD_LONG = 0xF3A0` [[IEEE Std 802.11-2020#^ieee-2754-qjb4e5nm|↗ IEEE 802.11-2020, с. 2754]] ^l49

`SFD_SHORT = 0x05CF` [[IEEE Std 802.11-2020#^ieee-2780-3r7zsiqp|↗ IEEE 802.11-2020, с. 2780]] ^l50

```python
SFD_PATTERNS = {                                  # слово w из 16 дескремблированных бит, первый по времени = бит 0
    SFD_LONG: ("long", False),
    SFD_SHORT: ("short", False),
    SFD_LONG ^ 0xFFFF: ("long", True),            # ПРОВЕРКА (tests): инверсия всех бит до дескремблера инвертирует выход
    SFD_SHORT ^ 0xFFFF: ("short", True),          #   — это случай фазы θ+π (неоднозначность оценки по z²)
}
SIGNAL_RATE = {0x0A: 1, 0x14: 2, 0x37: 5.5, 0x6E: 11}
```

↳ [[IEEE Std 802.11-2020#^ieee-2754-vveitadp|↗ IEEE 802.11-2020, с. 2754]] [[IEEE Std 802.11-2020#^ieee-2754-ubi9g2x5|↗ IEEE 802.11-2020, с. 2754]] ^l58

↳ [[IEEE Std 802.11-2020#^ieee-2781-hj8qtn6t|↗ IEEE 802.11-2020, с. 2781]] [[IEEE Std 802.11-2020#^ieee-2781-rhgz5d2c|↗ IEEE 802.11-2020, с. 2781]] ^l59

`OCTET_PER_US = {1: 1 / 8, 2: 1 / 4}` [[IEEE Std 802.11-2020#^ieee-2757-isfpz4bj|↗ IEEE 802.11-2020, с. 2757]] ^l60

`PSDU_MAX_OCTETS = 2 ** 12 - 1` [[IEEE Std 802.11-2020#^ieee-2764-qsii4cd5|↗ IEEE 802.11-2020, с. 2764]] (степень 2^12 при извлечении текста теряет верхний индекс) ^l61

`HDR_SYMBOLS = {"long": 48, "short": 24}` [[IEEE Std 802.11-2020#^ieee-2753-zi4cbs7a|↗ IEEE 802.11-2020, с. 2753]] (преамбула, заголовок); [[IEEE Std 802.11-2020#^ieee-2775-a6xmjeir|↗ IEEE 802.11-2020, с. 2775]] — 2 бита на символ ^l62

`SYNC_SYMBOLS = {"long": 128, "short": 56}` [[IEEE Std 802.11-2020#^ieee-2753-hseiqbkd|↗ IEEE 802.11-2020, с. 2753]] [[IEEE Std 802.11-2020#^ieee-2775-9tvx3iej|↗ IEEE 802.11-2020, с. 2775]] ^l63

```python
CFO_SYMBOLS = 48                                  # ВЫБОР: 48 символов перед концом SFD лежат внутри SYNC+SFD
                                                  #   и для длинной (128+16), и для короткой (56+16) преамбулы
TIMING_SEARCH = 5                                 # ВЫБОР: ±5 отсчётов покрывают разброс фаз, на которых
                                                  #   сработал поиск SFD (кластер ≤ 10 отсчётов, см. cluster)
TRACK_BLOCK = 32                                  # ВЫБОР: подстройка момента отсчёта раз в 32 символа (32 мкс).
```

↳ [[IEEE Std 802.11-2020#^ieee-2767-hj8ej2ht|↗ IEEE 802.11-2020, с. 2767]] ^l69

```python
                                                  #   точность HackRF в источниках нет, поэтому фактический сдвиг
                                                  #   записывается в timing_drift и виден в выводе


def descramble(raw_bits, prev7=None):
```

↳ """Дескремблер: [[IEEE Std 802.11-2020#^ieee-2756-2maumi5g|↗ IEEE 802.11-2020, с. 2756]] ^l75

↳ [[IEEE Std 802.11-2020#^ieee-2756-ig7zcsj6|↗ IEEE 802.11-2020, с. 2756]] ^l76

```python
    ВЫБОР: y[k] = b[k] ⊕ b[k−4] ⊕ b[k−7] — прямое прочтение G(z) для самосинхронизирующейся (feedthrough)
    схемы; ПРОВЕРКА (tests): обращает скремблер с той же G(z). prev7 — семь предшествующих бит."""
    b = np.asarray(raw_bits, dtype=np.uint8)
    hist = np.zeros(7, np.uint8) if prev7 is None else np.asarray(prev7, np.uint8)
    full = np.concatenate([hist, b])              # ВЫБОР: явная история вместо внутреннего состояния
    return full[7:] ^ full[3:-4] ^ full[0:-7]     # full[7+i]=b[i], full[3+i]=b[i−4], full[i]=b[i−7]


def bits_to_int_lsb_first(bits):
    """Первый переданный бит — младший разряд.
```

↳ Поля PHY: [[IEEE Std 802.11-2020#^ieee-2754-jcuv8g7p|↗ IEEE 802.11-2020, с. 2754]] ^l87

↳ Октеты MAC: [[IEEE Std 802.11-2020#^ieee-755-dbfsw9h6|↗ IEEE 802.11-2020, с. 755]] ^l88

```python
    return int(sum(int(b) << i for i, b in enumerate(bits)))


def bits_to_bytes(bits):
    n = len(bits) // 8
    return bytes(bits_to_int_lsb_first(bits[8 * i:8 * i + 8]) for i in range(n))


def crc16_ok(field_bits32, crc_bits16):
```

↳ [[IEEE Std 802.11-2020#^ieee-2755-a2swpbhi|↗ IEEE 802.11-2020, с. 2755]] ^l98

↳ [[IEEE Std 802.11-2020#^ieee-2755-vgqes27r|↗ IEEE 802.11-2020, с. 2755]] [[IEEE Std 802.11-2020#^ieee-2755-v8gwqc6f|↗ IEEE 802.11-2020, с. 2755]] ^l99

↳ [[IEEE Std 802.11-2020#^ieee-2756-rfpq56cc|↗ IEEE 802.11-2020, с. 2756]] ^l100

```python
    ВЫБОР: первый по времени бит FCS сравнивается со старшим разрядом регистра — так устроен пример
    рис. 15-3 (столбец «MSB LSB»); ПРОВЕРКА (tests): пример из стандарта проходит."""
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

`FCS_POLY = 0x04C11DB7` [[IEEE Std 802.11-2020#^ieee-790-tvp5hc7r|↗ IEEE 802.11-2020, с. 790]] без x^32 ^l114

`FCS_RESIDUE = 0xC704DD7B` [[IEEE Std 802.11-2020#^ieee-791-xhactgja|↗ IEEE 802.11-2020, с. 791]] ^l115

```python


def fcs_ok(mpdu_bits):
```

↳ [[IEEE Std 802.11-2020#^ieee-791-98bmew6d|↗ IEEE 802.11-2020, с. 791]] ^l119

```python
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
    z = s[1:] * torch.conj(s[:-1])                # ВИЗУАЛЬНО JHCW99MH с.2765, табл. 15-7: бит кодируется сменой фазы (0 -> 0, 1 -> π)
    q = z * z                                     # возведение в квадрат снимает модуляцию 0/π, остаётся 2θ
    cs = torch.cumsum(q, dim=0)
    qs = cs.clone()
    qs[window:] = cs[window:] - cs[:-window]      # скользящая сумма по 32 предыдущим символам (ВЫБОР: окно
                                                  #   короче самой короткой SYNC — 56 символов, см. SYNC_SYMBOLS)
    theta = torch.angle(qs) / 2                   # фаза сдвига частоты за символ, с неоднозначностью π
    zr = z * torch.exp(-1j * theta)
    b = (zr.real < 0).to(torch.int32)             # ВИЗУАЛЬНО JHCW99MH с.2765, табл. 15-7: сдвиг фазы π -> бит 1, 0 -> бит 0
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

`return np.correlate(x, TEMPLATE, mode="valid")` [[Цифровая обработка сигналов#^5frqd424-30-9rnk83v3|↗ Матвеев 2013, с. 30]] — тот же c[n], что на этапе 1 ^l180

```python


def _dbpsk_bits(s_prev, s, theta):
    z = s * np.conj(s_prev) * np.exp(-1j * theta)
    return (z.real < 0).astype(np.uint8)          # ВИЗУАЛЬНО JHCW99MH с.2765, табл. 15-7


def _dqpsk_bits(s_prev, s, theta):
    """ВИЗУАЛЬНО JHCW99MH с.2765, табл. 15-8: дибит (d0,d1): 00 -> 0, 01 -> π/2, 11 -> π, 10 -> 3π/2.
```

↳ [[IEEE Std 802.11-2020#^ieee-2765-9ufuczkb|↗ IEEE 802.11-2020, с. 2765]] (символ «+jω» в PDF — нестандартный глиф) ^l190

↳ [[IEEE Std 802.11-2020#^ieee-2765-dbw8p5kz|↗ IEEE 802.11-2020, с. 2765]] ^l191

```python
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

`d = int(np.argmax(energy)) - TIMING_SEARCH` [[Цифровая обработка сигналов#^5frqd424-97-5q5pr8rc|↗ Матвеев 2013, с. 97]] ^l211

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

`hraw = _dqpsk_bits(sh[:-1], sh[1:], theta)` [[IEEE Std 802.11-2020#^ieee-2775-8zwmj3ej|↗ IEEE 802.11-2020, с. 2775]] ^l234

`hy = descramble(hraw, prev7=raw[-7:])` [[IEEE Std 802.11-2020#^ieee-2754-nbtrxdqt|↗ IEEE 802.11-2020, с. 2754]] ^l235

```python
    signal = bits_to_int_lsb_first(hy[0:8])       # порядок и длины полей — см. SYNC_SYMBOLS (рис. 15-1)
    service = bits_to_int_lsb_first(hy[8:16])
    length = bits_to_int_lsb_first(hy[16:32])
    rec.update(signal=signal, service=service, length_us=length,
```

`crc16=bool(crc16_ok(hy[0:32], hy[32:48])))` [[IEEE Std 802.11-2020#^ieee-2781-fbazsqkj|↗ IEEE 802.11-2020, с. 2781]] ^l240

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

`n_oct = length * OCTET_PER_US[rate]` [[IEEE Std 802.11-2020#^ieee-2754-sbuu78gz|↗ IEEE 802.11-2020, с. 2754]] ^l249

```python
    if n_oct != int(n_oct) or not (1 <= n_oct <= PSDU_MAX_OCTETS):
        rec["stage"] = "bad_length"
        return rec
    n_oct = int(n_oct)
    n_sym = length                                # 1 символ = 1 мкс (SYMBOL_RATE), LENGTH — в мкс
    p0 = a + SPS * hdr_syms                       # последний символ заголовка; MPDU — со следующего:
```

↳ [[IEEE Std 802.11-2020#^ieee-2757-vxfxue4x|↗ IEEE 802.11-2020, с. 2757]] ^l256

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
        praw = _dbpsk_bits(sp[:-1], sp[1:], theta)  # ВИЗУАЛЬНО JHCW99MH с.2765, табл. 15-7
    else:
        praw = _dqpsk_bits(sp[:-1], sp[1:], theta)  # ВИЗУАЛЬНО JHCW99MH с.2765, табл. 15-8
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
