# Код zotero_add_highlights.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/BAR2NWVT).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Выделения в PDF для цитат, у которых их ещё нет (Zotero: дочерние аннотации-highlight к PDF-вложению).

Берёт цитаты ⟦КЛЮЧ с.N: «…»⟧ из файлов, переданных аргументами (например, PREREG_v3_approved.md), и
для каждой, которой нет в out/citation_annotations.json, находит слова цитаты на странице N PDF
(pdftotext -bbox-layout: координаты слов) и создаёт выделение. Прямоугольники — по строкам, в координатах
PDF (начало внизу слева), как у выделений, созданных в Zotero. Результат дописывается в
out/citation_annotations.json. Цитата, не найденная на странице, не выделяется — печатается НЕ НАЙДЕНО.

Запуск: python zotero_add_highlights.py [--dry] PREREG_v3_approved.md  (--dry — только показать найденное)
"""
import html
import json
import re
import subprocess
import sys
from pathlib import Path

import check_citations as cc
from zotero_sync_code import write

HERE = Path(__file__).resolve().parent
AMAP_PATH = HERE / "out" / "citation_annotations.json"
WORD = re.compile(r'<word xMin="([\d.]+)" yMin="([\d.]+)" xMax="([\d.]+)" yMax="([\d.]+)">(.*?)</word>')


def page_words(pdf, page):
    xml = subprocess.run(["pdftotext", "-f", str(page), "-l", str(page), "-bbox-layout", str(pdf), "-"],
                         capture_output=True, text=True, check=True).stdout
    height = float(re.search(r'<page width="[\d.]+" height="([\d.]+)"', xml).group(1))
    words = [(float(a), float(b), float(c), float(d), html.unescape(t)) for a, b, c, d, t in WORD.findall(xml)]
    return height, words


def locate(words, quote):
    """Индексы слов страницы, покрывающих цитату (сравнение без пробелов, после cc.norm)."""
    stream, owner = "", []
    for i, w in enumerate(words):
        t = cc.norm(w[4]).replace(" ", "")
        if t.endswith("-") and i + 1 < len(words) and words[i + 1][1] > w[1] + 1:
            t = t[:-1]                            # перенос слова в конце строки («Or-» / «dered»)
        stream += t
        owner += [i] * len(t)
    q = cc.norm(quote).replace(" ", "")
    pos = stream.find(q)
    if pos < 0:
        return None
    return sorted(set(owner[pos:pos + len(q)])), pos


def rects(words, idx, height):
    """Один прямоугольник на строку (слова с близким yMin), в координатах PDF."""
    lines = []
    for i in idx:
        x0, y0, x1, y1, _ = words[i]
        for ln in lines:
            if abs(ln[1] - y0) < 2:
                ln[0], ln[2], ln[3] = min(ln[0], x0), max(ln[2], x1), max(ln[3], y1)
                break
        else:
            lines.append([x0, y0, x1, y1])
    return [[round(x0, 3), round(height - y1, 3), round(x1, 3), round(height - y0, 3)] for x0, y0, x1, y1 in lines]


def main(paths):
    amap = json.loads(AMAP_PATH.read_text())
    todo = []
    for p in paths:
        for m in cc.CITE.finditer(Path(p).read_text()):
            k = f"{m['key']}|{m['page']}|{m['quote']}"
            if m["page"] and k not in amap and k not in todo:
                todo.append(k)
    for k in todo:
        key, page, quote = k.split("|", 2)
        got = cc.pdf_for(key)
        if not got:
            print("НЕТ PDF", k)
            continue
        pdf, att, _ = got
        height, words = page_words(pdf, int(page))
        hit = locate(words, quote)
        if not hit:
            print("НЕ НАЙДЕНО", k)
            continue
        idx, off = hit
        rr = rects(words, idx, height)
        if DRY:
            print("найдено", k[:90], "| слова:", " ".join(words[i][4] for i in idx)[:120], "| строк", len(rr))
            continue
        top = min(words[i][1] for i in idx)
        ann = [{"itemType": "annotation", "parentItem": att, "annotationType": "highlight",
                "annotationText": quote, "annotationComment": "", "annotationColor": "#ffd400",
                "annotationPageLabel": page, "annotationSortIndex": f"{int(page) - 1:05d}|{off:06d}|{int(top):05d}",
                "annotationPosition": json.dumps({"pageIndex": int(page) - 1, "rects": rr})}]
        r = write("POST", "items", ann)
        if "0" not in r["successful"]:
            print("ОШИБКА", k, r["failed"])
            continue
        amap[k] = {"annotation": r["successful"]["0"]["key"], "attachment": att, "comment": ""}
        AMAP_PATH.write_text(json.dumps(amap, ensure_ascii=False, indent=1))
        print("выделено", r["successful"]["0"]["key"], k[:90], f"строк {len(rr)}")


DRY = "--dry" in sys.argv

if __name__ == "__main__":
    main([a for a in sys.argv[1:] if a != "--dry"])

```
