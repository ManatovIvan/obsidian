# Deep One-Class Classification

Источник Zotero: [P5CB3PIF](zotero://select/library/items/P5CB3PIF). Цитаты — выделения в PDF. Сгенерировано — не править вручную.

«One-Class Deep SVDD simply employs a quadratic loss for penalizing the distance of every network representation» — с. 4 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=4&annotation=WFCC8525)
Используется в коде: [[Код svdd.py#^c-a14c89|svdd.py:112]] ^p5cb3pif-4-wfcc8525

«consistent with the assumption that the majority of training data is from one class» — с. 4 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=4&annotation=CVCMFQQM)
Используется в коде: [[Код svdd.py#^c-a14c89|svdd.py:112]] ^p5cb3pif-4-cvcmfqqm

«we can naturally define an anomaly score s for both variants of Deep SVDD by the distance of the point to the center of the hypersphere» — с. 4 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=4&annotation=FI3BATZX)
Используется в коде: [[Код svdd.py#^c-55acbb|svdd.py:117]] ^p5cb3pif-4-fi3batzx

«autoencoders with bias terms are vulnerable to converge to a constant mapping onto the mean» — с. 5 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=5&annotation=AF5B6EEB)
Пока не используется ^p5cb3pif-5-af5b6eeb

«For Deep SVDD, we remove the bias terms in all network units to prevent a hypersphere collapse» — с. 6 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=6&annotation=S2N3WVEA)
Используется в коде: [[Код svdd.py#^c-771261|svdd.py:22]], [[Код svdd.py#^c-0d2df9|svdd.py:26]], [[Код svdd.py#^c-7942ea|svdd.py:29]], [[Код svdd.py#^c-0ed07a|svdd.py:36]] ^p5cb3pif-6-s2n3wvea

«For the DCAE encoder, we employ the same network architectures as we use for Deep SVDD» — с. 6 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=6&annotation=PVSBEKRS)
Используется в коде: [[Код svdd.py#^c-a2e79f|svdd.py:46]] ^p5cb3pif-6-pvsbekrs

«The decoder is then created symmetrically, where we substitute max-pooling with upsampling» — с. 6 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=6&annotation=ZGZ9TZ97)
Используется в коде: [[Код svdd.py#^c-a2e79f|svdd.py:46]] ^p5cb3pif-6-zgz9tz97

«We train the DCAE using the MSE loss» — с. 6 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=6&annotation=4HSP4AZC)
Используется в коде: [[Код svdd.py#^c-8a83c0|svdd.py:109]] ^p5cb3pif-6-4hsp4azc

«evaluate the results quantitatively via AUC measure by using the ground truth labels in testing» — с. 6 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=6&annotation=CU58WF6V)
Пока не используется ^p5cb3pif-6-cu58wf6v

«Average AUCs in % with StdDevs (over 10 seeds) per method» — с. 7 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=7&annotation=76I8V7TC)
Используется в коде: [[Код experiments_v4.py#^c-9a7943|experiments_v4.py:27]], [[Код svdd.py#^c-52eac9|svdd.py:102]] ^p5cb3pif-7-76i8v7tc

«For DCAE we train 250 + 100 epochs» — с. 7 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=7&annotation=5ATZMSUH)
Используется в коде: [[Код svdd.py#^c-56017c|svdd.py:16]] ^p5cb3pif-7-5atzmsuh

«For the competing deep AD methods we initialize network weights by uniform Glorot weights» — с. 7 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=7&annotation=J256BT9F)
Используется в коде: [[Код svdd.py#^c-030c57|svdd.py:63]] ^p5cb3pif-7-j256bt9f

«Leaky ReLU activations are used with leakiness α = 0.1» — с. 7 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=7&annotation=NHKX7JG2)
Используется в коде: [[Код svdd.py#^c-69ee71|svdd.py:17]] ^p5cb3pif-7-nhkx7jg2

«We use a batch size of 200 and set the weight decay hyperparameter to λ = 10−6» — с. 7 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=7&annotation=RMPBSXC4)
Используется в коде: [[Код svdd.py#^c-c04dca|svdd.py:12]], [[Код svdd.py#^c-96be67|svdd.py:13]], [[Код svdd.py#^c-b3d84a|svdd.py:71]] ^p5cb3pif-7-rmpbsxc4

«for Deep SVDD 150 + 100» — с. 7 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=7&annotation=55NA95MG)
Используется в коде: [[Код svdd.py#^c-e2e18e|svdd.py:15]] ^p5cb3pif-7-55na95mg

«for Deep SVDD use the weights from the trained DCAE encoder for initialization» — с. 7 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=7&annotation=6I5WQDA2)
Используется в коде: [[Код svdd.py#^c-11b665|svdd.py:100]] ^p5cb3pif-7-6i5wqda2

«we set the hypersphere center c to the mean of the mapped data after performing an initial forward pass» — с. 7 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=7&annotation=392FRH9K)
Используется в коде: [[Код svdd.py#^c-20026f|svdd.py:110]] ^p5cb3pif-7-392frh9k

«we use the Adam optimizer (Kingma & Ba, 2014) with parameters as recommended in the original work and apply Batch Normalization» — с. 7 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=7&annotation=NQ639BR3)
Используется в коде: [[Код svdd.py#^c-771261|svdd.py:22]], [[Код svdd.py#^c-0d2df9|svdd.py:26]], [[Код svdd.py#^c-b3d84a|svdd.py:71]] ^p5cb3pif-7-nq639br3

«with initial learning rate η = 10−4 , and subsequently η = 10−5» — с. 7 [↗ PDF](zotero://open-pdf/library/items/AV2IMUSR?page=7&annotation=FEWQENTH)
Используется в коде: [[Код svdd.py#^c-f901c2|svdd.py:14]], [[Код svdd.py#^c-1e13a6|svdd.py:73]] ^p5cb3pif-7-fewqenth
