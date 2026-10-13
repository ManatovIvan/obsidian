# Код test_svdd.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/V776TIVZ).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Тесты Deep SVDD (svdd.py): сеть без смещений (условие против вырождения у Ruff et al.) и способность
обучиться отличать заведомо различимые сигналы (проверка инструмента до основной выборки, PREREG_v4 §5)."""
import numpy as np
import torch
from torch import nn

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


class _Ref2d(nn.Module):
    """Эталон: Homegrown-кодировщик и зеркальный декодер в прямой записи Conv2d / ConvTranspose2d."""

    def __init__(self, L):
        super().__init__()
        bn1, bn2, a = (lambda n: nn.BatchNorm1d(n, affine=False)), (lambda n: nn.BatchNorm2d(n, affine=False)), nn.LeakyReLU(0.1)
        self.enc = nn.Sequential(nn.Conv2d(1, 50, (1, 7), bias=False), bn2(50), a,
                                 nn.Conv2d(50, 50, (2, 7), bias=False), bn2(50), a, nn.Flatten(),
                                 nn.Linear(50 * (L - 12), 256, bias=False), bn1(256), a, nn.Linear(256, 80, bias=False))
        self.fc = nn.Sequential(nn.Linear(80, 256, bias=False), bn1(256), a,
                                nn.Linear(256, 50 * (L - 12), bias=False), bn1(50 * (L - 12)), a)
        self.dec = nn.Sequential(nn.ConvTranspose2d(50, 50, (2, 7), bias=False), bn2(50), a,
                                 nn.ConvTranspose2d(50, 1, (1, 7), bias=False))
        self.L = L

    def forward(self, x):
        z = self.enc(x)
        return z, self.dec(self.fc(z).view(-1, 50, 1, self.L - 12))


def test_conv1d_matches_conv2d():
    """Запись через Conv1d (svdd.py) даёт те же выходы, что прямая запись Conv2d, при тех же весах."""
    L = 40
    ref, enc, dec = _Ref2d(L), svdd.Encoder(L), svdd.Decoder(L)
    with torch.no_grad():
        enc.c1.weight.copy_(ref.enc[0].weight.view(50, 1, 7))
        enc.c2.weight.copy_(ref.enc[3].weight.reshape(50, 100, 7))
        enc.fc[1].weight.copy_(ref.enc[7].weight)
        enc.fc[4].weight.copy_(ref.enc[10].weight)
        dec.fc.load_state_dict(ref.fc.state_dict())
        dec.d2.weight.copy_(ref.dec[0].weight.reshape(50, 100, 7))
        dec.d1.weight.copy_(ref.dec[3].weight.view(50, 1, 7))
    x = torch.randn(16, 1, 2, L)
    for train in (True, False):
        for m in (ref, enc, dec):
            m.train(train)
        z_ref, y_ref = ref(x)
        z = enc(x)
        assert torch.allclose(z, z_ref, atol=1e-4)
        assert torch.allclose(dec(z_ref), y_ref, atol=1e-4)


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
