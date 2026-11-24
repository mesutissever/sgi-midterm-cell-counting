# Klasik Görüntü İşleme ile Hücre Segmentasyonu, Sayımı ve Zamansal Analizi

Görüntü İşleme dersi yüksek lisans ara sınav projesi.

## Projenin Amacı

Cell Tracking Challenge **Fluo-N2DH-GOWT1** veri setindeki ardışık floresan mikroskop
görüntülerinde hücrelerin yalnızca **klasik görüntü işleme yöntemleriyle** (yoğunluk
dönüşümleri, filtreleme, eşikleme, morfoloji vb.) segmentasyonu ve sayılması.
Farklı yöntem ve parametreler kontrollü deneylerle karşılaştırılır, nihai pipeline
doğrulama sonuçlarına göre seçilir ve bağımsız test görüntülerinde değerlendirilir.
Derin öğrenme veya önceden eğitilmiş modeller kullanılmamıştır.

## Veri Seti

- Veri seti sayfası: https://celltrackingchallenge.net/2d-datasets/
- Eğitim arşivi: https://data.celltrackingchallenge.net/training-datasets/Fluo-N2DH-GOWT1.zip

Veri seti lisans kuralları gereği bu repository'de **bulunmaz**. Arşiv indirilip
repository kök klasöründe aşağıdaki konuma açılmalıdır:

```
sgi-midterm-cell-counting/
├── main.ipynb
└── data/
    └── Fluo-N2DH-GOWT1/
        ├── 01/        (t000.tif ... t091.tif)
        ├── 01_GT/
        ├── 02/
        └── 02_GT/
```

## Random Seed

Seed = öğrenci numarasının son 5 hanesi + 2026 = 29015 + 2026 = **31041**

## Yazılım Gereksinimleri

Python 3.13.1 ile geliştirilmiş ve çalıştırılmıştır. Kütüphane sürümleri (`requirements.txt`):

| Kütüphane | Sürüm |
|---|---|
| numpy | 2.5.5 |
| opencv-python-headless | 5.1.0.95 |
| scipy | 1.18.1 |
| scikit-image | 0.26.0 |
| matplotlib | 3.11.2 |
| pandas | 3.1.1 |
| jupyter | 1.1.1 |
| ipykernel | 7.4.0 |

## Kurulum

```bash
git clone https://github.com/mesutissever/sgi-midterm-cell-counting.git
cd sgi-midterm-cell-counting

python -m venv .venv
# Windows
.venv\Scripts\activate
# Linux / macOS
source .venv/bin/activate

pip install -r requirements.txt
```

## Çalıştırma

1. Veri setini yukarıdaki konuma yerleştirin.
2. `main.ipynb` dosyasını Jupyter Notebook / VS Code ile açın.
3. Kernel olarak kurulumdaki sanal ortamı seçin.
4. **Run All** ile hücreleri baştan sona sırayla çalıştırın.

Notebook, ödev dokümanındaki bölüm numaralarıyla aynı sırada düzenlenmiştir.

## Nihai Pipeline

Dört aday pipeline arasından, doğrulama kümesinde (5 kare) en düşük MAE'yi veren **Pipeline D**
seçilmiştir (doğrulama MAE 0,8, RMSE 1,10, kare başına ≈ 210 ms):

```
Ham görüntü
 → Gaussian 15×15
 → sabit eşik T = 7
 → closing 11×11 (elips) → delik doldurma
 → distance transform + watershed (min_distance = 25)
 → bağlı bileşen (8-komşuluk), min. alan 100 piksel
 → hücre sayısı
```

Bütün parametreler `final_pipeline.json` dosyasındadır. Karşılaştırma için kullanılan baseline:
global Otsu + bağlı bileşen (min. alan 50).

## Üretilen Dosyalar

| Dosya | İçerik |
|---|---|
| `selected_frames.csv` | Seçilen görüntülerin dosya adları, kare numaraları, veri grupları ve seed |
| `experiments.csv` | Kontrollü deneylerin konfigürasyonları ve geliştirme MAE değerleri |
| `validation_results.csv` | Aday pipeline'ların doğrulama MAE / RMSE / süre sonuçları |
| `final_pipeline.json` | Nihai pipeline'ın işlem sırası ve parametreleri |
| `test_predictions_locked.csv` | Test referansları okunmadan kaydedilen baseline ve nihai tahminler |
| `test_results.csv` | 15 test karesinde gerçek sayı, baseline ve nihai tahmin, mutlak hatalar |
| `test_summary.csv` | Test özet metrikleri (MAE, RMSE, Bias, Exact-count Accuracy, ortalama süre) |
| `rapor/rapor.md` | Teknik raporun kaynağı (PDF sürümü ve veri seti görselleri ders teslim ortamında sunulur) |

Notebook çalıştırıldığında grafikler (yoğunluk dönüşümleri, segmentasyon ve morfoloji
karşılaştırmaları, doğrulama grafikleri, zamansal sayım grafikleri, hata analizi) hücre çıktısı
olarak üretilir. Veri seti görüntülerini içerdikleri için repository'ye yüklenen notebook'ta
çıktılar temizlenmiştir.

**Kilit koruması:** `final_pipeline.json` ve `test_predictions_locked.csv` bir kez oluşturulduktan
sonra notebook yeniden çalıştırıldığında üzerlerine yazılmaz; yeniden hesaplanan tahminlerin kilitli
dosyayla aynı olduğu doğrulanır. Test referansları, kilit dosyaları commit'lenip GitHub'a
gönderilmeden açılmaz (Bölüm 13).

## Test Güvenliği

- Sekans 02 (test havuzu) görüntüleri, pipeline kilitlenene kadar incelenmemiştir.
- Nihai pipeline parametreleri ve test tahminleri, test referansları okunmadan önce
  ayrı bir commit ile kaydedilmiştir.
- Veri setine ait görüntüler ve bunları içeren görseller repository'ye yüklenmemiştir;
  bu görseller yalnızca teknik raporda yer alır.
