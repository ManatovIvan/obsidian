# Код label_sessions.py

Оригинал и история изменений — [запись в Zotero](zotero://select/library/items/M4AK8KZB).

Сгенерировано obsidian_export.py — не править вручную.

```python
"""Разметка весенних сессий по содержимому кадров: каждый найденный кадр DSSS декодируется,
и для кадров с верной FCS записываются тип, адреса, мощность и оценка сдвига частоты.

Запуск: python label_sessions.py R1 [E1 ...] [--seconds S]
Вывод: out/frames_<ID>.csv (по строке на каждый кадр, прошедший CRC-16 заголовка),
       out/summary_<ID>.json (счётчики этапов, SHA-256 файла, параметры, версии библиотек).
"""
import argparse
import csv
import hashlib
import json
import sys
import time
from collections import Counter
from pathlib import Path

import numpy as np
import torch

import dsss_rx as rx
import iq_io
import mac80211

BLOCK = 1 << 25                                   # ВЫБОР: 33,5 млн отсчётов (1,7 с) на блок — ~2 ГБ на GPU с запасом
OVERLAP = 4096                                    # ВЫБОР: > (32 окно + 16 SFD + 7 дескремблер + 1) × 20 + 19 отсчётов,
                                                  #   чтобы SFD на стыке блоков нашёлся целиком в одном из них
OUT = Path(__file__).resolve().parent / "out"


def stage1(sid, n_samples, device):
    """Один последовательный проход по файлу: кандидаты SFD + SHA-256 (если читается весь файл)."""
    hasher = hashlib.sha256() if n_samples == iq_io.N_SAMPLES else None
    tail = np.zeros(0, np.complex64)
    abs_start = 0                                 # абсолютный индекс первого отсчёта текущего буфера
    found, done = [], 0
    for raw in iq_io.iter_blocks(sid, BLOCK, hasher):
        x = np.concatenate([tail, iq_io.to_complex(raw).astype(np.complex64)])
        pos, _ = rx.find_sfd_candidates(x, torch, device)
        found.append(pos + abs_start)
        done += len(raw) // 2
        tail = x[-OVERLAP:]
        abs_start += len(x) - OVERLAP
        if done >= n_samples:
            break
    if hasher is not None:
```

```python
        iq_io.check_sha(sid, hasher.hexdigest())  # файл тот же, что при записи эталонных SHA-256
```
↳ [[Вывод GJI5ZH7N|↗ Вывод GJI5ZH7N]] ^c-7906c3

```python
    allpos = np.concatenate(found) if found else np.zeros(0, np.int64)
    allpos = allpos[allpos < n_samples]
    return allpos, (hasher.hexdigest() if hasher else None)


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("sessions", nargs="+", choices=iq_io.SESSION_ORDER)
    ap.add_argument("--seconds", type=float, default=None, help="обработать только начало сессии (отладка)")
    args = ap.parse_args()
    OUT.mkdir(exist_ok=True)
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    for sid in args.sessions:
        t0 = time.time()
        n = iq_io.N_SAMPLES if args.seconds is None else int(args.seconds * iq_io.SAMPLE_RATE)
        cand, sha = stage1(sid, n, device)
        clusters = rx.cluster(cand)
        mm = iq_io.open_session(sid)
        stages = Counter()
        rows = []
        for a0 in clusters:
            rec = rx.decode_candidate(mm, a0)
            stages[rec["stage"]] += 1
            if not rec.get("crc16"):
                continue                          # ВЫБОР: без верного CRC-16 заголовка кадр не считается найденным
            row = {k: rec.get(k) for k in ("a", "start", "end", "preamble", "signal", "rate_mbps", "length_us",
                                           "n_octets", "fcs", "cfo_hz", "power", "timing_drift", "stage")}
            if rec.get("fcs"):
                row.update(mac80211.parse(rec["mpdu"]))   # ВЫБОР: поля MAC доверяем только при верной FCS
            row["mpdu_hex"] = rec["mpdu"].hex() if "mpdu" in rec else ""
            rows.append(row)
        # Один кадр может быть найден дважды (разные кластеры у одного SFD). ВЫБОР: ближе одного
        # символа — тот же кадр; из дублей оставляем копию с верной FCS, иначе первую по позиции
        rows.sort(key=lambda r: r["a"])
        dedup = []
        for r in rows:
            if dedup and r["a"] - dedup[-1]["a"] <= rx.SPS:
                if r["fcs"] and not dedup[-1]["fcs"]:
                    dedup[-1] = r
                continue
            dedup.append(r)
        suffix = "" if args.seconds is None else f"_first{args.seconds:g}s"
        cols = ["a", "start", "end", "preamble", "signal", "rate_mbps", "length_us", "n_octets", "fcs", "stage",
                "cfo_hz", "power", "timing_drift", "name", "ftype", "subtype", "a1", "a2", "a3", "seq", "ssid",
                "reason", "protected", "tods", "fromds", "mpdu_hex"]
        with open(OUT / f"frames_{sid}{suffix}.csv", "w", newline="") as f:
            w = csv.DictWriter(f, fieldnames=cols, extrasaction="ignore")
            w.writeheader()
            w.writerows(dedup)
        summary = {
            "session": sid, "samples_processed": n, "sha256": sha, "sha256_expected": iq_io.SHA256[sid],
            "sfd_candidates": int(len(cand)), "clusters": int(len(clusters)), "stages": dict(stages),
            "frames_crc16_ok": len(dedup), "frames_fcs_ok": sum(1 for r in dedup if r["fcs"]),
            "names_fcs_ok": dict(Counter(r.get("name") for r in dedup if r["fcs"])),
            "seconds_wall": round(time.time() - t0, 1),
            "versions": {"python": sys.version.split()[0], "numpy": np.__version__, "torch": torch.__version__,
                         "device": torch.cuda.get_device_name(0) if device.type == "cuda" else "cpu"},
        }
        (OUT / f"summary_{sid}{suffix}.json").write_text(json.dumps(summary, ensure_ascii=False, indent=1))
        print(json.dumps(summary, ensure_ascii=False))


if __name__ == "__main__":
    main()

```
