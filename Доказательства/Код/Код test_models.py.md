# Код test_models.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/VBQBDX4B).

Сгенерировано obsidian_export.py — не править вручную.

Модуль в целом: [[Предрегистрация|↗ предрегистрация §2]]

```python
"""Тесты предобработки и моделей (models.py): проверяется, что функции делают то, что заявлено в предрегистрации (§2–§3)."""
import numpy as np
import torch

import models

RNG = np.random.default_rng(20260504)


def test_energy_normalize_unit_power():
    x = (RNG.standard_normal((5, 1280)) + 1j * RNG.standard_normal((5, 1280))) * RNG.uniform(1, 50, (5, 1))
    p = np.mean(np.abs(models.energy_normalize(x)) ** 2, axis=1)
    assert np.allclose(p, 1.0)


def test_remove_cfo_cancels_tone():
    f = np.array([-177e3, 25e3])
    n = np.arange(1280)
    x = np.exp(2j * np.pi * f[:, None] * n[None, :] / 20e6)
    y = models.remove_cfo(x, f)
    assert np.allclose(y, 1.0)


def test_match_snr_reaches_target():
    """Сигнал мощности 1000 при шуме 5 (SNR ≈ 23 дБ) доводится до SNR 8 дБ."""
    n, L, noise = 200, 1280, 5.0
    sig = np.exp(1j * RNG.uniform(0, 2 * np.pi, (n, L))) * np.sqrt(1000)
    x = sig + (RNG.standard_normal((n, L)) + 1j * RNG.standard_normal((n, L))) * np.sqrt(noise / 2)
    p = np.mean(np.abs(x) ** 2, axis=1)
    target = 10 ** (8 / 10)
    y = models.match_snr(x, p, np.full(n, noise), np.ones(n, bool), np.full(n, target), RNG)
    snr = 1000 / (np.mean(np.abs(y - sig) ** 2, axis=1))
    assert abs(10 * np.log10(np.median(snr)) - 8) < 0.2


def test_match_snr_leaves_esp_frames_untouched():
    x = RNG.standard_normal((4, 100)) + 0j
    y = models.match_snr(x, np.full(4, 10.0), np.full(4, 1.0), np.array([False] * 4), np.full(4, 2.0), RNG)
    assert np.array_equal(x, y)


def test_shapes():
    assert models.Homegrown(1280)(torch.zeros(2, 1, 2, 1280)).shape == (2, 2)
    assert models.CDAE()(torch.zeros(2, 1, 1280, 2)).shape == (2, 1, 1280, 2)


def test_minmax_maps_training_data_to_unit_interval():
    iq = RNG.standard_normal((50, 1280)) + 1j * RNG.standard_normal((50, 1280))
    mm = models.minmax_fit(np.stack([iq.real, iq.imag], axis=2))
    x = models.to_tensor_m2(iq, mm)
    assert x.dtype == torch.float32                # иначе не совпадёт с типом весов сети
    x = x.numpy()
    assert x.min() >= -1e-6 and x.max() <= 1 + 1e-6   # допуск — округление float32



def test_significance_threshold():
    """Порог Deep Learning Tutorials: улучшение значимо, только если new < best·0,995."""
    assert models.significant(0.0099, 0.0100)                      # −1 %: значимо
    assert not models.significant(0.00996, 0.0100)                 # −0,4 %: не значимо
    assert models.significant(0.5, np.inf)                         # первое значение всегда значимо
    assert not models.significant(0.0, 0.0)                        # ошибка 0 улучшить нельзя

```
