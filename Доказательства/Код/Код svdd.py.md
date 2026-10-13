# Код svdd.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/UI87KMRE).

Сгенерировано obsidian_export.py — не править вручную.

Модуль в целом: [[Предрегистрация v4|↗ предрегистрация v4 §4 R-B]] · [[Предрегистрация v4|↗ Предрегистрация v4]] · [[Показание IBQ835R2|↗ Показание IBQ835R2]]

```python
"""R-B: одноклассовая CNN — One-Class Deep SVDD (Ruff et al.) на сети «Homegrown», PREREG_v4_approved.md §4.

Порядок: предобучение автоэнкодера (DCAE) -> веса кодировщика переносятся в φ -> центр c -> обучение
Deep SVDD. Оценка аномальности — ‖φ(x) − c‖².

Обоснования строк кода (источники, цитаты, решения) — в хранилище статьи Obsidian, а не в коде.
"""
import numpy as np
import torch
from torch import nn

```

```python
BATCH = 200
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-rmpbsxc4|↗ Ruff 2018, с. 7]] ^c-c04dca

```python
WEIGHT_DECAY = 1e-6
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-rmpbsxc4|↗ Ruff 2018, с. 7]] ^c-96be67

```python
LR = (1e-4, 1e-5)                                 # два этапа: поиск и дообучение
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-fewqenth|↗ Ruff 2018, с. 7]] ^c-f901c2

```python
EPOCHS_SVDD = (150, 100)
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-55na95mg|↗ Ruff 2018, с. 7]] ^c-e2e18e

```python
EPOCHS_DCAE = (250, 100)
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-5atzmsuh|↗ Ruff 2018, с. 7]] ^c-56017c

```python
LEAK = 0.1
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-nhkx7jg2|↗ Ruff 2018, с. 7]] ^c-69ee71

```python
REP = 80                                          # размер представления φ — последний скрытый слой Homegrown
```
↳ [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-5sd89kf8|↗ Al-Shawabka 2020, с. 3]] ^c-ac4ec8

```python


def _bn1(n):
```

```python
    return nn.BatchNorm1d(n, affine=False)        # без обучаемого сдвига (ВЫБОР, см. предрегистрацию)
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-nq639br3|↗ Ruff 2018, с. 7]] · [[Deep One-Class Classification#^p5cb3pif-6-s2n3wvea|↗ Ruff 2018, с. 6]] ^c-771261

```python


```

```python
class Encoder(nn.Module):
```
↳ [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-d7ua9p5g|↗ Al-Shawabka 2020, с. 3]] · [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-watm7kbz|↗ Al-Shawabka 2020, с. 3]] · [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-5sd89kf8|↗ Al-Shawabka 2020, с. 3]] · [[Deep One-Class Classification#^p5cb3pif-6-s2n3wvea|↗ Ruff 2018, с. 6]] · [[Показание IBQ835R2#^t-ibq835r2-2|↗ Показание IBQ835R2]] ^c-7942ea

```python
    """φ: Homegrown без смещений и без dropout; две свёртки по 50 фильтров (1×7 и 2×7), слои 256 и 80.
    Свёртки записаны через Conv1d (Журнал отступлений, п. 33): 1×7 — одна и та же свёртка по строке I и по
    строке Q; 2×7 — свёртка по 100 входам (50 каналов × строки I, Q), вход k = 2·канал + строка."""

    def __init__(self, length):
        super().__init__()
        self.length = length
        self.flat = 50 * (length - 12)            # две свёртки без дополнения укорачивают вход на 6 + 6
```

```python
        self.c1 = nn.Conv1d(1, 50, 7, bias=False)
```
↳ [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-watm7kbz|↗ Al-Shawabka 2020, с. 3]] · [[Deep One-Class Classification#^p5cb3pif-6-s2n3wvea|↗ Ruff 2018, с. 6]] · [[Показание R9STE4E2|↗ Показание R9STE4E2]] ^c-1fe43e

```python
        self.b1 = _bn1(50)
```

```python
        self.c2 = nn.Conv1d(100, 50, 7, bias=False)
```
↳ [[Exposing the Fingerprint  Dissecting the Impact of the Wireless Channel on Radio Fingerprinting#^m8af4mw7-3-watm7kbz|↗ Al-Shawabka 2020, с. 3]] · [[Deep One-Class Classification#^p5cb3pif-6-s2n3wvea|↗ Ruff 2018, с. 6]] · [[Показание R9STE4E2|↗ Показание R9STE4E2]] ^c-b91f4a

```python
        self.b2 = _bn1(50)
        self.act = nn.LeakyReLU(LEAK)
        self.fc = nn.Sequential(
            nn.Flatten(),
            nn.Linear(self.flat, 256, bias=False), _bn1(256), nn.LeakyReLU(LEAK),
            nn.Linear(256, REP, bias=False))      # ВЫБОР: выход φ — линейный, без BN и активации

    def forward(self, x):                         # x: (N, 1, 2, L)
        n, L = x.shape[0], self.length
        h = self.c1(x.reshape(n * 2, 1, L)).view(n, 2, 50, L - 6).transpose(1, 2)   # (N, 50, 2, L−6)
        h = self.act(self.b1(h.reshape(n, 50, 2 * (L - 6)))).view(n, 50, 2, L - 6)  # BN — по каналу, как BatchNorm2d
        h = self.act(self.b2(self.c2(h.reshape(n, 100, L - 6))))                    # (N, 50, L−12)
        return self.fc(h)


```

```python
class Decoder(nn.Module):
```
↳ [[Deep One-Class Classification#^p5cb3pif-6-pvsbekrs|↗ Ruff 2018, с. 6]] · [[Deep One-Class Classification#^p5cb3pif-6-zgz9tz97|↗ Ruff 2018, с. 6]] ^c-a2e79f

```python
    """Зеркало кодировщика: полносвязные 80 -> 256 -> 50·(L−12), затем транспонированные свёртки 2×7 и 1×7
    (через ConvTranspose1d, как в кодировщике)."""

    def __init__(self, length):
        super().__init__()
        self.length = length
        self.fc = nn.Sequential(
            nn.Linear(REP, 256, bias=False), _bn1(256), nn.LeakyReLU(LEAK),
            nn.Linear(256, 50 * (length - 12), bias=False), _bn1(50 * (length - 12)), nn.LeakyReLU(LEAK))
```

```python
        self.d2 = nn.ConvTranspose1d(50, 100, 7, bias=False)
```
↳ [[Deep One-Class Classification#^p5cb3pif-6-zgz9tz97|↗ Ruff 2018, с. 6]] · [[Показание R9STE4E2|↗ Показание R9STE4E2]] ^c-0b045d

```python
        self.b2 = _bn1(50)
        self.act = nn.LeakyReLU(LEAK)
```

```python
        self.d1 = nn.ConvTranspose1d(50, 1, 7, bias=False)   # ВЫБОР: выход линейный — вход не ограничен диапазоном
```
↳ [[Deep One-Class Classification#^p5cb3pif-6-zgz9tz97|↗ Ruff 2018, с. 6]] ^c-5360de

```python

    def forward(self, z):
        n, L = z.shape[0], self.length
        h = self.d2(self.fc(z).view(n, 50, L - 12))                                 # (N, 100, L−6)
        h = self.act(self.b2(h.reshape(n, 50, 2 * (L - 6)))).view(n, 50, 2, L - 6)
        h = h.transpose(1, 2).reshape(n * 2, 50, L - 6)                             # строки I, Q — отдельно
        return self.d1(h).view(n, 1, 2, L)


```

```python
def _glorot(module):
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-j256bt9f|↗ Ruff 2018, с. 7]] ^c-030c57

```python
    for m in module.modules():
        if isinstance(m, (nn.Conv1d, nn.ConvTranspose1d, nn.Linear)):
            nn.init.xavier_uniform_(m.weight)     # равномерная инициализация Glorot


def _run(params, loss_fn, x, epochs, rng, device, log, tag):
    """Adam с параметрами по умолчанию, затухание весов λ, два этапа скорости обучения.
    Смешанная точность (Журнал отступлений, п. 33): веса и шаг Adam — FP32, проход — FP16, масштабирование потерь."""
```

```python
    opt = torch.optim.Adam(params, lr=LR[0], weight_decay=WEIGHT_DECAY)
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-nq639br3|↗ Ruff 2018, с. 7]] · [[Deep One-Class Classification#^p5cb3pif-7-rmpbsxc4|↗ Ruff 2018, с. 7]] ^c-b3d84a

```python
    scaler = torch.amp.GradScaler()               # динамический масштаб; шаг с переполнением пропускается
```
↳ [[Mixed Precision Training#^aephriqm-1-9zdrrr86|↗ Micikevicius 2018, с. 1]] · [[Mixed Precision Training#^aephriqm-5-639p5u9t|↗ Micikevicius 2018, с. 5]] · [[Mixed Precision Training#^aephriqm-4-22m9fg48|↗ Micikevicius 2018, с. 4]] · [[Показание R9STE4E2|↗ Показание R9STE4E2]] ^c-e28f19

```python
    ep_total = 0
```

```python
    for lr, n_ep in zip(LR, epochs):
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-fewqenth|↗ Ruff 2018, с. 7]] ^c-1e13a6

```python
        for g in opt.param_groups:
            g["lr"] = lr
        for _ in range(n_ep):
            perm = rng.permutation(len(x))
            tot = 0.0
            for i in range(0, len(x), BATCH):
                xb = x[perm[i:i + BATCH]].to(device)
                opt.zero_grad()
```

```python
                with torch.autocast(device.type, dtype=torch.float16):
```
↳ [[Mixed Precision Training#^aephriqm-1-cr32afi3|↗ Micikevicius 2018, с. 1]] · [[Mixed Precision Training#^aephriqm-5-8ukj2t95|↗ Micikevicius 2018, с. 5]] · [[Показание R9STE4E2|↗ Показание R9STE4E2]] ^c-cd56e1

```python
                    loss = loss_fn(xb)
                scaler.scale(loss).backward()
                scaler.step(opt)
                scaler.update()
                tot += float(loss.detach()) * len(xb)
            ep_total += 1
            if ep_total % 10 == 0 or ep_total == sum(epochs):
                log(f"    {tag} эпоха {ep_total}: потеря {tot / len(x):.6g}")


def _forward_all(model, x, device):
    model.eval()
    out = []
    with torch.no_grad():
        for i in range(0, len(x), 1024):          # ВЫБОР: 1024 — только размер порции при оценке, на результат не влияет
```

```python
            with torch.autocast(device.type, dtype=torch.float16):
```
↳ [[Mixed Precision Training#^aephriqm-1-cr32afi3|↗ Micikevicius 2018, с. 1]] · [[Показание R9STE4E2|↗ Показание R9STE4E2]] ^c-ce2e2e

```python
                out.append(model(x[i:i + 1024].to(device)).float().cpu())   # представления — в FP32
```
↳ [[Mixed Precision Training#^aephriqm-5-bst4sizj|↗ Micikevicius 2018, с. 5]] ^c-aa6052

```python
    return torch.cat(out)


```

```python
def train(x, device, seed, log):
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-6i5wqda2|↗ Ruff 2018, с. 7]] · [[Показание IBQ835R2#^t-ibq835r2-2|↗ Показание IBQ835R2]] ^c-11b665

```python
    """x — тензор (N, 1, 2, L) кадров роутера. Возвращает (φ, c)."""
```

```python
    torch.manual_seed(seed)
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-76i8v7tc|↗ Ruff 2018, с. 7]] · [[Показание IBQ835R2#^t-ibq835r2-2|↗ Показание IBQ835R2]] ^c-52eac9

```python
    rng = np.random.default_rng(seed)
    enc, dec = Encoder(x.shape[-1]).to(device), Decoder(x.shape[-1]).to(device)
    _glorot(enc)
    _glorot(dec)
    ae = nn.Sequential(enc, dec)
    ae.train()
```

```python
    _run(ae.parameters(), lambda xb: torch.mean((ae(xb).float() - xb) ** 2), x, EPOCHS_DCAE, rng, device, log, "DCAE")
```
↳ [[Deep One-Class Classification#^p5cb3pif-6-4hsp4azc|↗ Ruff 2018, с. 6]] ^c-b30fc7

```python
    c = _forward_all(enc, x, device).mean(dim=0).to(device)   # центр — среднее представлений после прямого прохода
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-392frh9k|↗ Ruff 2018, с. 7]] ^c-20026f

```python
    enc.train()
```

```python
    _run(enc.parameters(), lambda xb: torch.mean(torch.sum((enc(xb).float() - c) ** 2, dim=1)), x, EPOCHS_SVDD, rng,
```
↳ [[Deep One-Class Classification#^p5cb3pif-4-wfcc8525|↗ Ruff 2018, с. 4]] · [[Deep One-Class Classification#^p5cb3pif-4-cvcmfqqm|↗ Ruff 2018, с. 4]] ^c-91f5c2

```python
         device, log, "SVDD")
    return enc, c


```

```python
def score(enc, c, x, device):
```
↳ [[Deep One-Class Classification#^p5cb3pif-4-fi3batzx|↗ Ruff 2018, с. 4]] ^c-55acbb

```python
    """Оценка аномальности — квадрат расстояния представления до центра."""
    z = _forward_all(enc, x, device).to(device)
    return torch.sum((z - c) ** 2, dim=1).cpu().numpy()

```
