# Код rc_cdae.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/AI5VDRIE).

Сгенерировано obsidian_export.py — не править вручную.

Модуль в целом: [[Предрегистрация v4|↗ предрегистрация v4 §4 R-C]] · [[Предрегистрация v4|↗ Предрегистрация v4]] · [[Показание R9STE4E2|↗ Показание R9STE4E2]]

```python
"""R-C: свёрточный автоэнкодер Yu et al. (models.CDAE), обучение только на кадрах роутера, PREREG_v4_approved.md §4.

Копия models.train_m2 / models.score_m2 с одним отличием — смешанной точностью (Журнал отступлений, п. 33):
веса и шаг Adam — FP32, прямой и обратный проход — FP16, масштабирование потерь; ошибка восстановления — FP32.

Обоснования строк кода (источники, цитаты, решения) — в хранилище статьи Obsidian, а не в коде.
"""
import numpy as np
import torch

```

```python
from models import BATCH, CDAE, LR, MAX_EPOCHS, PATIENCE, _batches, significant
```
↳ [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-ixqha2ne|↗ Yu 2019, с. 4]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-r6mqju5g|↗ Yu 2019, с. 4]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-ttggmsga|↗ Yu 2019, с. 4]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-mrnw7if3|↗ Yu 2019, с. 4]] · [[Practical recommendations for gradient-based training of deep architectures#^m6rcp432-10-kuk4ruqs|↗ Bengio 2012, с. 10]] · [[Practical recommendations for gradient-based training of deep architectures#^m6rcp432-10-qmi9kw9w|↗ Bengio 2012, с. 10]] · [↗ DeepLearningTutorials  code mlp.py (ранн](zotero://select/library/items/V5RD886G) · [[Early Stopping - But When#^up4ah8pi-5-4cb8krm9|↗ Prechelt 1998, с. 5]] ^c-c5330d

```python


```

```python
def score(model, x, device):
```
↳ [[Physical Layer-Based Device Fingerprinting for Wireless Security  From Theory to Practice#^4wi4s249-6-2zdp597x|↗ Zhang 2025, с. 6]] ^c-59dc16

```python
    """Оценка аномальности — MSE восстановления каждого окна (в FP32)."""
    model.eval()
    out = []
    with torch.no_grad():
        for i in range(0, len(x), 1024):          # ВЫБОР: 1024 — только размер порции при оценке, на результат не влияет
            xb = x[i:i + 1024].to(device)
```

```python
            with torch.autocast(device.type, dtype=torch.float16):
```
↳ [[Mixed Precision Training#^aephriqm-1-cr32afi3|↗ Micikevicius 2018, с. 1]] · [[Показание R9STE4E2|↗ Показание R9STE4E2]] ^c-b5ad0f

```python
                y = model(xb)
```

```python
            out.append(torch.mean((y.float() - xb) ** 2, dim=(1, 2, 3)).cpu())
```
↳ [[Mixed Precision Training#^aephriqm-5-bst4sizj|↗ Micikevicius 2018, с. 5]] ^c-cdfcea

```python
    return torch.cat(out).numpy()


```

```python
def train(xtr, xva, device, seed, log):
```
↳ [[Physical Layer-Based Device Fingerprinting for Wireless Security  From Theory to Practice#^4wi4s249-6-2zdp597x|↗ Zhang 2025, с. 6]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-ixqha2ne|↗ Yu 2019, с. 4]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-r6mqju5g|↗ Yu 2019, с. 4]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-ttggmsga|↗ Yu 2019, с. 4]] · [[Radio Frequency Fingerprint Identification Based on Denoising Autoencoders#^8m6ki8he-4-mrnw7if3|↗ Yu 2019, с. 4]] ^c-11617e

```python
    """Правило остановки и выбор лучших весов — как в models.train_m2."""
    torch.manual_seed(seed)
    rng = np.random.default_rng(seed)
    model = CDAE().to(device)
    opt = torch.optim.Adam(model.parameters(), lr=LR)
```

```python
    scaler = torch.amp.GradScaler()               # динамический масштаб; шаг с переполнением пропускается
```
↳ [[Mixed Precision Training#^aephriqm-1-9zdrrr86|↗ Micikevicius 2018, с. 1]] · [[Mixed Precision Training#^aephriqm-5-639p5u9t|↗ Micikevicius 2018, с. 5]] · [[Mixed Precision Training#^aephriqm-4-22m9fg48|↗ Micikevicius 2018, с. 4]] · [[Показание R9STE4E2|↗ Показание R9STE4E2]] ^c-5bb8ec

```python
    best, best_state, wait = np.inf, None, 0
    for ep in range(MAX_EPOCHS):
        model.train()
        for b in _batches(len(xtr), rng, BATCH):
            xb = xtr[b].to(device)
            opt.zero_grad()
```

```python
            with torch.autocast(device.type, dtype=torch.float16):
```
↳ [[Mixed Precision Training#^aephriqm-1-cr32afi3|↗ Micikevicius 2018, с. 1]] · [[Показание R9STE4E2|↗ Показание R9STE4E2]] ^c-d4572e

```python
                y = model(xb)
```

```python
            loss = torch.mean((y.float() - xb) ** 2)
```
↳ [[Mixed Precision Training#^aephriqm-5-bst4sizj|↗ Micikevicius 2018, с. 5]] · [[Physical Layer-Based Device Fingerprinting for Wireless Security  From Theory to Practice#^4wi4s249-6-2zdp597x|↗ Zhang 2025, с. 6]] ^c-b87eaa

```python
            scaler.scale(loss).backward()
            scaler.step(opt)
            scaler.update()
        va = float(np.mean(score(model, xva, device)))
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

```
