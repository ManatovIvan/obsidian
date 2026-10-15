# Код models.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/8EW7PQZE).

Сгенерировано obsidian_export.py — не править вручную.

Ссылаются на модуль целиком: [[Текст статьи#^u2-4|Текст статьи ^u2-4]]

Модуль в целом: [[Предрегистрация|↗ предрегистрация §2]] · [[Предрегистрация|↗ предрегистрация §3]] · [[Предрегистрация|↗ Предрегистрация]]

```python
"""Предобработка (C1, C2, C4) и модели M1–M3 по утверждённой предрегистрации.

Обоснования строк кода (источники, цитаты, решения) — в хранилище статьи Obsidian, а не в коде.
"""
import numpy as np
import torch
from torch import nn

import iq_io

# --- Предобработка ---------------------------------------------------------------------------


```

```python
def remove_cfo(iq, cfo_hz):
```
↳ [[Предрегистрация|↗ предрегистрация §2 C4]] ^c-4b0a68

```python
    """C4 (б): поворот каждого окна на оценку CFO кадра (оценка — из dsss_rx)."""
    n = np.arange(iq.shape[1])
    return iq * np.exp(-2j * np.pi * cfo_hz[:, None] * n[None, :] / iq_io.SAMPLE_RATE)


```

```python
def match_snr(iq, p_win, noise, is_router, pair_snr, rng):
```
↳ [[Radio frequency fingerprint identification for Internet of Things  A survey#^zvd6asea-17-4m8xmchv|↗ Xie 2023, с. 17]] ^c-e9eb1f

```python
    """C2: к кадрам роутера добавляется белый гауссов шум до SNR парного кадра ESP32:
    SNR = (P − Pш)/Pш; новая дисперсия шума σ² = (P − Pш)/SNR_цель − Pш (если > 0)."""
    out = iq.copy()
    for i in np.nonzero(is_router)[0]:
        sig = p_win[i] - noise[i]
        var = sig / pair_snr[i] - noise[i]
        if var > 0:
            out[i] += (rng.standard_normal(iq.shape[1]) + 1j * rng.standard_normal(iq.shape[1])) * np.sqrt(var / 2)
    return out


```

```python
def energy_normalize(iq):
```
↳ [[Design a Deep Neural Network with Simulated Data to Detect WLAN Router Impersonation#^8b73m9nd-4-e5srk9qu|↗ MathWorks, с. 4]] ^c-2e61cf

```python
    """C1: энергия окна нормируется к 1 (средняя мощность 1)."""
    p = np.mean(np.abs(iq) ** 2, axis=1, keepdims=True)
    return iq / np.sqrt(p)


```

```python
def to_tensor_m1(iq):
```
↳ [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-qfbxfmh9|↗ Al-Shawabka 2020, с. 3]] ^c-0caa6b

```python
    """M1: вход 2 × L (строки I и Q), чтобы ядра свёртки видели I и Q вместе."""
    x = np.stack([iq.real, iq.imag], axis=1).astype(np.float32)       # (N, 2, L)
    return torch.from_numpy(x[:, None])                               # (N, 1, 2, L)


# --- M1: CNN «Homegrown» ---------------------------------------------------------------------


```

```python
class Homegrown(nn.Module):
```
↳ [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-d7ua9p5g|↗ Al-Shawabka 2020, с. 3]] · [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-watm7kbz|↗ Al-Shawabka 2020, с. 3]] · [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-5sd89kf8|↗ Al-Shawabka 2020, с. 3]] · [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-xfun3crc|↗ Al-Shawabka 2020, с. 3]] · [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-t7pjdbsj|↗ Al-Shawabka 2020, с. 3]] ^c-be1a2b

```python
    """CNN «Homegrown»: две 1D-свёртки по 50 фильтров (ядра 1×7 и 2×7), ReLU, два полносвязных слоя
    256 и 80, softmax, dropout 0,5.
    ВЫБОР: свёртки без дополнения (padding) — в статье не указано; dropout — после слоя 256
    (в статье одно место не названо)."""

    def __init__(self, length):
        super().__init__()
        self.c1 = nn.Conv2d(1, 50, (1, 7))
        self.c2 = nn.Conv2d(50, 50, (2, 7))
        self.f1 = nn.Linear(50 * (length - 12), 256)
        self.f2 = nn.Linear(256, 80)
        self.out = nn.Linear(80, 2)
```

```python
        self.act = nn.ReLU()
```
↳ [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-w9exzxst|↗ Al-Shawabka 2020, с. 3]] ^c-ba3076

```python
        self.drop = nn.Dropout(0.5)

    def forward(self, x):
        x = self.act(self.c2(self.act(self.c1(x))))
        x = self.drop(self.act(self.f1(x.flatten(1))))
        return self.out(self.act(self.f2(x)))     # логиты; softmax — в функции потерь и при оценке


# --- M2: свёрточный автоэнкодер (Yu et al., табл. I) -----------------------------------------


```

```python
class CDAE(nn.Module):
```
↳ [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-r2rskxhm|↗ Yu 2019, с. 4]] · [↗ Yu 2019](zotero://select/library/items/8M6KI8HE) · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-4fddxmzu|↗ Yu 2019, с. 4]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-kk4wmij3|↗ Yu 2019, с. 4]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-8uic8a9p|↗ Yu 2019, с. 4]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-zhft5nqx|↗ Yu 2019, с. 4]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-erpih9vi|↗ Yu 2019, с. 4]] · [[Physical Layer-Based Device Fingerprinting for Wireless Security  From Theory to Practice#^4wi4s249-5-x7aiucm3|↗ Zhang 2025, с. 5]] ^c-fcf652

```python
    """Свёрточный автоэнкодер (Yu et al., табл. I), вход 1280 × 2: кодировщик Conv 128×(10×1), MaxPool 4×1,
    Conv 128×(3×2), MaxPool 4×1; декодер — зеркально с UpSampling, выход Conv 1×(3×1) с Sigmoid.
    Классификатор из табл. I не используется: атакующий при обучении недоступен.
    ВЫБОР: дополнение 'same' — иначе размер выхода не совпадёт со входом (в статье не указано)."""

    def __init__(self):
        super().__init__()
        r = nn.ReLU()
        self.net = nn.Sequential(
            nn.Conv2d(1, 128, (10, 1), padding="same"), r, nn.MaxPool2d((4, 1)),
            nn.Conv2d(128, 128, (3, 2), padding="same"), r, nn.MaxPool2d((4, 1)),
            nn.Conv2d(128, 128, (3, 2), padding="same"), r, nn.Upsample(scale_factor=(4, 1)),
            nn.Conv2d(128, 128, (10, 1), padding="same"), r, nn.Upsample(scale_factor=(4, 1)),
            nn.Conv2d(128, 1, (3, 1), padding="same"), nn.Sigmoid())

    def forward(self, x):                         # x: (N, 1, 1280, 2)
        return self.net(x)


```

```python
def minmax_fit(x):
```
↳ [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-2-asf56xtq|↗ Yu 2019, с. 2]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-2-cup66e65|↗ Yu 2019, с. 2]] ^c-f91fcd

```python
    """Min-max нормировка: min и max по каждому измерению берутся на обучающих данных,
    чтобы вход был в том же диапазоне [0, 1], что выход декодера (Sigmoid)."""
    return x.min(axis=0), x.max(axis=0)


def to_tensor_m2(iq, mm):
    x = np.stack([iq.real, iq.imag], axis=2).astype(np.float32)       # (N, 1280, 2)
    lo, hi = (np.asarray(v, np.float32) for v in mm)                  # float32 — тот же тип, что у весов сети
    x = ((x - lo) / np.maximum(hi - lo, np.float32(1e-12))).astype(np.float32)
    return torch.from_numpy(x[:, None])                               # (N, 1, 1280, 2)


# --- Обучение (общая процедура по Yu et al.) -------------------------------------------------
```

```python
BATCH, LR, PATIENCE = 64, 1e-3, 10
# Что считать улучшением при обучении мини-батчами — только значимое улучшение на валидации;
# порог значимости — относительное улучшение не менее 0,5 % (improvement_threshold = 0.995).
```
↳ [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-ixqha2ne|↗ Yu 2019, с. 4]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-r6mqju5g|↗ Yu 2019, с. 4]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-ttggmsga|↗ Yu 2019, с. 4]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-mrnw7if3|↗ Yu 2019, с. 4]] ^c-f61f50

```python
IMPROVEMENT_THRESHOLD = 0.995
# Терпение Yu сбрасывается только значимым улучшением; лучшие веса сохраняются при любом улучшении,
# как в Tutorials (best_validation_loss обновляется при любом this_validation_loss < best_validation_loss).
# Предел числа эпох — только страховка
```
↳ [[Practical recommendations for gradient-based training of deep architectures#^m6rcp432-10-kuk4ruqs|↗ Bengio 2012, с. 10]] · [[Practical recommendations for gradient-based training of deep architectures#^m6rcp432-10-qmi9kw9w|↗ Bengio 2012, с. 10]] · [↗ DeepLearningTutorials  code mlp.py (ранн](zotero://select/library/items/V5RD886G) ^c-11aea1

```python
MAX_EPOCHS = 3000
```
↳ [[Early Stopping - But When#^up4ah8pi-5-4cb8krm9|↗ Prechelt 1998, с. 5]] ^c-740325

```python


```

```python
def significant(new_loss, best_loss):
```
↳ [↗ DeepLearningTutorials  code mlp.py (ранн](zotero://select/library/items/V5RD886G) ^c-6699fb

```python
    """Новый минимум значим, если ошибка меньше лучшей не менее чем в 0,995 раза."""
    return new_loss < best_loss * IMPROVEMENT_THRESHOLD


def _batches(n, rng, size=BATCH):
    p = rng.permutation(n)
    for i in range(0, n, size):
        yield p[i:i + size]


def _predict(model, x, device, fn):
    model.eval()
    out = []
    with torch.no_grad():
        for i in range(0, len(x), 1024):          # ВЫБОР: 1024 — только размер порции при оценке, на результат не влияет
            out.append(fn(model, x[i:i + 1024].to(device)).cpu())
    return torch.cat(out).numpy()


```

```python
def train_m1(xtr, ytr, xva, yva, device, seed, log):
```
↳ [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-rmcjgpdd|↗ Al-Shawabka 2020, с. 3]] · [↗ DeepLearningTutorials  code mlp.py (ранн](zotero://select/library/items/V5RD886G) ^c-30b154

```python
    """M1: функция потерь — categorical cross-entropy.
    Ошибка на валидации для правила остановки — доля неверных ответов (zero-one loss).
    ВЫБОР: в каждой эпохе класс-большинство случайно прореживается до размера меньшего класса
    (в E1 кадров ESP32 в 18 раз больше, чем роутера)."""
    torch.manual_seed(seed)
    rng = np.random.default_rng(seed)
    model = Homegrown(xtr.shape[-1]).to(device)
    opt = torch.optim.Adam(model.parameters(), lr=LR)
    lossf = nn.CrossEntropyLoss()
    best, best_state, wait = np.inf, None, 0
    i0, i1 = np.nonzero(ytr == 0)[0], np.nonzero(ytr == 1)[0]
    m = min(len(i0), len(i1))
    for ep in range(MAX_EPOCHS):
        sel = np.concatenate([rng.choice(i0, m, replace=False), rng.choice(i1, m, replace=False)])
        model.train()
        for b in _batches(len(sel), rng):
            j = sel[b]
            opt.zero_grad()
            loss = lossf(model(xtr[j].to(device)), torch.from_numpy(ytr[j].astype(np.int64)).to(device))
            loss.backward()
            opt.step()
        pred = _predict(model, xva, device, lambda md, x: md(x).argmax(1))
        err = float(np.mean(pred != yva))
        sig = significant(err, best)
        log(f"    эпоха {ep}: ошибка на валидации {err:.5f}" + (" (значимое улучшение)" if sig else ""))
        if err < best:                            # лучшие веса — при любом улучшении
            best = err
            best_state = {k: v.detach().clone() for k, v in model.state_dict().items()}
        wait = 0 if sig else wait + 1             # терпение сбрасывает только значимое улучшение
        if wait >= PATIENCE:
            break
    model.load_state_dict(best_state)
    return model


def score_m1(model, x, device):
    """Оценка — вероятность класса «ESP32» (softmax)."""
    return _predict(model, x, device, lambda md, xx: torch.softmax(md(xx), 1)[:, 1])


```

```python
def train_m2(xtr, xva, device, seed, log):
```
↳ [[Physical Layer-Based Device Fingerprinting for Wireless Security  From Theory to Practice#^4wi4s249-6-2zdp597x|↗ Zhang 2025, с. 6]] ^c-7b16f0

```python
    """M2: обучение только на кадрах роутера; ошибка восстановления — MSE.
    ВЫБОР: ошибка на валидации для остановки — MSE (у автоэнкодера нет точности); правило — как в M1."""
    torch.manual_seed(seed)
    rng = np.random.default_rng(seed)
    model = CDAE().to(device)
    opt = torch.optim.Adam(model.parameters(), lr=LR)
    best, best_state, wait = np.inf, None, 0
    for ep in range(MAX_EPOCHS):
        model.train()
        for b in _batches(len(xtr), rng):
            xb = xtr[b].to(device)
            opt.zero_grad()
            loss = torch.mean((model(xb) - xb) ** 2)
            loss.backward()
            opt.step()
        va = float(np.mean(score_m2(model, xva, device)))
        sig = significant(va, best)
        log(f"    эпоха {ep}: MSE на валидации {va:.6f}" + (" (значимое улучшение)" if sig else ""))
        if va < best:                             # лучшие веса — при любом улучшении
            best = va
            best_state = {k: v.detach().clone() for k, v in model.state_dict().items()}
        wait = 0 if sig else wait + 1             # терпение сбрасывает только значимое улучшение
        if wait >= PATIENCE:
            break
    model.load_state_dict(best_state)
    return model


def score_m2(model, x, device):
    """Оценка аномальности — MSE восстановления каждого окна."""
    return _predict(model, x, device, lambda md, xx: torch.mean((md(xx) - xx) ** 2, dim=(1, 2, 3)))


```

```python
def score_m3(cfo_test, cfo_train_router):
```
↳ [[Предрегистрация|↗ предрегистрация §3]] ^c-b503b8

```python
    """M3: |CFO кадра − медиана CFO роутера на обучающих данных|."""
    return np.abs(cfo_test - np.median(cfo_train_router))

```
