# Код roc_v4.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/PFHTNIZG).

Сгенерировано obsidian_export.py — не править вручную.

Ссылаются на модуль целиком: [[Текст статьи#^u3-3|Текст статьи ^u3-3]]

Модуль в целом: [[Показание 9K4WBJ3W|↗ Показание 9K4WBJ3W]]

```python
"""Оценки кадров для ROC-кривых рисунка статьи: тест T2 (запись dev. board), вариант s1c1 (оба контроля).

Прогон v4 (experiments_v4.py) сохранял AUC и интервалы, но не оценки кадров. Здесь тот же расчёт повторяется
для одного варианта и первого зерна (SEEDS[0]): те же функции, тот же порядок вызовов генератора шума.
AUC повтора сравнивается с results_v4.jsonl; R-B и R-C могут отличаться в последних знаках — cudnn
недетерминирован (experiments.py, Журнал отступлений, п. 11).
Выход: out/roc_v4_T2_s1c1.npz (оценки и метки), строки в out/roc_v4_log.txt.

Запуск: python roc_v4.py

Обоснования строк кода (источники, цитаты, решения) — в хранилище статьи Obsidian, а не в коде.
"""
import json
import sys
import time
from pathlib import Path

import numpy as np

import experiments as v3
import experiments_v4 as e4
import metrics
import models
import paradis
import rc_cdae
import svdd

OUT = Path(__file__).resolve().parent / "out"
LOG = OUT / "roc_v4_log.txt"
```

```python
SNR, CFO_RM, VAR, SEED = True, True, "s1c1", e4.SEEDS[0]
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §3]] · [[Показание 9K4WBJ3W|↗ Показание 9K4WBJ3W]] ^c-29ab4f

```python


def log(msg):
    with open(LOG, "a") as f:
        f.write(msg + "\n")
    print(msg, flush=True)


def main():
    d = v3.load()
    lab = d["label"]
    tr, va, tests = e4.parts(d)
    tr_all = np.concatenate([tr, va])
    te = tests["T2"]
    log(f"=== повтор для ROC {time.strftime('%Y-%m-%d %H:%M:%S')}, {VAR}, зерно {SEED}, устройство {v3.DEVICE}")
    sc = {}
    # R-A — как в experiments_v4.main при cfo_rm=True: частотная ошибка исключена, шум C2 по порядку T1, T2
```

```python
    sig = paradis.fit_signature(e4.ra_features(d, tr_all, lab, False, np.random.default_rng(v3.SEED))[:, 1:])
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §4 R-A]] ^c-15aaa9

```python
    rng = np.random.default_rng(v3.SEED)
    f = {t: e4.ra_features(d, i, lab, SNR, rng)[:, 1:] for t, i in tests.items()}
    sc["R-A"] = paradis.score(f["T2"], sig)
    # R-B и R-C — как в experiments_v4.main: свой генератор шума на каждую реализацию, тесты готовятся до обучения
    for impl in ("R-B", "R-C"):
        rng = np.random.default_rng(v3.SEED)
        xtest = {t: v3.prepare(d, i, lab, SNR, CFO_RM, rng) for t, i in tests.items()}
        if impl == "R-B":
            x = v3.prepare(d, tr_all, lab, False, CFO_RM, rng)
```

```python
            enc, c = svdd.train(models.to_tensor_m1(x), v3.DEVICE, SEED, log)
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §4 R-B]] ^c-77b757

```python
            sc[impl] = svdd.score(enc, c, models.to_tensor_m1(xtest["T2"]), v3.DEVICE)
        else:
            xtr = v3.prepare(d, tr, lab, False, CFO_RM, rng)
            xva = v3.prepare(d, va, lab, False, CFO_RM, rng)
            mm = models.minmax_fit(np.stack([xtr.real, xtr.imag], axis=2))
```

```python
            m = rc_cdae.train(models.to_tensor_m2(xtr, mm), models.to_tensor_m2(xva, mm), v3.DEVICE, SEED, log)
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §4 R-C]] ^c-d148bd

```python
            sc[impl] = rc_cdae.score(m, models.to_tensor_m2(xtest["T2"], mm), v3.DEVICE)
    pub = {json.loads(l)["key"]: json.loads(l)["auc"] for l in (OUT / "results_v4.jsonl").read_text().splitlines() if l.strip()}
    for impl, s in sc.items():
        key = f"T2|{impl}|{VAR}" + ("" if impl == "R-A" else f"|seed={SEED}")
        log(f"    {key}: AUC повтора {metrics.auc(s, lab[te]):.6f}, в results_v4.jsonl {pub[key]:.6f}")
    np.savez(OUT / f"roc_v4_T2_{VAR}.npz", label=lab[te], **{k.replace("-", "_"): v for k, v in sc.items()})


if __name__ == "__main__":
    sys.exit(main())

```
