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

Seed = 29015 + 2026 = 31041

### 2.3 Seçilen görüntüler

**Tablo 1.** Rastgele seçilen görüntüler ve veri grupları.

### 2.4 Referans hücre sayıları

### 2.5 Veri gruplarının kullanım amacı

## 3. Yöntem ve Pipeline Tasarımı (16.3)

### 3.1 Baseline yöntemi
### 3.2 Uygulanan yöntemler ve alternatifler
### 3.3 Aday pipeline'lar (en az 3)
### 3.4 Nihai pipeline

**Şekil 1.** Nihai görüntü işleme pipeline'ının akış diyagramı.

## 4. Deneysel Karşılaştırmalar ve Parametre Analizi (16.4)

### 4.1 Yoğunluk dönüşümleri

**Şekil 2.** Orijinal görüntü ve farklı yoğunluk dönüşümlerinin sonuçları; ilgili histogramlar.

### 4.2 Filtre ve kernel karşılaştırmaları
### 4.3 Segmentasyon yöntemleri
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
