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
### 3.3 Aday pipeline'lar (en az 3)
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

**Şekil 3.** Farklı segmentasyon ve morfolojik işlem çıktılarının karşılaştırılması.

### 4.5 Kontrollü deney tablosu

**Tablo 2.** En az 10 deneysel konfigürasyonun yöntem, parametre ve performans karşılaştırması.

### 4.6 Sonuçların yorumlanması

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
