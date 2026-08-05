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

`og.jpg` yoksa site bozulmaz, link sadece düz metin olarak gider.

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

Aydınlık sitede en büyük risk açık zemin + açık video. Ölçüldü (WCAG 2.1, en kötü durum
= altındaki video **tamamen siyah** varsayımı; gerçek high-key videoda hepsi daha iyi):

| Metin | Zemin | Oran |
|---|---|---|
| `--ink` #1e2a24 | `--paper` | **14.25:1** |
| `--ink-dim` #5d6b64 | `--paper` | **5.36:1** |
| `--leaf-dark` #245a3e | `--paper` | **7.72:1** |
| beyaz | `--leaf-deep` #2f6b4c (düğme) | **6.31:1** |
| `--ink` | hero metin alanı, en kötü durum | **7.76:1** |
| `--hero-sub` #2b3832 | hero metin alanı, en kötü durum | **6.39:1** |
| `--ink` | mobil hero, en kötü durum | **12.45:1** |

Hepsi 4.5:1 eşiğinin üstünde.

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
