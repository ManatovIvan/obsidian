# Код report.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/7M4HAIVH).

Сгенерировано obsidian_export.py — не править вручную.

Модуль в целом: [[Предрегистрация|↗ предрегистрация §5.4]] · [[Предрегистрация|↗ Предрегистрация]]

```python
"""Итоговая таблица и решения по гипотезам строго по правилам предрегистрации (§5.4).

Вход: out/results.jsonl. Выход: out/report.txt. Ничего не пересчитывает и не подбирает:
только сводит посчитанные числа и применяет заранее записанные правила.

Где формулировка §5.4 не называет вариант (C2, C4), решение выводится для каждого варианта
отдельно и это прямо указано (запись об этом — в журнале отступлений).
"""
import json
from pathlib import Path

OUT = Path(__file__).resolve().parent / "out"


def rows():
    return [json.loads(l) for l in (OUT / "results.jsonl").read_text().splitlines() if l.strip()]


def get(R, split, model, snr, cfo):
    for r in R:
        if (r["split"], r["model"], r["snr_match"], r["cfo_removed"]) == (split, model, snr, cfo):
            return r
    return None


def lows(r):
    return [b["lo"] for b in r["boot"]]


def highs(r):
    return [b["hi"] for b in r["boot"]]


def fmt(r):
    if r is None:
        return "нет данных"
    return f"{r['auc']:.3f} [{min(lows(r)):.3f}; {max(highs(r)):.3f}]"


def main():
    R = rows()
    L = ["Сводка: AUC [самая низкая нижняя граница; самая высокая верхняя граница 95 % по всем 6 комбинациям C и B]",
         "", "split | модель | SNR выровнен | CFO снят | n тест (класс 1) | AUC [интервал]"]
    for r in R:
        L.append(f"{r['split']} | {r['model']} | {int(r['snr_match'])} | {int(r['cfo_removed'])} | "
                 f"{r['n_test']} ({r['n_pos']}) | {fmt(r)}")
    L.append("\nРешения (ПРЕДРЕГ §5.4; «устойчиво» = при всех 6 комбинациях C × B, §5.3):")
    # H1
    h1 = [get(R, s, "M1", True, True) for s in ("S2-E1", "S2-E3")]
    if all(h1):
        ok = all(min(lows(r)) > 0.5 for r in h1)
        L.append(f"H1: S2-E1 {fmt(h1[0])}, S2-E3 {fmt(h1[1])} (M1, SNR выровнен, CFO снят) -> "
                 + ("ПОДТВЕРЖДЕНА" if ok else "НЕ подтверждена") + "; аннотация: 0,93 (случайное разбиение)")
    # H2 — по каждому варианту
    for snr in (False, True):
        for cfo in (False, True):
            s1 = get(R, "S1", "M1", snr, cfo)
            s3 = [get(R, s, "M1", snr, cfo) for s in ("S3-A", "S3-B")]
            if s1 and all(s3):
                ok = all(r["auc"] < s1["auc"] and max(highs(r)) < min(lows(s1)) for r in s3)
                L.append(f"H2 [SNR={int(snr)}, CFO снят={int(cfo)}]: S1 {fmt(s1)}; S3-A {fmt(s3[0])}; S3-B {fmt(s3[1])} -> "
                         + ("ПОДТВЕРЖДЕНА" if ok else "НЕ подтверждена") + "; аннотация: 0,65")
    # H3 — M2 по вариантам, M3 один раз
    for snr in (False, True):
        for cfo in (False, True):
            s3 = [get(R, s, "M2", snr, cfo) for s in ("S3-A", "S3-B")]
            if all(s3):
                ok = all(max(lows(r)) <= 0.5 <= min(highs(r)) for r in s3)   # §5.3: интервал содержит 0,5 при каждом из 6 сочетаний
                L.append(f"H3 M2 [SNR={int(snr)}, CFO снят={int(cfo)}]: S3-A {fmt(s3[0])}; S3-B {fmt(s3[1])} -> "
                         + ("ПОДТВЕРЖДЕНА (≈ 0,5)" if ok else "НЕ подтверждена") + "; аннотация: ≈ 0,50")
    s3 = [get(R, s, "M3", False, False) for s in ("S3-A", "S3-B")]
    if all(s3):
        ok = all(max(lows(r)) <= 0.5 <= min(highs(r)) for r in s3)   # §5.3: интервал содержит 0,5 при каждом из 6 сочетаний
        L.append(f"H3 M3 (CFO): S3-A {fmt(s3[0])}; S3-B {fmt(s3[1])} -> "
                 + ("ПОДТВЕРЖДЕНА (≈ 0,5)" if ok else "НЕ подтверждена") + "; аннотация: ≈ 0,50")
    # H4 — подпись сессии; сравнение S3 и S2 при выровненном SNR выводится числами для решения автора
    s4 = [r for r in R if r["split"].startswith("S4")]
    if s4:
        for cfo in (False, True):
            g = [r for r in s4 if r["cfo_removed"] == cfo]
            sep = [r for r in g if min(lows(r)) > 0.5]
            L.append(f"H4, подпись сессии [CFO снят={int(cfo)}]: различимы {len(sep)} из {len(g)} пар; "
                     + "; ".join(f"{r['split'][3:]} {r['auc']:.3f}" for r in g))
        L.append("H4, вторая часть правила (провал S3 относительно S2 не объясняется SNR): см. строки H1/H2 "
                 "с SNR=1 и SNR=0 — сопоставление приводится в отчёте автору, правило §5.4 применяется к этим числам.")
    text = "\n".join(L)
    (OUT / "report.txt").write_text(text + "\n")
    print(text)


if __name__ == "__main__":
    main()

```
