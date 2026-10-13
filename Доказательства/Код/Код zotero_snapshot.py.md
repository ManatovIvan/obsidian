# Код zotero_snapshot.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/5JAWDA8D).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Снимок библиотеки Zotero: все записи и коллекции -> out/zotero_backup_<дата>_<время>.json.

Формат как у прежних снимков: {"date": ..., "items": [data записи, ...], "collections": [data коллекции, ...]}.
Снимок читает obsidian_export.py (берёт самый поздний по имени файла).

Запуск: python zotero_snapshot.py
"""
import json
import time
from pathlib import Path

import check_citations as cc

OUT = Path(__file__).resolve().parent / "out"


def fetch_all(path):
    rows, start = [], 0
    while True:
        page = cc.api(f"{path}?format=json&limit=100&start={start}")
        rows += [r["data"] for r in page]
        if len(page) < 100:
            return rows
        start += 100


def main():
    snap = {"date": time.strftime("%Y-%m-%d %H:%M:%S"), "items": fetch_all("items"),
            "collections": fetch_all("collections")}
    f = OUT / f"zotero_backup_{time.strftime('%Y-%m-%d_%H%M')}.json"
    f.write_text(json.dumps(snap, ensure_ascii=False, indent=1))
    print(f.name, "записей:", len(snap["items"]), "коллекций:", len(snap["collections"]))


if __name__ == "__main__":
    main()

```
