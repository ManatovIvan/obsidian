# Расчёт WGUFQ4BV: Расчёт: соответствие блоков «сессий» в коде реальным сессиям записи; число пакетов по сессиям после фильтра

Оригинал — [запись в Zotero](zotero://select/library/items/WGUFQ4BV).

`Цель: проверить, из каких реальных сессий взяты блоки, которые train_v2.py, train_autoencoder.py, cfo_analysis.py и compute_roc.py называют «сессиями» (трети массива: N = len // 3, первые PER = 50 000 пакетов каждой трети).` ^o-wgufq4bv-0

`Метод: для router.npy и esp32.npy (сырые пакеты до фильтра, сессии сконкатенированы в порядке R1,R2,R3 / E1,E2,E3 с размерами из вывода extract_packets.py: 355840/470687/510234 и 398056/534208/1042644) повторён фильтр mean(|x|^2) > 2000; каждому оставшемуся пакету приписана сессия; проверено, что результат поэлементно совпадает с router_clean.npy / esp32_clean.npy (выборочная проверка 4 индексов + совпадение общего числа).` ^o-wgufq4bv-2

`Вывод скрипта (сервер, hackrf_poc, 2026-10-12):` ^o-wgufq4bv-4

`router post-filter per session: [354665, 469344, 507942] total 1331951` ^o-wgufq4bv-5

`  block t1 [0:50000] sessions: {'R1': 50000}` ^o-wgufq4bv-6

`  block t2 [443983:493983] sessions: {'R2': 50000}` ^o-wgufq4bv-7

`  block t3 [887966:937966] sessions: {'R3': 50000}` ^o-wgufq4bv-8

`  check clean==raw[keep]: True` ^o-wgufq4bv-9

`esp32 post-filter per session: [394254, 532062, 510735] total 1437051` ^o-wgufq4bv-10

`  block t1 [0:50000] sessions: {'E1': 50000}` ^o-wgufq4bv-11

`  block t2 [479017:529017] sessions: {'E2': 50000}` ^o-wgufq4bv-12

`  block t3 [958034:1008034] sessions: {'E3': 50000}` ^o-wgufq4bv-13

`  check clean==raw[keep]: True` ^o-wgufq4bv-14

`Итог: каждый блок из 50 000 пакетов целиком лежит внутри своей реальной сессии; названия r1..r3 / e1..e3 в коде соответствуют R1..R3 / E1..E3. Используется 50 000 пакетов на сессию (первые по времени), а не вся сессия.` ^o-wgufq4bv-16

`Попутно: число пакетов после фильтра по сессиям — R1 354 665, R2 469 344, R3 507 942; E1 394 254, E2 532 062, E3 510 735 (фильтр удалил 531 909 из 1 042 644 пакетов E3).` ^o-wgufq4bv-17

`Скрипт (выполнен .venv/bin/python из каталога hackrf_poc):` ^o-wgufq4bv-19

`import numpy as np` ^o-wgufq4bv-20

`for name,sizes,clean in [('router',[355840,470687,510234],'router_clean'),('esp32',[398056,534208,1042644],'esp32_clean')]:` ^o-wgufq4bv-21

`    a=np.load(f'dataset/{name}.npy',mmap_mode='r'); assert len(a)==sum(sizes)` ^o-wgufq4bv-22

`    keep=np.zeros(len(a),bool)` ^o-wgufq4bv-23

`    for s in range(0,len(a),200000):` ^o-wgufq4bv-24

`        x=np.asarray(a[s:s+200000]); keep[s:s+200000]=np.mean(np.abs(x)**2,axis=1)>2000` ^o-wgufq4bv-25

`    c=np.load(f'dataset/{clean}.npy',mmap_mode='r'); assert keep.sum()==len(c)` ^o-wgufq4bv-26

`    lab=np.repeat(np.arange(3),sizes)[keep]` ^o-wgufq4bv-27

`    post=[int((lab==i).sum()) for i in range(3)]` ^o-wgufq4bv-28

`    print(name,'post-filter per session:',post,'total',sum(post))` ^o-wgufq4bv-29

`    N=len(c)//3; PER=50000` ^o-wgufq4bv-30

`    for blk,st in [('t1',0),('t2',N),('t3',2*N)]:` ^o-wgufq4bv-31

`        l=lab[st:st+PER]; print(f'  block {blk} [{st}:{st+PER}] sessions:',{f"{'RE'[name=='esp32']}{i+1}":int((l==i).sum()) for i in range(3) if (l==i).sum()})` ^o-wgufq4bv-32

`    idx=np.flatnonzero(keep)[[0,12345,len(c)//2,len(c)-1]]` ^o-wgufq4bv-33

`    print('  check clean==raw[keep]:',all(np.array_equal(np.asarray(c[j]),np.asarray(a[i])) for j,i in zip([0,12345,len(c)//2,len(c)-1],idx)))` ^o-wgufq4bv-34

**Происхождение и пояснения:**

`Тип: производный расчёт по весенним данным на диске (dataset/router.npy, esp32.npy, router_clean.npy, esp32_clean.npy от 4 мая 2026), выполнен Claude в сессии Claude Code 2026-10-12 (https://claude.ai/code/session_01D2gEwEtZbH7TnLruCtscax).` ^o-wgufq4bv-36

`Входные источники: вывод extract_packets.py (CJB95AVV) — размеры сессий до фильтра; вывод фильтра по мощности (XMHEQIKV) — порог 2000 и итоговые объёмы; код train_v2.py, train_autoencoder.py, cfo_analysis.py, compute_roc.py (коллекция 3) — схема «третей».` ^o-wgufq4bv-37
