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
aşağıdaki konuma açılmalıdır:

```
hucre_sayimi/
└── data/
    └── Fluo-N2DH-GOWT1/
        ├── 01/        (t000.tif ... t091.tif)
        ├── 01_GT/
        ├── 02/
        └── 02_GT/
```

## Random Seed

Seed = öğrenci numarasının son 5 hanesi + 2026 = 29015 + 2026 = **31041**

## Kurulum

```bash
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

*(Doğrulama aşamasından sonra eklenecek.)*

## Üretilen Dosyalar

| Dosya | İçerik |
|---|---|
| `selected_frames.csv` | Seçilen görüntülerin dosya adları, kare numaraları, veri grupları ve seed |
| `experiments.csv` | Kontrollü deneylerin konfigürasyonları ve geliştirme MAE değerleri |
| `validation_results.csv` | Aday pipeline'ların doğrulama MAE / RMSE / süre sonuçları |
| `final_pipeline.json` | Nihai pipeline'ın işlem sırası ve parametreleri |
| `test_predictions_locked.csv` | Test referansları okunmadan kaydedilen baseline ve nihai tahminler |
| `test_results.csv` | Test referansları ve performans sonuçları |

*(Dosyalar ilgili aşamalar tamamlandıkça eklenecektir.)*

## Test Güvenliği

- Sekans 02 (test havuzu) görüntüleri, pipeline kilitlenene kadar incelenmemiştir.
- Nihai pipeline parametreleri ve test tahminleri, test referansları okunmadan önce
  ayrı bir commit ile kaydedilmiştir.
- Veri setine ait görüntüler ve bunları içeren görseller repository'ye yüklenmemiştir;
  bu görseller yalnızca teknik raporda yer alır.
