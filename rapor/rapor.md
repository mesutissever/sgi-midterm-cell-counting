# Klasik Görüntü İşleme Yöntemleriyle Mikroskobik Görüntülerde Hücre Segmentasyonu, Sayımı ve Zamansal Analizi

**Ders:** Görüntü İşleme – Yüksek Lisans Ara Sınav Projesi
**Öğrenci:** Mesut İşsever – 264329015
**GitHub:** *(bağlantı eklenecek)*

> Taslak: Bölümler, ödev dokümanının 16. bölümündeki zorunlu yapıya göre düzenlenmiştir.
> Her aşama tamamlandıkça ilgili bölüm doldurulacaktır. Hedef uzunluk 6–8 sayfa;
> sığmayan tablo ve şekiller Ekler bölümüne konulacaktır.

---

## 1. Giriş ve Problem Tanımı (16.1)

- Hücre segmentasyonu ve sayımı problemi
- Çalışmanın amacı
- Kullanılan klasik görüntü işleme yaklaşımlarının kısa tanıtımı
- Çalışmanın kapsamı

## 2. Veri Seti ve Deneysel Tasarım (16.2)

### 2.1 Veri setinin özellikleri

Fluo-N2DH-GOWT1 veri seti, GFP ile işaretlenmiş fare kök hücre çekirdeklerinin floresan
mikroskop görüntülerinden oluşur. Eğitim arşivinde iki sekans (01 ve 02) bulunur; her biri
92 kare (t000–t091), 1024×1024 piksel ve 8 bit (uint8) gri seviyelidir. Referans olarak
`_GT/TRA` (tüm hücreler için takip etiketleri) ve `_GT/SEG` (seçili karelerde hassas sınırlı
manuel maskeler) kullanılmıştır; `_ST` ve `_ERR_SEG` klasörleri kullanılmamıştır.
Test sızıntısını önlemek için Sekans 02'de yalnızca kare sayısı, boyut ve veri tipi kontrol
edilmiş, yoğunluk dağılımı incelenmemiştir. Aşağıdaki bulgular Sekans 01'e aittir.

**Yoğunluk ve histogram.** Piksellerin %50'si tam 0, %90'ı ≤ 6, %99'u ≤ 68, %99,9'u ≤ 139'dur.
Histogram tek bir arka plan tepesi ve uzun bir kuyruktan oluşur; nesne sınıfı için ayrı bir
tepe yoktur (iki tepeli değildir). Histogramda yalnızca belirli gri seviyeler dolu olup
aradaki değerler boştur (tarak yapısı); bu, görüntülerin daha düşük bir bit derinliğinden
ölçeklendiğine işaret eder. Kare ortalama yoğunluğu 3,63–4,25, p99 değeri 61–74 arasında
değişir ve sekans boyunca artış eğilimi gösterir.

**Hücre özellikleri (01_GT/SEG, 30 kare, 144 hücre).** Hücre alanı ortanca 2847 piksel
(579–5306), eşdeğer çap ortalama 60 pikseldir. Hücreler hafif eliptik (eccentricity ort. 0,55)
ve dışbükeydir (solidity ort. 0,97). Ortalama hücre parlaklığı 3,4 ile 96 arasında yaklaşık
28 kat değişir; hücrelerin %37'sinin ortalama parlaklığı ≤ 6, yani arka planın büyük kısmıyla
aynı düzeydedir. Hücrelerin içinde sıfır yoğunluklu pikseller (çekirdekçikler ve soluk bölgeler)
bulunur.

**Olası güçlükler (Sekans 01, 92 kare).**
- *Temas eden hücreler:* TRA etiketleri 10 piksel genişletildiğinde karelerin %57'sinde en az
  bir birleşme oluşur (kare başına ort. 0,70, en fazla 2). Hücre ayırma yapılmazsa yalnızca bu
  nedenle sistematik eksik sayım beklenir.
- *Kontrast yetersizliği:* Odak düzleminden uzaklaşan hücreler arka plana yakın parlaklıktadır.
- *Gürültü:* Hücre dışında kare başına ortalama 2 (en fazla 11) küçük parlak leke (< 100 piksel)
  bulunur; en küçük gerçek hücre 579 pikseldir.
- *Düzensiz arka plan:* Hücre dışı 8×8 blok ortalamaları 0,93 ile 2,70 arasında değişir;
  sol-orta bölge belirgin biçimde daha parlaktır. Soluk hücrelerin parlaklığı (~5) ile bu
  farkın aynı mertebede olması, global eşik yerine yerel/adaptif eşiklemeyi gerekçelendirir.

### 2.2 Rastgele seçim yöntemi ve seed

Seed = 29015 + 2026 = 31041. Seçim `numpy.random.default_rng(31041)` ile oluşturulan tek bir
rastgele sayı akışıyla yapılmıştır. Önce Sekans 01'den 5 doğrulama karesi, ardından bu karelere
en az 5 kare uzaklıktaki (iki kare arasında en az 5 kare; numara farkı ≥ 6) karelerden 10
geliştirme karesi seçilmiştir. Sekans 02'de 3 başlangıç noktası seçilmiş ve her birinden ardışık
5 kare alınmıştır; bloklar arasında en az 5 kare boşluk koşulu sağlanmayan çekilişler aynı akıştan
yeniden yapılmıştır (Sekans 01: 1. denemede, Sekans 02: 3. denemede geçerli seçim). Seçim,
algoritma geliştirmeden önce yapılmış ve ayrı bir commit ile kaydedilmiştir.

### 2.3 Seçilen görüntüler

**Tablo 1.** Rastgele seçilen görüntüler ve veri grupları.

| Veri grubu | Sekans | Kareler | Sayı |
|---|---|---|---|
| Geliştirme | 01 | 14, 41, 43, 44, 45, 46, 62, 67, 72, 85 | 10 |
| Doğrulama | 01 | 4, 20, 32, 53, 78 | 5 |
| Test – Blok 1 | 02 | 42–46 | 5 |
| Test – Blok 2 | 02 | 62–66 | 5 |
| Test – Blok 3 | 02 | 86–90 | 5 |

Geliştirme kümesinin yarısı (41, 43–46) zamanca birbirine yakın karelerdir; bu durum rastgele
seçimin sonucudur ve geliştirme kümesinin çeşitliliğini sınırlar. Doğrulama kareleri ise
sekansın başından sonuna yayılmıştır.

### 2.4 Referans hücre sayıları

Referans sayı, ilgili karenin `TRA` takip etiket görüntüsünde sıfır (arka plan) dışındaki
farklı hücre kimliklerinin sayısıdır. Yalnızca geliştirme ve doğrulama karelerinin referansları
okunmuş; test referanslarının okunması kod içinde bir kilit değişkeniyle engellenmiştir.
Geliştirme karelerinde referans sayısı 21–24 (ort. 22,7), doğrulama karelerinde 20–24
(ort. 22,2) arasındadır. Parçalı (birden fazla kopuk bölgeden oluşan) etiket bulunmamaktadır.

Görsel kontrolde iki durum belirlenmiştir: (i) Sol kenarda sınırla kesilmiş soluk bir hücre ve
bazı karelerde üst kenarda kısmen görünen nesneler referansta etiketlenmemiştir; bu nesneleri
bulan bir yöntem referansa göre fazla sayım yapmış görünür. (ii) Küçük ve yoğun görünen bazı
etiketli nesneler (ör. t020'de 22, t085'te 35) artefakt değil, `man_track.txt` soy bilgisine göre
yeni bölünmüş yavru hücrelerdir. Referans sayılar değiştirilmeden kullanılmış, bu durumlar hata
analizinde dikkate alınmıştır.

### 2.5 Veri gruplarının kullanım amacı

## 3. Yöntem ve Pipeline Tasarımı (16.3)

### 3.1 Baseline yöntemi

Baseline; gri seviyeli görüntüye Otsu eşikleme, ardından 8-komşuluk bağlı bileşen analizi
uygulayıp bileşenleri sayar. Doküman izin verdiği için sabit bir minimum nesne alanı eşiği
kullanılmış ve bu eşik yalnızca geliştirme kümesinde 0–500 piksel arasındaki 12 değer taranarak
seçilmiştir. En düşük geliştirme MAE değeri 50 pikselde elde edilmiştir (MAE 3,4; RMSE 5,04;
Bias −3,0; tam doğru oranı %10; ortalama süre ≈10 ms). Baseline bu haliyle sabitlenmiştir.

Otsu eşiği karelere göre 30–35 arasında çıkmıştır; bu değer soluk hücrelerin parlaklık
düzeyinin (ortanca ≈14) çok üstündedir. Filtreden önce kare başına 620–1003 bileşen oluşması,
eşiğin soluk hücreleri tek parça yerine çok sayıda küçük parçaya böldüğünü gösterir. Görsel
inceleme, düşük MAE'nin kısmen hataların birbirini götürmesinden kaynaklandığını ortaya
koymuştur: t067'de sayım tam doğru (21) çıkmasına rağmen bazı soluk hücreler hiç bulunamamış,
bazıları ise birden fazla parça olarak sayılmıştır. En büyük hata, soluk hücre oranının yüksek
olduğu t014'te görülmüştür (10/23, hata −13). Bu nedenle deneylerde yalnızca sayım metrikleri
değil, görsel kontrol de kullanılmıştır.
### 3.2 Uygulanan yöntemler ve alternatifler

Pipeline aşamaları sırasıyla incelenmiş, her aşamada önceki aşamaların en iyi seçimi sabit
tutulmuştur (Bölüm 4.1–4.4):

- **Yoğunluk dönüşümü:** contrast stretching, gamma (0,5), logaritmik dönüşüm, histogram eşitleme,
  CLAHE, histogram eşleme.
- **Filtre:** Gaussian (3, 5, 9, 15), median (3, 5, 9), Laplacian keskinleştirme, unsharp masking.
- **Segmentasyon:** global Otsu, sabit eşik, Triangle, Li, adaptif Gaussian eşik, blok bazlı Otsu
  (4×4 ve 8×8 blok, taban eşikli).
- **Morfoloji:** erosion, dilation, opening, closing (5×5 ve 11×11 elips), iki işlem sırası, delik
  doldurma, distance transform + watershed.
- **Son işlem:** bağlı bileşen analizi (8-komşuluk) ve minimum alan filtresi.

Bütün pipeline'lar tek bir yapılandırma sözlüğüyle tanımlanmıştır (yoğunluk, filtre türü ve boyutu,
eşik yöntemi ve parametreleri, closing boyutu, watershed `min_distance`, min. alan). Nihai
yapılandırma `final_pipeline.json` dosyasına bu biçimde kaydedilmiştir.

### 3.3 Aday pipeline'lar (en az 3)

Dört aday pipeline oluşturulmuştur. Hepsi aynı son işlem zincirini (closing 11×11 → delik
doldurma → watershed, `min_distance` = 25 → min. alan 100) paylaşır; yalnızca ön işlem veya eşik
yöntemi farklıdır. Böylece doğrulama kümesindeki fark doğrudan bu seçime bağlanabilir.

| Pipeline | Deney | Filtre | Eşik | Gerekçe | Geliştirme MAE |
|---|---|---|---|---|---|
| A | E36 | Gaussian 15×15 | Blok Otsu 8×8, taban 4 | Geliştirme kümesinde en iyi | 0,5 |
| B | E40 | Median 9×9 | Blok Otsu 8×8, taban 4 | Kenar koruyan farklı filtre | 0,6 |
| C | E41 | Gaussian 15×15 | Li (global) | Parametresiz eşik; genelleme | 3,1 |
| D | E42 | Gaussian 15×15 | Sabit eşik T=7 | En basit ve en hızlı eşik | 1,7 |

### 3.4 Nihai pipeline

**Şekil 1.** Nihai görüntü işleme pipeline'ının akış diyagramı.

## 4. Deneysel Karşılaştırmalar ve Parametre Analizi (16.4)

### 4.1 Yoğunluk dönüşümleri

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

**Şekil 2.** Orijinal görüntü ve farklı yoğunluk dönüşümlerinin sonuçları; ilgili histogramlar.

### 4.2 Filtre ve kernel karşılaştırmaları

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
### 4.3 Segmentasyon yöntemleri

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
Bölüm 4.2'deki "darboğaz eşiktir" yorumunu doğrular. En iyi sonuç blok bazlı Otsu (8×8) ile elde
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

### 4.4 Morfolojik işlemler

Morfolojik işlemler, Bölüm 4.3'ün en iyi maskesine (Gaussian 15×15 → Blok Otsu 8×8, E21)
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
hücreler silinmiş (kaçırılan 1,9), bazı hücreler parçalara ayrılmıştır (bölünme 1,1). Şekil 3a'da
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
ile hem bölünme hem birleşme sıfıra inmiştir. Şekil 3b bu adımları t041 karesinde göstermektedir:
Blok Otsu maskesinde delikli ve birbirine değen iki hücre, closing ve doldurmadan sonra tek bir
"kum saati" biçimine dönüşmekte, distance transform iki ayrı tepe üretmekte, watershed iki hücreyi
boyun noktasından ayırmaktadır.

E35, MAE 0,7 ile en iyi sonucu vermiştir. Kalan hatanın büyük kısmı "fazla" nesnelerden (1,4)
oluşmaktadır; bunlar Bölüm 2.4'te belirtilen, görüntü kenarında olup referansta işaretlenmemiş
hücreler ve gürültü lekeleridir. Bu hata morfolojik işlemlerle giderilemez. Watershed, işlem süresini
kare başına yaklaşık 13 ms'den 222 ms'ye çıkarmıştır; bu süre hâlâ pratik kullanım için kabul
edilebilir düzeydedir.

**Şekil 3a.** t041 karesinde birbirine değen iki hücre bölgesinde 14 morfolojik işlemin çıktısı
(renkli: bulunan nesneler, kırmızı çizgi: referans hücre sınırları).

**Şekil 3b.** Watershed adımları: (1) Blok Otsu maskesi, (2) closing 11×11 + delik doldurma,
(3) distance transform ve tepeler, (4) watershed sonucu.

### 4.5 Kontrollü deney tablosu

Toplam 43 deney yapılmıştır; tamamı, her deneyin 10 geliştirme karesindeki tahminleriyle birlikte
`experiments.csv` dosyasındadır. Tablo 2'de her aşamanın temsilî deneyleri ile tam pipeline
deneyleri verilmiştir. Bütün değerler geliştirme kümesinde hesaplanmıştır.

**Tablo 2.** En az 10 deneysel konfigürasyonun yöntem, parametre ve performans karşılaştırması.

| Deney ID | Yoğunluk | Filtre | Segment. | Morfoloji | Param. | Count MAE |
|---|---|---|---|---|---|---|
| E01 | Orijinal | – | Otsu | – | min. alan 50 | 3,4 |
| E02 | Contrast stretching | – | Otsu | – | p99,5 → 255 | 2,4 |
| E04 | Logaritmik | – | Otsu | – | – | 460,1 |
| E11 | Orijinal | Gaussian 15×15 | Otsu | – | – | 5,5 |
| E16 | Orijinal | Unsharp masking | Otsu | – | σ=2, miktar 1 | 2,9 |
| E17 | Orijinal | Gaussian 15×15 | Sabit eşik | – | T=7 | 1,7 |
| E19 | Orijinal | Gaussian 15×15 | Li | – | otomatik | 1,8 |
| E20 | Orijinal | Gaussian 15×15 | Adaptif Gaussian | – | blok 101, C=−10 | 1,5 |
| E21 | Orijinal | Gaussian 15×15 | Blok Otsu 8×8 | – | taban 4 | 1,2 |
| E24 | Orijinal | Gaussian 15×15 | Blok Otsu 8×8 | Erosion 11×11 | – | 0,8 |
| E30 | Orijinal | Gaussian 15×15 | Blok Otsu 8×8 | Closing 11×11 | – | 0,9 |
| E34 | Orijinal | Gaussian 15×15 | Blok Otsu 8×8 | Watershed (ham maske) | md=35 | 1,4 |
| E35 | Orijinal | Gaussian 15×15 | Blok Otsu 8×8 | Closing → doldurma → watershed | md=25, min. alan 50 | 0,7 |
| **E36 (A)** | Orijinal | Gaussian 15×15 | Blok Otsu 8×8 | Closing → doldurma → watershed | md=25, min. alan 100 | **0,5** |
| E37 | CLAHE | Gaussian 15×15 | Blok Otsu 8×8 | Closing → doldurma → watershed | E36 ile aynı | 45,7 |
| E38 | Contrast stretching | Gaussian 15×15 | Blok Otsu 8×8 | Closing → doldurma → watershed | E36 ile aynı | 51,4 |
| E39 | Orijinal | Gaussian 9×9 | Blok Otsu 8×8 | Closing → doldurma → watershed | E36 ile aynı | 1,2 |
| E40 (B) | Orijinal | Median 9×9 | Blok Otsu 8×8 | Closing → doldurma → watershed | E36 ile aynı | 0,6 |
| E41 (C) | Orijinal | Gaussian 15×15 | Li | Closing → doldurma → watershed | E36 ile aynı | 3,1 |
| E42 (D) | Orijinal | Gaussian 15×15 | Sabit eşik T=7 | Closing → doldurma → watershed | E36 ile aynı | 1,7 |
| E43 | Orijinal | Gaussian 15×15 | Blok Otsu 8×8, interpolasyonlu | Closing → doldurma → watershed | E36 ile aynı | 1,1 |

### 4.6 Sonuçların yorumlanması

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
düşmüştür. Li yöntemi soluk hücreleri kaçırmaktadır (kaçırılan 3,4; Bias −3,1). Bölüm 4.3'te
(min. alan 50) Li'nin MAE'si 1,8 iken burada 3,1'dir: Li ile kısmen bulunan soluk hücreler küçük
parçalar hâlinde kalmakta ve min. alan 100 tarafından silinmektedir. Sabit eşik (T=7) de 2,1
hücre kaçırmıştır. Her iki sonuç, Bölüm 4.3'teki temel bulguyu doğrular: bu veri setinde asıl
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

## 5. Doğrulama ve Nihai Yöntemin Belirlenmesi (16.5)

**Tablo 3.** Aday pipeline'ların doğrulama performansları (MAE, RMSE, süre).

**Şekil 4.** Örnek bir görüntü üzerinde nihai pipeline'ın aşama aşama çıktıları.

- Seçim gerekçesi
- Seçilen bütün parametreler

## 6. Bağımsız Test Sonuçları (16.6)

**Tablo 4.** On beş test karesine ait gerçek ve tahmin edilen hücre sayıları.

**Tablo 5.** Baseline ve nihai pipeline'ın test performans metrikleri (MAE, RMSE, Bias,
Exact-count Accuracy, ortalama süre).

### 6.1 Test ve doğrulama sonuçlarının karşılaştırılması

### 6.2 Zamansal analiz

**Şekil 5.** Üç test bloğu için gerçek ve tahmin edilen hücre sayılarının zamansal değişimi.

### 6.3 Başarılı, hatalı ve zorlayıcı örnekler

**Şekil 6.** Başarılı, hatalı ve zorlayıcı sayım örneklerinin karşılaştırılması.

## 7. Tartışma ve Sonuç (16.7)

- Başlıca bulgular
- Seçilen yöntemlerin avantajları ve sınırlamaları
- Hatalı sayımların nedenleri
- Bağımsız testte gözlenen genelleme başarısı
- Geliştirilebilecek yönler

## 8. Kaynakça (16.8)

1. Cell Tracking Challenge, Fluo-N2DH-GOWT1 veri seti. https://celltrackingchallenge.net/2d-datasets/
2. N. Otsu, "A Threshold Selection Method from Gray-Level Histograms," *IEEE Trans. Systems, Man, and Cybernetics*, 9(1), 62–66, 1979.

## Ekler

- Ek A: 15 test karesinin tamamının görsel sonuçları
- Ek B: Ek deney tabloları ve şekiller
