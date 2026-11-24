# Klasik Görüntü İşleme Yöntemleriyle Mikroskobik Görüntülerde Hücre Segmentasyonu, Sayımı ve Zamansal Analizi

<p style="text-align:center">Görüntü İşleme – Yüksek Lisans Ara Sınav Projesi<br>
Mesut İşsever – 264329015<br>
GitHub: https://github.com/mesutissever/sgi-midterm-cell-counting</p>

## 1. Giriş ve Problem Tanımı
Floresan mikroskop görüntülerinde hücrelerin bölütlenmesi (segmentasyon) ve sayılması, hücre
çoğalması, ilaç yanıtı ve hücre döngüsü gibi biyolojik süreçlerin nicel olarak izlenmesinde temel
bir adımdır. Elle sayım zaman alıcı ve kişiden kişiye değişen bir işlemdir; bu nedenle otomatik
sayım yöntemlerine ihtiyaç duyulur. Problem göründüğünden zordur: hücrelerin parlaklığı aynı
görüntüde bile onlarca kat değişebilir, odak düzleminden uzaklaşan hücreler arka plana yakın
soluklukta görünür, birbirine değen hücreler tek nesne gibi algılanabilir ve gürültü lekeleri
hücre gibi sayılabilir. Görüntüler bir video dizisinin kareleri olduğundan sayımın zaman içinde
tutarlı olması da beklenir.

Bu çalışmanın amacı, Cell Tracking Challenge veri setlerinden Fluo-N2DH-GOWT1 üzerinde, derin
öğrenme kullanmadan, yalnızca klasik görüntü işleme yöntemleriyle hücreleri bölütleyen ve sayan
bir pipeline geliştirmek, bu pipeline'ı bilimsel olarak gerekçelendirilmiş bir deney düzeniyle
seçmek ve bağımsız bir test kümesinde değerlendirmektir.

Kullanılan yöntemler beş grupta toplanmıştır. **Yoğunluk dönüşümleri** (contrast stretching,
gamma, logaritmik dönüşüm, histogram eşitleme, CLAHE, histogram eşleme) kontrastı değiştirir.
**Uzamsal filtreler** (Gaussian, median, Laplacian keskinleştirme, unsharp masking) gürültüyü
azaltır ya da kenarları vurgular. **Eşikleme** yöntemleri (sabit eşik, Otsu, Triangle, Li, adaptif
ve blok bazlı eşik) hücreleri arka plandan ayırır. **Morfolojik işlemler** (erosion, dilation,
opening, closing, delik doldurma) ikili maskeyi düzeltir. **Distance transform ile watershed** ise
birbirine değen hücreleri ayırır. Sayım, bağlı bileşen analiziyle yapılır.

Çalışmanın kapsamı kare bazında hücre sayımı ve ardışık karelerdeki sayım değişimlerinin
analizidir; bireysel hücre takibi kapsam dışıdır. Deney düzeni üç bağımsız veri grubuna
dayanır. Yöntemler 10 geliştirme karesinde karşılaştırılmış, aday pipeline'lar 5 doğrulama
karesinde seçilmiştir. Seçilen yöntem, referansları kilitli tutulan farklı bir sekansın 15 test
karesinde yalnızca bir kez değerlendirilmiştir.

## 2. Veri Seti ve Deneysel Tasarım

### 2.1 Veri setinin özellikleri
Fluo-N2DH-GOWT1, GFP ile işaretlenmiş fare kök hücre çekirdeklerinin floresan mikroskop
görüntülerinden oluşur [1–4]. İki sekansın (01, 02) her biri 92 kare, 1024×1024 piksel ve 8 bit gri
seviyelidir. Referans olarak `TRA` (bütün hücreler için takip etiketleri) ve `SEG` (seçili karelerde
manuel maskeler) kullanılmıştır. Test sızıntısını önlemek için Sekans 02'de yalnızca kare sayısı, boyut
ve veri tipi kontrol edilmiştir. Sekans 01 analizinin bulguları şunlardır: Piksellerin %50'si tam 0,
%99'u ≤ 68'dir. Histogram tek bir arka plan tepesi ve uzun bir kuyruktan oluşur, iki tepeli değildir.
Hücre çapı ≈ 60 piksel, alanı 579–5306 pikseldir (ortanca 2847). Hücre parlaklığı 3,4 ile 96 arasında
yaklaşık 28 kat değişir ve hücrelerin %37'si arka planla aynı düzeyde soluktur. Başlıca güçlükler
dörttür: karelerin %57'sinde birbirine değen hücreler; odak dışındaki soluk hücreler; kare başına
ortalama 2 küçük gürültü lekesi (< 100 piksel); bölgeden bölgeye değişen arka plan parlaklığı. Arka
plan farkının soluk hücrelerin parlaklığıyla aynı mertebede olması, yerel eşiklemeyi gerekçelendirir.

### 2.2 Rastgele seçim yöntemi, seed ve seçilen görüntüler
Seed = 29015 + 2026 = **31041**. Seçim, `numpy.random.default_rng(31041)` ile tek bir rastgele sayı
akışından yapılmıştır. Önce Sekans 01'den 5 doğrulama karesi seçilmiş, ardından bunlara en az 5 kare
uzaklıktaki karelerden 10 geliştirme karesi alınmıştır. Sekans 02'de 3 başlangıç noktasından birer
ardışık 5 karelik blok seçilmiştir; bloklar arasında en az 5 kare boşluk koşulu sağlanmayan
çekilişler aynı akıştan yenilenmiştir. Seçim, algoritma geliştirmeden önce yapılmış ve ayrı bir
commit ile `selected_frames.csv` dosyasına kaydedilmiştir (Tablo 1). Geliştirme karelerinin yarısının
(41, 43–46) zamanca yakın olması rastgele seçimin bir sonucudur ve kümenin çeşitliliğini sınırlar.

**Tablo 1.** Rastgele seçilen görüntüler ve veri grupları.

| Veri grubu | Sekans | Kareler | Sayı | Referans sayısı |
|---|---|---|---|---|
| Geliştirme | 01 | 14, 41, 43, 44, 45, 46, 62, 67, 72, 85 | 10 | 21–24 (ort. 22,7) |
| Doğrulama | 01 | 4, 20, 32, 53, 78 | 5 | 20–24 (ort. 22,2) |
| Test – Blok 1 / 2 / 3 | 02 | 42–46 / 62–66 / 86–90 | 15 | kilitli (Bölüm 6) |

### 2.3 Referans hücre sayıları ve veri gruplarının kullanımı
Referans sayı, karenin `TRA` etiket görüntüsündeki sıfır dışı farklı hücre kimliklerinin sayısıdır.
Görsel kontrolde iki durum belirlenmiştir. Görüntü kenarında kesilmiş bazı hücreler referansta
etiketsizdir; bunları bulan bir yöntem fazla sayım yapmış görünür. Küçük ve yoğun görünen bazı
etiketli nesneler ise `man_track.txt` soy bilgisine göre artefakt değil, yeni bölünmüş yavru
hücrelerdir. Referanslar değiştirilmeden kullanılmıştır. Veri grupları şöyle kullanılmıştır:

- **Geliştirme:** Bütün yöntem karşılaştırmaları ve parametre taramaları yalnızca bu kümede yapılmıştır (E01–E43).
- **Doğrulama:** Hiçbir parametre seçiminde kullanılmamış; yalnızca adayları önceden belirlenmiş bir kuralla karşılaştırmak için bir kez kullanılmıştır.
- **Test:** Yöntem kilitlenene kadar görüntüler incelenmemiş, referanslar kod içindeki bir kilitle okunamaz tutulmuştur. Tahminler referanslar açılmadan ayrı bir commit ile kaydedilmiştir. Farklı bir sekanstan gelmesi, genellemeyi ölçer.

## 3. Yöntem ve Pipeline Tasarımı

### 3.1 Baseline yöntemi
Baseline, gri seviyeli görüntüye global Otsu eşikleme [5], ardından 8-komşuluk bağlı bileşen analizi
uygular. Minimum nesne alanı geliştirme kümesinde 0–500 piksel aralığında taranmış, 50 piksel
seçilmiştir (MAE 3,4; RMSE 5,04; Bias −3,0; ≈ 10 ms). Otsu eşiği (30–35) soluk hücrelerin parlaklığının
çok üstündedir. Bu yüzden soluk hücreler ya kaçırılmakta ya da çok sayıda parçaya bölünmektedir.
Bazı karelerde bu iki hata birbirini götürür; örneğin t067'de sayım doğru olduğu halde segmentasyon
hatalıdır. Bu nedenle bütün deneylerde sayım metriklerinin yanında tanı ölçütleri ve görsel kontrol
kullanılmıştır.

### 3.2 Uygulanan yöntemler, alternatifler ve aday pipeline'lar
Pipeline aşamaları sırasıyla incelenmiş, her aşamada önceki aşamaların en iyi seçimi sabit tutulmuştur:

- **Yoğunluk dönüşümü:** contrast stretching, gamma (0,5), logaritmik, histogram eşitleme, CLAHE [8], histogram eşleme.
- **Filtre:** Gaussian (3, 5, 9, 15), median (3, 5, 9), Laplacian keskinleştirme, unsharp masking.
- **Segmentasyon:** global Otsu [5], sabit eşik, Triangle [7], Li [6], adaptif Gaussian, blok bazlı Otsu (4×4, 8×8, taban eşikli).
- **Morfoloji:** erosion, dilation, opening, closing (5×5 ve 11×11 elips), iki işlem sırası, delik doldurma, distance transform + watershed [9, 10].
- **Son işlem:** 8-komşuluk bağlı bileşen ve minimum alan filtresi.

Her pipeline tek bir yapılandırma sözlüğüyle tanımlanmıştır. Dört aday pipeline aynı son işlem
zincirini paylaşır: closing 11×11 → delik doldurma → watershed (`min_distance` = 25) → min. alan 100.
Adaylar yalnızca filtre ve eşik yöntemiyle ayrılır; doğrulamadaki fark doğrudan bu seçime
bağlanabilir.

| Pipeline | Deney | Filtre | Eşik | Gerekçe | Geliştirme MAE |
|---|---|---|---|---|---|
| A | E36 | Gaussian 15×15 | Blok Otsu 8×8, taban 4 | Geliştirme kümesinde en iyi | 0,5 |
| B | E40 | Median 9×9 | Blok Otsu 8×8, taban 4 | Farklı filtre türü | 0,6 |
| C | E41 | Gaussian 15×15 | Li (global) | Parametresiz eşik | 3,1 |
| D | E42 | Gaussian 15×15 | Sabit eşik T = 7 | En basit eşik | 1,7 |

### 3.3 Nihai pipeline
Doğrulama sonucunda (Bölüm 5) **Pipeline D** seçilmiştir (Şekil 1). Gaussian filtre çekirdekçik ve
gürültü dokusunu yumuşatarak hücrelerin parçalanmasını önler. Sabit eşik hücreleri arka plandan
ayırır. Closing ve delik doldurma maskedeki çatlak ve delikleri kapatır; bu, watershed'in her hücrede
tek bir tepe bulması için gereklidir (Bölüm 4.3). Watershed birbirine değen hücreleri ayırır, min. alan
filtresi ise gürültü lekelerini atar.

<img src="figures/sekil1_pipeline.png" style="width:100%">

**Şekil 1.** Nihai görüntü işleme pipeline'ının akış diyagramı (parametreler `final_pipeline.json` dosyasından).

## 4. Deneysel Karşılaştırmalar ve Parametre Analizi

Deneyler, pipeline aşamaları sırasıyla ve her seferinde tek bir değişiklik yapılarak geliştirme
kümesinde yürütülmüştür (toplam 43 deney, `experiments.csv`). Sayım hatalarının birbirini götürmesini
ayırt etmek için bulunan nesneler TRA etiketleriyle eşleştirilmiş; MAE'nin yanında kare başına
**kaçırılan**, **bölünen**, **birleşen** ve **fazla** nesne sayıları da hesaplanmıştır. Her aşamanın
ayrıntılı tabloları ve analizleri Ek B'de verilmiştir; Tablo 2 temsilî deneyleri özetler.

**Tablo 2.** En az 10 deneysel konfigürasyonun yöntem, parametre ve performans karşılaştırması (geliştirme kümesi; tamamı `experiments.csv` ve Ek B'de; G15: Gaussian 15×15, BO8: blok Otsu 8×8 taban 4, C+D+W: closing 11×11 → delik doldurma → watershed).

| Deney ID | Yoğunluk | Filtre | Segment. | Morfoloji | Param. | Count MAE |
|---|---|---|---|---|---|---|
| E01 (baseline) | Orijinal | – | Otsu | – | min. alan 50 | 3,4 |
| E02 | Contrast stretching | – | Otsu | – | [0, p99,5] | 2,4 |
| E11 | Orijinal | G15 | Otsu | – | – | 5,5 |
| E17 | Orijinal | G15 | Sabit eşik | – | T=7 | 1,7 |
| E19 | Orijinal | G15 | Li | – | otomatik | 1,8 |
| E20 | Orijinal | G15 | Adaptif Gauss. | – | blok 101, C=−10 | 1,5 |
| E21 | Orijinal | G15 | BO8 | – | – | 1,2 |
| E24 | Orijinal | G15 | BO8 | Erosion 11×11 | – | 0,8 |
| E30 | Orijinal | G15 | BO8 | Closing 11×11 | – | 0,9 |
| E35 | Orijinal | G15 | BO8 | C+D+W | md=25, min. alan 50 | 0,7 |
| **E36 (A)** | Orijinal | G15 | BO8 | C+D+W | md=25, min. alan 100 | **0,5** |
| E37 | CLAHE | G15 | BO8 | C+D+W | E36 ile aynı | 45,7 |
| E40 (B) | Orijinal | Median 9×9 | BO8 | C+D+W | E36 ile aynı | 0,6 |
| E41 (C) | Orijinal | G15 | Li | C+D+W | E36 ile aynı | 3,1 |
| **E42 (D)** | Orijinal | G15 | Sabit eşik | C+D+W | T=7, E36 ile aynı | 1,7 |
| E43 | Orijinal | G15 | BO8 interpolasyonlu | C+D+W | E36 ile aynı | 1,1 |

### 4.1 Yoğunluk dönüşümleri ve filtreler

Baseline ile birlikte en iyi yoğunluk dönüşümü contrast stretching olmuştur (MAE 3,4 → 2,4).
Görsel olarak hücreleri en belirgin gösteren gamma, logaritmik dönüşüm ve histogram eşitleme ise
sayımı en çok bozmuştur (MAE 41–460). Piksellerin yarısı tam 0 olduğundan bu dönüşümler histogramı
0'daki dev bir tepe ve ondan uzak bir kuyruk biçimine getirmekte, Otsu eşiği 0'a düşmekte ve arka
plandaki bütün gürültü ön plan olmaktadır (Şekil 2, t014'te 411 nesne). Filtrelerde kernel boyutu
büyüdükçe gürültü bileşenleri 779'dan 24'e, soluk hücrelerin parçalanması (bölünme) 5,0'dan 2,3'e
düşmüş, ancak MAE artmıştır. Tanı ölçütleri nedeni göstermiştir: kaçırılan hücre sayısı filtreden
bağımsız olarak ≈ 8'dir. Filtresiz durumda bu eksiklik parçalanan hücrelerin fazla sayılmasıyla
gizlenmekte, yumuşatma bunu açığa çıkarmaktadır. **Darboğaz gürültü değil, tek bir global eşiğin
(≈ 30) soluk hücrelerin parlaklığının üstünde kalmasıdır.** Keskinleştirme filtreleri gürültüyü de
güçlendirdiğinden bu veri setine uygun değildir.

<img src="figures/sekil2_yogunluk.png" style="width:92%">

**Şekil 2.** t014 karesinde orijinal görüntü ve altı yoğunluk dönüşümü (üst), log ölçekli histogramlar
ve Otsu eşikleri (orta), Otsu maskeleri (alt). Gamma, log ve histogram eşitlemede Otsu eşiği 0'a düşmüştür.

### 4.2 Segmentasyon yöntemleri

Ortak ön işlem (Gaussian 15×15) sonrasında global Otsu dışındaki bütün yöntemler kaçırılan hücre
sayısını belirgin biçimde azaltmıştır. En iyi sonucu blok bazlı Otsu vermiştir (8×8 blok, taban eşik 4;
MAE 1,2; kaçırılan 7,9 → 0,8). Her blokta ayrı eşik hesaplandığından soluk hücre içeren bloklarda eşik
düşmektedir. Hücre içermeyen bloklarda Otsu gürültüyü ikiye böldüğü için taban eşik zorunludur ve
sonuç bu değere duyarlıdır (taban 4: 1,2; taban 5: 3,2). Adaptif eşik ikinci en düşük MAE'yi vermiş,
ancak hücreleri halka biçiminde parçalamıştır (6,5 kaçırılan, 5,6 bölünme; Şekil 3a). Bu düşük MAE
hataların birbirini götürmesinden kaynaklanmaktadır. Sabit eşik (T=7, MAE 1,7) ve Li (1,8) iyi
sonuç vermiş; Triangle ise neredeyse bütün hücreleri bulup gürültüyü de nesne saymıştır.

<img src="figures/sekil3a_segmentasyon.png" style="width:100%">

**Şekil 3a.** t014'te Gaussian 15×15 sonrası yedi segmentasyon yöntemi (renkli: bulunan nesneler,
kırmızı: referans hücreler). Global Otsu soluk hücreleri kaçırır, adaptif eşik hücreleri halka
biçiminde parçalar, blok Otsu soluk hücrelerin çoğunu bulur.

### 4.3 Morfolojik işlemler

Morfolojik işlemler en iyi maskeye (Gaussian 15×15 → Blok Otsu 8×8) iki boyutta (5×5 ve 11×11 elips)
uygulanmıştır. Büyük yapılandırma elemanı etkileri belirginleştirmiştir: closing bölünmeyi
sıfırlamış (0,7 → 0), dilation fazla nesneyi artırmış (1,4 → 3,3), opening ve erosion soluk hücreleri
silmiştir. Erosion 11×11'in düşük MAE'si (0,8) yine hataların birbirini götürmesinden
kaynaklanmaktadır (1,9 kaçırılan, 1,1 bölünme). İşlem sırası değişmeli değildir: opening → closing
MAE 1,2, closing → opening 1,7 vermiştir. Delik doldurma tek başına sayımı hiç değiştirmemiştir,
çünkü bir nesnenin içindeki delik bağlı bileşen sayısını etkilemez. Ancak watershed'den önce
gereklidir: delikli ve girintili ham maskede distance transform her hücrede birden fazla tepe
üretmekte ve watershed hücreleri bölmektedir (md=10'da MAE 13,1). Maske önce closing ve delik
doldurma ile düzeltildiğinde her hücrede tek tepe kalmış; `min_distance` = 25 ile bölünme ve
birleşme sıfıra inmiştir (E35, MAE 0,7; Şekil 3b–3c). Son olarak min. alan 100'e çıkarılmıştır
(E36, MAE 0,5); bu değer gürültü lekeleri (< 100 px) ile yeni bölünmüş yavru hücreler (250–400 px)
arasındadır.

<img src="figures/sekil3b_morfoloji.png" style="width:100%">

**Şekil 3b.** t041'de birbirine değen iki hücre bölgesinde 14 morfolojik işlemin çıktısı ve
geliştirme MAE değerleri.

<img src="figures/sekil3c_watershed.png" style="width:66%">

**Şekil 3c.** Watershed adımları: (1) Blok Otsu maskesi, (2) closing 11×11 + delik doldurma, (3)
distance transform ve tepeler, (4) watershed sonucu. Düzeltilmiş maskede iki hücre "kum saati"
biçimindedir ve boyun noktasından ayrılır.

### 4.4 Tam pipeline deneyleri

Aşamalar birleştirildikten sonra referans pipeline'dan (E36) tek değişiklik yapılarak yedi deney
daha yapılmıştır. CLAHE ve contrast stretching tam pipeline'a eklendiğinde MAE 45–51'e çıkmıştır.
Blok Otsu'nun taban eşiği orijinal parlaklık ölçeğinde seçilmiş mutlak bir değerdir; bu dönüşümler
arka plan gürültüsünü bu eşiğin üzerine taşımaktadır. **Pipeline parametreleri birbirinden bağımsız
değildir.** Gaussian 9×9 (MAE 1,2) 15×15'ten kötü, median 9×9 (0,6) ona yakın çıkmıştır. Blok
sınırlarındaki eşik sıçramalarını gidermek için denenen interpolasyonlu blok Otsu (E43) ise soluk
hücrelerin eşiğini komşu parlak blokların eşiğiyle karıştırdığı için sonucu kötüleştirmiştir
(MAE 1,1).

## 5. Doğrulama ve Nihai Yöntemin Belirlenmesi
Dört aday, geliştirme kümesinde seçilen parametreleri değiştirilmeden, hiçbir parametre seçiminde
kullanılmamış 5 doğrulama karesinde çalıştırılmıştır. Seçim kuralı, sonuçlar görülmeden önce dokümana
uygun olarak belirlenmiştir: en düşük doğrulama MAE'si; eşitlikte RMSE, işlem süresi ve karmaşıklık.

**Tablo 3.** Aday pipeline'ların doğrulama performansları (MAE, RMSE, süre); kaçırılan ve fazla: kare başına.

| Pipeline | Validation MAE | Validation RMSE | Ort. süre (ms) | Bias | Kaçırılan | Fazla | Geliştirme MAE | Seçim |
|---|---|---|---|---|---|---|---|---|
| A (G15 + Blok Otsu) | 1,6 | 1,79 | 217 | +0,8 | 1,6 | 2,2 | 0,5 | |
| B (Median 9 + Blok Otsu) | 1,2 | 1,41 | 271 | +0,8 | 1,4 | 1,6 | 0,6 | |
| C (G15 + Li) | 2,8 | 3,22 | 219 | −2,8 | 3,6 | 0,8 | 3,1 | |
| **D (G15 + sabit T=7)** | **0,8** | **1,10** | 212 | 0,0 | 2,0 | 1,6 | 1,7 | **Seçildi** |
| Baseline (Otsu) | 3,8 | 5,39 | 10 | −2,6 | 8,6 | 0,2 | 3,4 | – |

**Seçim gerekçesi.** Kural gereği en düşük doğrulama MAE'sine sahip **Pipeline D** seçilmiştir. D,
RMSE ve süre açısından da adayların en iyisidir; eşitlik kuralına gerek kalmamıştır. Bütün adaylar
baseline'ı geçmiştir. Geliştirmede en iyi olan A (0,5) doğrulamada üçüncü sıraya düşmüştür (1,6). A'nın
birçok parametresi geliştirme kümesinde ayarlanmıştır ve bu karelerin çoğu sekansın 41–46 aralığında
kümelenmiştir; blok Otsu'nun taban eşiğine duyarlılığı da yüksektir. Bu fark, geliştirme kümesine
aşırı uyumun (overfitting) göstergesidir ve bağımsız doğrulamanın neden gerekli olduğunu gösterir.
Sınırlılıklar da vardır. 5 karede D ile B arasındaki fark yalnızca 2 hücredir. D'nin sıfır Bias'ı ise
kısmen 2,0 kaçırılan ve 1,6 fazla nesnenin birbirini götürmesinden kaynaklanmaktadır. Seçim kuralının
sonuçlar görüldükten sonra değiştirilmesi seçim yanlılığı yaratacağından kurala bağlı kalınmıştır.
Doğrulama ayrıntıları Ek C.1'dedir.

**Seçilen bütün parametreler (`final_pipeline.json`).** Yoğunluk dönüşümü yok → Gaussian 15×15
(σ ≈ 2,6) → sabit eşik T = 7 → closing 11×11 elips → delik doldurma (`binary_fill_holes`) → distance
transform + watershed (`min_distance` = 25) → 8-komşuluk bağlı bileşen, min. alan 100 piksel.
Baseline: global Otsu + 8-komşuluk bağlı bileşen, min. alan 50 piksel. Bu parametreler
`final_pipeline.json` dosyasına yazılmıştır. Nihai pipeline ve baseline 15 test karesinde referanslar
okunmadan çalıştırılmış, tahminler `test_predictions_locked.csv` dosyasına kaydedilmiştir. İki dosya,
test referansları açılmadan önce **ayrı bir commit** ile GitHub'a yüklenmiştir. Notebook yeniden
çalıştırıldığında kilitli dosyaların üzerine yazılmaz; tahminlerin kilitli değerlerle aynı olduğu
doğrulanır.

<img src="figures/sekil4_asamalar.png" style="width:68%">

**Şekil 4.** Örnek bir görüntü (geliştirme karesi t041) üzerinde nihai pipeline'ın aşama aşama
çıktıları. Gaussian filtre dokuyu yumuşatır; T = 7 parlak hücreleri tam ayırır, ancak merkezdeki soluk
bir hücre noktalara dağılır. Closing ve doldurmadan sonra distance transform her hücrede tek tepe
üretir. Sonuç 22 hücredir (referans 24). Eksik sayımın nedeni, eşiğin altında kalan sağ alttaki iki
çok soluk hücredir (kırmızı konturlar).

## 6. Bağımsız Test Sonuçları
Test referansları, kilit commit'i (`e075cc3`, 23.11.2026 13:05) GitHub'a gönderildikten sonra ilk
kez okunmuştur. Değerlendirmede tahminler yeniden hesaplanmamış, doğrudan
`test_predictions_locked.csv` dosyasından alınmıştır. Notebook kilit açılmadan önce dosyaların
commit'lenmiş, değiştirilmemiş ve gönderilmiş olduğunu otomatik olarak kontrol etmektedir.

**Tablo 4.** On beş test karesine ait gerçek ve tahmin edilen hücre sayıları.

| Kare | Blok | Gerçek sayı | Baseline tahmini | Nihai tahmin | Baseline mutlak hata | Nihai mutlak hata |
|---|---|---|---|---|---|---|
| t042 | 1 | 27 | 36 | 23 | 9 | 4 |
| t043 | 1 | 27 | 34 | 25 | 7 | 2 |
| t044 | 1 | 26 | 37 | 23 | 11 | 3 |
| t045 | 1 | 26 | 38 | 22 | 12 | 4 |
| t046 | 1 | 28 | 43 | 24 | 15 | 4 |
| t062 | 2 | 31 | 24 | 23 | 7 | 8 |
| t063 | 2 | 30 | 24 | 23 | 6 | 7 |
| t064 | 2 | 30 | 19 | 23 | 11 | 7 |
| t065 | 2 | 30 | 25 | 24 | 5 | 6 |
| t066 | 2 | 30 | 25 | 24 | 5 | 6 |
| t086 | 3 | 29 | 18 | 25 | 11 | 4 |
| t087 | 3 | 30 | 24 | 27 | 6 | 3 |
| t088 | 3 | 30 | 23 | 24 | 7 | 6 |
| t089 | 3 | 29 | 22 | 24 | 7 | 5 |
| t090 | 3 | 28 | 20 | 24 | 8 | 4 |

**Tablo 5.** Baseline ve nihai pipeline'ın test performans metrikleri (MAE, RMSE, Bias, Exact-count Accuracy, ortalama süre).

| Yöntem | MAE | RMSE | Bias | Exact-count Accuracy | Ortalama süre (ms) |
|---|---|---|---|---|---|
| Baseline | 8,47 | 8,93 | −1,27 | 0,00 | 11 |
| Nihai pipeline (D) | **4,87** | **5,14** | −4,87 | 0,00 | 223 |

Nihai pipeline, testte baseline'ın hatasını **%43 azaltmıştır** (MAE 8,47 → 4,87; RMSE 8,93 → 5,14) ve
15 karenin 11'inde daha küçük hata vermiştir. Baseline'ın düşük Bias'ı (−1,27) yanıltıcıdır: blok 1'de
7–15 hücre fazla, blok 2–3'te 5–11 hücre eksik saymakta, hatalar birbirini götürmektedir. Nihai
pipeline ise her karede eksik saymaktadır (Bias = −MAE). İki yöntem de hiçbir karede tam doğru sayım
yapamamıştır. Nihai yöntem baseline'dan 20 kat yavaştır (223 ms), ancak pratik kullanım için yeterlidir.

### 6.1 Test ve doğrulama sonuçlarının karşılaştırılması
Pipeline D'nin MAE'si geliştirmede 1,70, doğrulamada 0,80, testte 4,87'dir. Test hatası doğrulamanın
yaklaşık 6 katıdır. Tanı ölçütleri hatanın türünü açıkça göstermektedir: kare başına 5,7 hücre
kaçırılmakta, buna karşılık bölünme (0,27), birleşme (0) ve fazla nesne (0,6) neredeyse hiç yoktur. Şekil
ve ayırma aşamaları testte de doğru çalışmaktadır; sorun yalnızca eşik aşamasındadır. Her referans
hücrenin Gaussian sonrası ortalama parlaklığı ölçüldüğünde nedeni nicel olarak ortaya çıkmıştır (Ek C.2).
Seq 02, Seq 01'den 2–3 kat daha karanlıktır: hücre parlaklık medyanı 29,3'e karşı 10,0, görüntü p99
değeri 68,5'e karşı 29,8'dir. Kaçırılan bütün hücrelerin parlaklığı 6,5'in altındadır. Testte T = 7'nin
altındaki hücre oranı (%20), kaçırılan hücre oranıyla (%20) birebir örtüşmektedir. Bölüm 5'te sınırlılık
olarak belirtilen risk gerçekleşmiştir: mutlak bir parlaklık değerine dayanan sabit eşik, parlaklığı
farklı bir sekansa genellenememiştir. Derste vurgulandığı gibi, sabit eşik ancak görüntülerin
histogramları birbirine benzetildiğinde farklı görüntülere uygulanabilir. Doğrulama kümesi de Seq
01'den alındığı için bu farkı yakalayamamıştır. Kurallar gereği test sonuçlarından sonra yöntem
değiştirilmemiş, diğer adaylar test kümesinde çalıştırılmamıştır.

### 6.2 Zamansal analiz
Her test bloğu 5 ardışık kareden oluşan ayrı bir zaman segmenti olarak incelenmiştir (Şekil 5).
Referanstaki her sayım değişiminin nedeni TRA etiketleri ve `man_track.txt` soy ağacından
belirlenmiştir (geçiş tablosu Ek C.3). Gerçek sayı bloklar içinde yavaş değişir (ortalama |Δ| 0,25–0,75,
en fazla 2). Test bloklarında bölünme yoktur. Değişimlerin tamamı hücrelerin görüntüden kaybolması (5),
bir süre görünmedikten sonra yeniden belirmesi (2) ve görüş alanına girmesi (1) ile açıklanmaktadır;
hücreler üç boyutlu hareket ederek odaktan çıkıp girmektedir. **Baseline** ardışık karelerde 5–6 hücrelik,
biyolojik olarak mümkün olmayan sıçramalar yapmaktadır. **Nihai pipeline** çok daha kararlıdır
(ortalama |Δ| 0,25–1,75). Ancak blok 2'de gerçek sayının 7 altında, ona paralel bir çizgi izlemektedir.
Hücre kimliği bazındaki analiz bu sabit farkın nedenini göstermektedir: bir karede kaçırılıp sonraki
karede de görünen 67 hücreden 66'sı (%99) yine kaçırılmıştır. Hata sistematiktir; aynı soluk hücreler
her karede kaçırılır. Yeniden belirdikleri anda soluk olan hücreler ilk karelerinden itibaren
kaçırılmaktadır. 20 numaralı hücre ise odaktan çıkarken önce kaçırılmış, ardından referanstan da
kaybolmuştur (Şekil C.3). Nihai tahmindeki +2/−3'lük ani değişimler gerçek değişimlerden değil, sahte
bölünme ve küçük parçalardan kaynaklanmaktadır.

<img src="figures/sekil5_zamansal.png" style="width:100%">

**Şekil 5.** Üç test bloğu için gerçek ve tahmin edilen hücre sayılarının zamansal değişimi (üstteki notlar: soy ağacına göre gerçek değişimin nedeni).

**Zamansal karar stratejisi.** Parametreleri yalnızca Seq 01'in soy ağacından belirlenen iki kurallı bir
strateji önerilmiştir. (1) *Tutarlılık kontrolü:* Seq 01'in 92 karesinde gerçek sayı ardışık iki kare
arasında en fazla 2 değişmiştir. Bu yüzden |Δ tahmin| > 2 olan geçişler "şüpheli" olarak işaretlenir.
(2) *Zamansal medyan:* Her karenin sayımı, kendisi ve iki komşusunun medyanıyla düzeltilir. Kilitli
tahminler değiştirilmeden ayrı bir analiz olarak uygulandığında kontrol, baseline'da 5, nihai
pipeline'da 1 şüpheli geçiş yakalamıştır. Medyan, kareden kareye titremeyi belirgin biçimde azaltmış
(ortalama |Δ| nihai 1,08 → 0,33, baseline 2,67 → 0,88, gerçek 0,58), ancak **MAE'yi düzeltmemiştir**
(nihai 4,87 → 5,00). Zamansal yöntemler rastgele hataları giderebilir, fakat her karede aynı hücrelerin
kaçırılmasından doğan sistematik hatayı gideremez. Strateji bu nedenle bir kalite kontrol adımı olarak
önerilmektedir: segmentasyon hatalarını tespit eder ve sayım dizisini kararlı hale getirir.

### 6.3 Başarılı, hatalı ve zorlayıcı örnekler
Örnek bölgeler nesnel bir ölçütle seçilmiştir: 260×260 piksellik pencerelerde doğru bulunan, kaçırılan
ve bölünen hücreler sayılmıştır (Şekil 6). **Başarılı (t062):** Yeterince parlak 7 hücrenin tamamı tek ve
doğru nesne olarak bulunmuştur. **Hatalı (t062):** 4 hücrenin hiçbiri bulunamamıştır. Bu hücreler
parlatılmış görüntüde düzgün ve yuvarlak olarak açıkça seçilir; tek sorunları mutlak parlaklıklarının
T = 7'nin altında kalmasıdır. Test hatasının neredeyse tamamı bu türdendir. **Zorlayıcı (t043):** Eşiğe
çok yakın parlaklıktaki bir hücre düzensiz parçalar hâlinde yakalanmış, watershed onu ikiye bölmüştür.
Eşiğe yakın hücreler hem kaçırılma hem de bölünme hatalarının ortak kaynağıdır.

<img src="figures/sekil6_ornekler.png" style="width:72%">

**Şekil 6.** Başarılı, hatalı ve zorlayıcı sayım örneklerinin karşılaştırılması (üst: gösterim için parlatılmış görüntü; alt: yeşil nesne sınırı, kırmızı referans hücre).

## 7. Tartışma ve Sonuç

### 7.1 Deneysel sonuçların tartışılması
**1. Yoğunluk dönüşümleri.** Baseline ile en başarılı dönüşüm contrast stretching olmuştur (3,4 → 2,4).
Görsel olarak en çarpıcı olan gamma, log ve histogram eşitleme ise Otsu eşiğini 0'a çekerek sayımı en
çok bozmuştur (Şekil 2). Tam pipeline'da hiçbir dönüşüm yarar sağlamamış; CLAHE, mutlak taban eşikli blok
Otsu ile birleşince MAE'yi 45,7'ye çıkarmıştır. Bir dönüşümün etkisi sonraki aşamalarla etkileşimine
bağlıdır.

**2. Filtre ve kernel boyutu.** Büyük kernel gürültü bileşenlerini (779 → 24) ve parçalanmayı (5,0 → 2,3)
azaltmıştır. Global Otsu ile MAE yine de artmıştır, çünkü filtre parçalanmanın gizlediği eksik sayımı
açığa çıkarmıştır. Yerel ya da sabit eşikle ise büyük kernel açıkça yararlıdır (G9 ile 1,2, G15 ile 0,5).
Keskinleştirme gürültüyü güçlendirdiği için uygun değildir.

**3. Segmentasyon.** Geliştirmede blok Otsu (kaçırılan 7,9 → 0,8), doğrulamada sabit eşik en başarılı
olmuştur. Adaptif eşik hücreleri halka biçiminde parçalamış; Otsu ve Triangle tek tepeli histograma
uymamıştır. "En başarılı" yöntem, değerlendirildiği veri kümesine bağlı çıkmıştır.

**4. Morfoloji sırası ve eleman boyutu.** 11×11 eleman etkileri belirginleştirmiştir: closing bölünmeyi
sıfırlamış, dilation fazla nesneyi artırmış, erosion ve opening soluk hücreleri silmiştir. İşlem sırası
değişmeli değildir (opening → closing 1,2; closing → opening 1,7). Delik doldurma tek başına etkisizdir,
ama watershed'den önce zorunludur (ham maske 1,4 → düzeltilmiş 0,7; Şekil 3c).

**5. Ölçülebilir iyileşme.** En büyük iyileşme global eşikten yerel ya da sabit eşiğe geçişte
gerçekleşmiştir (5,5 → 1,2 / 1,7). Ardından closing (0,9), closing + doldurma + watershed (0,7) ve min.
alan 100 (0,5) gelir (Ek B.7). Gaussian 15×15 tek başına MAE'yi artırmış, ancak sonraki iyileşmelerin ön
koşulu olmuştur. Erosion 11×11 ve adaptif eşiğin düşük MAE'leri ise hataların birbirini götürmesinden
kaynaklanan görünür iyileşmelerdir. Seçimlerde bu nedenle tanı ölçütleri de kullanılmıştır.

**6. Doğrulamada başarılı yöntem testte başarılı mı?** Kısmen. D doğrulamada en iyisidir (0,8), testte ise
hata yaklaşık 6 kat artmıştır (4,87). Şekil ve ayırma aşamaları testte de çalışmış, ancak eşik daha karanlık
Seq 02'ye genellenememiştir. Aynı sekanstan alınan doğrulama, sekans içi aşırı uyumu (A) yakalayabilmiş,
sekanslar arası genellemeyi ölçememiştir.

**7. Nihai pipeline baseline'dan iyi mi?** Evet; üç veri grubunun üçünde de (MAE 3,4 → 1,7; 3,8 → 0,8;
8,47 → 4,87). Nihai yöntem zamansal olarak da çok daha tutarlıdır (ortalama |Δ| 1,08'e karşı 2,67).
Bedeli işlem süresidir (223 ms'ye karşı 11 ms).

**8. Hatalı sayımların nedenleri.** (i) Eşiğin altında kalan soluk ve odak dışı hücreler: test hatasının
neredeyse tamamı bundandır ve ardışık karelerde %99 oranında tekrarlanır. (ii) Eşiğe yakın hücrelerin
parçalanıp bölünmesi. (iii) Referansta etiketsiz kenar nesneleri (fazla sayım). (iv) Baseline'da global
eşiğin kareden kareye değişmesi.

**9. Başka mikroskobik görüntülerde sınırlamalar.** En önemli sınırlama mutlak eşiktir (T = 7). Farklı
mikroskop, pozlama ya da boya yoğunluğu bu eşiği geçersiz kılar; Seq 02 bunun somut örneğidir.
Kernel, closing elemanı, `min_distance` ve min. alan ≈ 60 piksellik hücre çapına göre seçilmiştir; farklı
büyütmede yeniden ayarlanmaları gerekir. Yöntem iki boyutludur ve odak dışı hücreleri yapısı gereği
kaçırır. Çok yoğun kümelerde watershed yetersiz kalabilir.

### 7.2 Sonuç
Bu çalışmada, Fluo-N2DH-GOWT1 veri setinde klasik görüntü işleme yöntemleriyle hücre sayımı için
43 kontrollü deney yapılmış, 4 aday pipeline oluşturulmuş ve doğrulama kümesinde en düşük MAE'yi
veren Gaussian 15×15 → sabit eşik → closing → delik doldurma → watershed → min. alan pipeline'ı
seçilmiştir. Nihai yöntem, bağımsız testte baseline'ın hatasını %43 azaltmış ve zamansal olarak
çok daha tutarlı sayımlar üretmiştir. Ancak farklı parlaklıktaki ikinci sekansa tam olarak
genellenememiş ve her karede ortalama 4,9 hücre eksik saymıştır.

Başlıca bulgular şöyle özetlenebilir: (1) Bu veri setinde sayım hatasının asıl kaynağı gürültü
değil, eşikleme aşamasıdır. (2) Morfolojik işlemlerin etkisi sıralarına bağlıdır ve watershed'den
önce maskenin düzeltilmesi gereklidir. (3) Sayıma dayalı MAE, hataların birbirini götürmesi
nedeniyle tek başına yanıltıcı olabilir; tanı ölçütleri ve görsel inceleme gereklidir. (4)
Geliştirme kümesinde en iyi olan yöntem doğrulamada geriye düşebilir; bağımsız doğrulama ve test
kümeleri bu nedenle vazgeçilmezdir. (5) Zamansal yöntemler rastgele hataları bastırabilir, ancak
sistematik eksik sayımı düzeltemez.

**Geliştirilebilecek yönler.** (i) Sabit eşik, görüntüler arası parlaklık farkını gideren bir
normalizasyonla birleştirilmelidir. Örneğin histogram eşleme (E07) ya da görüntü başına arka plan
ve hücre parlaklığına göre ölçeklenen bir eşik kullanılabilir. Bu, derste sabit eşik için önerilen
koşuldur ve bu çalışmadaki genelleme sorununu doğrudan hedefler. (ii) Doğrulama kümesi birden
fazla sekanstan oluşturulursa sekanslar arası genelleme seçim aşamasında ölçülebilir. (iii) Blok
bazlı Otsu'da daha küçük bloklar ya da hücre başına yerel eşik, parlak ve soluk hücrelerin aynı
blokta bulunması sorununu azaltabilir. (iv) Zamansal bilgi, yalnızca sayımın düzeltilmesinde değil,
bir karede bulunan hücrenin komşu karelerde aranmasında da kullanılabilir. Bu yaklaşım, odaktan
geçici olarak çıkan hücrelerin kaçırılmasını azaltabilir.

## 8. Kaynakça
1. Cell Tracking Challenge, Fluo-N2DH-GOWT1 veri seti. https://celltrackingchallenge.net/2d-datasets/
2. V. Ulman et al., "An objective comparison of cell-tracking algorithms," *Nature Methods*, 14, 1141–1152, 2017.
3. M. Maška et al., "A benchmark for comparison of cell tracking algorithms," *Bioinformatics*, 30(11), 1609–1617, 2014.
4. E. Bártová et al., "Recruitment of Oct4 protein to UV-damaged chromatin in embryonic stem cells," *PLoS ONE*, 6(12), e27954, 2011.
5. N. Otsu, "A Threshold Selection Method from Gray-Level Histograms," *IEEE Trans. Systems, Man, and Cybernetics*, 9(1), 62–66, 1979.
6. C. H. Li and C. K. Lee, "Minimum cross entropy thresholding," *Pattern Recognition*, 26(4), 617–625, 1993.
7. G. W. Zack, W. E. Rogers and S. A. Latt, "Automatic measurement of sister chromatid exchange frequency," *J. Histochemistry & Cytochemistry*, 25(7), 741–753, 1977.
8. K. Zuiderveld, "Contrast Limited Adaptive Histogram Equalization," in *Graphics Gems IV*, Academic Press, 474–485, 1994.
9. L. Vincent and P. Soille, "Watersheds in digital spaces: an efficient algorithm based on immersion simulations," *IEEE Trans. Pattern Analysis and Machine Intelligence*, 13(6), 583–598, 1991.
10. P. Soille, *Morphological Image Analysis: Principles and Applications*, 2nd ed., Springer, 2003.
11. R. C. Gonzalez and R. E. Woods, *Digital Image Processing*, 4th ed., Pearson, 2018.
12. G. Bradski, "The OpenCV Library," *Dr. Dobb's Journal of Software Tools*, 2000.
13. S. van der Walt et al., "scikit-image: image processing in Python," *PeerJ*, 2, e453, 2014.
14. P. Virtanen et al., "SciPy 1.0: fundamental algorithms for scientific computing in Python," *Nature Methods*, 17, 261–272, 2020.
15. C. R. Harris et al., "Array programming with NumPy," *Nature*, 585, 357–362, 2020.

<div class="sayfa-sonu"></div>

## Ekler

### Ek A. 15 test karesinin tamamının görsel sonuçları

<img src="figures/ekA_test15.png" style="width:100%">

**Şekil A.1.** Nihai pipeline'ın 15 test karesindeki çıktısı (yeşil: bulunan nesne sınırı, kırmızı: referans hücre). Başlıklarda nihai tahmin, referans, baseline tahmini ve tanı ölçütleri yer alır.

### Ek B. Ayrıntılı deney analizleri (geliştirme kümesi)

#### B.1 Yoğunluk dönüşümleri
Yedi yoğunluk konfigürasyonu, geri kalan adımlar baseline ile aynı tutularak (Otsu, 8-komşuluk,
min. alan 50) geliştirme kümesinde karşılaştırılmıştır (E01–E07). Her dönüşüm orijinal görüntüye
ayrı ayrı uygulanmıştır.

| ID | Dönüşüm | MAE | RMSE | Bias | Ort. Otsu eşiği |
|---|---|---|---|---|---|
| E01 | Orijinal | 3,4 | 5,04 | −3,0 | 31,8 |
| E02 | Contrast stretching [0, p99,5] | **2,4** | **2,90** | −1,8 | 75,8 |
| E03 | Gamma (γ=0,5) | 41,1 | 122,73 | +38,3 | 49,9 |
| E04 | Logaritmik | 460,1 | 464,97 | +460,1 | 0 |
| E05 | Histogram eşitleme | 460,1 | 464,97 | +460,1 | 0 |
| E06 | CLAHE (clip 2, 8×8) | 4,7 | 5,54 | −4,3 | 41,1 |
| E07 | Histogram eşleme (ref. t046) | 2,9 | 3,42 | −1,9 | 30,4 |

Görsel olarak hücreleri en belirgin gösteren dönüşümler (gamma, log, histogram eşitleme) sayımı
en çok bozan dönüşümler olmuştur. Bunun nedeni Otsu yönteminin bu dönüşümlerle etkileşimidir:
piksellerin yaklaşık yarısı tam 0'dır; bu dönüşümler 0'ı yerinde bırakırken en küçük sıfır dışı
değeri (3) yukarı taşır (örneğin log ile 64'e). Histogram, 0'da çok büyük bir tepe ile ondan uzak
kalan tüm diğer değerler biçimine gelir ve Otsu eşiği 0 çıkar; böylece arka plandaki sıfır dışı
tüm gürültü pikselleri ön plan olur (t014'te 411 nesne). Log ve histogram eşitlemenin birebir aynı
sonucu vermesi de bundan kaynaklanır: eşik 0 olduğunda kesin artan her dönüşüm aynı maskeyi
(orijinal > 0) üretir. Gamma'da bu durum yalnızca t014'te oluşmuştur.

Contrast stretching, p99,5'in üstünü doyurarak histogram şeklini değiştirir ve Otsu eşiğini
orijinal ölçekte düşürür; soluk hücrelerin daha fazlası yakalanmıştır (t014: 10 → 19). Histogram
eşleme kareler arası parlaklık farkını giderdiği için en karanlık karede (t014) hatayı −13'ten −1'e
indirmiş, diğer karelerde etkisi sınırlı kalmıştır. CLAHE arka plan gürültüsünün yerel kontrastını
da artırdığı için Otsu eşiğini yükseltmiş ve eksik sayımı artırmıştır. Tüm konfigürasyonlarda
soluk hücrelerin parçalı segmentasyonu devam etmektedir; bu sorun filtreleme adımında ele
alınmıştır.

#### B.2 Filtre ve kernel karşılaştırmaları
Filtreler, yoğunluk dönüşümü uygulanmadan ve baseline segmentasyonu (Otsu, min. alan 50) sabit
tutularak karşılaştırılmıştır (E08–E16). Sayım hatalarının birbirini götürmesini ayırt etmek
için bulunan nesneler TRA etiketleriyle eşleştirilmiş; kaçırılan, bölünen ve fazla nesneler ayrı
ayrı sayılmıştır (kare başına ortalama, referans ort. 22,7).

| ID | Filtre | MAE | Bias | Ham bileşen | Kaçırılan | Bölünme |
|---|---|---|---|---|---|---|
| E01 | Yok | 3,4 | −3,0 | 779 | 8,2 | 5,0 |
| E08 | Gaussian 3×3 | 4,0 | −1,8 | 175 | 8,1 | 5,9 |
| E09 | Gaussian 5×5 | 4,1 | −2,9 | 99 | 8,3 | 5,2 |
| E10 | Gaussian 9×9 | 4,5 | −4,1 | 42 | 8,1 | 3,7 |
| E11 | Gaussian 15×15 | 5,5 | −5,5 | 24 | 7,9 | 2,3 |
| E12 | Median 3×3 | 4,1 | −1,3 | 194 | 8,5 | 7,1 |
| E13 | Median 5×5 | 3,5 | −2,9 | 88 | 8,2 | 5,1 |
| E14 | Median 9×9 | 6,1 | −6,1 | 38 | 8,1 | 1,9 |
| E15 | Laplacian keskinleştirme | 5,2 | −0,6 | 3231 | 11,5 | 10,5 |
| E16 | Unsharp masking | 2,9 | −1,1 | 1392 | 9,9 | 8,1 |

Kernel boyutu büyüdükçe ham bileşen sayısı 779'dan 24'e, bölünme 5,0'dan 2,3'e düşmüştür; yani
yumuşatma gürültüyü ve soluk hücrelerin parçalanmasını büyük ölçüde gidermiştir. Buna karşın
MAE artmıştır. Tanı ölçütleri bu çelişkiyi açıklar: kaçırılan hücre sayısı filtreden bağımsız
olarak ≈8'de sabittir. Filtresiz durumda bu eksiklik, parçalanan hücrelerin fazla sayılmasıyla
kısmen dengelenmekte, filtreleme parçalanmayı giderince eksik sayım açığa çıkmaktadır. Görsel
incelemede en soluk hücreler yumuşatılmış görüntüde açıkça seçilebildiği halde Otsu maskesinde
hiç yer almamaktadır; darboğaz gürültü değil, tek global eşiğin (≈30) bu hücrelerin parlaklığının
üstünde kalmasıdır. Keskinleştirme filtreleri gürültüyü de güçlendirdiği için parçalanmayı ve
kaçırılan hücre sayısını artırmıştır; E16'nın düşük MAE'si yine hataların birbirini götürmesinden
kaynaklanmaktadır. Median 9×9 yaklaşık 7 kat daha yavaştır (≈76 ms).

#### B.3 Segmentasyon yöntemleri
Segmentasyon yöntemleri ortak bir ön işlemden (Gaussian 15×15) sonra, min. alan 50 sabit
tutularak karşılaştırılmıştır. Parametreli yöntemlerin parametreleri yalnızca geliştirme kümesinde
taranarak seçilmiştir.

| ID | Yöntem | Parametre | MAE | Bias | Kaçırılan | Bölünme | Birleşme | Fazla |
|---|---|---|---|---|---|---|---|---|
| E11 | Otsu (global) | otomatik | 5,5 | −5,5 | 7,9 | 2,3 | 0,0 | 0,1 |
| E17 | Sabit eşik | T=7 | 1,7 | +1,3 | 2,0 | 0,4 | 0,0 | 2,9 |
| E18 | Triangle | otomatik | 4,1 | +4,1 | 0,6 | 2,1 | 0,0 | 2,6 |
| E19 | Li | otomatik | 1,8 | −0,8 | 3,3 | 1,4 | 0,0 | 1,1 |
| E20 | Adaptif Gaussian | blok 101, C=−10 | 1,5 | −0,3 | 6,5 | 5,6 | 0,0 | 0,6 |
| E21 | Blok Otsu 8×8 | taban 4 | **1,2** | +0,8 | **0,8** | 0,7 | 0,5 | 1,4 |
| E22 | Blok Otsu 4×4 | taban 5 | 1,9 | −1,7 | 3,9 | 2,2 | 0,0 | 0,0 |

Global Otsu dışındaki tüm yöntemler kaçırılan hücre sayısını belirgin biçimde azaltmıştır; bu,
Ek B.2'deki "darboğaz eşiktir" yorumunu doğrular. En iyi sonuç blok bazlı Otsu (8×8) ile elde
edilmiştir: kaçırılan hücre 7,9'dan 0,8'e inmiştir. Her blokta ayrı eşik hesaplandığı için soluk
hücre içeren bloklarda eşik düşmekte, parlak hücre içeren bloklarda yükselmektedir. Hücre
içermeyen bloklarda Otsu gürültüyü ikiye böldüğünden taban eşik zorunludur (taban 0'da MAE > 100);
taban değerine duyarlılık yüksektir (8×8 için taban 4: 1,2; taban 5: 3,2).

Adaptif eşikleme en düşük ikinci MAE'yi vermiş, ancak tanı ölçütleri bunun hataların birbirini
götürmesinden kaynaklandığını göstermiştir (6,5 kaçırılan, 5,6 bölünme). Her piksel kendi
çevresinin ortalamasıyla karşılaştırıldığı için büyük ve homojen hücrelerin iç kısmı eşiğin altında
kalmakta, hücreler halka biçimli parçalara bölünmektedir. Ayrıca en iyi C değeri taranan aralığın
sınırındadır. Yalnızca MAE'ye göre seçim yapılsaydı bu yöntem tercih edilebilirdi; bu nedenle
seçimde tanı ölçütleri de dikkate alınmıştır. Triangle yöntemi neredeyse tüm hücreleri bulmuş
(0,6 kaçırılan), ancak düşük eşik nedeniyle gürültü ve parçalanma artmıştır. Sabit eşik (T=7) iyi
sonuç vermekle birlikte, parlaklığı farklı sekanslara veya mikroskoplara genellenmesi en zayıf
yöntemdir; Li yöntemi ise her görüntü için eşiği otomatik hesapladığından genelleme açısından
avantajlıdır.

#### B.4 Morfolojik işlemler
Morfolojik işlemler, Ek B.3'ün en iyi maskesine (Gaussian 15×15 → Blok Otsu 8×8, E21)
uygulanmıştır; önceki adımlar sabit tutulmuştur. Hücreler yuvarlak olduğundan elips biçimli
yapılandırma elemanı, iki boyutta (5×5 ve 11×11) kullanılmıştır. Watershed için `min_distance`
parametresi geliştirme kümesinde taranmıştır (ham maske: 10–50 aralığında MAE 13,1'den 1,4'e;
düzeltilmiş maske: en iyi değer 25, MAE 0,7).

| ID | İşlem | MAE | Bias | Kaçırılan | Bölünme | Birleşme | Fazla | Süre (ms) |
|---|---|---|---|---|---|---|---|---|
| E21 | Yok | 1,2 | +0,8 | 0,8 | 0,7 | 0,5 | 1,4 | 13 |
| E23 | Erosion 5×5 | 1,1 | −0,1 | 1,3 | 0,2 | 0,5 | 1,5 | 24 |
| E24 | Erosion 11×11 | 0,8 | +0,4 | 1,9 | 1,1 | 0,0 | 1,2 | 24 |
| E25 | Dilation 5×5 | 2,4 | +2,4 | 0,5 | 0,4 | 0,6 | 3,1 | 24 |
| E26 | Dilation 11×11 | 3,0 | +3,0 | 0,3 | 0,7 | 0,7 | 3,3 | 25 |
| E27 | Opening 5×5 | 1,1 | +0,7 | 0,9 | 0,5 | 0,5 | 1,6 | 24 |
| E28 | Opening 11×11 | 1,0 | +0,2 | 1,6 | 0,6 | 0,4 | 1,6 | 26 |
| E29 | Closing 5×5 | 1,2 | +0,2 | 0,8 | 0,2 | 0,6 | 1,4 | 24 |
| E30 | Closing 11×11 | 0,9 | +0,1 | 0,7 | 0,0 | 0,6 | 1,4 | 30 |
| E31 | Opening → Closing 11×11 | 1,2 | −0,4 | 1,6 | 0,1 | 0,5 | 1,6 | 33 |
| E32 | Closing → Opening 11×11 | 1,7 | −0,3 | 1,2 | 0,1 | 0,6 | 1,4 | 30 |
| E33 | Delik doldurma | 1,2 | +0,8 | 0,8 | 0,7 | 0,5 | 1,4 | 62 |
| E34 | Watershed (ham maske, md=35) | 1,4 | +1,4 | 0,8 | 1,2 | 0,4 | 1,4 | 184 |
| E35 | Closing 11×11 → doldurma → watershed (md=25) | **0,7** | +0,7 | **0,7** | **0,0** | **0,0** | 1,4 | 222 |

Temel işlemlerin etkisi beklenen yöndedir. Dilation nesneleri büyüttüğü için 50 pikselin altındaki
gürültü lekelerini min. alan sınırının üzerine taşımış ve yakın hücreleri birleştirmiştir ("fazla"
1,4 → 3,3; Bias +3,0). Opening küçük lekeleri silmesi beklenirken, 11×11 boyutunda soluk ve küçük
hücreleri de silmiştir (kaçırılan 0,8 → 1,6). Closing ise hücre içindeki çatlakları kapattığından
bölünmeyi tamamen ortadan kaldırmış (0,7 → 0,0), ancak birbirine değen hücreleri ayıramamıştır
(birleşme 0,6).

Erosion 11×11, tek başına uygulanan işlemler arasında en düşük MAE'yi (0,8) vermiştir; ancak tanı
ölçütleri bunun yine hataların birbirini götürmesinden kaynaklandığını göstermektedir: soluk
hücreler silinmiş (kaçırılan 1,9), bazı hücreler parçalara ayrılmıştır (bölünme 1,1). Şekil 3c'de
t041 karesindeki iki hücrenin erosion sonrası birden fazla parçaya ayrıldığı görülmektedir. Bu
nedenle erosion nihai seçim için uygun değildir.

İşlem sırası sonucu değiştirmektedir: opening → closing MAE 1,2, closing → opening MAE 1,7
vermiştir. Morfolojik işlemler değişmeli değildir; önce closing yapıldığında hücreler arasındaki
dar boşluklar kapanıp nesneler birleşmekte, ardından gelen opening bu birleşmeyi geri
alamamaktadır.

Delik doldurma (E33) sayımı hiç değiştirmemiştir; çünkü bir nesnenin içindeki delik nesnenin
bağlantılılığını, dolayısıyla bağlı bileşen sayısını etkilemez. Ancak watershed öncesinde
önemlidir: distance transform, deliğin kenarını da arka plan kabul ettiği için delikli bir hücrede
birden fazla tepe oluşur ve hücre bölünür. Watershed ham maskeye uygulandığında (E34) bu nedenle
küçük `min_distance` değerlerinde aşırı bölünme görülmüş (md=10'da MAE 13,1); aşırı bölünmeyi
önleyecek kadar büyük `min_distance` (35) seçildiğinde ise bölünme (1,2) devam etmiş, birbirine
değen hücreler de tam ayrılamamıştır. Maske önce closing ve delik doldurma ile düzeltildiğinde
(E35) her hücrede tek bir tepe kalmış, hücre yarıçapının yaklaşık yarısı olan `min_distance` = 25
ile hem bölünme hem birleşme sıfıra inmiştir. Şekil 3c bu adımları t041 karesinde göstermektedir:
Blok Otsu maskesinde delikli ve birbirine değen iki hücre, closing ve doldurmadan sonra tek bir
"kum saati" biçimine dönüşmekte, distance transform iki ayrı tepe üretmekte, watershed iki hücreyi
boyun noktasından ayırmaktadır.

E35, MAE 0,7 ile en iyi sonucu vermiştir. Kalan hatanın büyük kısmı "fazla" nesnelerden (1,4)
oluşmaktadır; bunlar Bölüm 2.3'te belirtilen, görüntü kenarında olup referansta işaretlenmemiş
hücreler ve gürültü lekeleridir. Bu hata morfolojik işlemlerle giderilemez. Watershed, işlem süresini
kare başına yaklaşık 13 ms'den 222 ms'ye çıkarmıştır; bu süre hâlâ pratik kullanım için kabul
edilebilir düzeydedir.

#### B.5 Tam pipeline deneylerinin yorumu
**Minimum alan (E36).** E35 üzerinde 50–600 piksel aralığı taranmıştır. MAE 100–200 aralığında en
düşüktür (0,5); eşitlik durumunda en küçük değer (100) seçilmiştir. Bu aralık veri analiziyle
uyumludur: gürültü lekeleri 100 pikselden küçük, bölünme sonrası yavru hücreler 250–400 piksel,
en küçük hücre 579 pikseldir. Min. alan 300 ve üzerine çıkarıldığında yavru hücreler silinmeye
başlamış, Bias negatife dönmüştür (600'de −0,8). E36'da fazla nesne 1,4'ten 1,2'ye inmiş, buna
karşılık kaçırılan hücre 0,7'den 0,9'a çıkmıştır; iyileşmenin bir kısmı bu nedenle hataların
birbirini götürmesinden kaynaklanmaktadır.

**Yoğunluk dönüşümlerinin tam pipeline'daki etkisi (E37, E38).** CLAHE ve contrast stretching tek
başına global Otsu ile denendiğinde iyileşme sağlamıştı (E02: 2,4). Tam pipeline'a eklendiklerinde
ise MAE 45–51'e çıkmıştır ve hatanın neredeyse tamamı fazla nesnedir. Bunun nedeni, blok bazlı
Otsu'nun taban eşiğinin (4) orijinal parlaklık ölçeğinde seçilmiş mutlak bir değer olmasıdır. Bu
dönüşümler arka plan gürültüsünü de güçlendirdiği için hücre içermeyen bloklarda gürültü taban
eşiğin üzerine çıkmakta ve yüzlerce leke oluşmaktadır. Bu sonuç, pipeline parametrelerinin
birbirinden bağımsız olmadığını gösterir: bir aşama değiştiğinde sonraki aşamaların parametreleri
yeniden ayarlanmalıdır.

**Filtre (E39, E40).** Gaussian 15×15 yerine 9×9 kullanıldığında MAE 1,2'ye çıkmıştır; daha az
yumuşatılan maskede gürültü ve çatlaklar artmaktadır. Median 9×9 ile Gaussian 15×15'e çok yakın
bir sonuç (0,6) elde edilmiştir; ancak median biraz daha fazla bölünme (0,4) ve birleşme (0,2)
üretmiş ve daha yavaştır (296 ms'ye karşı 226 ms).

**Eşik (E41, E42).** Blok Otsu yerine global eşik kullanıldığında performans belirgin biçimde
düşmüştür. Li yöntemi soluk hücreleri kaçırmaktadır (kaçırılan 3,4; Bias −3,1). Ek B.3'te
(min. alan 50) Li'nin MAE'si 1,8 iken burada 3,1'dir: Li ile kısmen bulunan soluk hücreler küçük
parçalar hâlinde kalmakta ve min. alan 100 tarafından silinmektedir. Sabit eşik (T=7) de 2,1
hücre kaçırmıştır. Her iki sonuç, Ek B.3'teki temel bulguyu doğrular: bu veri setinde asıl
darboğaz eşiklemedir ve hücre parlaklıkları geniş bir aralıkta değiştiği için yerel (blok bazlı)
eşik gereklidir.

**Görsel inceleme.** Pipeline A'nın 10 geliştirme karesindeki çıktısı tek tek incelenmiştir
(notebook Bölüm 10 (b)). Bölünme ve birleşme hiçbir karede görülmemiştir. Hataların tamamı ya
görüntü kenarında olup referansta işaretlenmemiş hücrelerden (fazla, karelerin hepsinde 1–2) ya
da soluk hücrelerden (kaçırılan) kaynaklanmaktadır. En zor kare t014'tür (3 kaçırılan hücre);
baseline yöntemi de en büyük hatayı bu karede yapmıştı (Bölüm 3.1).

**Blok sınırı yapaylığı (E43).** Görsel incelemede bazı hücrelerin düz bir çizgi boyunca kesildiği
görülmüştür. Bu çizgiler blok bazlı Otsu'nun 128×128 piksellik blok sınırlarıdır: sınıra denk gelen
soluk bir hücrenin iki yarısı farklı eşiklerle kesilmekte, eşiği yüksek bloktaki yarısı
kaybolmaktadır. Kalan "kaçırılan" ve "fazla" hatalarının bir kısmı bu nedenle oluşmaktadır. Bu
yapaylığı gidermek için CLAHE'de kullanılan yöntem denenmiş, blok eşikleri doğrusal interpolasyonla
her piksele yumuşak biçimde dağıtılmıştır (E43). Beklenenin aksine sonuç kötüleşmiştir (taban 4:
MAE 0,5 → 1,1; kaçırılan 0,9 → 2,2). İnterpolasyon soluk bir hücrenin eşiğini komşu parlak blokların
yüksek eşiğiyle karıştırmaktadır. Klasik yöntemde ise bloğunun tamamen içinde kalan soluk hücre, o
bloğun düşük eşiğini olduğu gibi almaktadır. Eşik haritaları ayrıca t014'teki 3 kaçırılan hücrenin
asıl nedeninin blok sınırı olmadığını göstermiştir: bu hücreler, parlak hücrelerle aynı blokta
bulunan çok soluk hücrelerdir. Böyle bir blokta Otsu, parlak hücreleri arka plandan ayıran yüksek bir
eşik (≈ 20–30) seçmekte, soluk hücreler bu eşiğin altında kalmaktadır. Bu hata her iki yöntemde de
aynıdır (iki yöntemde de 3 kaçırılan) ve daha küçük bloklar ya da hücre başına eşik gerektirir.
Taban 5 ile MAE 0,5'e inmiş, ancak bunun arkasında
2,2 kaçırılan ve 0,9 bölünme bulunduğu, yani hataların birbirini götürdüğü görülmüştür. Bu nedenle
klasik Blok Otsu korunmuş, blok sınırı yapaylığı yöntemin bir sınırlılığı olarak kaydedilmiştir
(Bölüm 7).

#### B.6 Segmentasyon parametre taramaları
<img src="figures/ekB7_taramalar.png" style="width:100%">

**Şekil B.1.** Sabit eşik T, adaptif eşik (blok, C) ve blok Otsu taban eşiği taramaları (geliştirme MAE).

#### B.7 İyileşme zinciri
<img src="figures/ekB6_iyilesme.png" style="width:100%">

**Şekil B.2.** Baseline'dan aday pipeline'lara her değişikliğin geliştirme MAE'sine etkisi; sağda baseline ve nihai yöntemin geliştirme / doğrulama / test MAE'si.

#### B.8 Pipeline A'nın geliştirme kümesindeki çıktısı ve blok eşik haritaları
<img src="figures/ekB1_pipelineA_dev.png" style="width:100%">

<img src="figures/ekB2_esik_haritasi.png" style="width:100%">

**Şekil B.3.** Üst: Pipeline A'nın 10 geliştirme karesi. Alt: t014 için klasik (basamaklı) ve interpolasyonlu blok Otsu eşik haritaları ve sonuçları (sarı: blok sınırları).

### Ek C. Doğrulama ve test için ek analizler

#### C.1 Doğrulama kümesi ayrıntıları
Kare bazında tahminler (referans / A / B / C / D / baseline): t004: 23 / 21 / 22 / 19 / 22 / 13;
t020: 23 / 24 / 23 / 20 / 23 / 17; t032: 24 / 25 / 26 / 19 / 23 / 24; t053: 21 / 22 / 22 / 20 / 21 / 24;
t078: 20 / 23 / 22 / 19 / 22 / 20.

**Kare bazında davranış.** Tahmin çizgileri incelendiğinde D referansı en yakından izleyen adaydır;
iki karede (t020, t053) tam doğru sayım yapmış, diğerlerinde en fazla 2 hücre sapmıştır. Pipeline C (Li)
her karede 19–20 hücre bulmuştur; referans değiştiğinde tahmin değişmemekte, yani sistematik olarak eksik
saymaktadır. Baseline ise 13 ile 24 arasında dalgalanarak en kararsız davranışı göstermiştir. A ve B'de
doğrulama MAE'si geliştirme MAE'sinin 2–3 katına çıkmış, C ve D'de ise düşmüştür.

**Görsel inceleme ve tanı ölçütleri.** D'nin doğrulama karelerindeki görsel çıktısı (Şekil C.1) iki önemli
noktayı ortaya koymuştur. Birincisi, kaçırılan hücrelerin çoğu her karede aynı bölgededir: görüntünün sağ
alt kısmındaki çok soluk hücre grubu. Bu hücrelerin parlaklığı T = 7 eşiğinin altındadır; hücrelerin
sıvı içinde üç boyutlu hareket ederek odak düzleminin dışına çıkmasıyla açıklanabilir. Hata rastgele
değil sistematiktir; ardışık karelerde aynı hücreler kaçırılmaktadır. İkincisi, tam doğru sayımlar
hatasız segmentasyon anlamına gelmemektedir. Örneğin t020'de tahmin ve referans 23'tür, ancak 3 hücre
kaçırılmış ve 3 fazla nesne (küçük parçalar ve görüntü kenarındaki işaretlenmemiş hücre) bulunmuştur.
Sayım doğru olsa da 6 segmentasyon hatası birbirini götürmüştür. Bu, yalnızca sayıma dayalı MAE'nin
sınırlılığını ve tanı ölçütlerinin gerekliliğini göstermektedir.

<img src="figures/ekB3_dogrulama_grafik.png" style="width:100%">

<img src="figures/ekB3_dogrulama_kareler.png" style="width:100%">

**Şekil C.1.** Doğrulama kareleri: referans ve aday tahminleri, geliştirme ve doğrulama MAE'si (üst); Pipeline D'nin 5 doğrulama karesindeki çıktısı (alt).

#### C.2 Test kümesinde hücre parlaklığı
Her referans hücrenin, nihai pipeline'ın eşiği uyguladığı Gaussian 15×15 sonrası görüntüdeki ortalama parlaklığı (notebook Bölüm 13 (c)):

| Küme | Hücre | Parlaklık medyanı | T = 7 altındaki oran | Kaçırılan oran | Görüntü p99 |
|---|---|---|---|---|---|
| Geliştirme (Seq 01) | 227 | 29,3 | 0,12 | 0,09 | 68,5 |
| Doğrulama (Seq 01) | 111 | 26,9 | 0,12 | 0,09 | 67,0 |
| Test (Seq 02) | 431 | 10,0 | 0,20 | 0,20 | 29,8 |

<img src="figures/ekB4_parlaklik.png" style="width:100%">

**Şekil C.2.** Seq 01 ve Seq 02 hücre parlaklık dağılımları (sol) ve test kümesinde bulunan / kaçırılan hücreler (sağ). Kaçırılan hücrelerin tamamı T = 7 eşiğinin solundadır.

#### C.3 Zamansal geçişler ve kaçırılan hücrelerin kalıcılığı
| Geçiş | Δ gerçek | Δ baseline | Δ nihai | Gerçek değişimin nedeni |
|---|---|---|---|---|
| t042→t043 | 0 | −2 | +2 | – |
| t043→t044 | −1 | +3 | −2 | 1 hücre kayboldu |
| t044→t045 | 0 | +1 | −1 | – |
| t045→t046 | +2 | +5 | +2 | 1 yeni hücre girdi, 1 hücre yeniden belirdi |
| t062→t063 | −1 | 0 | 0 | 1 hücre kayboldu |
| t063→t064 | 0 | −5 | 0 | – |
| t064→t065 | 0 | +6 | +1 | – |
| t065→t066 | 0 | 0 | 0 | – |
| t086→t087 | +1 | +6 | +2 | 1 hücre yeniden belirdi |
| t087→t088 | 0 | −1 | −3 | – |
| t088→t089 | −1 | −1 | 0 | 1 hücre kayboldu |
| t089→t090 | −1 | −2 | 0 | 1 hücre kayboldu |

**Baseline'ın zamansal tutarsızlığı.** Baseline ardışık karelerde 5–6 hücrelik sıçramalar yapmaktadır
(t063→t064: −5, t064→t065: +6, t086→t087: +6). Gerçek sayının bu kadar değişmesi biyolojik olarak
mümkün değildir. Değişimin yönü yalnızca geçişlerin %25–75'inde gerçek değişimle uyuşmaktadır. Bu
sıçramalar, global Otsu eşiğinin kareden kareye değişen histograma duyarlı olmasından
kaynaklanmaktadır: eşik biraz yer değiştirdiğinde hücreler parçalanmakta ya da gürültü lekeleri
nesne olarak sayılmaktadır.

<img src="figures/ekB5_isi_haritasi.png" style="width:80%">

**Şekil C.3.** En az bir kez kaçırılan referans hücreler ve 15 test karesindeki durumları (kırmızı: kaçırıldı, yeşil: bulundu, beyaz: karede yok).
