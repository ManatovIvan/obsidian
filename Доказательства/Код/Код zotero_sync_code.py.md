# Код zotero_sync_code.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/WN45CZGJ).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Синхронизация исходного кода verification/*.py с коллекцией 7 Zotero («Код и выводы (оригиналы)»).

В Zotero хранится только оригинал: полный текст файла (поле «Аннотация»), размер и SHA-256, история
версий (SHA-256 прежних версий). Никаких ссылок и заметок — все связи живут в хранилище Obsidian.
Запись создаётся, если файла в Zotero ещё нет; если SHA-256 не изменился, запись не трогается.

Запуск: python zotero_sync_code.py
"""
import hashlib
import json
import re
import time
import urllib.error
import urllib.request
from pathlib import Path

import check_citations as cc

HERE = Path(__file__).resolve().parent
COLL = "BMSHC34Q"
MAP = HERE / "out" / "zotero_code_items.json"
AUTH = [{"creatorType": "author", "name": "Claude (Claude Code, модель claude-opus-5-5)"},
        {"creatorType": "contributor", "firstName": "Иван", "lastName": "Манатов"}]


def write(method, path, body, version=None):
    hdr = {"Zotero-API-Key": cc.KEY, "Content-Type": "application/json"}
    if version is not None:
        hdr["If-Unmodified-Since-Version"] = str(version)
    for attempt in range(5):
        try:
            r = urllib.request.urlopen(urllib.request.Request(
                f"https://api.zotero.org/users/{cc.USER}/{path}", data=json.dumps(body, ensure_ascii=False).encode(),
                method=method, headers=hdr), timeout=600)
            out = r.read()
            return json.loads(out) if out else None
        except urllib.error.URLError as e:
            if isinstance(e, urllib.error.HTTPError) or attempt == 4:
                raise
            time.sleep(5 * (attempt + 1))


def main():
    known = json.loads(MAP.read_text()) if MAP.exists() else {}
    files = sorted(p.name for p in HERE.glob("*.py"))
    for fn in list(known):
        if fn not in files:                       # файл удалён — запись в Zotero остаётся как есть (её убирают вручную)
            known.pop(fn)
    for fn in files:
        p = HERE / fn
        sha = hashlib.sha256(p.read_bytes()).hexdigest()
        head = f"Файл: hackrf_poc/verification/{fn}, {p.stat().st_size} байт, SHA-256 = {sha}."
        if fn in known:
            d = cc.api(f"items/{known[fn]}?format=json")
            extra = d["data"].get("extra") or ""
            if sha in extra.split("\n")[0]:
                print(fn, "без изменений")
                continue
            hist = [l.strip() for l in extra.split("\n") if "SHA-256 =" in l]
            stamp = time.strftime("%Y-%m-%d %H:%M")
            new_extra = head + "\nИстория версий:\n" + "\n".join([f"{stamp}: {hist[0]}"] + hist[1:] if hist else [])
            write("PATCH", f"items/{known[fn]}", {"abstractNote": p.read_text(), "extra": new_extra}, d["version"])
            print(fn, known[fn], "обновлён")
        else:
            it = [{"itemType": "document", "title": f"Код verification/{fn}", "creators": AUTH,
                   "abstractNote": p.read_text(), "type": "Python-модуль", "date": time.strftime("%Y-%m-%d"),
                   "language": "ru", "extra": head, "collections": [COLL]}]
            known[fn] = write("POST", "items", it)["successful"]["0"]["key"]
            print(fn, known[fn], "создан")
        MAP.write_text(json.dumps(known, ensure_ascii=False, indent=1))
    MAP.write_text(json.dumps(known, ensure_ascii=False, indent=1))


if __name__ == "__main__":
    main()

```
