# Код diag_m1_collapse.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/JFGGMVB4).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Диагностика M1 после прогона experiments.py (ничего не меняет в результатах).

1. По журналу out/experiments_log.txt для каждой строки M1: число эпох, эпоха лучших весов
   (первая эпоха с наименьшей ошибкой на валидации) и набор различных значений ошибки.
2. Для всех разбиений S4 M1 обучается заново тем же кодом и с тем же зерном, что в experiments.py,
   и на тестовых кадрах считаются: число различных оценок (одна оценка на все кадры — выход
   не зависит от входа) и доля «мёртвых» нейронов после каждого слоя ReLU (нейрон, выход которого
   равен 0 на всех тестовых кадрах).

Выход: out/diag_m1_collapse.txt.
"""
import re
import sys

import numpy as np
import torch

import experiments as E
import models

OUT = E.OUT / "diag_m1_collapse.txt"


def say(f, msg):
    f.write(msg + "\n")
    print(msg, flush=True)


def log_summary(f):
    say(f, "1. Журнал обучения M1 (out/experiments_log.txt)")
    say(f, "строка | эпох | эпоха лучших весов | различные ошибки на валидации")
    key, errs = None, []
    for line in open(E.OUT / "experiments_log.txt"):
        m = re.match(r"--- (\S+):", line)
        if m:
            key, errs = m.group(1), []
            continue
        m = re.search(r"ошибка на валидации ([\d.]+)", line)
        if m:
            errs.append(float(m.group(1)))
        if line.strip().startswith("AUC =") and key and "|M1|" in key:
            say(f, f"{key} | {len(errs)} | {int(np.argmin(errs))} | {sorted(set(errs))}")


def dead_share(model, x):
    """Доля нейронов, выход которых после ReLU равен 0 на всех кадрах x (порядок слоёв — Homegrown.forward)."""
    model.eval()
    with torch.no_grad():
        a1 = model.act(model.c1(x))
        a2 = model.act(model.c2(a1))
        f1 = model.act(model.f1(a2.flatten(1)))
        f2 = model.act(model.f2(f1))
    return {n: float((o.reshape(o.shape[0], -1) == 0).all(0).float().mean())
            for n, o in (("c1", a1), ("c2", a2), ("f1", f1), ("f2", f2))}


def s4_retrain(f, d):
    say(f, "")
    say(f, "2. S4: повторное обучение M1 (тот же код и зерно, что в experiments.py)")
    say(f, "строка | различных оценок на тесте | мин. оценка | макс. оценка | доля мёртвых нейронов c1, c2, f1, f2")
    for name, tr, va, te, lab in E.splits(d):
        if not name.startswith("S4"):
            continue
        for cfo_rm in (False, True):
            rng = np.random.default_rng(E.SEED)
            xtr = E.prepare(d, tr, lab, False, cfo_rm, rng)
            xva = E.prepare(d, va, lab, False, cfo_rm, rng)
            xte = E.prepare(d, te, lab, False, cfo_rm, rng)
            m = models.train_m1(models.to_tensor_m1(xtr), lab[tr], models.to_tensor_m1(xva), lab[va],
                                E.DEVICE, E.SEED, lambda s: None)
            xt = models.to_tensor_m1(xte)
            sc = models.score_m1(m, xt, E.DEVICE)
            dead = dead_share(m, xt.to(E.DEVICE))
            say(f, f"{name}|M1|snr=0|cfo_removed={int(cfo_rm)} | {len(np.unique(sc))} | {sc.min():.6f} | {sc.max():.6f} | "
                   + ", ".join(f"{v:.3f}" for v in dead.values()))


def main():
    d = E.load()
    with open(OUT, "w") as f:
        log_summary(f)
        s4_retrain(f, d)


if __name__ == "__main__":
    sys.exit(main())

```
