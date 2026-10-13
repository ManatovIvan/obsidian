# Код obsidian_export.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/Q4EU5Z25).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Генератор хранилища Obsidian (репозиторий ManatovIvan/obsidian) — все ссылки и связи живут там.

Входы:
  out/zotero_backup_*.json      — снимок Zotero (zotero_snapshot.py): тексты показаний, выводов, предрегистрации;
  out/zotero_output_items.json  — ключи выводов экспериментов (zotero_sync_outputs.py);
  out/citation_annotations.json — выделения в PDF: цитата -> ключ выделения;
  obsidian/_связи/код.json      — связи «строка кода -> источник» (якорь — код строки без комментария);
  verification/*.py             — чистый код без ссылок;
  out/zotero_code_items.json    — ключи записей кода в Zotero.
Выходы (перезаписываются целиком, руками не править): Источники/, Доказательства/Код, Доказательства/Показания,
Доказательства/Выводы, Доказательства/Расчёты, заметки предрегистраций (PREREGS). Журнал отступлений ведётся вручную.
Статья/ генератор не трогает.

Метки блоков: ^c-<хэш> — строка кода, ^<источник>-<стр>-<выделение> — цитата PDF, ^t-<ключ>-<n> — строка
показания, ^o-<ключ>-<n> — строка вывода или расчёта. Метки стабильны при повторной генерации.
"""
import hashlib
import io
import json
import re
import tokenize
from collections import defaultdict
from pathlib import Path

HERE = Path(__file__).resolve().parent
VAULT = HERE.parent / "obsidian"
CITE = re.compile(r"⟦(?P<key>[A-Z0-9]{8})(?: с\.(?P<page>\d+))?: «(?P<quote>[^»]+)»⟧")
VIS = re.compile(r"ВИЗУАЛЬНО (?P<key>[A-Z0-9]{8}) с\.(?P<page>\d+)")
SHORT = {"JHCW99MH": "ieee"}
LABEL = {"8B73M9ND": "MathWorks", "EQKQ2C4M": "Lichtman, PySDR", "JHCW99MH": "IEEE 802.11-2020"}
TESTIMONY_COLL, CALC_COLL = "564MFDE5", "3EWN8SPF"
OUTPUT_KEYS = ["GZJDPPNF", "WMRVFDS5", "JAE6NFVD", "GJI5ZH7N"]   # выводы, которые остаются артефактами
_OUT_MAP = HERE / "out" / "zotero_output_items.json"              # выводы экспериментов (zotero_sync_outputs.py)
OUTPUT_KEYS += list(json.loads(_OUT_MAP.read_text()).values()) if _OUT_MAP.exists() else []
PREREGS = {"6GGC73ZD": "Предрегистрация", "QDQWWNXQ": "Предрегистрация v4"}   # запись Zotero -> заметка

snap = json.loads(sorted((HERE / "out").glob("zotero_backup_*.json"))[-1].read_text())
ITEMS = {d["key"]: d for d in snap["items"]}
AMAP = json.loads((HERE / "out" / "citation_annotations.json").read_text())
LINKS = json.loads((VAULT / "_связи" / "код.json").read_text())
CODE_ITEMS = json.loads((HERE / "out" / "zotero_code_items.json").read_text())


def safe(t):
    return re.sub(r'[\\/:*?"<>|#^\[\]]', " ", t).strip()


def title(key):
    d = ITEMS[key]
    return safe(d.get("shortTitle") or d.get("title") or key)


def label(key):
    if key in LABEL:
        return LABEL[key]
    d = ITEMS[key]
    cr = [c for c in d.get("creators", []) if c.get("lastName")]
    return f"{cr[0]['lastName']} {(d.get('date') or '')[:4]}".strip() if cr else title(key)[:40]


def zsel(key):
    return f"zotero://select/library/items/{key}"


def h6(*parts):
    return hashlib.sha1("|".join(map(str, parts)).encode()).hexdigest()[:6]


def note_of(key):
    """Заметка хранилища, которая представляет запись Zotero (если такая есть)."""
    d = ITEMS.get(key, {})
    if TESTIMONY_COLL in d.get("collections", []):
        return f"Показание {key}"
    if key in OUTPUT_KEYS:
        return f"Вывод {key}"
    if CALC_COLL in d.get("collections", []):
        return f"Расчёт {key}"
    if key in PREREGS:
        return PREREGS[key]
    return None


SEP = "\x00"   # граница «Аннотация» / «Экстра»; не «---», потому что «---» встречается в самих текстах


def text_lines(key):
    d = ITEMS[key]
    return (d.get("abstractNote") or "").split("\n") + [SEP] + (d.get("extra") or "").split("\n")


def line_block(key, quote):
    """Метка строки артефакта, в которой есть цитата (без учёта повторных пробелов)."""
    q = re.sub(r"\s+", " ", quote).strip()
    pref = "t" if (note_of(key) or "").startswith("Показание") else "o"
    for n, l in enumerate(text_lines(key)):
        if q in re.sub(r"\s+", " ", l):
            return f"{pref}-{key.lower()}-{n}"
    return None


def ref_link(r):
    if r["kind"] == "cite":
        k, pg, q = r["key"], r["page"], r["quote"]
        if pg:
            a = AMAP.get(f"{k}|{pg}|{q}")
            if a:
                return f"[[{title(k)}#^{SHORT.get(k, k.lower())}-{pg}-{a['annotation'].lower()}|↗ {label(k)}, с. {pg}]]"
            return f"[↗ {label(k)}, с. {pg}]({zsel(k)})"
        nt = note_of(k)
        b = line_block(k, q) if nt else None
        if b:
            return f"[[{nt}#^{b}|↗ {nt}]]"
        return f"[[{nt}|↗ {nt}]]" if nt else f"[↗ {label(k)}]({zsel(k)})"
    if r["kind"] == "visual":
        k = r["key"]
        att = next((a["attachment"] for m, a in AMAP.items() if m.startswith(k + "|")), None)
        target = f"zotero://open-pdf/library/items/{att}?page={r['page']}" if att else zsel(k)
        loc = f", {r['loc']}" if r.get("loc") else ""
        return f"[↗ {label(k)}, с. {r['page']}{loc} — сверить глазами]({target})"
    if r["kind"] == "prereg":
        nt = PREREGS[r.get("key", "6GGC73ZD")]          # связи без ключа — к v3
        return f"[[{nt}|↗ {nt.lower()} {r['section']}]]"
    if r["kind"] == "item":
        k = r["key"]
        nt = note_of(k)
        if nt:
            return f"[[{nt}|↗ {nt}]]"
        if k in ITEMS and ITEMS[k].get("itemType") == "attachment":
            return f"[↗ PDF {label(ITEMS[k]['parentItem'])}]({zsel(k)})"
        return f"[↗ {label(k) if k in ITEMS else k}]({zsel(k)})"
    return ""


def code_part(line):
    try:
        toks = list(tokenize.generate_tokens(io.StringIO(line + "\n").readline))
        c = [t for t in toks if t.type == tokenize.COMMENT]
        return (line[:c[0].start[1]] if c else line).strip()
    except (tokenize.TokenError, IndentationError, SyntaxError):
        return line.split("  #")[0].strip()


def code_blocks():
    """{(файл, строка): (метка, [ссылки])} и ссылки на модуль целиком."""
    marks, module = {}, defaultdict(list)
    for e in LINKS:
        if e["anchor"] == "<модуль>":
            module[e["file"]] += e["refs"]
            continue
        lines = (HERE / e["file"]).read_text().split("\n")
        hits = [i for i, l in enumerate(lines, 1) if code_part(l) == e["anchor"]]
        if not hits:
            raise RuntimeError(f"якорь не найден: {e['file']}: {e['anchor']}")
        ln = hits[min(e["occurrence"], len(hits) - 1)]
        bid = f"c-{h6(e['file'], e['anchor'], e['occurrence'])}"
        old = marks.get((e["file"], ln), (bid, []))
        marks[(e["file"], ln)] = (old[0], old[1] + e["refs"])
    return marks, module


def export_code(marks, module):
    out = VAULT / "Доказательства" / "Код"
    out.mkdir(parents=True, exist_ok=True)
    for f in out.glob("*.md"):
        f.unlink()
    for fn in sorted({e["file"] for e in LINKS} | {f for f in CODE_ITEMS if (HERE / f).exists()}):  # весь код из Zotero
        src = (HERE / fn).read_text().split("\n")
        zk = CODE_ITEMS.get(fn)
        md = [f"# Код {fn}", ""]
        if zk:
            md += [f"Оригинал и история изменений — [запись в Zotero]({zsel(zk)}).", ""]
        md += ["Сгенерировано obsidian_export.py — не править вручную.", ""]
        if module.get(fn):
            md += ["Модуль в целом: " + " · ".join(dict.fromkeys(ref_link(r) for r in module[fn])), ""]
        buf = []

        def flush():
            if buf:
                md.extend(["```python", *buf, "```", ""])
                buf.clear()
        n = 0
        while n < len(src):
            line = src[n]
            n += 1
            if (fn, n) in marks:
                flush()
                bid, refs = marks[(fn, n)]
                block = [line]
                # строки-комментарии сразу под отмеченной строкой — её продолжение, остаются в том же блоке
                while n < len(src) and src[n].strip().startswith("#") and (fn, n + 1) not in marks:
                    block.append(src[n])
                    n += 1
                md.extend(["```python", *block, "```",
                           "↳ " + " · ".join(dict.fromkeys(ref_link(r) for r in refs)) + f" ^{bid}", ""])
            else:
                buf.append(line)
        flush()
        (out / f"Код {fn}.md").write_text("\n".join(md))


def export_sources(marks):
    used = defaultdict(list)
    for (fn, ln), (bid, refs) in sorted(marks.items()):
        for r in refs:
            if r["kind"] == "cite" and r["page"]:
                used[(r["key"], r["page"], r["quote"])].append(f"[[Код {fn}#^{bid}|{fn}:{ln}]]")
    out = VAULT / "Источники"
    for f in out.glob("*.md"):
        if f.name != "О папке.md":
            f.unlink()
    by_src = defaultdict(list)
    for mkey, a in AMAP.items():
        k, pg, q = mkey.split("|", 2)
        by_src[k].append((int(pg), q, a))
    for k, rows in by_src.items():
        md = [f"# {title(k)}", "", f"Источник Zotero: [{k}]({zsel(k)}). Цитаты — выделения в PDF. "
              "Сгенерировано — не править вручную.", ""]
        for pg, q, a in sorted(rows):
            url = f"zotero://open-pdf/library/items/{a['attachment']}?page={pg}&annotation={a['annotation']}"
            u = list(dict.fromkeys(used.get((k, str(pg), q), [])))
            md += [f"«{q}» — с. {pg} [↗ PDF]({url})",
                   ("Используется в коде: " + ", ".join(u) if u else "Пока не используется")
                   + f" ^{SHORT.get(k, k.lower())}-{pg}-{a['annotation'].lower()}", ""]
        (out / f"{title(k)}.md").write_text("\n".join(md))


def export_artifacts():
    for sub in ("Показания", "Выводы", "Расчёты"):
        d = VAULT / "Доказательства" / sub
        d.mkdir(parents=True, exist_ok=True)
        for f in d.glob("*.md"):
            f.unlink()
    for k, d in ITEMS.items():
        nt = note_of(k)
        if not nt or k in PREREGS:
            continue
        sub = {"Показание": "Показания", "Вывод": "Выводы", "Расчёт": "Расчёты"}[nt.split()[0]]
        pref = "t" if sub == "Показания" else "o"
        md = [f"# {nt}: {d.get('title', '')}", "", f"Оригинал — [запись в Zotero]({zsel(k)}).", ""]
        for n, l in enumerate(text_lines(k)):
            if l == SEP:
                md += ["**Происхождение и пояснения:**", ""]
                continue
            if not l.strip():
                continue
            body = f"`{l}`" if sub in ("Выводы", "Расчёты") else l   # моноширинно: в тексте выводов бывают [[ ]]
            if body.strip() == "---":
                body = "\\---"                     # иначе Markdown превратит строку в горизонтальную черту
            md += [f"{body} ^{pref}-{k.lower()}-{n}", ""]
        (VAULT / "Доказательства" / sub / f"{nt}.md").write_text("\n".join(md))


def linkify(text):
    def cite(m):
        return f"«{m['quote']}» " + ref_link({"kind": "cite", "key": m["key"], "page": m["page"], "quote": m["quote"]})
    text = CITE.sub(cite, text)
    return VIS.sub(lambda m: ref_link({"kind": "visual", "key": m["key"], "page": m["page"], "loc": ""}), text)


def export_documents():
    for k, nt in PREREGS.items():
        (VAULT / f"{nt}.md").write_text(
            f"> Утверждённый оригинал (без ссылок) — [запись в Zotero]({zsel(k)}).\n\n" + linkify(ITEMS[k]["abstractNote"]))
    # «Журнал отступлений.md» ведётся вручную в хранилище и генератором не перезаписывается


def main():
    marks, module = code_blocks()
    export_code(marks, module)
    export_sources(marks)
    export_artifacts()
    export_documents()
    print("строк кода со ссылками:", len(marks), "| модулей:", len({k[0] for k in marks}))


if __name__ == "__main__":
    main()

```
