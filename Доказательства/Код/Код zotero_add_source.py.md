# Код zotero_add_source.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/84JT8DXC).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Новый источник литературы -> коллекция 1 Zotero («Литература»): запись с метаданными и PDF-вложение.

Метаданные — JSON-файл с полями записи Zotero (itemType, title, creators, date, ...); коллекция
добавляется здесь. PDF загружается как вложение (imported_file) тем же протоколом, что выводы запусков.

Запуск: python zotero_add_source.py <метаданные.json> <файл.pdf>
"""
import json
import sys
from pathlib import Path

from zotero_sync_code import write
from zotero_sync_outputs import attach_file

LIT = "4BSVRSTV"


def main(meta_path, pdf_path):
    it = json.loads(Path(meta_path).read_text())
    it["collections"] = [LIT]
    r = write("POST", "items", [it])
    if "0" not in r["successful"]:
        sys.exit(f"ОШИБКА {r['failed']}")
    key = r["successful"]["0"]["key"]
    attach_file(key, Path(pdf_path), "application/pdf")
    print("запись", key, it["title"])


if __name__ == "__main__":
    main(sys.argv[1], sys.argv[2])

```
