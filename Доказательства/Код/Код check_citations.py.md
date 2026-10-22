# Код check_citations.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/IUZ6UWM7).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Проверка цитат и связей в хранилище Obsidian по оригиналам в Zotero.

1. Каждая цитата в Источники/*.md («…» — с. N) ищется дословно на странице N PDF-вложения записи Zotero.
   PDF скачивается из Zotero, MD5 сверяется с Zotero. Сравнение после нормализации, не меняющей смысла
   (Unicode NFKC, тире -> «-», типографские апострофы -> «'», пробелы схлопнуты, перенос слова склеен).
   Текст страницы: pdftotext -layout, затем порядок чтения; для сканов без текстового слоя — OCR (tesseract),
   такие находки помечены «OCR, сверить по скану».
2. Каждая связь в _связи/код.json, ссылающаяся на цитату PDF, должна указывать на цитату, которая есть в Источники/;
   якорь каждой связи — на существующую строку кода.
3. Все внутренние ссылки хранилища [[заметка#^метка]] (вне блоков кода) должны вести на существующие блоки.

Запуск: python check_citations.py   (выход 1, если есть хотя бы один FAIL)
"""
import hashlib
import json
import re
import subprocess
import sys
import unicodedata
import urllib.error
import urllib.request
from pathlib import Path

HERE = Path(__file__).resolve().parent
CACHE = HERE / "sources_cache"
CITE = re.compile(r"⟦(?P<key>[A-Z0-9]{8})(?: с\.(?P<page>\d+))?: «(?P<quote>[^»]+)»⟧")


def creds():
    env = {}
    for line in (Path.home() / ".config/zotero/credentials").read_text().splitlines():
        if "=" in line:
            k, v = line.split("=", 1)
            env[k.strip()] = v.strip().strip('"')
    return env["ZOTERO_API_KEY"], env["ZOTERO_USER_ID"]


KEY, USER = creds()


def api(path, raw=False):
    """Запрос к Zotero; при сетевой ошибке — до 5 повторов с паузой (ВЫБОР: в этой сети бывают
    кратковременные сбои DNS; без повтора они давали ложный FAIL, не связанный с цитатой)."""
    import time
    req = urllib.request.Request(f"https://api.zotero.org/users/{USER}/{path}", headers={"Zotero-API-Key": KEY})
    for attempt in range(5):
        try:
            data = urllib.request.urlopen(req, timeout=600).read()
            return data if raw else json.loads(data)
        except urllib.error.URLError:
            if attempt == 4:
                raise
            time.sleep(5 * (attempt + 1))


def norm(s):
    s = unicodedata.normalize("NFKC", s).replace("­", "")
    s = re.sub(r"[‐-―−]", "-", s)
    s = re.sub(r"[‘’‛′]", "'", s)   # типографские апострофы -> прямой
    s = re.sub(r"(\w)-\s*\n\s*(\w)", r"\1\2", s)   # перенос слова через дефис в конце строки
    return re.sub(r"\s+", " ", s).strip()


_pdf = {}


def pdf_for(key):
    """Скачивает PDF-вложение записи и проверяет MD5 по данным Zotero."""
    if key in _pdf:
        return _pdf[key]
    att = [c["data"] for c in api(f"items/{key}/children?format=json")
           if c["data"].get("contentType") == "application/pdf"]
    if not att:
        _pdf[key] = None
        return None
    a = att[0]
    CACHE.mkdir(exist_ok=True)
    p = CACHE / f"{key}_{a['key']}.pdf"
    if not p.exists() or hashlib.md5(p.read_bytes()).hexdigest() != a["md5"]:
        p.write_bytes(api(f"items/{a['key']}/file", raw=True))
    if hashlib.md5(p.read_bytes()).hexdigest() != a["md5"]:
        raise RuntimeError(f"{key}: MD5 скачанного PDF не совпал с Zotero")
    _pdf[key] = (p, a["key"], a["md5"])
    return _pdf[key]


def page_text(pdf, page, layout=True):
    """layout=True сохраняет расположение (удобно для таблиц); layout=False — порядок чтения
    (нужен для двухколоночных статей, где в режиме layout строки двух колонок сливаются)."""
    mode = ["-layout"] if layout else []
    return subprocess.run(["pdftotext", "-f", str(page), "-l", str(page), *mode, str(pdf), "-"],
                          capture_output=True, text=True, check=True).stdout


def ocr_text(pdf, page):
    """Для сканов без текстового слоя: страница рендерится в 300 dpi (pdftoppm) и
    распознаётся tesseract (английский). Распознанный текст — производный, поэтому в отчёте
    помечается «OCR», и такую цитату стоит выборочно сверить глазами по скану."""
    import tempfile
    with tempfile.TemporaryDirectory() as d:
        subprocess.run(["pdftoppm", "-f", str(page), "-l", str(page), "-r", "300", "-png", str(pdf), f"{d}/p"],
                       check=True, capture_output=True)
        png = sorted(Path(d).glob("p*.png"))[0]
        return subprocess.run(["tesseract", str(png), "-", "-l", "eng"], capture_output=True, text=True, check=True).stdout


def find_line(text, quote):
    """Номер строки (с 1), с которой начинается фрагмент, или None."""
    lines = text.split("\n")
    q = norm(quote)
    for i in range(len(lines)):                   # фрагмент может занимать до 12 строк
        if q in norm("\n".join(lines[i:i + 12])) and q not in norm("\n".join(lines[i + 1:i + 12])):
            return i + 1                          # строка i — первая, без которой фрагмент уже не найти
    return None


VAULT = HERE.parent / "obsidian"
SRC_LINE = re.compile(r"^«(?P<quote>.+)» — с\. (?P<page>\d+) \[↗ PDF\]")
SRC_KEY = re.compile(r"Источник Zotero: \[(?P<key>[A-Z0-9]{8})\]")
WIKI = re.compile(r"\[\[([^#\]|]+)(?:#\^([A-Za-z0-9-]+))?")


def vault_notes():
    return {p: p.read_text() for p in VAULT.rglob("*.md") if ".git" not in p.parts}


def without_code(text):
    text = re.sub(r"```.*?```", "", text, flags=re.S)       # блоки кода — в них ссылок Obsidian не бывает
    return re.sub(r"`[^`\n]*`", "", text)                    # и моноширинный текст тоже


def main():
    n_ok = n_fail = 0

    def ok(msg):
        nonlocal n_ok
        n_ok += 1
        print("OK  ", msg)

    def fail(msg):
        nonlocal n_fail
        n_fail += 1
        print("FAIL", msg)
    notes = vault_notes()
    quotes = set()
    # 1. цитаты источников по PDF
    for p, text in sorted(notes.items()):
        if p.parent.name != "Источники":
            continue
        m = SRC_KEY.search(text)
        if not m:
            continue
        key = m["key"]
        for line in text.splitlines():
            q = SRC_LINE.match(line)
            if not q:
                continue
            quote, page = q["quote"], int(q["page"])
            quotes.add((key, str(page), quote))
            where = f"{p.name} {key} с.{page}"
            try:
                pdf, att, md5 = pdf_for(key)
                mode, hit = "layout", find_line(page_text(pdf, page), quote)
                if hit is None:
                    mode, hit = "порядок чтения", find_line(page_text(pdf, page, layout=False), quote)
                if hit is None and len(norm(page_text(pdf, page))) < 500:
                    mode, hit = "OCR, сверить по скану", find_line(ocr_text(pdf, page), quote)
                if hit is None:
                    fail(f"{where}: фрагмент не найден — «{quote}»")
                else:
                    ok(f"{where}: PDF {att} md5 {md5[:8]}, строка {hit} ({mode})")
            except Exception as e:  # noqa: BLE001 — любая причина провала печатается
                fail(f"{where}: {e} — «{quote}»")
    # 2. связи кода
    import obsidian_export as ox
    links = json.loads((VAULT / "_связи" / "код.json").read_text())
    for e in links:
        if e["anchor"] != "<модуль>":
            src = (HERE / e["file"]).read_text().split("\n")
            if not any(ox.code_part(l) == e["anchor"] for l in src):
                fail(f"связь {e['file']}: якорь не найден в коде — {e['anchor']}")
                continue
        for r in e["refs"]:
            if r["kind"] == "cite" and r["page"] and (r["key"], r["page"], r["quote"]) not in quotes:
                fail(f"связь {e['file']}: цитаты нет в Источники/ — {r['key']} с.{r['page']} «{r['quote']}»")
    ok(f"связи кода: {len(links)} проверены")
    # 3. внутренние ссылки хранилища
    names = {p.stem: p for p in notes}
    blocks = {p.stem: set(re.findall(r"\^([A-Za-z0-9-]+)\s*$", t, re.M)) for p, t in notes.items()}
    total = 0
    for p, text in notes.items():
        if p.name == "README.md":
            continue
        for name, bid in WIKI.findall(without_code(text)):
            total += 1
            if not bid and "." in name and any(VAULT.rglob(name)):   # вложение (рисунок): файл есть в хранилище
                continue
            if name not in names or (bid and bid not in blocks[name]):
                fail(f"{p.relative_to(VAULT)}: битая ссылка [[{name}#^{bid}]]")
    ok(f"внутренние ссылки: {total} проверены")
    print(f"\nИтого: {n_ok} OK, {n_fail} FAIL")
    sys.exit(1 if n_fail else 0)


if __name__ == "__main__":
    main()

```
