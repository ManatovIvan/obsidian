# Код metrics.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/MIB5TS3X).

Сгенерировано obsidian_export.py — не править вручную.

Модуль в целом: [[Предрегистрация|↗ Предрегистрация]]

```python
"""Метрика (ROC AUC) и неопределённость (блочный бутстрэп) по утверждённой предрегистрации, §5.

Обоснования строк кода (источники, цитаты, решения) — в хранилище статьи Obsidian, а не в коде.
"""
import math

import numpy as np
from scipy.stats import rankdata


```

```python
def auc(scores, labels):
```
↳ [[An introduction to ROC analysis#^343efdiu-8-em2j99s8|↗ Fawcett 2006, с. 8]] · [[An introduction to ROC analysis#^343efdiu-8-7cskaxiq|↗ Fawcett 2006, с. 8]] ^c-50a250

```python
    """AUC как вероятность того, что случайный положительный пример оценён выше случайного
    отрицательного. Вычисление через ранги (эквивалент теста Уилкоксона);
    равные оценки получают средний ранг (вклад 1/2), ПРОВЕРКА (tests): совпадает с перебором пар."""
    s = np.asarray(scores, float)
    y = np.asarray(labels).astype(bool)
    n_pos, n_neg = int(y.sum()), int((~y).sum())
    if n_pos == 0 or n_neg == 0:
        return float("nan")                       # ВЫБОР: без обоих классов AUC не определён
    ranks = rankdata(s, method="average")         # средний ранг для равных оценок (ВЫБОР: scipy вместо цикла — скорость)
    return float((ranks[y].sum() - n_pos * (n_pos + 1) / 2) / (n_pos * n_neg))


```

```python
def block_length(n, c):
```
↳ [↗ Nordman 2014, с. 6, формула (2.3) — сверить глазами](zotero://open-pdf/library/items/CDHJ349I?page=6) · [[Convergence rates of empirical block length selectors for block bootstrap#^ddsa936w-6-frdk5ten|↗ Nordman 2014, с. 6]] · [[Предрегистрация|↗ предрегистрация §5.2.]] · [↗ Künsch 1989, с. 1 — сверить глазами](zotero://open-pdf/library/items/MPJNS67T?page=1) ^c-cc2a39

```python
    """ℓ = ⌈C·n^(1/3)⌉: порядок MSE-оптимальной длины блока; C ∈ {0,5; 1; 2}.
    Условия Künsch l(n) → ∞, l(n)/n → 0 выполняются: n^(1/3)/n → 0."""
    return max(1, math.ceil(c * n ** (1 / 3)))


```

```python
def block_bootstrap_auc(scores, labels, order_key, c, n_boot, seed):
```
↳ [[The Jackknife and the Bootstrap for General Stationary Observations#^t5mh3e88-1-8na4xig7|↗ Künsch 1989, с. 1]] · [[The Jackknife and the Bootstrap for General Stationary Observations#^t5mh3e88-1-d9up7pp7|↗ Künsch 1989, с. 1]] · [↗ Künsch 1989, с. 2, (журнальная с.1218) — сверить глазами](zotero://open-pdf/library/items/MPJNS67T?page=2) ^c-4dfe90

```python
    """Блочный бутстрэп AUC. Кадры упорядочиваются по времени (order_key), блоки по ℓ подряд
    идущих кадров выбираются случайно с возвращением из n − ℓ + 1 перекрывающихся блоков.
    Возвращает (AUC на всех данных, P2,5, P97,5, ℓ, массив повторов)."""
    idx = np.argsort(order_key, kind="mergesort")
    s = np.asarray(scores, float)[idx]
    y = np.asarray(labels)[idx]
    n = len(s)
    ell = block_length(n, c)
    k = math.ceil(n / ell)                        # ВЫБОР: ⌈n/ℓ⌉ блоков, затем обрезка до n — та же длина выборки
    rng = np.random.default_rng(seed)
    reps = np.empty(n_boot)
    for b in range(n_boot):
        starts = rng.integers(0, n - ell + 1, k)  # n − ℓ + 1 перекрывающихся блоков
        take = (starts[:, None] + np.arange(ell)[None, :]).reshape(-1)[:n]
        reps[b] = auc(s[take], y[take])
    reps = reps[~np.isnan(reps)]                  # ВЫБОР: повтор без одного из классов пропускается (их число сообщается)
```

```python
    lo, hi = np.percentile(reps, [2.5, 97.5])  # интервал — 2,5 и 97,5 процентили
```
↳ [[Предрегистрация|↗ предрегистрация §5.2]] ^c-aaec8c

```python
    return auc(s, y), float(lo), float(hi), ell, reps

```
