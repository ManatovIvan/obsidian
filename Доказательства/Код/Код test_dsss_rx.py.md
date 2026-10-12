# Код test_dsss_rx.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/FEKQQRB9).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Тесты приёмника. Соответствие стандарту проверяют только тесты с примерами из самого стандарта
(test_crc16_example_from_standard, test_fcs_residue — цитаты в самих тестах). Остальные тесты проверяют
механику: синтетический передатчик написан по тем же цитатам стандарта, что и приёмник,
поэтому ошибку прочтения стандарта он не поймает — её ловит доля кадров с верными CRC на реальных записях."""
import numpy as np
import torch

import dsss_rx as rx
import mac80211

RNG = np.random.default_rng(20260504)             # ВЫБОР: фиксированное зерно (дата записи) для повторяемости


def test_conv1d_is_correlation():
    x = (RNG.standard_normal(500) + 1j * RNG.standard_normal(500)).astype(np.complex64)
    pos_ref = np.correlate(x, rx.TEMPLATE, "valid")
    xr = torch.tensor(x.real).view(1, 1, -1)
    xi = torch.tensor(x.imag).view(1, 1, -1)
    w = torch.tensor(rx.TEMPLATE).view(1, 1, -1)
    got = (torch.nn.functional.conv1d(xr, w) + 1j * torch.nn.functional.conv1d(xi, w)).view(-1).numpy()
    assert np.allclose(got, pos_ref, atol=1e-4)


def test_template_covers_each_chip():
    counts = np.bincount((np.arange(rx.SPS) * rx.CHIPS) // rx.SPS, minlength=rx.CHIPS)
    assert counts.sum() == 20 and set(counts) <= {1, 2}  # 20/11 ≈ 1,8 отсчёта на чип


def scramble(data_bits, state7):
    """Скремблер передатчика (STD, обратная связь z^−4, z^−7): s[k] = d[k] ⊕ s[k−4] ⊕ s[k−7]."""
    s = list(state7)
    for d in data_bits:
        s.append(int(d) ^ s[-4] ^ s[-7])
    return np.array(s[7:], np.uint8)


def test_descrambler_inverts_scrambler_and_inversion():
    d = RNG.integers(0, 2, 300).astype(np.uint8)
    st = RNG.integers(0, 2, 7).astype(np.uint8)
    s = scramble(d, st)
    assert np.array_equal(rx.descramble(s, prev7=st), d)
    assert np.array_equal(rx.descramble(1 - s, prev7=1 - st), 1 - d)  # основание для SFD_PATTERNS с инверсией


def test_short_sfd_is_time_reverse_of_long():
    bits_long = [(rx.SFD_LONG >> i) & 1 for i in range(16)]
    assert rx.bits_to_int_lsb_first(bits_long[::-1]) == rx.SFD_SHORT


def test_crc16_example_from_standard():
    """Пример из стандарта для MPDU 192 мкс."""
```

```python
    data = [int(c) for c in "01010000000000000000001100000000"]
```
↳ [[IEEE Std 802.11-2020#^ieee-2755-qv8b3jc7|↗ IEEE 802.11-2020, с. 2755]] ^c-ef3d5e

```python
    crc = [int(c) for c in "0101101101010111"]
```
↳ [[IEEE Std 802.11-2020#^ieee-2755-ur6v43dn|↗ IEEE 802.11-2020, с. 2755]] ^c-38a9f0

```python
    assert rx.crc16_ok(data, crc)
    assert not rx.crc16_ok(data, crc[:-1] + [1 - crc[-1]])


```

```python
def fcs_tx_bits(octets):
```
↳ [[IEEE Std 802.11-2020#^ieee-790-xn6dv3dq|↗ IEEE 802.11-2020, с. 790]] · [[IEEE Std 802.11-2020#^ieee-790-vf5gbz3v|↗ IEEE 802.11-2020, с. 790]] ^c-47a7ba

```python
    """Передатчик FCS:
    Биты полей — в порядке передачи, младший бит октета первым (см. dsss_rx.bits_to_int_lsb_first)."""
    reg = 0xFFFFFFFF
    for byte in octets:
        for i in range(8):
            fb = ((reg >> 31) & 1) ^ ((byte >> i) & 1)
            reg = (reg << 1) & 0xFFFFFFFF
            if fb:
                reg ^= rx.FCS_POLY
    reg ^= 0xFFFFFFFF
    return [(reg >> (31 - i)) & 1 for i in range(32)]


def fcs_tx_bytes(octets):
    return rx.bits_to_bytes(fcs_tx_bits(octets))


def test_fcs_residue():
    body = bytes(RNG.integers(0, 256, 40).tolist())
    bits = [(b >> i) & 1 for b in body for i in range(8)] + fcs_tx_bits(body)
    assert rx.fcs_ok(bits)
    bits[5] ^= 1
    assert not rx.fcs_ok(bits)


def crc16_bits(bits32):
    reg = 0xFFFF
    for b in bits32:
        fb = ((reg >> 15) & 1) ^ int(b)
        reg = (reg << 1) & 0xFFFF
        if fb:
            reg ^= 0x1021
    crc = reg ^ 0xFFFF
    return [(crc >> (15 - i)) & 1 for i in range(16)]


def bits_of(value, n):
    return [(value >> i) & 1 for i in range(n)]


def synth_frame(mpdu, cfo_hz, snr_db, flip_phase, rate=1):
    """Синтетический кадр DSSS с длинной преамбулой на сетке 20 MSPS."""
    signal = {1: 0x0A, 2: 0x14}[rate]
    length = len(mpdu) * 8 // rate
    hdr = bits_of(signal, 8) + bits_of(0, 8) + bits_of(length, 16)
    hdr += crc16_bits(hdr)
    psdu = [b for byte in mpdu for b in bits_of(byte, 8)]
    pre = [1] * 128 + bits_of(rx.SFD_LONG, 16)
    data = np.array(pre + hdr + psdu, np.uint8)
    s = scramble(data, RNG.integers(0, 2, 7))
    n_pre = 128 + 16 + 48
    phases, ph = [], 0.0
    for b in s[:n_pre]:
        ph += np.pi * b
        phases.append(ph)
    dq = {(0, 0): 0, (0, 1): np.pi / 2, (1, 1): np.pi, (1, 0): 3 * np.pi / 2}
    rest = s[n_pre:]
    if rate == 1:
        for b in rest:
            ph += np.pi * b
            phases.append(ph)
    else:
        for i in range(0, len(rest), 2):
            ph += dq[(int(rest[i]), int(rest[i + 1]))]
            phases.append(ph)
    sym = np.exp(1j * (np.array(phases) + (np.pi if flip_phase else 0)))
    k = np.arange(len(sym) * rx.SPS)
    chips = rx.BARKER[((k % rx.SPS) * rx.CHIPS) // rx.SPS]
    sig = 40 * sym[k // rx.SPS] * chips * np.exp(2j * np.pi * cfo_hz * k / rx.SAMPLE_RATE)
    noise_std = 40 / 10 ** (snr_db / 20) / np.sqrt(2)
    pad = 3000
    out = np.zeros(len(sig) + 2 * pad, np.complex64)
    out[pad:pad + len(sig)] = sig
    out += (RNG.standard_normal(len(out)) + 1j * RNG.standard_normal(len(out))) * noise_std
    last_sfd = pad + (128 + 16 - 1) * rx.SPS
    return out, last_sfd


class FakeMM:
    """Подменяет memmap: read_samples берёт срез int8 I,Q; здесь отдаём float, это допустимо в тесте."""
    def __init__(self, x):
        self.raw = np.empty(2 * len(x), np.float32)
        self.raw[0::2], self.raw[1::2] = x.real, x.imag

    def __getitem__(self, sl):
        return self.raw[sl]


def _roundtrip(rate, cfo, flip):
    body = bytes([0xC0, 0x00, 0x3A, 0x01]) + bytes.fromhex("ffffffffffff") + bytes.fromhex("28285de06cc6") * 2 + b"\x10\x00\x07\x00"
    mpdu = body + fcs_tx_bytes(body)
    x, last = synth_frame(mpdu, cfo, 15, flip, rate)
    rec = rx.decode_candidate(FakeMM(x), last + 3)
    assert rec["stage"] == "fcs_ok", rec
    assert abs(rec["cfo_hz"] - cfo) < 5000
    m = mac80211.parse(rec["mpdu"])
    assert m["name"] == "deauth" and m["a2"] == "28:28:5d:e0:6c:c6" and m["reason"] == 7


def test_roundtrip_dbpsk_cfo_and_phase_flip():
    for cfo in (-300e3, -20e3, 0, 90e3, 400e3):   # ВЫБОР: включая |CFO| > 250 кГц, где оценка по z² неоднозначна
        for flip in (False, True):
            _roundtrip(1, cfo, flip)


def test_roundtrip_dqpsk():
    for cfo in (-150e3, 60e3):
        _roundtrip(2, cfo, False)


def test_stage1_finds_synthetic_frame():
    body = bytes(30)
    mpdu = body + fcs_tx_bytes(body)
    x, last = synth_frame(mpdu, 70e3, 15, False)
    pos, words = rx.find_sfd_candidates(x, torch, torch.device("cpu"))
    assert any(abs(p - last) <= 10 and w == rx.SFD_LONG for p, w in zip(pos, words)), (pos[:10], last)
```
