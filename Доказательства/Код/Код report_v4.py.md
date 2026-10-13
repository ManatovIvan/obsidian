# Код report_v4.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/HFMCK2ZH).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Сводка результатов v4 и решения по правилам предрегистрации (PREREG_v4_approved.md, §5, §6.3, §6.4).

Вход: out/results_v4.jsonl. Выход: out/report_v4.txt.

Обоснования строк кода (источники, цитаты, решения) — в хранилище статьи Obsidian, а не в коде.
"""
import json
from pathlib import Path

import numpy as np

OUT = Path(__file__).resolve().parent / "out"
IMPLS = ("R-A", "R-B", "R-C")
TESTS = ("T1", "T2")
VARIANTS = ("s0c0", "s0c1", "s1c0", "s1c1")
RULES = (("Г", "s0c0"), ("Г-а", "s0c1"), ("Г-б", "s1c0"), ("s1c1 (оба контроля)", "s1c1"))
N_SEEDS = 10
N_BOOT = 6                                        # сочетаний C × B


def test_state(rows):
    """Состояние одного теста: «не определено» (§5), если хоть одна строка с постоянной оценкой;
    иначе выполнено ли «нижняя граница > 0,5» при всех зёрнах и всех сочетаниях C × B (§6.3)."""
    if any(r["constant"] for r in rows):
        return "undefined"
    return all(b["lo"] > 0.5 for r in rows for b in r["boot"])


def main():
    rows = [json.loads(l) for l in (OUT / "results_v4.jsonl").read_text().splitlines() if l.strip()]
    out = ["Сводка v4: AUC (положительный класс — ESP32). R-A — одна строка; R-B, R-C — среднее ± с. к. о. по зёрнам,",
           "минимальная нижняя граница интервала по всем зёрнам и сочетаниям C × B.", ""]
    missing = []
    for impl in IMPLS:
        out.append(f"{impl}:")
        for v in VARIANTS:
            for t in TESTS:
                g = [r for r in rows if r["impl"] == impl and r["variant"] == v and r["test"] == t]
                need = 1 if impl == "R-A" else N_SEEDS
                if len(g) < need or any(len(r["boot"]) < N_BOOT for r in g):
                    missing.append(f"{impl} {v} {t}: строк {len(g)} из {need}")
                    continue
                aucs = np.array([r["auc"] for r in g])
                lo = min(b["lo"] for r in g for b in r["boot"])
                hi = max(b["hi"] for r in g for b in r["boot"])
                const = sum(r["constant"] for r in g)
                txt = f"{aucs[0]:.4f}" if impl == "R-A" else f"{aucs.mean():.4f} ± {aucs.std():.4f} (мин {aucs.min():.4f}, макс {aucs.max():.4f})"
                out.append(f"  {v} {t}: AUC {txt}; границы интервалов [{lo:.4f}; {hi:.4f}]"
                           + (f"; постоянная оценка в {const} строках" if const else ""))
    if missing:
        out += ["", "НЕ ХВАТАЕТ СТРОК — решения не выносятся:"] + ["  " + m for m in missing]
    else:
        out += ["", "Решения по §6.4 (для каждой реализации отдельно):"]
        for impl in IMPLS:
            for name, v in RULES:
                st = {t: test_state([r for r in rows if r["impl"] == impl and r["variant"] == v and r["test"] == t])
                      for t in TESTS}
                if "undefined" in st.values():
                    verdict = "не определено (постоянная оценка, §5)"
                elif all(st.values()):
                    verdict = "подтверждена (T1 и T2)"
                elif any(st.values()):
                    verdict = "не подтверждена; условие выполнено только в " + ", ".join(t for t in TESTS if st[t])
                else:
                    verdict = "не подтверждена (ни в T1, ни в T2)"
                out.append(f"  {impl} {name} [{v}]: {verdict}")
    text = "\n".join(out)
    (OUT / "report_v4.txt").write_text(text + "\n")
    print(text)


if __name__ == "__main__":
    main()

```
