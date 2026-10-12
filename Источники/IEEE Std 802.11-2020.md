# IEEE Std 802.11-2020

Источник Zotero: [JHCW99MH](zotero://select/library/items/JHCW99MH). Сгенерировано из выделений Zotero — не править вручную.

«1) the station (STA) is not transmitting a frame, and 2) the station (STA) is not receiving a frame» — с. 159 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=159&annotation=ZTF6HT9V)
Используется в коде: [[Код dataset_build.py#^l84|dataset_build.py:84]] ^ieee-159-ztf6ht9v

«The Individual/Group bit is always transferred first and is bit 0 of the MAC address.» — с. 755 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=755&annotation=DBFSW9H6)
Используется в коде: [[Код dsss_rx.py#^l88|dsss_rx.py:88]] ^ieee-755-dbfsw9h6

«The octets in fields longer than a single octet are sent to the PHY in order from the octet containing the lowest numbered bits» — с. 755 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=755&annotation=WIXUI67W)
Используется в коде: [[Код mac80211.py#^l49|mac80211.py:49]] ^ieee-755-wixui67w

«the first three fields (Frame Control, Duration/ID, and Address 1) and the last field (FCS)» — с. 757 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=757&annotation=CMKMPVIJ)
Используется в коде: [[Код mac80211.py#^l35|mac80211.py:35]] ^ieee-757-cmkmpvij

«00 Management 0101 Probe Response» — с. 759 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=759&annotation=3RG4CQCI)
Используется в коде: [[Код mac80211.py#^l12|mac80211.py:12]] ^ieee-759-3rg4cqci

«the value of the protocol version is 0 for MAC frames as described in 9.2» — с. 759 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=759&annotation=FVF3QGS4)
Используется в коде: [[Код mac80211.py#^l50|mac80211.py:50]] ^ieee-759-fvf3qgs4

«00 Management 1000 Beacon» — с. 760 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=760&annotation=MN85IHE4)
Используется в коде: [[Код mac80211.py#^l13|mac80211.py:13]] ^ieee-760-mn85ihe4

«00 Management 1010 Disassociation» — с. 760 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=760&annotation=GCQSG7R6)
Используется в коде: [[Код mac80211.py#^l15|mac80211.py:15]] ^ieee-760-gcqsg7r6

«00 Management 1100 Deauthentication» — с. 760 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=760&annotation=U47KT73S)
Используется в коде: [[Код mac80211.py#^l17|mac80211.py:17]] ^ieee-760-u47kt73s

«01 Control 1101 Ack» — с. 760 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=760&annotation=UIQGH6N3)
Используется в коде: [[Код mac80211.py#^l21|mac80211.py:21]] ^ieee-760-uiqgh6n3

«10 Data 0000 Data» — с. 760 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=760&annotation=HT9AI99M)
Используется в коде: [[Код mac80211.py#^l23|mac80211.py:23]] ^ieee-760-ht9ai99m

«10 Data 1000 QoS Data» — с. 760 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=760&annotation=KAKAQRH4)
Используется в коде: [[Код mac80211.py#^l25|mac80211.py:25]] ^ieee-760-kakaqrh4

«The TA field contains an IEEE MAC address that identifies the STA that has transmitted, onto the WM, the» — с. 770 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=770&annotation=PWPQ93ZI)
Используется в коде: [[Код mac80211.py#^l44|mac80211.py:44]] ^ieee-770-pwpq93zi

«As a typical implementation, at the transmitter, the initial remainder of the division is preset to all 1s» — с. 790 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=790&annotation=XN6DV3DQ)
Используется в коде: [[Код test_dsss_rx.py#^l60|test_dsss_rx.py:60]] ^ieee-790-xn6dv3dq

«G(x) = x32 + x26 + x23 + x22 + x16 + x12 + x11 + x10 + x8 + x7 + x5 + x4 + x2 + x + 1» — с. 790 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=790&annotation=TVP5HC7R)
Используется в коде: [[Код dsss_rx.py#^l114|dsss_rx.py:114]] ^ieee-790-tvp5hc7r

«The 1s complement of this remainder is transmitted, with the highest-order bit first, as the FCS field.» — с. 790 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=790&annotation=VF5GBZ3V)
Используется в коде: [[Код test_dsss_rx.py#^l61|test_dsss_rx.py:61]] ^ieee-790-vf5gbz3v

«At the receiver, the initial remainder is preset to all 1s and the serial incoming bits of the calculation fields and FCS, when divided by G(x), results (in the absence of transmission errors) in a unique nonzero remainder value.» — с. 791 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=791&annotation=98BMEW6D)
Используется в коде: [[Код dsss_rx.py#^l119|dsss_rx.py:119]] ^ieee-791-98bmew6d

«x31 + x30 + x26 + x25 + x24 + x18 + x15 + x14 + x12 + x11 + x10 + x8 + x6 + x5 + x4 + x3 + x + 1» — с. 791 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=791&annotation=XHACTGJA)
Используется в коде: [[Код dsss_rx.py#^l115|dsss_rx.py:115]] ^ieee-791-xhactgja

«Frame Control Duration RA TA FCS» — с. 797 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=797&annotation=DJPD4AQT)
Используется в коде: [[Код mac80211.py#^l32|mac80211.py:32]] ^ieee-797-djpd4aqt

«Frame Control Duration RA BSSID(TA) FCS» — с. 799 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=799&annotation=3EF2PBNS)
Используется в коде: [[Код mac80211.py#^l33|mac80211.py:33]] ^ieee-799-3ef2pbns

«Frame Control Duration/ID BSSID(RA) TA FCS» — с. 799 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=799&annotation=P9XAZBVC)
Используется в коде: [[Код mac80211.py#^l31|mac80211.py:31]] ^ieee-799-p9xazbvc

«Frame Control Duration RA TA BAR Control BAR Information FCS» — с. 800 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=800&annotation=VQX2NAGT)
Используется в коде: [[Код mac80211.py#^l29|mac80211.py:29]] ^ieee-800-vqx2nagt

«Frame Control Duration RA TA BA Control BA Information FCS» — с. 803 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=803&annotation=XN3NNQZT)
Используется в коде: [[Код mac80211.py#^l30|mac80211.py:30]] ^ieee-803-xn3nnqzt

«The Frame Control, Duration, Address 1, Address 2, Address 3, and Sequence Control fields are present in all data frame subtypes.» — с. 815 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=815&annotation=9GB7PRNZ)
Используется в коде: [[Код mac80211.py#^l67|mac80211.py:67]] ^ieee-815-9gb7prnz

«2 2 6 6 6 2 0 or 4 variable 4» — с. 822 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=822&annotation=C7WG4TW2)
Используется в коде: [[Код mac80211.py#^l59|mac80211.py:59]] ^ieee-822-c7wg4tw2

«The Address 2 field of the Management frame is the TA (=SA)» — с. 823 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=823&annotation=Q22PBCN2)
Используется в коде: [[Код mac80211.py#^l66|mac80211.py:66]] ^ieee-823-q22pbcn2

«1 Timestamp See 9.4.1.10 for Timestamp field format.» — с. 825 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=825&annotation=99GCDHZS)
Используется в коде: [[Код mac80211.py#^l73|mac80211.py:73]] ^ieee-825-99gcdhzs

«1 Reason code» — с. 856 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=856&annotation=ICAX6BIW)
Используется в коде: [[Код mac80211.py#^l84|mac80211.py:84]] ^ieee-856-icax6biw

«The length of the Beacon Interval field is 2 octets.» — с. 868 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=868&annotation=K8KI3P2S)
Используется в коде: [[Код mac80211.py#^l75|mac80211.py:75]] ^ieee-868-k8ki3p2s

«The length of the Capability Information field is 2 octets.» — с. 868 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=868&annotation=6R6DQ9BX)
Используется в коде: [[Код mac80211.py#^l76|mac80211.py:76]] ^ieee-868-6r6dq9bx

«The length of the Reason Code field is 2 octets.» — с. 871 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=871&annotation=HPTQGZWK)
Используется в коде: [[Код mac80211.py#^l85|mac80211.py:85]] ^ieee-871-hptqgzwk

«The length of the Timestamp field is 8 octets.» — с. 880 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=880&annotation=KXG73QVA)
Используется в коде: [[Код dataset_build.py#^l41|dataset_build.py:41]], [[Код mac80211.py#^l74|mac80211.py:74]] ^ieee-880-kxg73qva

«This field represents the timing synchronization function (TSF) timer» — с. 880 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=880&annotation=TFTUIR4K)
Используется в коде: [[Код analyze_labels.py#^l57|analyze_labels.py:57]] ^ieee-880-tftuir4k

«Octets: 1 1 0 or 1 variable» — с. 940 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=940&annotation=BMMMZIMV)
Используется в коде: [[Код mac80211.py#^l78|mac80211.py:78]] ^ieee-940-bmmmzimv

«SSID (see 9.4.2.2) 0» — с. 940 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=940&annotation=7JK274NE)
Используется в коде: [[Код mac80211.py#^l79|mac80211.py:79]] ^ieee-940-7jk274ne

«Each STA shall maintain a TSF timer with modulus 264 counting in increments of microseconds.» — с. 2050 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2050&annotation=R8ZI8MQN)
Используется в коде: [[Код analyze_labels.py#^l58|analyze_labels.py:58]] ^ieee-2050-r8zi8mqn

«counting in increments of microseconds» — с. 2050 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2050&annotation=HK6ZVA2U)
Используется в коде: [[Код dataset_build.py#^l43|dataset_build.py:43]] ^ieee-2050-hk6zva2u

«128 bits 16 bits 8 bits 8 bits 16 bits 16 bits» — с. 2753 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2753&annotation=HSEIQBKD)
Используется в коде: [[Код dsss_rx.py#^l63|dsss_rx.py:63]] ^ieee-2753-hseiqbkd

«144 bits 48 bits» — с. 2753 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2753&annotation=ZI4CBS7A)
Используется в коде: [[Код dsss_rx.py#^l62|dsss_rx.py:62]] ^ieee-2753-zi4cbs7a

«All transmitted bits shall be scrambled using the feedthrough scrambler» — с. 2754 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2754&annotation=NBTRXDQT)
Используется в коде: [[Код dsss_rx.py#^l235|dsss_rx.py:235]] ^ieee-2754-nbtrxdqt

«The LSB shall be transmitted first in time.» — с. 2754 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2754&annotation=JCUV8G7P)
Используется в коде: [[Код dsss_rx.py#^l87|dsss_rx.py:87]] ^ieee-2754-jcuv8g7p

«The SFD shall be X'F3A0' (MSB to LSB). The LSB shall be transmitted first in time.» — с. 2754 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2754&annotation=QJB4E5NM)
Используется в коде: [[Код dsss_rx.py#^l49|dsss_rx.py:49]] ^ieee-2754-qjb4e5nm

«The entire PHY preamble and header shall be transmitted using the 1 Mb/s DBPSK modulation» — с. 2754 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2754&annotation=Z9Z2QHCE)
Используется в коде: [[Код dsss_rx.py#^l25|dsss_rx.py:25]] ^ieee-2754-z9z2qhce

«X'0A' (MSB to LSB) for 1 Mb/s DBPSK» — с. 2754 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2754&annotation=VVEITADP)
Используется в коде: [[Код dsss_rx.py#^l58|dsss_rx.py:58]] ^ieee-2754-vveitadp

«X'14' (MSB to LSB) for 2 Mb/s DQPSK» — с. 2754 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2754&annotation=UBI9G2X5)
Используется в коде: [[Код dsss_rx.py#^l58|dsss_rx.py:58]] ^ieee-2754-ubi9g2x5

«indicates the number of microseconds required to transmit the MPDU» — с. 2754 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2754&annotation=SBUU78GZ)
Используется в коде: [[Код dsss_rx.py#^l249|dsss_rx.py:249]] ^ieee-2754-sbuu78gz

«0101 0000 0000 0000 0000 0011 0000 0000 (leftmost bit transmitted first in time)» — с. 2755 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2755&annotation=QV8B3JC7)
Используется в коде: [[Код test_dsss_rx.py#^l52|test_dsss_rx.py:52]] ^ieee-2755-qv8b3jc7

«0101 1011 0101 0111 (leftmost bit transmitted first in time)» — с. 2755 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2755&annotation=UR6V43DN)
Используется в коде: [[Код test_dsss_rx.py#^l53|test_dsss_rx.py:53]] ^ieee-2755-ur6v43dn

«The CRC-16 FCS shall be the 1s complement of the remainder» — с. 2755 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2755&annotation=A2SWPBHI)
Используется в коде: [[Код dsss_rx.py#^l98|dsss_rx.py:98]] ^ieee-2755-a2swpbhi

«The protected bits shall be processed in transmit order.» — с. 2755 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2755&annotation=V8GWQC6F)
Используется в коде: [[Код dsss_rx.py#^l99|dsss_rx.py:99]] ^ieee-2755-v8gwqc6f

«x16 + x12 + x5 + 1» — с. 2755 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2755&annotation=VGQES27R)
Используется в коде: [[Код dsss_rx.py#^l99|dsss_rx.py:99]] ^ieee-2755-vgqes27r

«1111111111111111 ; initialize preset to 1s» — с. 2756 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2756&annotation=RFPQ56CC)
Используется в коде: [[Код dsss_rx.py#^l100|dsss_rx.py:100]] ^ieee-2756-rfpq56cc

«The polynomial G(z) = z –7 + z –4 + 1 shall be used to scramble all bits transmitted by the DSSS PHY.» — с. 2756 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2756&annotation=2MAUMI5G)
Используется в коде: [[Код dsss_rx.py#^l75|dsss_rx.py:75]] ^ieee-2756-2maumi5g

«which requires no prior knowledge of the transmitter initialization of the scrambler for receive processing» — с. 2756 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2756&annotation=IG7ZCSJ6)
Используется в коде: [[Код dsss_rx.py#^l76|dsss_rx.py:76]] ^ieee-2756-ig7zcsj6

«by multiplying by 8 for 1 Mb/s and by 4 for 2 Mb/s» — с. 2757 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2757&annotation=ISFPZ4BJ)
Используется в коде: [[Код dsss_rx.py#^l60|dsss_rx.py:60]] ^ieee-2757-isfpz4bj

«starting with the first symbol (1 bit for DBPSK or 2 bits for DQPSK) of the MPDU» — с. 2757 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2757&annotation=VXFXUE4X)
Используется в коде: [[Код dsss_rx.py#^l256|dsss_rx.py:256]] ^ieee-2757-vxfxue4x

«aPSDUMaxLength 212 – 1 octets» — с. 2764 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2764&annotation=QSII4CD5)
Используется в коде: [[Код dsss_rx.py#^l61|dsss_rx.py:61]] ^ieee-2764-qsii4cd5

«+1, –1, +1, +1, –1, +1, +1, +1, –1, –1, –1» — с. 2765 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2765&annotation=NSHCWN8J)
Используется в коде: [[Код dsss_rx.py#^l31|dsss_rx.py:31]] ^ieee-2765-nshcwn8j

«The first chip shall be aligned at the start of a transmitted symbol.» — с. 2765 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2765&annotation=WP8JE8FB)
Используется в коде: [[Код dsss_rx.py#^l37|dsss_rx.py:37]] ^ieee-2765-wp8je8fb

«The leftmost chip shall be output first in time.» — с. 2765 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2765&annotation=A63D3EJ6)
Используется в коде: [[Код dsss_rx.py#^l32|dsss_rx.py:32]] ^ieee-2765-a63d3ej6

«The symbol duration shall be exactly 11 chips long.» — с. 2765 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2765&annotation=7RNVVTJ7)
Используется в коде: [[Код dsss_rx.py#^l29|dsss_rx.py:29]] ^ieee-2765-7rnvvtj7

«d0 is first in time» — с. 2765 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2765&annotation=DBW8P5KZ)
Используется в коде: [[Код dsss_rx.py#^l191|dsss_rx.py:191]] ^ieee-2765-dbw8p5kz

«shall be defined as counterclockwise rotation.» — с. 2765 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2765&annotation=9UFUCZKB)
Используется в коде: [[Код dsss_rx.py#^l190|dsss_rx.py:190]] ^ieee-2765-9ufuczkb

«The PN code chip clock frequency tolerance shall be ± 25 ppm.» — с. 2767 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2767&annotation=HJ8EJ2HT)
Используется в коде: [[Код dsss_rx.py#^l69|dsss_rx.py:69]] ^ieee-2767-hj8ej2ht

«The transmitted spectral products shall be less than –30 dBr» — с. 2767 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2767&annotation=S9K5B7F7)
Используется в коде: [[Код dsss_rx.py#^l39|dsss_rx.py:39]] ^ieee-2767-s9k5b7f7

«48 bits at 2 Mb/s» — с. 2775 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2775&annotation=A6XMJEIR)
Используется в коде: [[Код dsss_rx.py#^l62|dsss_rx.py:62]] ^ieee-2775-a6xmjeir

«56 bits 16 bits» — с. 2775 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2775&annotation=9TVX3IEJ)
Используется в коде: [[Код dsss_rx.py#^l63|dsss_rx.py:63]] ^ieee-2775-9tvx3iej

«The short PHY header uses the 2 Mb/s Barker code spreading with DQPSK modulation» — с. 2775 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2775&annotation=8ZWMJ3EJ)
Используется в коде: [[Код dsss_rx.py#^l234|dsss_rx.py:234]] ^ieee-2775-8zwmj3ej

«The field is the bit pattern 0000 0101 1100 1111. The right end bit shall be transmitted first in time.» — с. 2780 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2780&annotation=3R7ZSIQP)
Используется в коде: [[Код dsss_rx.py#^l50|dsss_rx.py:50]] ^ieee-2780-3r7zsiqp

«The CRC in the short header shall be the same as the CRC field defined in 16.2.3.7.» — с. 2781 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2781&annotation=FBAZSQKJ)
Используется в коде: [[Код dsss_rx.py#^l240|dsss_rx.py:240]] ^ieee-2781-fbazsqkj

«X'37' (MSB to LSB) for 5.5 Mb/s» — с. 2781 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2781&annotation=HJ8QTN6T)
Используется в коде: [[Код dsss_rx.py#^l59|dsss_rx.py:59]] ^ieee-2781-hj8qtn6t

«X'6E' (MSB to LSB) for 11 Mb/s» — с. 2781 [↗ PDF](zotero://open-pdf/library/items/VI9FAQRH?page=2781&annotation=RHGZ5D2C)
Используется в коде: [[Код dsss_rx.py#^l59|dsss_rx.py:59]] ^ieee-2781-rhgz5d2c
