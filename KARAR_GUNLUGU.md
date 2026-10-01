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

---

## Karar 2: Cevap doğruluğu metriği — KAPANDI

| Seçenek | Artı | Eksi |
|---|---|---|
| a) Tam eşleşme (exact match) | Tartışmasız, hızlı, tekrarlanabilir | Sayı biçimi farkı ("1.2 milyar" vs "1,200,000,000") yanlış sayılır |
| b) Sayısal tolerans (örn. %1 sapma) | Biçim farkına dayanıklı, hâlâ nesnel | Sayısal olmayan cevaplarda çalışmaz, eşik keyfi |
| c) LLM-judge | Her cevap tipine uyar | Yargıç model yanılır, kendi ailesinden cevapları kayırabilir, tekrarlanabilirliği düşük |

**Karar: b + a karışımı.** Sayısal cevaplarda tolerans, metin cevaplarda tam
eşleşme veya normalize edilmiş karşılaştırma. LLM-judge kullanılırsa insan
puanlamasıyla kalibre edilmeden tek başına raporlanmaz (30-50 örnekte elle kontrol,
uyum oranı raporlanır).

**Veriyle teyit edilecek:** 150 cevabın kaçı sayısal, %1 eşiği uygun mu, birim ve
ölçek (milyon/milyar) normalizasyonu nasıl yapılacak.

**Revize edildi (2026-10-01):** bu karar yetersiz çıktı; geliştirme kümesinde 99 cevabın yalnızca
34'ü kısa sayısal cevap. Katmanlı revize hâli için bkz. "Karar 4 kapanış ve Karar 2 revizesi"
bölümü ve `src/cevap_metrik.py`.

---

## Karar 3: Fine-tune modeli ve batch stratejisi — KISMEN KAPANDI

BGE-M3 (XLM-RoBERTa-large, ~568M parametre) T4'ün 16 GB'ına gradient
checkpointing ve FP16 ile sığar, ama **kontrastif eğitimde asıl mesele batch
size**: in-batch negatif sayısı doğrudan batch'e bağlı.

**Karar: tek modelle sınırlı kalınmaz.** BGE-M3 ana aday (daha önce başka projede
kullanıldı, aşinalık var), ama başka modeller de eklenerek çeşitlilik sağlanır.
Bu, "neden bu model?" sorusuna ölçümle cevap verir.

**Aday listesi (model kartlarından doğrulandı, 2026-10-01):**

| Model | Parametre | Boyut | Maks. uzunluk | Lisans | Not |
|---|---|---|---|---|---|
| BGE-M3 | kartta yok (~568M, XLM-R large) | 1024 | 8192 | MIT | Çok dilli; dense/sparse/ColBERT |
| bge-base-en-v1.5 | 0,1B | 768 | 512 | MIT | Sorguya "Represent this sentence for searching relevant passages:" öneki |
| e5-base-v2 | 0,1B | 768 | 512 | MIT | `query: ` / `passage: ` önekleri zorunlu |
| gte-base-en-v1.5 | 137M | 768 | 8192 | Apache 2.0 | `trust_remote_code=True` gerekli |

**Plan (karar):**

- **Sıfır atış baseline:** dört model de.
- **Fine-tune:** BGE-M3, bge-base-en-v1.5, gte-base-en-v1.5 (üç aile, farklı
  boyut ve bağlam uzunluğu). e5-base-v2 yalnızca sıfır atışta kalır.
- **Reranker (Karar 7 adım 3):** bge-reranker-v2-m3 (0,6B, Apache 2.0, maks. 512).

**gte uzak kod notu:** modelin kodu kendi deposunda değil, ayrı bir depoda
(`Alibaba-NLP/new-impl`; config'teki `auto_map`). Model sürümünü sabitlemek kodu
sabitlemez; yüklerken `code_revision` ile kod deposunun commit'i de sabitlenmeli ve
çalıştırmadan önce incelenmelidir. Model deposunun doğrulanacak son commit'i
(2026-10-01'de görülen): `a829fd0e060bb84554da0dfd354d0de0f7712b7f`.

**gte kod incelemesi (2026-10-01, sonuç: temiz):**

| | |
|---|---|
| Model deposu | `Alibaba-NLP/gte-base-en-v1.5`, commit `a829fd0e060bb84554da0dfd354d0de0f7712b7f`. Ağırlıklar `safetensors` (pickle yok) |
| Kod deposu | `Alibaba-NLP/new-impl`, commit `40ced75c3017eb27626c9d4ea981bde21a2662f4`, 2 dosya: `configuration.py` (145 satır), `modeling.py` (1418 satır) |
| sha256 | configuration.py `3411088045ffb8a9a0aa9936eae275896b39983a2ee5b08f091b44e6289e4fe4`, modeling.py `374670b416fcc82f081c9cd28b5fd61c2bd91bbe18eb4798fcc48a81f9c250a0` |
| Taranan riskli desenler | `subprocess`, `os.system/popen`, `eval/exec`, `__import__`, `pickle`, `socket`, `urllib`, `requests`, `open(`, `shutil`, `importlib`, `base64`, `ctypes`, `environ`, `torch.load`: **yok** (eşleşenler yalnızca yorumdaki bağlantılar) |
| Bağımlılıklar | `torch`, `transformers`; `xformers` isteğe bağlı (import hatasında `None`, ilgili ayar varsayılan kapalı) |

Sınır: tarama desen aramasıdır, satır satır tam okuma değil; ağ/dosya/süreç erişimi
olmaması ve ağırlıkların safetensors olması riski düşürür ama sıfırlamaz. Kod,
`dense_baseline.py` içinde her iki commit'e sabitlenmiştir.

**Her modelin öneki:** kodda modele özel sorgu/pasaj öneki tablosu tutulur ve
sonuçlarda belirtilir; yanlış önek modeli haksız yere düşürür.

**Hâlâ açık:**

- Her aday için azami batch size ölçümü (dummy veriyle 50 adım, bellek taşmadan
  çıkılabilen en büyük batch).
- (Kapandı) Chunk tanımı: kelime sayısı, aşağıda ve Karar 5'te.

| Sonuç | Karar |
|---|---|
| Batch ≥ 32 sığıyor | Modelle devam |
| Batch < 32 | Küçük model, büyük batch; muhtemelen daha iyi sonuç |

Not: gradient accumulation bu sorunu **çözmez**. Mikro-batch'ler ayrı ayrı ileri
geçişten geçer ve birbirlerini görmez; accumulation gradient istatistiğini büyük
batch'e benzetir, negatif havuzunu değil. Gerçekten daha çok negatif için
GradCache gibi bir teknik gerekir.

**Ölçüm 2: sıfır atış dense baseline (2026-10-01, `src/dense_baseline.py`)**

Geliştirme kümesi (99 soru, 127 kanıt), chunk 200 kelime, fp16, kartın önerdiği öneklerle,
başka ayar yok. Köşeli parantez: Recall@5 için %95 bootstrap aralığı. Gömme süresi yerel
RTX 2060 (6 GB).

| Yöntem | Ortak havuz R@5 | R@10 | R@50 | MRR | Tek belge R@5 | R@10 | R@50 | MRR | Gömme |
|---|---|---|---|---|---|---|---|---|---|
| BM25 | 0,031 [0,01-0,07] | 0,039 | 0,118 | 0,024 | 0,157 [0,09-0,23] | 0,205 | 0,370 | 0,130 | - |
| bge-base-en | 0,157 [0,10-0,23] | 0,260 | 0,488 | 0,108 | 0,480 [0,39-0,57] | 0,661 | 0,953 | 0,371 | 620 sn |
| e5-base | 0,213 [0,14-0,28] | 0,354 | 0,598 | 0,144 | 0,567 [0,48-0,65] | 0,709 | 0,937 | 0,339 | 627 sn |
| BGE-M3 | 0,197 [0,13-0,27] | 0,268 | 0,535 | 0,157 | 0,433 [0,34-0,52] | 0,606 | 0,906 | 0,324 | 1671 sn |
| gte-base-en | 0,189 [0,12-0,26] | 0,276 | 0,449 | 0,122 | 0,449 [0,36-0,54] | 0,567 | 0,890 | 0,358 | 919 sn |

**Bulgular:**

- Dört dense modelin hepsi BM25'in belirgin üstünde (aralıklar çakışmıyor).
- Dense modeller birbirinden **istatistiksel olarak ayırt edilemiyor** (Recall@5
  aralıkları büyük ölçüde çakışıyor). e5-base her iki uzayda sayısal olarak önde ama
  fark anlamlı değil. "e5 daha iyi" iddiası bu veriyle yapılamaz.
- BGE-M3 (en büyük, en yavaş: 1671 sn) sıfır atışta en iyi değil (tek belge R@5 en düşük
  ikinci). Fine-tune için ana aday olmasının gerekçesi sıfır atış sonucu değil, önceki
  deneyim ve fine-tune sonrası potansiyel olmalıdır; bu fine-tune ölçümüyle sınanacak.
- Tek belgede R@50 ≈ 0,89-0,95, R@5 ≈ 0,43-0,57: doğru sayfa ilk 50'de hemen hep var,
  ilk 5'te yarısı. Reranker için alan var (Karar 7, hipotez destekleniyor).
- Ortak havuzda tüm modeller düşük (R@5 ≈ 0,16-0,21): darboğaz doğru belgeyi bulmak.
  **Revize notu (2026-10-01):** bu çıkarım Deney 1'de ölçümle çürütüldü: künyesiz bile belge düzeyi
  isabet @5 = 0,5-0,8; baskın zorluk belge içinde doğru sayfayı bulmak.
- gte kodu sabit commit'lerle (model ve kod) çalıştırıldı.

---

## Karar 4: SEC EDGAR kapsama dahil mi — KAPANDI (2026-10-01): kapsam dışı

FinanceBench kendi PDF'leriyle geliyorsa EDGAR'a gerek olmayabilir. Bir haftalık
projede belirsiz kapsam en büyük risk.

**(Eski durum; 2026-10-01'de kapandı: kapsam dışı. Aşağıdaki "Karar 4 kapanış" bölümüne bkz.)**
**Karar:** şimdilik verilmedi. PDF'ler indirilip incelendikten sonra (1A) karar
verilecek. Varsayılan eğilim: PDF'ler yeterliyse EDGAR kapsam dışı, gerekçesi
README'de. EDGAR kuralları zaten doğrulandı: saniyede 10 istek, zorunlu
`User-Agent`, `Accept-Encoding: gzip, deflate`, `Host: www.sec.gov`.
