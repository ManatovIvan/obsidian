# Код zotero_sync_outputs.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/BKX84VUF).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Выводы запусков -> коллекция 7 Zotero («Код и выводы (оригиналы)») — только оригиналы, без ссылок.

Каждый файл из OUTPUTS — отдельная запись: полный текст файла (поле «Аннотация»), в «Экстра» — путь, размер,
SHA-256 и команда, которой файл получен. Если SHA-256 не изменился, запись не трогается; если изменился —
текст заменяется, прежний SHA-256 уходит в «Историю версий».
Ключи записей: out/zotero_output_items.json (их читает obsidian_export.py).

Запуск: python zotero_sync_outputs.py
"""
import hashlib
import json
import time
import urllib.parse
import urllib.request
from pathlib import Path

import check_citations as cc
from zotero_sync_code import AUTH, COLL, write

HERE = Path(__file__).resolve().parent
MAP = HERE / "out" / "zotero_output_items.json"
OUTPUTS = {
    "out/dataset_report.txt": ("Вывод: состав набора кадров dataset.npz (dataset_build.py)",
                               "python dataset_build.py"),
    "out/experiments_log.txt": ("Вывод: журнал обучения всех 65 вариантов (experiments.py)",
                                "python experiments.py"),
    "out/results.jsonl": ("Вывод: результаты 65 вариантов — AUC и интервалы бутстрэпа (experiments.py)",
                          "python experiments.py"),
    "out/report.txt": ("Вывод: сводка AUC и решения по H1–H4 по §5.4 предрегистрации (report.py)",
                       "python report.py"),
    "out/diag_m1_collapse.txt": ("Вывод: диагностика M1 — эпоха лучших весов, коллапс выхода в S4 (diag_m1_collapse.py)",
                                 "python diag_m1_collapse.py"),
    "out/experiments_v4_log.txt": ("Вывод: журнал запусков v4 — проверка признаков R-A, обучение R-B и R-C (experiments_v4.py)",
                                   "python experiments_v4.py --check; python experiments_v4.py"),
    "out/results_v4.jsonl": ("Вывод: результаты v4 — 168 строк, AUC и интервалы бутстрэпа (experiments_v4.py)",
                             "python experiments_v4.py"),
    "out/report_v4.txt": ("Вывод: сводка AUC и решения по §6.4 предрегистрации v4 (report_v4.py)",
                          "python report_v4.py"),
    "out/roc_v4_log.txt": ("Вывод: повтор T2 s1c1 (первое зерно) для ROC-кривых рисунка статьи — AUC повтора и v4 (roc_v4.py)",
                           "python roc_v4.py"),
}


def _form(path, fields):
    """POST /items/<key>/file с телом application/x-www-form-urlencoded (протокол загрузки файлов Zotero Web API)."""
    req = urllib.request.Request(f"https://api.zotero.org/users/{cc.USER}/{path}",
                                 data=urllib.parse.urlencode(fields).encode(), method="POST",
                                 headers={"Zotero-API-Key": cc.KEY, "If-None-Match": "*",
                                          "Content-Type": "application/x-www-form-urlencoded"})
    out = urllib.request.urlopen(req, timeout=600).read()
    return json.loads(out) if out else None


def attach_file(parent, p, content_type="text/plain"):
    """Файл p как вложение (imported_file) к записи parent: создание вложения, авторизация, загрузка, регистрация."""
    data = p.read_bytes()
    att = [{"itemType": "attachment", "parentItem": parent, "linkMode": "imported_file", "title": p.name,
            "contentType": content_type, "filename": p.name}]
    if content_type == "text/plain":
        att[0]["charset"] = "utf-8"
    key = write("POST", "items", att)["successful"]["0"]["key"]
    auth = _form(f"items/{key}/file", {"md5": hashlib.md5(data).hexdigest(), "filename": p.name,
                                      "filesize": len(data), "mtime": int(p.stat().st_mtime * 1000)})
    if not auth.get("exists"):
        body = auth["prefix"].encode() + data + auth["suffix"].encode()
        urllib.request.urlopen(urllib.request.Request(auth["url"], data=body, method="POST",
                                                      headers={"Content-Type": auth["contentType"]}), timeout=600)
        _form(f"items/{key}/file", {"upload": auth["uploadKey"]})
    print("   вложение", key, p.name)
    return key


def main():
    known = json.loads(MAP.read_text()) if MAP.exists() else {}
    for rel, (title, cmd) in OUTPUTS.items():
        p = HERE / rel
        sha = hashlib.sha256(p.read_bytes()).hexdigest()
        head = (f"Файл: hackrf_poc/verification/{rel}, {p.stat().st_size} байт, SHA-256 = {sha}.\n"
                f"Получен командой: {cmd} (каталог hackrf_poc/verification, сервер).")
        if rel in known:
            d = cc.api(f"items/{known[rel]}?format=json")
            extra = d["data"].get("extra") or ""
            if sha in extra.split("\n")[0]:
                print(rel, "без изменений")
                continue
            hist = [l.strip() for l in extra.split("\n") if "SHA-256 =" in l]
            stamp = time.strftime("%Y-%m-%d %H:%M")
            new_extra = head + "\nИстория версий:\n" + "\n".join([f"{stamp}: {hist[0]}"] + hist[1:])
            write("PATCH", f"items/{known[rel]}", {"abstractNote": p.read_text(), "extra": new_extra}, d["version"])
            print(rel, known[rel], "обновлён")
        else:
            it = {"itemType": "document", "title": title, "creators": AUTH, "abstractNote": p.read_text(),
                  "type": "вывод запуска", "date": time.strftime("%Y-%m-%d"), "language": "ru",
                  "extra": head, "collections": [COLL]}
            r = write("POST", "items", [it])
            if "0" not in r["successful"] and r["failed"]["0"]["code"] == 413:   # текст длиннее поля «Аннотация»
                it["abstractNote"] = f"Полный текст — во вложенном файле {p.name} (не помещается в поле «Аннотация»)."
                r = write("POST", "items", [it])
                known[rel] = r["successful"]["0"]["key"]
                attach_file(known[rel], p)
            else:
                known[rel] = r["successful"]["0"]["key"]
            print(rel, known[rel], "создан")
        MAP.write_text(json.dumps(known, ensure_ascii=False, indent=1))


if __name__ == "__main__":
    main()

```
