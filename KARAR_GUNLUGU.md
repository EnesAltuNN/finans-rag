# Karar Günlüğü

Her karar burada, gerekçesiyle ve varsa ölçümüyle kaydedilir. Amaç: *"Neden hibrit arama seçtin?"*
sorusunun cevabı bir cümle değil, bir ölçüm olsun.

**Kayıt biçimi:** seçenekler → takaslar → karar → gerekçe → (varsa) ölçüm.
Karar değişirse eskisi silinmez, altına "revize edildi" notu düşülür.

## Karar 1: Korpus ve değerlendirme seti — KAPANDI

| Seçenek | Artı | Eksi |
|---|---|---|
| a) Sadece FinanceBench (150 soru + PDF'leri) | Soru-cevap-kanıt üçlüsü hazır, retrieval doğrudan ölçülür, küçük ve hızlı | 150 soru istatistiksel olarak az, fine-tune verisi çıkmaz |
| b) Sadece FinQA (8.281 örnek) | Binlerce soru, fine-tune'a yeter, kanıt satırları işaretli | Kısa tablo-metin parçaları, "büyük belgede arama" gerçekçiliği düşük |
| c) İkisi birden: FinQA ile eğit, FinanceBench ile test et | Dağılım kaymasını bilerek ölçersin, ikisinin de gücü var | İki formatla uğraşmak gerekir (gün 2'de) |

**Karar: c**, sızıntı riski ciddi bir denetime tabi tutularak.

**Gerekçe:** Retrieval'ın zor kısmı (büyük belgede arama) yalnızca FinanceBench'te
var; FinQA'da bağlam soruya önceden verilmiş. c, eğitim ve test dağılımını bilerek
ayırır ve fine-tune'ın başka dağılıma aktarılıp aktarılmadığını ölçer.

**Sızıntı denetimi (1A'da yapılacak, ayrıntılar veriyle netleşecek):**

- Şirket-yıl kesişimi: FinQA ve FinanceBench'in şirket-yıl listeleri karşılaştırılır.
- Örtüşen örnekler eğitimden çıkarılır ya da örtüşme miktarı açıkça raporlanır.
- Yalnızca belge düzeyi değil, soru ve kanıt metni düzeyinde de benzerlik bakılır.
- Hem ham sayılar hem "örtüşenler çıkarıldıktan sonraki" sonuç raporlanır.
- Sonuç çok örtüşme gösterirse bu karar yeniden açılır.
- Bilinen sınır: ön-eğitilmiş modellerin SEC metinlerini ön-eğitimde görmüş olma
  ihtimali doğrulanamaz. Bu, README'de sınırlılık olarak belirtilir.

**Denetim sonucu (2026-09-30; `src/sizinti_kontrol.py`, `src/sizinti_metin.py`):**

| Düzey | Bulgu |
|---|---|
| Aynı sayfa / aynı metin | Yok. 150 sorunun kanıt sayfalarında FinQA ile kapsama ≥0.3 olan eşleşme 0 (en yüksek ~0.2 = SEC kapak sayfası standart metni) |
| Aynı rapor, farklı sayfa | 2 soru: General Mills 2019 (`04103`), Walmart 2018 (`06247`). FinQA'dan 22 örnek (train 17, dev 3, test 2) |
| Aynı şirket, farklı yıl | 12 şirket, 39 soru. Alan benzerliği, sızıntı sayılmadı |

Metin yöntemi pozitif kontrolle doğrulandı: General Mills 2019 10-K'nın bilinen
kaynak sayfalarında FinQA örnekleri 0.39-0.65 kapsama verdi, yanlış sayfalarda ~0.
Yöntem şirket eşleme tablosundan bağımsızdır. Doğrulama tek belgede yapıldı.

**Politika (karar):**

1. FinanceBench sorusu olan belgeyle aynı şirket-yıl raporundan gelen 22 FinQA
   örneği kullanılmaz (`sonuclar/finqa_haric.json`).
2. Sonuçlar iki satırda raporlanır: tüm sorular ve çıkarılmış hâl.
3. Şirket düzeyi örtüşme (39 soru) çıkarılmaz, alt küme olarak ayrıca raporlanır.
4. Kalan sınır: ön-eğitim sızıntısı doğrulanamaz.

Düzeltme notu: denetim sırasında çıkarılacak örnek sayısı önce 27 diye
yazılmıştı; doğrusu 22 (17 + 5).
