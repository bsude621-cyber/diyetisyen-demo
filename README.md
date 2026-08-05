# Diyetisyen Demo Sitesi (Kütahya) — 6 Senaryo

Tek dosyalık statik demo. Build yok, bağımlılık yok, framework yok.

**Hepsini bir arada görmek için:** [`senaryolar.html`](senaryolar.html)

---

## Çalıştırma

`file://` ile açma — `frames/` fetch'i CORS'a takılır. Local server şart:

```bash
python -m http.server 8021
```

`http://localhost:8021/senaryolar.html`

---

## Senaryolar

3 hero videosu × 2 scroll videosu = 6 kombinasyon. `index.html` URL parametresiyle seçilir:

| | Hero (`?h=`) | Scroll (`?s=`) |
|---|---|---|
| **1 / a** | Mermer tezgâh, sabah ışığı, buğu | Boş tezgâhtan hazır tabağa yürüyüş |
| **2 / b** | Mevsim sebzeleri, tahıl kâseleri | Tabağın üstünden yavaş yükselme |
| **3** | Cam sürahi, yeşil yaprak, ferah oda | — |

```
index.html?h=1&s=a   ← varsayılan
index.html?h=3&s=a   ← kontrast açısından en güvenli
```

Scroll videosuna göre sahne metinleri de değişir (`COPY` sabiti, `index.html` içinde):
`s=a` → Tanışma / Kişiye Özel Program / Takip · `s=b` → Değerlendirme / Plan / Devam

Sağ alttaki **senaryo anahtarı** ile geçiş yapabilirsin; scroll konumunu koruduğu için
aynı noktada A/B karşılaştırması yapılabilir.

---

## Müşteriye özel markalama — iki yol

### 1) URL ile (dosyaya dokunmadan)

Soğuk iletişimde en hızlısı. Tek deploy, N kişiselleştirilmiş link:

```
index.html?ad=Dyt.%20Ay%C5%9Fe%20Y%C4%B1lmaz&wa=905321234567&tel=05321234567
```

| Parametre | Ne yapar |
|---|---|
| `ad` | Marka adı (nav, footer, sekme başlığı) |
| `unvan` | Alt başlık — varsayılan "Beslenme ve Diyet Uzmanı" |
| `sehir` | Şehir adı |
| `tel` | Görünen telefon + `tel:` linki (10 hane ise başına 90 eklenir) |
| `wa` | WhatsApp numarası, ülke kodlu, sadece rakam |
| `adres` | İletişim kartındaki adres satırı |
| `saat1` / `saat2` | Çalışma saatleri |

Değerler `textContent` ile yazılır ve uzunluk sınırlıdır — HTML enjeksiyonu mümkün değil.
`h` / `s` senaryo parametreleriyle birlikte kullanılabilir; senaryo anahtarına basınca
müşteri parametreleri korunur.

**Not:** URL parametreleri `<title>` dışındaki meta/OG/JSON-LD alanlarını değiştirmez.
Kalıcı teslimde bunlar elle düzenlenmeli (aşağıya bak).

### 2) BRAND objesi ile (kalıcı teslim)

`index.html` içindeki `<script>` başındaki `BRAND` objesi. Ad, telefon, WhatsApp, adres
ve saatler oradan gelir; sayfadaki tüm alanlar otomatik dolar.

---

## Klasör yapısı

```
index.html          demo site (senaryo + marka parametreli)
senaryolar.html     6 senaryo karşılaştırma galerisi
og.jpg              WhatsApp/sosyal link önizleme kartı (1200x630) — video gelince üretilir
media/
  hero1..3.mp4/.webm   16 sn dikişsiz boomerang loop, 1600x900, sessiz
  preview/h1..3.mp4    galeri önizlemeleri (560px)
  preview/sa,sb.mp4    galeri önizlemeleri
frames/a/0001..0120.jpg  scroll-scrub kareleri (1280px)
frames/b/0001..0120.jpg
_raw/               ham Higgsfield çıktıları — DAĞITILMAZ (.gitignore + .vercelignore)
_tools/             npm ffmpeg/ffprobe — DAĞITILMAZ
```

Kareler bölüme **bir buçuk ekran kala** indirilmeye başlar (IntersectionObserver);
hero'da duran ziyaretçi kare indirmez. Mobilde `STEP = 2` ile kare seyreltilir → yarı bant.

**Ölçülen boyutlar:** dağıtılan toplam **16.3 MB**. Ziyaretçi başına indirilen
(1 hero webm + 1 kare seti): masaüstü **3.7–6.5 MB**, mobil (`STEP=2`) **2.1–3.7 MB**.
En hafif senaryo `h=1&s=a` ve `h=3&s=a`, en ağır `h=2&s=b`.

---

## Videolar nasıl işlenir

ffmpeg sistemde yok, npm ile kurulur:

```bash
mkdir _tools && cd _tools && npm init -y
npm i @ffmpeg-installer/ffmpeg @ffprobe-installer/ffprobe
node -e "console.log(require('@ffmpeg-installer/ffmpeg').path)"
```

Gelen ffmpeg 2018 sürümü — `reverse`, `delogo`, `concat` var; yeni opsiyonlar yok.
Higgsfield çıktıları tipik olarak 1920×1080, 24 fps, 8.04 sn, 193 kare geliyor.
**Önce `ffprobe` ile doğrula**; kare sayısı farklıysa aşağıdaki `191` değerini düzelt.

### Hero → dikişsiz boomerang loop

Düz `concat` yaparsan dönüş noktasında kare tekrar eder ve 1 karelik takılma olur.
Ters klipten ilk ve son kare atılmalı:

```bash
# 1) ileri
ffmpeg -y -i _raw/hero_raw_alt1.mp4 -an -vf "scale=1600:-2" \
  -c:v libx264 -crf 20 -preset veryfast -pix_fmt yuv420p _tools/tmp/f1.mp4
# 2) geri — n=0 ve n=192 atılır → 191 kare
ffmpeg -y -i _tools/tmp/f1.mp4 -an \
  -vf "reverse,select='between(n\,1\,191)',setpts=N/FRAME_RATE/TB" \
  -c:v libx264 -crf 20 -preset veryfast -pix_fmt yuv420p _tools/tmp/r1.mp4
# 3) birleştir (list1.txt: file 'f1.mp4' / file 'r1.mp4')
ffmpeg -y -f concat -safe 0 -i _tools/tmp/list1.txt -an \
  -c:v libx264 -crf 25 -preset slow -pix_fmt yuv420p -movflags +faststart media/hero1.mp4
# 4) webm
ffmpeg -y -i media/hero1.mp4 -an -c:v libvpx-vp9 -crf 36 -b:v 0 -row-mt 1 \
  -deadline good -cpu-used 3 media/hero1.webm
```

Sonuç: 384 kare = tam 16.000 sn, 1600×900.

> **hero3 farklı işlendi.** Higgsfield'in ürettiği 0. kare hafifçe sapmıştı; loop noktasında
> ortalamanın 4 katı fark veriyordu (ölçüldü: 0.604 / ort 0.147). İleri klipten ilk kare de
> atıldı → 382 kare = 15.917 sn. Yeni loop farkı 0.267 / ort 0.148 → temiz.
> Komut farkı sadece şu iki filtrede:
> ```
> ileri : -vf "select=gte(n\,1),scale=1600:-2,setpts=N/FRAME_RATE/TB"
> geri  : -vf "reverse,select=between(n\,1\,190),setpts=N/FRAME_RATE/TB"
> ```
> **Yeni video geldiğinde önce dikişi ölç**, körlemesine bu varyantı kullanma.

### Scroll → 120 kare

```bash
ffmpeg -y -i _raw/scroll_raw_alt1.mp4 -vf "fps=15,scale=1280:-2" -q:v 5 \
  -frames:v 120 "frames/a/%04d.jpg"
```

- 8 sn × 15 fps = 120 kare → JS'teki `FRAME_COUNT = 120` ile **birebir**
- `-frames:v 120` **şart** — yoksa 121. kare üretilip eşleşme kayar
- Sıfır-pad **4 hane** (`%04d` ↔ `padStart(4,'0')`)
- Video 5 sn geldiyse `fps=24` kullan (5×24=120), `FRAME_COUNT` değişmez
- **1280px / `-q:v 5`** — mimarlık demosundaki 1440/q4'ten hafif. Bu sitenin trafiği
  Instagram bio linkinden, yani mobil veri. Kare seti ~3 MB olmalı.

Watermark çıkarsa CSS ile kapatma, **kaynakta sil**:
`-vf "delogo=x=1715:y=875:w=175:h=165,fps=15,scale=1280:-2"`

### Scroll A ivmelenmişti — hareket eşitleme ile düzeltildi

`scroll_raw_alt1.mp4` prompt'taki "sabit hız" talimatına rağmen yavaş-hızlı-yavaş
(ease-in-out) geldi. Kare-arası hareket ölçüldü, çeyrekler: **1.20 / 5.54 / 6.69 / 1.38**
→ en hızlı/en yavaş oranı **5.55×**. Düz `fps=15` ile bölününce scroll'da ilk çeyrek
donuk kalıp orta bölüm fırlıyordu. Ayrıca 15 adet neredeyse-donuk kare vardı.

Çözüm — kareleri eşit **zaman** yerine eşit **görsel hareket** aralıklarıyla seç:

1. Kaynağı 60 fps'e ara-kare üreterek çıkar (havuzu 193 → 478 kareye büyütür):
   ```
   ffmpeg -y -i _raw/scroll_raw_alt1.mp4 \
     -vf "scale=1280:-2,minterpolate=fps=60:mi_mode=mci:mc_mode=aobmc:me_mode=bidir:vsbmc=1" \
     -q:v 4 _tools/tmp/a60/%04d.jpg
   ```
   (~160 sn sürer. Ara kareler artefaktsız çıktı, gözle doğrulandı.)
2. Her kare çifti için görsel farkı ölç, kümülatif hareket eğrisini kur.
3. Eğri üzerinde eşit aralıklı 120 nokta seç, o kareleri `frames/a/0001..0120.jpg` olarak yaz.

Sonuç: **120/120 benzersiz kare**, çeyrekler 3.62 / 3.97 / 4.03 / 3.66 → oran **1.11×**,
donuk kare yok. Script: bu repoda değil, `scratchpad/fixa2.js` mantığı yukarıda özetli.

Ara kare üretmeden doğrudan hareket eşitleme yaparsan (193 kareden 120 seçmek) hızlı
bölüm için kaynak yetmez ve **22 kare tekrar eder** — ölçüldü, öyle yapma.

Scroll B temiz geldi: çeyrekler 2.20 / 2.76 / 2.65 / 2.69 → oran **1.26×**, düz
`fps=15,scale=1280:-2 -q:v 6` yeterliydi.

### Videoyu kabul etmeden önce ölç

Gözle "sabit görünüyor" yetmiyor. Her yeni klip için:

- **Scroll:** kareleri 32×18 gri olarak çıkar, kare-arası farkın çeyrek ortalamalarını
  karşılaştır. **En hızlı/en yavaş < 1.6×** olmalı. Üstündeyse hareket eşitle.
- **Hero:** loop'u dairesel gez, komşu kare farkı ortalamanın %15'inin altına düşen çift
  varsa **tekrar eden kare** (takılma) demektir. Dikiş ve loop noktası ortalamanın 3
  katını geçmemeli.

### Galeri önizlemeleri

```bash
ffmpeg -y -i media/hero1.mp4 -an -vf "scale=560:-2" -c:v libx264 -crf 31 \
  -preset slow -pix_fmt yuv420p -movflags +faststart media/preview/h1.mp4
ffmpeg -y -i _raw/scroll_raw_alt1.mp4 -an -vf "scale=560:-2" -c:v libx264 -crf 31 \
  -preset slow -pix_fmt yuv420p -movflags +faststart media/preview/sa.mp4
```

> **Dosya adlandırma** — kare klasörleri `a`/`b` ise önizleme klipleri de
> `sa.mp4`/`sb.mp4` olmalı, `s1`/`s2` değil.

### OG kartı (WhatsApp link önizlemesi)

Link WhatsApp'tan gönderiliyor — kapaklı önizleme açılma oranını doğrudan etkiler.
Hero videosundan bir kare alıp 1200×630'a kırp:

```bash
ffmpeg -y -i media/hero1.mp4 -ss 3 -frames:v 1 \
  -vf "scale=1200:630:force_original_aspect_ratio=increase,crop=1200:630" -q:v 3 og.jpg
```

Üstüne isim yazmak istersen (Georgia, Windows'ta hazır):

```bash
ffmpeg -y -i og.jpg -vf "drawbox=x=0:y=430:w=1200:h=200:color=white@0.82:t=fill,\
drawtext=fontfile='C\:/Windows/Fonts/georgia.ttf':text='Dyt. Ad Soyad':\
fontcolor=0x1e2a24:fontsize=64:x=70:y=478,\
drawtext=fontfile='C\:/Windows/Fonts/arial.ttf':text='Beslenme ve Diyet Uzmani · Kutahya':\
fontcolor=0x245a3e:fontsize=30:x=72:y=556" -q:v 3 og.jpg
```

**Üretildi:** hero1'in 3. saniyesinden alınan kare, 1200×630, 35 KB — metinsiz bıraktım,
isim zaten `og:title` ile kartın yanında görünüyor. Teslimde üstüne isim yazmak istersen
yukarıdaki `drawtext` komutunu kullan (`drawtext` bu ffmpeg derlemesinde mevcut, doğrulandı).

`og.jpg` yoksa site bozulmaz, link sadece düz metin olarak gider.
**Dikkat:** `og:image` ve `canonical` mutlak URL olmalı — Vercel alan adı belli olunca
`index.html` içindeki `ORNEK-ALANADI.vercel.app` geçen 4 satırı değiştir.

---

## Ayar noktaları

| Ne | Nerede |
|---|---|
| Marka bilgileri | JS `BRAND` objesi (script'in en başı) |
| Scroll hızı / uzunluğu | `.scene { height:500vh }` (mobil `400vh`) — büyük = yavaş scrub |
| Kare sayısı | JS `FRAME_COUNT` (ffmpeg fps ile senkron olmalı) |
| Kare indirme eşiği | `IntersectionObserver` `rootMargin:'150% 0px'` |
| Metin sahne zamanları | `band()`/`bell()`: `0.14–0.26`, `0.30–0.56`, `0.58–0.78`, kart `0.80–0.92` |
| Sahne metinleri | JS `COPY` sabiti (scroll videosuna göre iki set) |
| Mobil kare seyreltme | JS `STEP = isMobile ? 2 : 1` |
| DPR tavanı | `resize()` içindeki `Math.min(devicePixelRatio, 2)` |
| Palet | `:root` → `--paper --ink --leaf-*` |
| Hero okunabilirlik | `.hero-scrim` beyaz gradyan alfaları (aşağıdaki tabloya bak) |

---

## Kontrast — ölçülmüş değerler

### Düz zeminde

| Metin | Zemin | Oran |
|---|---|---|
| `--ink` #1e2a24 | `--paper` | **14.25:1** |
| `--ink-dim` #5d6b64 | `--paper` | **5.36:1** |
| `--leaf-dark` #245a3e | `--paper` | **7.72:1** |
| beyaz | `--leaf-deep` #2f6b4c (düğme) | **6.31:1** |

### Video üstünde — beyaz yıkama nasıl ayarlandı

İlk sürümde yıkamayı "video tamamen siyah olabilir" varsayımıyla kurmuştum: metin
bölgesinde efektif alfa **0.90**'a çıkıyordu ve videolar yıkanmış görünüyordu.
Videolar gelince varsayım yerine **gerçek ölçüm** kullanıldı:

1. Her videonun/kare setinin metin bölgesi, **tüm karelerde**, x-bandına bölünerek tarandı;
   her bantta en karanlık ~satır-yüksekliği hücrenin luminansı bulundu.
2. Metin kutularının gerçek konumları tarayıcıdan alındı (`getBoundingClientRect`).
3. Gradyan durakları, her metnin kendi konumundaki en karanlık zemine karşı
   eşiği geçecek **en düşük** alfa ile kuruldu.

Sonuç: çekirdek metin bölgesinde alfa **0.90 → 0.58–0.64**, sağ taraf **tamamen açık**
(eskiden %74'te hâlâ 0.43 vardı, şimdi %80'de sıfır).

Metin bölgesinde ölçülen en karanlık video luminansları:

| | hero1 | hero2 | hero3 | frames/a | frames/b |
|---|---|---|---|---|---|
| en karanlık L | 0.052 | 0.052 | 0.427 | 0.004 | 0.034 |

Hero 3 (su) o kadar aydınlık ki teknik olarak hiç yıkama gerektirmiyor; gradyan
en karanlık senaryoya (frames/a, L=0.004) göre kurulduğu için orada bol payla geçiyor.

**Doğrulama sonucu — 5 video × 8 metin × masaüstü ve mobil, hepsi eşiğin üstünde:**

| Metin | En düşük oran | Eşik |
|---|---|---|
| `.hero-kick` (yeşil vurgu) | 6.71:1 | 4.5 |
| `.hero h1` | 5.38:1 | 3.0 (büyük metin) |
| `.hero-sub` | 6.81:1 | 4.5 |
| `.hero-note` | 7.40:1 | 4.5 |
| nav marka / linkleri | 7.30:1 | 4.5 |
| `.stage-num` | 6.67:1 | 4.5 |
| sahne `h2` | 5.11:1 | 3.0 (büyük metin) |
| sahne `p` | **4.86:1** ← en düşük | 4.5 |

Başlıklar büyük metin olduğu için WCAG eşiği 3:1; yine de hepsi 4.5'in üstünde.

**Gradyanın taşıyamadığı iki yer, yıkamayı artırmak yerine noktasal çözüldü:**

- **Yeşil vurgu metni** (`--leaf-dark`) hero1'de 0.67 alfa istiyordu — tüm frame'i
  yıkamak yerine `.hero-kick` ve `.stage-num`'a yumuşak hap zemini verildi
  (`rgba(232,242,236,.92)`) → videodan bağımsız 6.7:1.
- **Nav linkleri** videonun üstünde `--ink-dim` ile 2.36:1'de kalıyordu. Şeffaf navda
  `--ink`/600, sayfa kayıp zemin kâğıda dönünce `--ink-dim`/500 oluyor. Ek yıkama yok.
- `.scroll-cue` sağ kenarda, orada scrim artık sıfır → kendi hap zeminini taşıyor.

Yeni video geldiğinde bu ölçümü tekrarla; alfaları körlemesine kopyalama.

> **İKİ RENK METİN İÇİN KULLANILMAZ:**
> `--leaf` #4a8f6b (paper üstünde 3.70:1) ve `--warm` #d9a566 (2.11:1).
> Bunlar sadece çizgi/ikon/kenarlık/zemin. Vurgu metni için `--leaf-dark`,
> düğme zemini için `--leaf-deep` kullan.

---

## Sağlık sektörü kuralları — bu sitede BİLEREK YOK

Diyetisyenlik Türkiye'de düzenlenmiş bir sağlık mesleği; sağlık hizmeti tanıtımında
reklam kısıtları var. Aşağıdakiler kasten eklenmedi — **geri ekleme**:

1. Danışan yorumu, puan, "N mutlu danışan", yıllık deneyim sayacı
2. Öncesi/sonrası fotoğrafı (bölüm bile açılmadı)
3. Sonuç garantisi ("1 ayda 10 kilo", "garantili zayıflama")
4. Tedavi/tıbbi iddia — kullanılan doğru dil: *tıbbi beslenme tedavisi*,
   hekim tedavisinin **yerine geçmez, ona eşlik eder**
5. Yapay zekâ ile üretilmiş "diyetisyen" veya "danışan" görseli
6. Doğrulanmamış diploma/sertifika/üniversite adı — hepsi `[KÖŞELİ PARANTEZ]` yer tutucu

Footer'da genel bilgilendirme notu var. BMI aracı sonucu "tanı değildir" uyarısıyla verir.
Ön değerlendirme formu hiçbir veri **göndermez ve saklamaz** — sadece WhatsApp mesajı kurar
(KVKK yükü yok).

---

## Müşteriye teslim listesi

1. **Senaryoyu sabitle** — `HERO` / `SCRL` sabitlerine tek değer yaz, URL parametresini kaldır.
2. **Demo anahtarını sil** — `<div class="demo-bar">` bloğu + `.demo-bar` CSS'i + `demoBar()` JS.
3. **Senaryolar sekmesini sil** — nav ve footer'daki `.nav-demo` linkleri + `senaryolar.html`
   dosyası + kullanılmayan hero/frames setleri.
4. **`BRAND` objesini doldur** — ad, unvan, şehir, tel, telHref, wa, adres, saat1, saat2.
5. **`[KÖŞELİ PARANTEZ]` yer tutucularını doldur** — hepsi sayfada yeşil çerçeveli görünür,
   gözden kaçmaz. Diyetisyenden öğrenmeden doldurma:
   - Hakkımda: kısa tanıtım, `[ÜNİVERSİTE]`, `[YIL]`, `[SERTİFİKA]`, `[MESLEK ODASI]`, `[DİL]`
   - SSS: `[GÖRÜŞME SIKLIĞI]`, `[ÜCRET BİLGİSİ]`, `[ÜCRETLİ / ÜCRETSİZ]`
   - İletişim: `[AÇIK ADRES]`
6. **Meta/SEO'yu elle güncelle** — `<title>`, `description`, `canonical`, tüm `og:*`
   ve JSON-LD bloğu. Bunlar `BRAND`'den gelmiyor.
7. **`og.jpg`'yi üret** (yukarıdaki ffmpeg komutu).
8. **Fotoğrafları koy** — `<div class="about-visual">` → `<img class="about-visual" src="dyt.jpg" alt="...">`.
   Yapay zekâ üretimi insan görseli kullanma.
9. **Instagram linki** ekle (nav veya footer) — bu sektörde trafiğin çoğu oradan gelir.

**WhatsApp'tan HTML dosyası göndermek işe yaramaz** — `index.html` tek başına videoları
içermez. Her zaman canlı link gönderilir.

---

*Mekanik kaynağı: `mimar-demo` (scroll-scrub reçetesi + hero loop + senaryo sistemi).
Palet, içerik, araçlar ve markalama katmanı bu proje için yeniden yazıldı.*
