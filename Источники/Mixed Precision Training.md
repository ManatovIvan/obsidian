# Mixed Precision Training

Источник Zotero: [AEPHRIQM](zotero://select/library/items/AEPHRIQM). Цитаты — выделения в PDF. Сгенерировано — не править вручную.

«we propose loss-scaling to preserve gradient values with small magnitudes» — с. 1 [↗ PDF](zotero://open-pdf/library/items/ZTUCFXQB?page=1&annotation=9ZDRRR86)
Используется в коде: [[Код rc_cdae.py#^c-5bb8ec|rc_cdae.py:33]], [[Код svdd.py#^c-e28f19|svdd.py:85]] ^aephriqm-1-9zdrrr86

«we recommend maintaining a single-precision copy of weights that accumulates» — с. 1 [↗ PDF](zotero://open-pdf/library/items/ZTUCFXQB?page=1&annotation=CR32AFI3)
Используется в коде: [[Код rc_cdae.py#^c-b5ad0f|rc_cdae.py:21]], [[Код rc_cdae.py#^c-d4572e|rc_cdae.py:40]], [[Код svdd.py#^c-cd56e1|svdd.py:96]], [[Код svdd.py#^c-ce2e2e|svdd.py:112]] ^aephriqm-1-cr32afi3

«ensuring that no hyper-parameters (such as gradient clipping threshold, weight decay, etc.) have to be adjusted» — с. 4 [↗ PDF](zotero://open-pdf/library/items/ZTUCFXQB?page=4&annotation=22M9FG48)
Используется в коде: [[Код rc_cdae.py#^c-5bb8ec|rc_cdae.py:33]], [[Код svdd.py#^c-e28f19|svdd.py:85]] ^aephriqm-4-22m9fg48

«Large reductions (sums across elements of a vector) should be carried out in FP32» — с. 5 [↗ PDF](zotero://open-pdf/library/items/ZTUCFXQB?page=5&annotation=BST4SIZJ)
Используется в коде: [[Код rc_cdae.py#^c-cdfcea|rc_cdae.py:23]], [[Код rc_cdae.py#^c-b87eaa|rc_cdae.py:42]], [[Код svdd.py#^c-aa6052|svdd.py:113]] ^aephriqm-5-bst4sizj

«One option is to skip the weight update when an overflow is detected and simply move on to the next iteration» — с. 5 [↗ PDF](zotero://open-pdf/library/items/ZTUCFXQB?page=5&annotation=639P5U9T)
Используется в коде: [[Код rc_cdae.py#^c-5bb8ec|rc_cdae.py:33]], [[Код svdd.py#^c-e28f19|svdd.py:85]] ^aephriqm-5-639p5u9t

«mostly come up in batch-normalization layers when accumulating statistics and softmax layers» — с. 5 [↗ PDF](zotero://open-pdf/library/items/ZTUCFXQB?page=5&annotation=8UKJ2T95)
Используется в коде: [[Код svdd.py#^c-cd56e1|svdd.py:96]] ^aephriqm-5-8ukj2t95
