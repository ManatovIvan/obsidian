# Код test_metrics.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/K3N2RSUH).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Тесты metrics.py: AUC сверяется с определением Fawcett (перебор всех пар), бутстрэп — со свойствами."""
import itertools

import numpy as np

import metrics

RNG = np.random.default_rng(20260504)


def auc_by_pairs(s, y):
    """Прямое определение: доля пар (положительный, отрицательный), где положительный выше; равенство — 1/2."""
    pos = [a for a, t in zip(s, y) if t]
    neg = [a for a, t in zip(s, y) if not t]
    tot = sum(1.0 if p > n else 0.5 if p == n else 0.0 for p, n in itertools.product(pos, neg))
    return tot / (len(pos) * len(neg))


def test_auc_matches_pairwise_definition_with_ties():
    for _ in range(50):
        n = RNG.integers(5, 60)
        s = RNG.integers(0, 6, n).astype(float)   # много равных значений
        y = RNG.integers(0, 2, n)
        if y.min() == y.max():
            continue
        assert abs(metrics.auc(s, y) - auc_by_pairs(s, y)) < 1e-12


def test_auc_extremes():
    assert metrics.auc([0, 1, 2, 3], [0, 0, 1, 1]) == 1.0
    assert metrics.auc([3, 2, 1, 0], [0, 0, 1, 1]) == 0.0
    assert metrics.auc([1, 1, 1, 1], [0, 1, 0, 1]) == 0.5


def test_block_length_grows_slower_than_n():
    assert metrics.block_length(1000, 1) == 10
    assert metrics.block_length(8000, 0.5) == 10
    assert metrics.block_length(10**6, 2) == 200


def test_bootstrap_interval_contains_point_and_is_reproducible():
    s = RNG.normal(size=500) + np.r_[np.zeros(250), np.ones(250)] * 0.5
    y = np.r_[np.zeros(250), np.ones(250)]
    t = RNG.permutation(500)
    a1, lo1, hi1, ell, _ = metrics.block_bootstrap_auc(s, y, t, 1, 300, seed=1)
    a2, lo2, hi2, _, _ = metrics.block_bootstrap_auc(s, y, t, 1, 300, seed=1)
    assert lo1 <= a1 <= hi1 and (lo1, hi1) == (lo2, hi2) and ell == 8

```
