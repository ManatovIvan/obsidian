# Код test_paradis.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/9NN3INSC).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Тесты признаков PARADIS (paradis.py): на синтетических кадрах с заранее заданными искажениями
признаки должны вернуть заданные значения (PREREG_v4_approved.md §4, R-A)."""
import numpy as np

import dsss_rx as rx
import paradis

RNG = np.random.default_rng(20260504)
N = 8


def synth(signs, ref=1.0, offset=0.0, sym_gain=None):
    """Окно из 64 символов: символ j = ref·(знак_j·усиление_j + смещение) в отсчётах шаблона Баркера.
    Смещение и усиление задаются прямо в символах: c_j = Σ x·tmpl = коэффициент·‖tmpl‖²."""
    g = np.ones_like(signs) if sym_gain is None else sym_gain
    sym = ref * (signs * g + offset)
    e = float(rx.TEMPLATE @ rx.TEMPLATE)
    return (sym[:, :, None] * rx.TEMPLATE[None, None, :] / e).reshape(len(signs), -1)


def random_bits():
    return ["".join(RNG.choice(["0", "1"], 64)) for _ in range(N)]


def test_ideal_signs_follow_dbpsk():
    s = paradis.ideal_signs(["0" + "1" + "0" * 62])
    assert s[0, 0] == 1 and np.all(s[0, 1:] == -1)          # одна смена фазы на символе 1


def test_clean_frame():
    bits = random_bits()
    sgn = paradis.ideal_signs(bits)
    f = paradis.features(synth(sgn, ref=3 * np.exp(0.7j)), bits, np.full(N, -177e3))
    assert np.allclose(f[:, 0], -177e3)                      # частотная ошибка передаётся как есть
    assert np.allclose(f[:, 1], 1.0)                          # совпадение с идеальным SYNC
    assert np.allclose(f[:, 2:], 0.0, atol=1e-9)              # нет смещения, ошибок амплитуды и фазы


def test_offset_recovered():
    bits = random_bits()
    sgn = paradis.ideal_signs(bits)
    f = paradis.features(synth(sgn, ref=2.0, offset=0.1 + 0.05j), bits, np.zeros(N))
    # МНК-амплитуда ref·(1 + 0,1·mean(sgn)) немного отличается от ref, поэтому проверка — с допуском
    assert np.allclose(f[:, 2], abs(0.1 + 0.05j), rtol=0.1)


def test_magnitude_and_phase_errors():
    bits = random_bits()
    sgn = paradis.ideal_signs(bits)
    alt = np.where(np.arange(64) % 2 == 0, 1, -1)[None, :]
    mag = paradis.features(synth(sgn, sym_gain=1 + 0.2 * alt), bits, np.zeros(N))
    assert np.allclose(mag[:, 3], 0.2, atol=1e-9) and np.allclose(mag[:, 4], 0.0, atol=1e-9)
    ph = paradis.features(synth(sgn, sym_gain=np.exp(0.1j * alt)), bits, np.zeros(N))
    # МНК-амплитуда при повороте ±0,1 рад равна cos 0,1, поэтому |m| = 1/cos 0,1 у всех символов
    assert np.allclose(ph[:, 4], 0.1, atol=1e-9) and np.allclose(ph[:, 3], 1 / np.cos(0.1) - 1, atol=1e-9)


def test_signature_ranks_outlier_higher():
    f = RNG.normal(0, 1, (400, 5))
    sig = paradis.fit_signature(f)
    far = paradis.score(np.full((1, 5), 6.0), sig)
    assert far[0] > np.percentile(paradis.score(f, sig), 99)

```
