# Код test_svdd.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/V776TIVZ).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Тесты Deep SVDD (svdd.py): сеть без смещений (условие против вырождения у Ruff et al.) и способность
обучиться отличать заведомо различимые сигналы (проверка инструмента до основной выборки, PREREG_v4 §5)."""
import numpy as np
import torch

import metrics
import svdd

RNG = np.random.default_rng(20260504)


def test_no_bias_anywhere():
    for m in (svdd.Encoder(100), svdd.Decoder(100)):
        names = [n for n, _ in m.named_parameters()]
        assert names and all(n.endswith("weight") for n in names)          # только веса свёрток и полносвязных
        assert all(not getattr(x, "affine", False) for x in m.modules())   # у BatchNorm нет обучаемого сдвига


def test_decoder_restores_shape():
    x = torch.zeros(3, 1, 2, 100)
    enc, dec = svdd.Encoder(100), svdd.Decoder(100)
    assert enc(x).shape == (3, svdd.REP) and dec(enc(x)).shape == x.shape


def _tones(n, f, L=100):
    t = np.arange(L)
    ph = RNG.uniform(0, 2 * np.pi, (n, 1))
    z = np.exp(1j * (2 * np.pi * f * t[None, :] + ph)) + 0.1 * (RNG.standard_normal((n, L)) + 1j * RNG.standard_normal((n, L)))
    return torch.from_numpy(np.stack([z.real, z.imag], axis=1).astype(np.float32)[:, None])


def test_learns_separable_classes(monkeypatch):
    """Обучение только на «нормальном» тоне; другой тон должен получать бóльшие оценки (AUC > 0,9),
    выход не постоянен. Эпох меньше, чем в рецепте, — проверяется работоспособность, а не качество."""
    monkeypatch.setattr(svdd, "EPOCHS_DCAE", (15, 5))
    monkeypatch.setattr(svdd, "EPOCHS_SVDD", (15, 5))
    monkeypatch.setattr(svdd, "LR", (1e-3, 1e-4))
    dev = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    enc, c = svdd.train(_tones(600, 0.05), dev, 1, lambda m: None)
    sc = svdd.score(enc, c, torch.cat([_tones(200, 0.05), _tones(200, 0.2)]), dev)
    y = np.r_[np.zeros(200), np.ones(200)]
    assert np.std(sc) > 0
    assert metrics.auc(sc, y) > 0.9

```
