# Код experiments.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/E565W2EF).

Сгенерировано obsidian_export.py — не править вручную.

Ссылаются на модуль целиком: [[Текст статьи#^u2-4|Текст статьи ^u2-4]]

Модуль в целом: [[Предрегистрация|↗ Предрегистрация]]

```python
"""Эксперименты по утверждённой предрегистрации (PREREG_v3_approved.md, §2–§5).

Вход: out/dataset.npz (dataset_build.py). Выход: out/results.jsonl (по строке на модель × разбиение ×
вариант), out/experiments_log.txt (ход обучения). Повторный запуск пропускает уже посчитанные строки.

Обоснования строк кода (источники, цитаты, решения) — в хранилище статьи Obsidian, а не в коде.
"""
import hashlib
import json
import sys
import time
from pathlib import Path

import numpy as np
import torch

import metrics
import models

OUT = Path(__file__).resolve().parent / "out"
```

```python
SESSIONS = ["R1", "E1", "R2", "R3", "E3"]  # порядок записи без E2
```
↳ [[Предрегистрация|↗ предрегистрация §1]] ^c-48922e

```python
SESSION_S = 120.0
```
↳ [[Показание 83U3GZXN#^t-83u3gzxn-2|↗ Показание 83U3GZXN]] ^c-feae7f

```python
SEED = 20260504                                   # ВЫБОР: одно фиксированное зерно для всего
```

```python
BOOT_C = (0.5, 1, 2)
```
↳ [[Предрегистрация|↗ предрегистрация §5.2–5.3]] ^c-0e6937

```python
BOOT_N = (1000, 2000)
```
↳ [[Предрегистрация|↗ предрегистрация §5.3]] ^c-5275ba

```python
DEVICE = torch.device("cuda" if torch.cuda.is_available() else "cpu")
torch.backends.cudnn.deterministic = False        # ВЫБОР: в 2,8 раза быстрее (замер 36,5 → 12,9 с/эпоху M2);
torch.backends.cudnn.benchmark = False            #   зёрна фиксированы, повтор может отличаться в последних знаках (Журнал отступлений, п. 11)


def log(msg):
    with open(OUT / "experiments_log.txt", "a") as f:
        f.write(msg + "\n")
    print(msg, flush=True)


def load():
    d = dict(np.load(OUT / "dataset.npz"))
    d["t"] = d["a"] / 20e6                        # время кадра в записи, с (20 MSPS — iq_io.SAMPLE_RATE)
    d["order"] = np.array([SESSIONS.index(s) for s in d["session"]]) * 1000 + d["t"]  # время с учётом порядка сессий
    return d


def time_split(d, idx, tr_frac, va_frac):
    """Деление кадров idx по времени внутри своей сессии: [0, tr) — обучение, [tr, tr+va) — валидация, остальное — тест."""
    t = d["t"][idx]
    a, b = tr_frac * SESSION_S, (tr_frac + va_frac) * SESSION_S
    return idx[t < a], idx[(t >= a) & (t < b)], idx[t >= b]


```

```python
def splits(d):
```
↳ [[Предрегистрация|↗ предрегистрация §4.]] ^c-db7c07

```python
    """Возвращает список (имя, обучение, валидация, тест, метки)."""
    n = len(d["label"])
    out = []
    rng = np.random.default_rng(SEED)
    p = rng.permutation(n)                        # S1: случайное 70/20/10
    out.append(("S1", p[:int(0.7 * n)], p[int(0.7 * n):int(0.9 * n)], p[int(0.9 * n):], d["label"]))
    for sid in ("E1", "E3"):                      # S2: внутри сессии по времени 70/20/10
        tr, va, te = time_split(d, np.nonzero(d["session"] == sid)[0], 0.7, 0.2)
        out.append((f"S2-{sid}", tr, va, te, d["label"]))
    for name, train_s, test_s in (("S3-A", ("R1", "E1", "R2"), ("R3", "E3")),
                                  ("S3-B", ("R2", "R3", "E3"), ("R1", "E1"))):
        idx = np.nonzero(np.isin(d["session"], train_s))[0]
        # ВЫБОР: валидация — последние 2/9 времени обучающих сессий (то же отношение 70:20, что в S1/S2)
        tr, va, _ = time_split(d, idx, 7 / 9, 2 / 9)
        te = np.nonzero(np.isin(d["session"], test_s))[0]
        out.append((name, tr, va, te, d["label"]))
    router = d["label"] == 0
    for i, sa in enumerate(SESSIONS):             # S4: сессия как класс, только роутер, все пары
        for sb in SESSIONS[i + 1:]:
            parts = [time_split(d, np.nonzero(router & (d["session"] == s))[0], 0.7, 0.2) for s in (sa, sb)]
            lab = (d["session"] == sb).astype(np.int8)
            out.append((f"S4-{sa}-{sb}", *[np.concatenate([parts[0][k], parts[1][k]]) for k in range(3)], lab))
    return out


def pair_snr(d, idx, lab, rng):
    """C2: каждому кадру роутера в части idx ставится SNR случайного кадра ESP32 той же части."""
    esp = idx[lab[idx] == 1]
    snr = np.full(len(d["label"]), np.nan)
    if len(esp) == 0:
        return snr
    s_esp = (d["p_win"][esp] - d["noise"][esp]) / d["noise"][esp]
    rt = idx[lab[idx] == 0]
    snr[rt] = s_esp[rng.integers(0, len(esp), len(rt))]
    return snr


def prepare(d, idx, lab, snr_match, cfo_removed, rng):
    iq = d["iq"][idx]
    if snr_match:
        ps = pair_snr(d, idx, lab, rng)[idx]
        iq = models.match_snr(iq, d["p_win"][idx], d["noise"][idx], lab[idx] == 0, ps, rng)
    if cfo_removed:
        iq = models.remove_cfo(iq, d["cfo"][idx])
    return models.energy_normalize(iq)


def bootstrap(scores, lab, order, seed):
    res = []
    for c in BOOT_C:
        for nb in BOOT_N:
            a, lo, hi, ell, reps = metrics.block_bootstrap_auc(scores, lab, order, c, nb, seed)
            res.append(dict(C=c, B=nb, ell=ell, lo=lo, hi=hi, n_valid=int(len(reps))))
    return a, res


def main():
    h = hashlib.sha256((OUT / "dataset.npz").read_bytes()).hexdigest()
    d = load()
    done = set()
    rpath = OUT / "results.jsonl"
    if rpath.exists():
        done = {json.loads(l)["key"] for l in rpath.read_text().splitlines() if l.strip()}
    log(f"=== запуск {time.strftime('%Y-%m-%d %H:%M:%S')}, dataset.npz SHA-256 {h}, устройство {DEVICE}")
    for name, tr, va, te, lab in splits(d):
        variants = [("M3", False, False)]
        snr_opts = (False,) if name.startswith("S4") else (False, True)  # в S4 оба класса — роутер
        for snr in snr_opts:
            for cfo_rm in (False, True):
                variants.append(("M1", snr, cfo_rm))
                if not name.startswith("S4"):
                    variants.append(("M2", snr, cfo_rm))
        if name.startswith("S4"):
```

```python
            variants = [v for v in variants if v[0] == "M1"]  # только M1
```
↳ [[Предрегистрация|↗ предрегистрация §4 S4]] ^c-30633a

```python
        for mdl, snr, cfo_rm in variants:
            key = f"{name}|{mdl}|snr={int(snr)}|cfo_removed={int(cfo_rm)}"
            if key in done:
                continue
            t0 = time.time()
            rng = np.random.default_rng(SEED)
            log(f"--- {key}: обучение {len(tr)}, валидация {len(va)}, тест {len(te)} "
                f"(ESP32/класс 1 в тесте: {int(lab[te].sum())})")
            if mdl == "M3":
                sc = models.score_m3(d["cfo"][te], d["cfo"][tr][lab[tr] == 0])
            else:
                xtr = prepare(d, tr, lab, snr, cfo_rm, rng)
                xva = prepare(d, va, lab, snr, cfo_rm, rng)
                xte = prepare(d, te, lab, snr, cfo_rm, rng)
                if mdl == "M1":
                    m = models.train_m1(models.to_tensor_m1(xtr), lab[tr], models.to_tensor_m1(xva), lab[va],
                                        DEVICE, SEED, log)
                    sc = models.score_m1(m, models.to_tensor_m1(xte), DEVICE)
                else:
                    rtr, rva = lab[tr] == 0, lab[va] == 0
                    mm = models.minmax_fit(np.stack([xtr[rtr].real, xtr[rtr].imag], axis=2))
                    m = models.train_m2(models.to_tensor_m2(xtr[rtr], mm), models.to_tensor_m2(xva[rva], mm),
                                        DEVICE, SEED, log)
                    sc = models.score_m2(m, models.to_tensor_m2(xte, mm), DEVICE)
            a, boots = bootstrap(sc, lab[te], d["order"][te], SEED)
            row = dict(key=key, split=name, model=mdl, snr_match=snr, cfo_removed=cfo_rm, auc=a,
                       n_test=int(len(te)), n_pos=int(lab[te].sum()), boot=boots,
                       seconds=round(time.time() - t0, 1), dataset_sha256=h)
            with open(rpath, "a") as f:
                f.write(json.dumps(row, ensure_ascii=False) + "\n")
            log(f"    AUC = {a:.4f}; интервалы: " +
                "; ".join(f"C={b['C']},B={b['B']}: [{b['lo']:.4f}; {b['hi']:.4f}]" for b in boots))


if __name__ == "__main__":
    sys.exit(main())

```
