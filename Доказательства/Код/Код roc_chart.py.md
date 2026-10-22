# Код roc_chart.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/RHZCJH3X).

Сгенерировано obsidian_export.py — не править вручную.

Ссылаются на модуль целиком: [[Текст статьи#^u3-4|Текст статьи ^u3-4]]

Модуль в целом: [[Показание 9K4WBJ3W|↗ Показание 9K4WBJ3W]] · [↗ II Международная научно-практическая кон](zotero://select/library/items/H8X4N5KH)

```python
"""Рисунок статьи: ROC-кривые как диаграмма Word со встроенной таблицей Excel (П11: рисунок редактируется
средствами Word или Excel, чёрно-белый — методы различаются типом линии).

Точки ROC — алгоритм 1 Fawcett 2006 (сортировка по убыванию оценки, точка после каждой группы равных оценок).
Диаграмму строит python-pptx (разметка DrawingML одна и та же в .pptx и .docx), затем её часть и встроенная
книга Excel переносятся в документ Word.

Обоснования строк кода (источники, цитаты, решения) — в хранилище статьи Obsidian, а не в коде.
"""
import re

import numpy as np
from docx.opc.constants import RELATIONSHIP_TYPE as RT
from docx.opc.packuri import PackURI
from docx.opc.part import Part
from docx.oxml import parse_xml
from pptx import Presentation
from pptx.chart.data import XyChartData
from pptx.dml.color import RGBColor
from pptx.enum.chart import XL_CHART_TYPE, XL_LEGEND_POSITION
from pptx.enum.dml import MSO_LINE_DASH_STYLE as DASH
from pptx.util import Cm, Pt

STEP = 0.005          # ВЫБОР: прореживание точек для показа — соседние точки отстоят не меньше чем на 0,005


def roc_points(scores, labels):
    """Fawcett 2006, алгоритм 1: (FPR, TPR) после каждой группы равных оценок, от (0, 0) до (1, 1)."""
```

```python
    o = np.argsort(-scores, kind="stable")
```
↳ [[An introduction to ROC analysis#^343efdiu-6-25wk3pbd|↗ Fawcett 2006, с. 6]] ^c-a339ed

```python
    s, y = scores[o], labels[o].astype(bool)
    tp, fp = np.cumsum(y), np.cumsum(~y)
```

```python
    last = np.r_[s[1:] != s[:-1], True]           # конец группы равных оценок
```
↳ [[An introduction to ROC analysis#^343efdiu-6-637q8wct|↗ Fawcett 2006, с. 6]] ^c-f81311

```python
    return np.r_[0, fp[last] / fp[-1]], np.r_[0, tp[last] / tp[-1]]
```
↳ [[An introduction to ROC analysis#^343efdiu-2-qqfmxthb|↗ Fawcett 2006, с. 2]] ^c-dc67a5

```python


def thin(fpr, tpr):
    keep, lf, lt = [0], fpr[0], tpr[0]
    for i in range(1, len(fpr) - 1):
        if fpr[i] - lf >= STEP or tpr[i] - lt >= STEP:
            keep.append(i); lf, lt = fpr[i], tpr[i]
    keep.append(len(fpr) - 1)
    return fpr[keep], tpr[keep]


def chart_blobs(series, width, height, xtitle, ytitle):
    """series: [(подпись, fpr, tpr, тип линии)] -> (XML диаграммы, книга .xlsx)."""
    cd = XyChartData()
    for name, x, y, _ in series:
        s = cd.add_series(name)
        for a, b in zip(x, y):
            s.add_data_point(float(a), float(b))
    prs = Presentation()
    gf = prs.slides.add_slide(prs.slide_layouts[6]).shapes.add_chart(
```

```python
        XL_CHART_TYPE.XY_SCATTER_LINES_NO_MARKERS, 0, 0, Cm(width), Cm(height), cd)
```
↳ [↗ II Международная научно-практическая кон](zotero://select/library/items/H8X4N5KH) ^c-55df57

```python
    ch = gf.chart
    ch.has_title = False
    ch.font.name, ch.font.size = "Times New Roman", Pt(10)
    ch.has_legend = True
    ch.legend.position, ch.legend.include_in_layout = XL_LEGEND_POSITION.RIGHT, False
    for plot_series, (_, _, _, dash) in zip(ch.plots[0].series, series):
        ln = plot_series.format.line
```

```python
        ln.color.rgb, ln.width, ln.dash_style = RGBColor(0, 0, 0), Pt(1.25 if dash != DASH.ROUND_DOT else 0.75), dash
```
↳ [↗ II Международная научно-практическая кон](zotero://select/library/items/H8X4N5KH) ^c-8c9d81

```python
        plot_series.smooth = False
    for ax, title in ((ch.category_axis, xtitle), (ch.value_axis, ytitle)):
        ax.minimum_scale, ax.maximum_scale, ax.major_unit = 0, 1, 0.2
        ax.has_major_gridlines = False
        ax.tick_labels.number_format, ax.tick_labels.number_format_is_linked = "0.0", False
        ax.format.line.color.rgb = RGBColor(0, 0, 0)
        ax.has_title = True
        ax.axis_title.text_frame.text = title
        r = ax.axis_title.text_frame.paragraphs[0].runs[0]
        r.font.size, r.font.bold, r.font.name = Pt(10), False, "Times New Roman"
    part = gf.chart_part
    return part.blob, part.chart_workbook.xlsx_part.blob


def add_chart(doc, paragraph, series, width, height, xtitle, ytitle):
    """Вставляет диаграмму в абзац как встроенный объект (wp:inline)."""
    xml, xlsx = chart_blobs(series, width, height, xtitle, ytitle)
    pkg = doc.part.package
    n = len([p for p in pkg.iter_parts() if str(p.partname).startswith("/word/charts/")]) + 1
    book = Part(PackURI(f"/word/embeddings/Microsoft_Excel_Worksheet{n}.xlsx"),
                "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet", xlsx, pkg)
    chart = Part(PackURI(f"/word/charts/chart{n}.xml"),
                 "application/vnd.openxmlformats-officedocument.drawingml.chart+xml", b"", pkg)
    rid_book = chart.relate_to(book, RT.PACKAGE)
    chart._blob = re.sub(rb'(<c:externalData r:id=")[^"]+', b"\\g<1>" + rid_book.encode(), xml)
    rid = doc.part.relate_to(chart, RT.CHART)
    cx, cy = int(Cm(width)), int(Cm(height))
    paragraph.add_run()._r.append(parse_xml(
        f'<w:drawing xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main" '
        f'xmlns:wp="http://schemas.openxmlformats.org/drawingml/2006/wordprocessingDrawing" '
        f'xmlns:a="http://schemas.openxmlformats.org/drawingml/2006/main" '
        f'xmlns:c="http://schemas.openxmlformats.org/drawingml/2006/chart" '
        f'xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships">'
        f'<wp:inline distT="0" distB="0" distL="0" distR="0"><wp:extent cx="{cx}" cy="{cy}"/>'
        f'<wp:effectExtent l="0" t="0" r="0" b="0"/><wp:docPr id="{1000 + n}" name="Диаграмма {n}"/>'
        f'<wp:cNvGraphicFramePr/><a:graphic><a:graphicData uri="http://schemas.openxmlformats.org/drawingml/2006/chart">'
        f'<c:chart r:id="{rid}"/></a:graphicData></a:graphic></wp:inline></w:drawing>'))

```
