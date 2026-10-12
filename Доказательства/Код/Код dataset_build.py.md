# Код dataset_build.py

Файл verification/dataset_build.py на сервере. Сгенерировано obsidian_export.py — не править вручную.

```python
"""Сборка набора кадров по утверждённой предрегистрации (6GGC73ZD, PREREG_v3_approved.md, §1–§2).

Вход: out/frames_<ID>.csv (разметка, GZJDPPNF) и файлы сессий. Выход:
  out/dataset.npz        — окна IQ (1280 отсчётов) и сведения о каждом кадре;
  out/dataset_report.txt — объёмы классов, проверка C3 (биты SYNC), шум и SNR (C2).

Обоснования: ⟦КЛЮЧ с.N: «цитата»⟧ — проверяется check_citations.py; ПРЕДРЕГ §N — пункт
утверждённой предрегистрации; ВЫБОР — решение этого кода с причиной.
"""
import csv
import hashlib
import json
from collections import Counter
from pathlib import Path

import numpy as np

import dsss_rx as rx
import iq_io

OUT = Path(__file__).resolve().parent / "out"
```

`ROUTER = "28:28:5d:e0:6c:c6"` [↗ Манатов 2026](zotero://select/library/items/JV2GWUWN) ^l22

`SESSIONS = ["R1", "E1", "R2", "R3", "E3"]` [↗ Манатов 2026](zotero://select/library/items/MTFI5CTD) (E2) ^l23

```python
WIN_SYMBOLS = 64                                  # ПРЕДРЕГ §2 C3: последние 64 символа SYNC+SFD
```

`WIN = WIN_SYMBOLS * rx.SPS` 1280 отсчётов; [[Deep Learning for RF Fingerprinting  A Massive Experimental Study#^5ag9m5dk-7-wnwnxzpg|↗ Jian 2020, с. 7]] ^l25

```python
NOISE_BLOCK = 2000                                # ВЫБОР: 100 мкс — короче любого промежутка между кадрами роутера (beacon раз в ~0,1 с)
NOISE_GUARD = 20_000                              # ВЫБОР: 1 мс отступа от любого найденного кадра
NOISE_TRIES = 20_000                              # ВЫБОР: случайных блоков-кандидатов на сессию
SEED = 20260504                                   # ВЫБОР: фиксированное зерно (дата записи) для повторяемости


def load_frames(sid):
    with open(OUT / f"frames_{sid}.csv", newline="") as f:
        return list(csv.DictReader(f))


def tsf_offset_s(r):
    """(TSF − время в записи), округлённое до секунды — «семья» передатчика (ПРЕДРЕГ §1, L2).
    Timestamp — первые 8 октетов тела Beacon/Probe Response, младший октет первым (см. mac80211.parse)."""
    b = bytes.fromhex(r["mpdu_hex"])
```

`tsf = int.from_bytes(b[24:32], "little")` [[IEEE Std 802.11-2020#^ieee-880-kxg73qva|↗ IEEE 802.11-2020, с. 880]] ^l41

```python
    t_us = int(r["a"]) / iq_io.SAMPLE_RATE * 1e6
```

`return round((tsf - t_us) / 1e6)` [[IEEE Std 802.11-2020#^ieee-2050-hk6zva2u|↗ IEEE 802.11-2020, с. 2050]] ^l43

```python


def label_session(rows, prev_router_offset):
    """Метки L1–L3 для кадров с верной FCS и TA = MAC роутера. Возвращает (список, offset роутера)."""
    ok = [r for r in rows if r["fcs"] == "True" and r["a2"] == ROUTER]
    fams = Counter(tsf_offset_s(r) for r in ok if r["name"] in ("beacon", "probe_resp"))
    router_fams = {f for f in fams if prev_router_offset is None or f >= prev_router_offset}
    # ПРЕДРЕГ §1 L2: семья роутера продолжает возрастающий ряд; в первой сессии семья должна быть одна
    if prev_router_offset is None and len(fams) != 1:
        raise RuntimeError(f"в первой сессии {len(fams)} семей TSF, правило L2 неприменимо")
    out = []
    for r in ok:
        if (r["name"] == "deauth" and r["a1"] == "ff:ff:ff:ff:ff:ff" and r["reason"] == "2"
                and r["seq"] == "4095"):
            lab, rule = 1, "L1"                   # ПРЕДРЕГ §1 L1
        elif r["name"] in ("beacon", "probe_resp") and tsf_offset_s(r) not in router_fams:
            lab, rule = 1, "L2"                   # ПРЕДРЕГ §1 L2
        else:
            lab, rule = 0, "L3"                   # ПРЕДРЕГ §1 L3
        out.append((r, lab, rule))
    new_offset = max(router_fams) if router_fams else prev_router_offset
    return out, new_offset


def window(mm, a):
    """Окно последних 64 символов SYNC+SFD: символы −63..0, отсчёты [a − 63·20, a + 20)."""
    return iq_io.read_samples(mm, a - (WIN_SYMBOLS - 1) * rx.SPS, a + rx.SPS)


def sync_raw_bits(mm, a, cfo_hz):
    """Скремблированные (сырые) биты DBPSK символов −62..0 — для проверки C3."""
    x = iq_io.read_samples(mm, a - WIN_SYMBOLS * rx.SPS, a + 2 * rx.SPS)
    c = np.correlate(x, rx.TEMPLATE, mode="valid")  # тот же согласованный фильтр, что в dsss_rx
    s = c[rx.SPS * np.arange(WIN_SYMBOLS + 1)]     # символы −64..0
    theta = 2 * np.pi * cfo_hz / rx.SYMBOL_RATE
    return rx._dbpsk_bits(s[:-1], s[1:], theta)    # биты символов −63..0 (табл. 15-7, см. dsss_rx)


def noise_power(mm, rows, rng):
    """C2: мощность шума на интервалах без найденных кадров —
```

↳ [[IEEE Std 802.11-2020#^ieee-159-ztf6ht9v|↗ IEEE 802.11-2020, с. 159]] ^l84

```python
    Медиана по блокам (ВЫБОР: устойчива к редким кадрам OFDM, которые приёмник DSSS не видит)."""
    busy = []
    for r in rows:
        a = int(r["a"])
        s = int(r["start"]) if r["start"] else a - (128 + 15) * rx.SPS
        e = int(r["end"]) if r["end"] else a + 200 * rx.SPS   # ВЫБОР: для кадров без MPDU — 200 мкс после SFD
        busy.append((s - NOISE_GUARD, e + NOISE_GUARD))
    busy.sort()
    starts = np.array([b[0] for b in busy])
    ends = np.maximum.accumulate(np.array([b[1] for b in busy]))
    powers = []
    for p0 in rng.integers(0, iq_io.N_SAMPLES - NOISE_BLOCK, NOISE_TRIES):
        k = np.searchsorted(starts, p0 + NOISE_BLOCK) - 1   # последний интервал, начавшийся до конца блока
        if k >= 0 and ends[k] > p0:
            continue                              # блок задевает занятый интервал
        x = iq_io.read_samples(mm, p0, p0 + NOISE_BLOCK)
        powers.append(float(np.mean(np.abs(x) ** 2)))
    return np.array(powers)


def main():
    rng = np.random.default_rng(SEED)
    lines, recs = [], []
    prev = None
    for sid in iq_io.SESSION_ORDER:              # порядок записи нужен правилу L2 (возрастающий ряд TSF)
        rows = load_frames(sid)
        labeled, prev = label_session(rows, prev)
        if sid not in SESSIONS:
            lines.append(f"{sid}: исключена (MTFI5CTD); offset роутера {prev} с учтён для правила L2")
            continue
        mm = iq_io.open_session(sid)
        pn = noise_power(mm, rows, rng)
        lines.append(f"{sid}: шум (медиана по {len(pn)} блокам по 100 мкс) = {np.median(pn):.2f} ед.АЦП², "
                     f"P10 = {np.percentile(pn, 10):.2f}, P90 = {np.percentile(pn, 90):.2f}; offset роутера {prev} с")
        for r, lab, rule in labeled:
            a = int(r["a"])
            w = window(mm, a)
            if len(w) != WIN:
                continue
            bits = sync_raw_bits(mm, a, float(r["cfo_hz"]))
            recs.append(dict(session=sid, a=a, label=lab, rule=rule, name=r["name"],
                             cfo=float(r["cfo_hz"]), p_win=float(np.mean(np.abs(w) ** 2)),
                             noise=float(np.median(pn)), iq=w.astype(np.complex64),
                             bits="".join(map(str, bits.tolist()))))
    # --- C3: совпадают ли скремблированные биты окна у роутера и ESP32 ---
    lines.append("\nC3: последовательности сырых бит DBPSK символов −63..0 (окно сети):")
    by = {}
    for r in recs:
        by.setdefault((r["session"], r["label"]), Counter())[r["bits"]] += 1
    allseq = Counter()
    for (sid, lab), cnt in sorted(by.items()):
        top, n = cnt.most_common(1)[0]
        allseq.update({top: 0})
        lines.append(f"  {sid} {'ESP32' if lab else 'роутер'}: кадров {sum(cnt.values())}, различных "
                     f"последовательностей {len(cnt)}, самая частая ({n} кадров) = {top}")
    tops = {cnt.most_common(1)[0][0] for cnt in by.values()}
    same = len(tops) == 1
    lines.append(f"  самые частые последовательности во всех группах совпадают: {same}")
    lines.append("  решение по ПРЕДРЕГ §2 C3: " + ("вход используется как есть (последовательность общая)" if same
                 else "модуляция снимается умножением на идеальный знак ±1"))
    # --- объёмы и SNR ---
    lines.append("\nОбъёмы (session, класс, правило): " + json.dumps(
        {f"{k[0]}/{'ESP32' if k[1] else 'router'}/{k[2]}": v
         for k, v in sorted(Counter((r["session"], r["label"], r["rule"]) for r in recs).items())}, ensure_ascii=False))
    for sid in SESSIONS:
        for lab in (0, 1):
            g = [r for r in recs if r["session"] == sid and r["label"] == lab]
            if g:
                snr = [10 * np.log10(max(r["p_win"] - r["noise"], 1e-9) / r["noise"]) for r in g]
                lines.append(f"  SNR окна, дБ: {sid} {'ESP32' if lab else 'роутер'}: медиана {np.median(snr):.1f}, "
                             f"P10 {np.percentile(snr, 10):.1f}, P90 {np.percentile(snr, 90):.1f}")
    np.savez(OUT / "dataset.npz",
             iq=np.stack([r["iq"] for r in recs]), label=np.array([r["label"] for r in recs], np.int8),
             session=np.array([r["session"] for r in recs]), a=np.array([r["a"] for r in recs], np.int64),
             rule=np.array([r["rule"] for r in recs]), name=np.array([r["name"] for r in recs]),
             cfo=np.array([r["cfo"] for r in recs]), p_win=np.array([r["p_win"] for r in recs]),
             noise=np.array([r["noise"] for r in recs]), bits=np.array([r["bits"] for r in recs]),
             strip_modulation=np.array(not same))
    h = hashlib.sha256((OUT / "dataset.npz").read_bytes()).hexdigest()
    lines.append(f"\nout/dataset.npz: {len(recs)} кадров, SHA-256 {h}")
    text = "\n".join(lines)
    (OUT / "dataset_report.txt").write_text(text + "\n")
    print(text)


if __name__ == "__main__":
    main()

```
