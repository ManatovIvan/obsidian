# Код experiments_v4.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/MRV4HRDT).

Сгенерировано obsidian_export.py — не править вручную.

Модуль в целом: [[Предрегистрация v4|↗ Предрегистрация v4]] · [[Предрегистрация v4|↗ предрегистрация v4 §2]] · [[Предрегистрация v4|↗ предрегистрация v4 §6]]

```python
"""Эксперименты по утверждённой предрегистрации v4 (PREREG_v4_approved.md, §2–§6).

Обучение — только кадры роутера из R1, R2, R3; тест T1 = E1, T2 = E3 (роутер — класс 0, ESP32 — класс 1).
Вход: out/dataset.npz (dataset_build.py, тот же, что в v3). Выход: out/results_v4.jsonl (строка на
реализацию × тест × вариант × зерно), out/experiments_v4_log.txt. Повторный запуск пропускает посчитанное.

Запуск: python experiments_v4.py [--check]   (--check — только проверка признаков на обучающих кадрах роутера)

Обоснования строк кода (источники, цитаты, решения) — в хранилище статьи Obsidian, а не в коде.
"""
import hashlib
import json
import sys
import time
from pathlib import Path

import numpy as np

import experiments as v3
import models
import paradis
import svdd

OUT = Path(__file__).resolve().parent / "out"
```

```python
TRAIN_S = ("R1", "R2", "R3")
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §2]] · [[Physical Layer-Based Device Fingerprinting for Wireless Security  From Theory to Practice#^4wi4s249-6-dzf28r5a|↗ Zhang 2025, с. 6]] ^c-94231c

```python
TESTS = {"T1": "E1", "T2": "E3"}
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §2]] ^c-2b8564

```python
SEEDS = [v3.SEED + k for k in range(10)]          # ВЫБОР: 10 зёрен подряд от зерна v3
```
↳ [[Deep One-Class Classification#^p5cb3pif-7-76i8v7tc|↗ Ruff 2018, с. 7]] · [[Показание IBQ835R2#^t-ibq835r2-2|↗ Показание IBQ835R2]] ^c-9a7943

```python
LOG = OUT / "experiments_v4_log.txt"
RES = OUT / "results_v4.jsonl"


def log(msg):
    with open(LOG, "a") as f:
        f.write(msg + "\n")
    print(msg, flush=True)


def parts(d):
    """Обучение и валидация (последние 20 % времени каждой из R1–R3), тесты T1 и T2."""
    idx = np.nonzero(np.isin(d["session"], TRAIN_S) & (d["label"] == 0))[0]
```

```python
    tr, va, _ = v3.time_split(d, idx, 0.8, 0.2)
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §2]] ^c-685d26

```python
    tests = {t: np.nonzero(d["session"] == s)[0] for t, s in TESTS.items()}
    return tr, va, tests


def ra_features(d, idx, lab, snr, rng):
    """Признаки PARADIS считаются после снятия CFO (захват несущей) в обоих вариантах C4."""
```

```python
    iq = v3.prepare(d, idx, lab, snr, True, rng)
```
↳ [[Wireless device identification with radiometric signatures#^uxhxnmqg-6-8tvb7fip|↗ Brik 2008, с. 6]] ^c-b6e8d2

```python
    return paradis.features(iq, d["bits"][idx], d["cfo"][idx])


def check(d):
    """Проверка инструмента на обучающих кадрах роутера (тестовые кадры не используются)."""
    tr, va, _ = parts(d)
    idx = np.concatenate([tr, va])
    f = ra_features(d, idx, d["label"], False, np.random.default_rng(v3.SEED))
    log(f"--- проверка R-A на {len(idx)} обучающих кадрах роутера: NaN {int(np.isnan(f).sum())}")
    for k, name in enumerate(paradis.NAMES):
        log(f"    {name}: медиана {np.median(f[:, k]):.6g}, P1 {np.percentile(f[:, k], 1):.6g}, "
            f"P99 {np.percentile(f[:, k], 99):.6g}")


def write_row(row):
    with open(RES, "a") as f:
        f.write(json.dumps(row, ensure_ascii=False) + "\n")


def evaluate(key, sc, d, te, lab, done, extra, t0, h):
    if key in done:
        return
```

```python
    a, boots = v3.bootstrap(sc, lab[te], d["order"][te], v3.SEED)
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §6.2]] · [[Предрегистрация v4|↗ предрегистрация v4 §6.3]] ^c-a3d1e5

```python
    row = dict(key=key, auc=a, score_std=float(np.std(sc)), n_unique=int(len(np.unique(sc))),
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §5]] · [[Предрегистрация v4|↗ предрегистрация v4 §6.1]] ^c-62ba1e

```python
               constant=bool(np.std(sc) == 0), n_test=int(len(te)), n_pos=int(lab[te].sum()), boot=boots,
               seconds=round(time.time() - t0, 1), dataset_sha256=h, **extra)
    write_row(row)
    log(f"    {key}: AUC = {a:.4f}, с. к. о. оценки {row['score_std']:.4g}; интервалы: " +
        "; ".join(f"C={b['C']},B={b['B']}: [{b['lo']:.4f}; {b['hi']:.4f}]" for b in boots))


def main():
    h = hashlib.sha256((OUT / "dataset.npz").read_bytes()).hexdigest()
    d = v3.load()
    lab = d["label"]
    log(f"=== запуск v4 {time.strftime('%Y-%m-%d %H:%M:%S')}, dataset.npz SHA-256 {h}, устройство {v3.DEVICE}")
    if "--check" in sys.argv:
        check(d)
        return
    done = {json.loads(l)["key"] for l in RES.read_text().splitlines() if l.strip()} if RES.exists() else set()
    tr, va, tests = parts(d)
```

```python
    tr_all = np.concatenate([tr, va])             # R-A и R-B валидация не нужна — обучение на всех кадрах R1–R3
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §2]] ^c-be4f9a

```python
    log(f"обучение: {len(tr_all)} кадров роутера (R-C: {len(tr)} + валидация {len(va)}); "
        + ", ".join(f"{t}: {len(i)} кадров, ESP32 {int(lab[i].sum())}" for t, i in tests.items()))
```

```python
    for snr in (False, True):
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §3]] ^c-7ae551

```python
        for cfo_rm in (False, True):
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §3]] ^c-92971c

```python
            var = f"s{int(snr)}c{int(cfo_rm)}"
            extra = lambda impl, seed=None, t=None: dict(impl=impl, test=t, variant=var, snr_match=snr,
                                                       cfo_removed=cfo_rm, seed=seed)
            # --- R-A: признаки PARADIS ---
            t0 = time.time()
            keys = [f"{t}|R-A|{var}" for t in tests]
            if not all(k in done for k in keys):
```

```python
                cols = slice(1, None) if cfo_rm else slice(None)   # Г-а: частотная ошибка исключается
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §4 R-A]] ^c-217fa7

```python
                sig = paradis.fit_signature(ra_features(d, tr_all, lab, False, np.random.default_rng(v3.SEED))[:, cols])
                rng = np.random.default_rng(v3.SEED)  # шум C2 — тот же, что у R-B и R-C (тот же порядок T1, T2)
                for t, te in tests.items():
                    f = ra_features(d, te, lab, snr, rng)[:, cols]
                    evaluate(f"{t}|R-A|{var}", paradis.score(f, sig), d, te, lab, done, extra("R-A", None, t), t0, h)
            # --- R-B и R-C: 10 зёрен ---
            for seed in SEEDS:
                for impl in ("R-B", "R-C"):
                    keys = [f"{t}|{impl}|{var}|seed={seed}" for t in tests]
                    if all(k in done for k in keys):
                        continue
                    t0 = time.time()
                    log(f"--- {impl} {var} зерно {seed}")
                    rng = np.random.default_rng(v3.SEED)  # шум C2 — один для всех зёрен и реализаций
                    xtest = {t: v3.prepare(d, te, lab, snr, cfo_rm, rng) for t, te in tests.items()}
                    if impl == "R-B":
```

```python
                        x = v3.prepare(d, tr_all, lab, False, cfo_rm, rng)
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §3]] ^c-594548

```python
                        enc, c = svdd.train(models.to_tensor_m1(x), v3.DEVICE, seed, log)
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §4 R-B]] ^c-171b15

```python
                        scores = {t: svdd.score(enc, c, models.to_tensor_m1(xt), v3.DEVICE) for t, xt in xtest.items()}
                    else:
                        xtr = v3.prepare(d, tr, lab, False, cfo_rm, rng)
                        xva = v3.prepare(d, va, lab, False, cfo_rm, rng)
                        mm = models.minmax_fit(np.stack([xtr.real, xtr.imag], axis=2))
```

```python
                        m = models.train_m2(models.to_tensor_m2(xtr, mm), models.to_tensor_m2(xva, mm),
```
↳ [[Предрегистрация v4|↗ предрегистрация v4 §4 R-C]] ^c-e273ef

```python
                                            v3.DEVICE, seed, log)
                        scores = {t: models.score_m2(m, models.to_tensor_m2(xt, mm), v3.DEVICE) for t, xt in xtest.items()}
                    for t, te in tests.items():
                        evaluate(f"{t}|{impl}|{var}|seed={seed}", scores[t], d, te, lab, done,
                                 extra(impl, seed, t), t0, h)


if __name__ == "__main__":
    sys.exit(main())

```
