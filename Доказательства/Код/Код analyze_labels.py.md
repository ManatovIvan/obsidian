# Код analyze_labels.py

Файл verification/analyze_labels.py на сервере. Сгенерировано obsidian_export.py — не править вручную.

```python
"""Сводка разметки out/frames_<ID>.csv: кто и что передавал в каждой сессии.

Используются только кадры с верной FCS (CRC-32): у них тип и адреса прочитаны без ошибок
(проверка FCS — dsss_rx.fcs_ok; 32-битный CRC). Ничего не обучается и
не подбирается. Вывод: out/label_summary.txt (дословно сохраняется в Zotero как дочерняя запись).
"""
import csv
from collections import Counter, defaultdict
from pathlib import Path

import numpy as np

import iq_io

OUT = Path(__file__).resolve().parent / "out"


def q(x):
    """Медиана [Q1; Q3] — ВЫБОР: устойчиво к выбросам и не предполагает нормальности."""
    x = np.asarray(x, float)
    if len(x) == 0:
        return "—"
    return f"{np.median(x):.0f} [{np.percentile(x, 25):.0f}; {np.percentile(x, 75):.0f}]"


def load(sid):
    with open(OUT / f"frames_{sid}.csv", newline="") as f:
        return list(csv.DictReader(f))


def main():
    lines = []
    for sid in iq_io.SESSION_ORDER:
        rows = load(sid)
        ok = [r for r in rows if r["fcs"] == "True"]
        bad = [r for r in rows if r["fcs"] != "True"]
        lines.append(f"=== {sid}: кадров с верным CRC-16 заголовка {len(rows)}, из них с верной FCS {len(ok)}")
        lines.append(f"    FCS ok — преамбула {dict(Counter(r['preamble'] for r in ok))}, "
                     f"скорость Мбит/с {dict(Counter(r['rate_mbps'] for r in ok))}")
        lines.append(f"    без верной FCS — этапы {dict(Counter(r['stage'] for r in bad))}, "
                     f"мощность {q([float(r['power']) for r in bad if r['power']])}")
        groups = defaultdict(list)
        for r in ok:
            groups[(r["a2"] or "(нет TA)", r["name"])].append(r)
        lines.append("    TA (A2) | тип | число | мощность, ед.АЦП² мед.[Q1;Q3] | CFO, кГц мед.[Q1;Q3] | SSID")
        for (ta, name), part in sorted(groups.items(), key=lambda kv: -len(kv[1])):
            ssids = ",".join(sorted({r["ssid"] for r in part if r["ssid"]}))[:40]
            lines.append(f"    {ta} | {name} | {len(part)} | {q([float(r['power']) for r in part])} | "
                         f"{q([float(r['cfo_hz']) / 1e3 for r in part])} | {ssids}")
        da = [r for r in ok if r["name"] in ("deauth", "disassoc")]
        if da:
            lines.append(f"    адресаты deauth/disassoc (A1): {dict(Counter(r['a1'] for r in da).most_common(5))}")
            lines.append(f"    reason code: {dict(Counter(r['reason'] for r in da).most_common(5))}")
            lines.append(f"    номер последовательности deauth/disassoc: {dict(Counter(r['seq'] for r in da).most_common(5))}")
        # Признаки содержимого beacon/probe_resp, не зависящие от физического уровня:
        # длина MPDU и поле Timestamp — первое поле тела (см. mac80211.parse):
```

↳ [[IEEE Std 802.11-2020#^ieee-880-tftuir4k|↗ IEEE 802.11-2020, с. 880]] ^l57

↳ [[IEEE Std 802.11-2020#^ieee-2050-r8zi8mqn|↗ IEEE 802.11-2020, с. 2050]] ^l58

```python
        # (2^64 — верхний индекс теряется при извлечении текста); младший октет первым — см. mac80211.parse.
        # Разность TSF − время в записи постоянна для одного передатчика (оба идут в микросекундах),
        # ВЫБОР: группируем по этой разности с шагом 1 с — сдвиги часов за 120 с много меньше секунды.
        fam = defaultdict(list)
        for r in ok:
            if r["name"] in ("beacon", "probe_resp"):
                b = bytes.fromhex(r["mpdu_hex"])
                tsf = int.from_bytes(b[24:32], "little")
                t_us = int(r["a"]) / iq_io.SAMPLE_RATE * 1e6
                fam[(round((tsf - t_us) / 1e6), r["name"], len(b))].append(r)
        if fam:
            lines.append("    beacon/probe_resp: (TSF − t) с | тип | длина MPDU | число | мощность | CFO кГц | номера seq мин–макс")
            for (off, name, ln), part in sorted(fam.items(), key=lambda kv: -len(kv[1])):
                seqs = [int(r["seq"]) for r in part]
                lines.append(f"    {off} | {name} | {ln} | {len(part)} | {q([float(r['power']) for r in part])} | "
                             f"{q([float(r['cfo_hz']) / 1e3 for r in part])} | {min(seqs)}–{max(seqs)}")
    text = "\n".join(lines)
    (OUT / "label_summary.txt").write_text(text + "\n")
    print(text)


if __name__ == "__main__":
    main()
```
