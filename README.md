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
  hero1..3.mp4/.webm     16 sn dikişsiz boomerang loop, 1600x900, sessiz
  hero1..3m.mp4/.webm    aynı loop, mobil kopya (1600x900, crf 31 / vp9 crf 42)
  preview/h1..3.mp4      galeri önizlemeleri (560px)
  preview/sa,sb.mp4      galeri önizlemeleri
frames/a/0001..0120.jpg  scroll-scrub kareleri (1280px) — masaüstü
frames/b/0001..0120.jpg
frames/am/, frames/bm/   MOBİL kare seti: 720px, q6, sadece tek numaralar (60 kare)
_raw/               ham Higgsfield çıktıları — DAĞITILMAZ (.gitignore + .vercelignore)
_tools/             npm ffmpeg/ffprobe + mobil ekran görüntüleri — DAĞITILMAZ
```

Kareler bölüme **bir buçuk ekran kala** indirilmeye başlar (IntersectionObserver);
hero'da duran ziyaretçi kare indirmez. Mobilde `STEP = 2` **ve** ayrı `frames/am|bm`
seti kullanılır (dosya adları masaüstüyle aynı, sadece tek numaralar var; mobil dosya
bulunamazsa masaüstü karesine düşülür).

**Ölçülen boyutlar (Eylül 2026):** dağıtılan toplam **21.5 MB**.
Ziyaretçi başına indirilen (1 hero + 1 kare seti):

| | en hafif | en ağır |
|---|---|---|
| Masaüstü (webm + 120 kare) | `h=1&s=a` **4.3 MB** | `h=2&s=b` **6.5 MB** |
| Mobil, webm (Android/Chrome) | `h=1&s=a` **1.1 MB** | `h=2&s=b` **1.7 MB** |
| Mobil, mp4 (iOS Safari) | `h=3&s=a` **1.3 MB** | `h=2&s=b` **2.1 MB** |

Mobilde en ağır senaryo bile **2.5 MB hedefinin altında**. (Tarayıcıdan ölçüldü:
`performance.getEntriesByType('resource')`, 375×812, h=2&s=b → 1751 KB.)

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

### Hero → mobil kopya (`hero1m` …)

Çözünürlük **düşürülmez** (telefon dikeyde videoyu cover-crop ediyor; 1280'e inince
görünen alan 1× CSS pikselin altına düşüyor, ölçüldü). Sadece bit hızı düşürülür:

```bash
ffmpeg -y -i media/hero1.mp4 -an -c:v libx264 -crf 31 -preset slow \
  -pix_fmt yuv420p -movflags +faststart media/hero1m.mp4
ffmpeg -y -i media/hero1m.mp4 -an -c:v libvpx-vp9 -crf 42 -b:v 0 -row-mt 1 \
  -deadline good -cpu-used 4 media/hero1m.webm
```

Ölçülen: mp4 528–899 KB (masaüstünde 1.1–2.0 MB), webm 296–592 KB.

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

### Scroll → mobil kare seti (`frames/am`, `frames/bm`)

Mobilde `STEP=2` zaten tek numaralı kareleri kullanıyor; o kareler 720 px'e indirilip
ayrı klasöre yazılır. **Dosya adları değişmez** — JS sadece klasör adına `m` ekler:

```bash
for n in $(seq 1 2 120); do f=$(printf "%04d" $n); \
  ffmpeg -y -i frames/a/$f.jpg -vf "scale=720:-2" -q:v 6 frames/am/$f.jpg; done
```

Ölçülen: `frames/a` 3993 KB → `frames/am` 811 KB · `frames/b` 5575 KB → `frames/bm` 1159 KB.

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
| Mobil kare seyreltme | JS `STEP = isMobile ? 2 : 1` · mobil klasör `DIR` |
| Mobil eşik (video + kare) | JS `MOB` — `(max-width:768px),(max-height:540px)` |
| Mobil CSS eşiği | `@media(max-width:900px),(max-height:540px)` |
| Mobil alt bar yüksekliği | `:root --mbar-h` (body alt payı + demo anahtarı buna bağlı) |
| DPR tavanı | `resize()` içindeki `Math.min(devicePixelRatio, 2)` |
| Palet | `:root` → `--paper --ink --leaf-*` |
| Hero okunabilirlik | `.hero-scrim` beyaz gradyan alfaları (aşağıdaki tabloya bak) |
| Mobil hero kadrajı | JS: `heroVideo.style.objectPosition` — hero başına yüzde |
| Mobil ilk ekran ölçüleri | son `@media(max-width:680px)` bloğu (punto bloğunun ARKASINDA) |
| Nav yüksekliği | JS `navHeight()` → `--navh` (hero metninin üst dolgusu buna bağlı) |
| Mobil alt bar eylemleri | `<div class="mbar">` — WhatsApp + `.mbar-randevu` (`#iletisim`) |

---

## Mobil uyum (Eylül 2026)

Site masaüstü için kurulmuştu; telefonda kullanılabilir hale getirildi.
**Masaüstü görünümü değişmedi** — değişikliklerin tamamı medya sorgusu içinde.
İçerik, metin, bölüm sırası ve sağlık sektörü kuralları aynı.

### Ne değişti

1. **Medya sorgusu iki koşullu:** `@media(max-width:900px),(max-height:540px)`.
   İkinci koşul telefonun **yatay modu** (812×375) için — orada da dokunma hedefi ve
   yazı kuralları geçerli. 1440×900 masaüstü hiçbir koşula girmiyor.
2. **Dokunma hedefleri ≥44×44:** nav hamburger, marka bloğu, hizmet kartlarındaki
   "Bu konuda yazın", ön değerlendirme seçenek çipleri, form alanları, SSS başlıkları,
   footer linkleri, iletişim kartlarındaki telefon/WhatsApp linkleri, demo senaryo
   anahtarındaki tüm düğmeler. Görsel boyut korundu, alan `min-height` + padding ile büyüdü.
3. **Yazı boyutları:** gövde metni ≥16 px, etiket/ikincil metin ≥13 px, satır yüksekliği ≥1.5.
   (Hero başlığı ikinci turda 29 px'e indirildi — aşağıdaki "İlk ekran düzeltmesi".)
   8–12.5 px'e düşen 63 yer düzeltildi (marka alt satırı, `kick` etiketleri, `stage-num`,
   BMI ölçek rakamları, `cred` anahtarları, iletişim etiketleri, footer, demo anahtarı).
   Nav marka alt satırı mobilde büyük harf yerine normal yazılıyor — 13 px'de büyük harf
   + harf aralığı 375 px'e sığmıyordu.
4. **iOS video kuralı:** `<source>` listesi kaldırıldı. Format `canPlayType` ile seçiliyor
   (Safari → her zaman mp4), **tek `src`** veriliyor, `error` olayında sıradaki adaya
   geçiliyor: `hero<N>m.<fmt>` → `hero<N>m.<alt>` → `hero<N>.<fmt>` → `hero<N>.<alt>` → CSS fallback.
   Otomatik oynatma engellenirse (iOS Düşük Güç Modu) ilk dokunuş/tık/tuşta başlıyor.
5. **Mobil medya bütçesi:** ayrı mobil hero kopyası + ayrı 720 px kare seti üretildi.
   `STEP=2` tek başına yetmiyordu: iOS mp4'e düştüğü için en ağır senaryo 3.3 MB'ye
   çıkıyordu. Şimdi en ağır senaryo **2.1 MB**. `navigator.connection.saveData` açıksa
   hero videosu ve kare seti hiç indirilmiyor (prosedürel sahne çiziliyor).
6. **Mobil menü:** zaten vardı; `aria-expanded` + `aria-label` güncellemesi, **Esc ile kapanma**,
   açılınca ilk linke odak, kapanınca odağın düğmeye dönmesi ve menü içinde Tab döngüsü eklendi.
   Arka plan kaydırma kilidi korundu.
7. **Mobil sabit alt bar:** zaten vardı; yüksekliği `--mbar-h` değişkenine bağlandı,
   `body` alt payı ve demo anahtarının konumu bu değişkenden hesaplanıyor.
   Sol/sağ/alt `env(safe-area-inset-*)` payları eklendi.
8. **Demo senaryo anahtarı** mobilde **katlanabilir**: kapalıyken tek bir 44 px'lik
   "Senaryo" hapı, alt barın 10 px üstünde (ölçüldü: panel alt kenarı 738 px, alt bar 741 px).
   Menü açıkken ve klavye açıkken gizleniyor.
9. **Klavye:** sayı girişleri `inputmode="decimal"` + `enterkeyhint="done"`, font 16 px
   (iOS yakınlaştırmasını engeller). Alan odaklanınca `body.kb` sınıfı ile alt bar ve demo
   anahtarı kalkıyor; BMI sonucu ilk göründüğünde `scrollIntoView({block:'nearest'})` ile
   görünür kalıyor.
10. **Güvenli alan:** `viewport-fit=cover` + `--pad`, `.section`, `.mbar`, `.demo-bar`,
    `.scrub-card` için `env(safe-area-inset-*)`.
11. **Yatay mod:** `min-height:600px` hero'yu 375 px yüksekliğinde ekrana sığmaz hale
    getiriyordu → kısa ekranda kaldırıldı, hero dikeyde ortalandı, başlık küçültüldü.
    Hero metni ve iki CTA ilk ekranda görünür.
12. **`senaryolar.html`** de aynı kurallara göre düzeltildi (kart etiketleri, meta satırları,
    "Bu senaryoyu aç" düğmesi, footer).

### Ölçüm (tarayıcıdan, tahmin değil)

| Genişlik | Yatay taşma | <44 px hedef | <12 px yazı | Konsol |
|---|---|---|---|---|
| 320 | yok | 0 | 0 | temiz |
| 360 | yok | 0 | 0 | temiz |
| 375 | yok | 0 | 0 | temiz |
| 390 | yok | 0 | 0 | temiz |
| 414 | yok | 0 | 0 | temiz |
| 812×375 (yatay) | yok | 0 | 0 | temiz |

Punto alt sınırı üçüncü turda gövde 15 px / etiket 12 px'e çekildi (hero küçülsün diye).
Sayfanın geri kalanı hâlâ 16 / 13 px; 15 px'in altındaki tek gövde metni yok, 13 px'in
altına inen tek yer hero etiketinin 320 px'teki hâli (12.5 px).
Konsol: favicon 404'ü de kalktı — sekme ikonu gömülü SVG olarak eklendi.

Önce: **35** adet 44 px altı hedef, **63** adet 13 px altı yazı (375×812).
Ölçüm sadece açılış durumunda değil; **mobil menü açık**, **demo anahtarı açık**,
**BMI sonucu görünür** ve **ön değerlendirme seçili** durumlarında da tekrarlandı — hepsi 0.

Ekran görüntüleri: `_tools/tmp/mobil/` (375 hero · 390 hero · 375 BMI · 375 iletişim ·
375 menü · yatay 812×375 · 1440 hero).

### İlk ekran düzeltmesi — "video görünmüyor" (ikinci tur)

Mobil uyum turunda punto ve dokunma hedefleri büyütülürken hero şişti; gerçek telefonda
metin + düğmeler ilk ekranın **%47**'sini kaplıyordu ve arkadaki videoda ne olduğu
anlaşılmıyordu. Dokunma hedefi (≥44 px), gövde (≥16 px) ve etiket (≥13 px) kuralları
bozulmadan **başlık, dolgu ve boşluklar** küçültüldü:

| Öğe | Önce | Sonra |
|---|---|---|
| `.hero h1` | 34 px / satır 1.1 | **29 px** / 1.14 (≤360 px'te 27 px) |
| Hap etiket (`.hero-kick`) | 13 px BÜYÜK HARF, **iki satır**, 55 px | 13.5 px normal yazım, **tek satır**, 34 px |
| CTA'lar | `flex:1` + `min-width:150` → "WhatsApp'tan Yaz" iki satır, blok 78 px | içerik genişliğinde, **yan yana**, 109×50 + 180×50 |
| `.hero-sub` | 16 px / satır 1.68 | 16 px / **1.5** |
| Alt dolgu | `13vh + 68px` | **`5vh + 68px`** |
| Bloklar arası boşluk | 22 / 22 / 32 / 24 px | 12 / 13 / 18 / 13 px |
| Üst yıkama (0–38%) | 0.66 → 0.56 | **0.44 → 0.14** (metin bölgesinde 0.58–0.62 korundu) |

**Metin bloğu / ilk ekran alanı** (`(kicker+h1+alt metin+CTA+not) kutusu ÷ viewport`):

| Ekran | Önce | Sonra | Hedef |
|---|---|---|---|
| 375×812 | %47.1 | **%34.9** | ≤%45 |
| 390×844 | — | **%33.9** | ≤%45 |
| 320×750 | — | **%41.7** | ≤%45 |
| 414×896 | — | **%32.1** | ≤%45 |
| 812×375 (yatay) | %60.6 | **%39.7** | ≤%45 |

**Kadraj (asıl kazanç):** dikey telefonda 16:9 video cover ile kırpılınca kadrajın
yalnızca **%37–62** bandı görünüyor; üç videoda da özne (kâse / sebzeler / sürahi)
sağda kalıp ekrandan düşüyordu. Dikey şeritlerde kenar-yoğunluğu ölçüldü —
hero1 %60–100, hero2 %50–100, hero3 %50–80 — ve mobilde `object-position`
hero1 `%76`, hero2 `%66`, hero3 `%62` olarak ayarlandı. Masaüstünde değişiklik yok.

### İlk ekran — üçüncü tur: metin yukarı, video aşağı, CTA'lar alt barda

İkinci turdan sonra metin bloğu ekranın **ortasındaydı**: üstünde navigasyonun altında
boş bir video bandı, altında da video kalıyordu — yani video ikiye bölünüyordu.
Ayrıca ekranda **iki WhatsApp düğmesi** vardı (hero + sabit alt bar). Üç değişiklik:

**1. Metin bloğu navigasyonun hemen altına alındı.** `.hero-inner` mobilde
`justify-content:flex-start` + `padding-top:calc(var(--navh) + 17px)`.
`--navh` JS ile ölçülüyor (marka adı uzayınca nav büyür; nav küçülmüş `.scrolled`
hâldeyken ölçüm yapılmaz ki kaydırırken metin zıplamasın). Ölçülen boşluk **16–17 px**.
Altta kalan alan tek parça video.

**2. Punto bir tık daha küçüldü** (alt sınır: gövde 15 px, etiket 12 px):

| Öğe | 2. tur | 3. tur |
|---|---|---|
| `.hero h1` | 29 px / 1.14 | **25 px** / 1.32 (≤360 px'te 24 px) |
| `.hero-sub` | 16 px / 1.5 | **15 px** / 1.42 |
| `.hero-kick` | 13.5 px | **13 px** (≤360 px'te 12.5) |
| Bloklar arası | 12 / 13 / 18 px | **10 / 10 / 12 px** |

**3. Hero'daki iki düğme kaldırıldı, sabit alt bar iki eylem taşıyor:**
**WhatsApp'tan Yaz** (birincil, yeşil) + **Randevu Al** (beyaz zemin, hedef `#iletisim`
— hero'daki hedefin aynısı). Hero'da aksiyon kaybolmasın diye tek satırlık
**"Nasıl çalışıyoruz →"** bağlantısı bırakıldı (`#surec`, 44 px yüksek, blok yüksekliğini
artırmıyor çünkü `.hero-note` mobilde gizlendi — aynı bilgi "Online Danışmanlık"
kartında ve iletişimdeki "Kütahya dışında mısınız?" şeridinde zaten var).

> **Telefon ikonu neden kaldırıldı:** 320 px'te alt barda kullanılabilir genişlik
> 320 − 24 (dolgu) − 20 (iki boşluk) = 276 px. Telefon düğmesi 52 px yer kaplayınca
> iki metin düğmesine 224 px kalıyor; "WhatsApp'tan Yaz" (≈165 px) + "Randevu Al"
> (≈105 px) = 270 px sığmıyor, düğmeler iki satıra kırılıyordu. Telefonsuz ikili
> 375'te 239×50 + 102×50, 390'da 254×50 + 102×50 olarak rahat oturuyor.
> `tel:` bağlantısı iletişim bölümünde (kart + "Hemen Ara" düğmesi) duruyor.

**Perde ters çevrildi:** metin artık üstte olduğu için yıkama da üste alındı —
`0.56 → 0.58 (%30) → 0.52 (%40) → 0.10 (%50) → 0.08 (%86)`, altta sayfaya geçiş için
`%96`'da 0.55 kâğıt. Yani ilk ekranın alt yarısı **neredeyse yıkamasız** video.

| Ekran | Metin bloğu / ilk ekran | Hedef | Metnin altındaki kesintisiz video |
|---|---|---|---|
| 375×812 | **%26.0** | ≤%30 | **415 px (%51.1)** |
| 390×844 | **%25.1** | ≤%30 | **447 px (%52.9)** |
| 320×750 | **%29.2** | ≤%30 | 327 px (%43.6) |
| 414×896 | **%21.7** | ≤%30 | 521 px (%58.1) |
| 812×375 (yatay) | %39.7 | — | sağda 322 px genişlik |

Yatayda alt bar yok, o yüzden hero düğmeleri orada duruyor (metin solda, video sağda).

> Yeni hero videosu gelince bu iki şeyi tekrarla: (1) şerit ölçümüyle `object-position`,
> (2) yıkama kontrastı. İkisi birbirine bağlı — kadraj kaydırınca metnin arkasındaki
> piksel değişir.

### Bilinen kalan konular

- Mobil kare seti 720 px; 3× ekranda scroll sahnesi masaüstü kadar keskin değil.
  Bilinçli takas — 4 MB yerine 0.8 MB. Daha keskin isteniyorsa `scale=900` ile yeniden üret.
- Mobil hero kopyaları crf 31; hızlı hareket eden yeni bir videoda blok oluşabilir,
  **yeni video gelince boyut/kaliteyi tekrar ölç**.
- `viewport-fit=cover` + `env()` değerleri gerçek çentikli cihazda doğrulanmadı
  (masaüstü tarayıcıda inset'ler 0 döner).
- Dikey telefonda hero videosunun öznesi doğal olarak kadrajın alt yarısında kalıyor;
  `object-position` yatayda kaydırıyor, dikeyde kırpma zaten yok (16:9 → yükseklik tam
  oturuyor). Yani özne metnin arkasına denk gelebiliyor — kontrast ölçüldü, sorun yok,
  ama daha temiz bir kadraj isteniyorsa dikey (9:16) ayrı hero çekimi gerekir.
- Uzun bir unvan (`?unvan=` ile) hap etiketini yine iki satıra kırabilir; ölçüm
  "Beslenme ve Diyet Uzmanı · Kütahya" (34 karakter) ile yapıldı. Nav yüksekliği
  `--navh` ile ölçüldüğü için hero metni yine navigasyonun altında kalır.
- `.hero-note` ("Online danışmanlık ile Türkiye'nin her yerinden…") mobilde gizli —
  ilk ekranı kısaltmak için. Masaüstünde duruyor, bilgi ayrıca "Online Danışmanlık"
  hizmet kartında ve iletişimdeki "Kütahya dışında mısınız?" şeridinde var.
  Müşteri ille de istiyorsa `.hero-cta,.hero-note{display:none}` satırından çıkar.
- Mobil alt barda `tel:` yok (yer ölçümü yukarıda). Telefonla arama iletişim
  bölümündeki karttan ve "Hemen Ara" düğmesinden yapılıyor.

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

### Mobil — yeniden ölçüldü (Eylül 2026)

Mobil kare seti ve hero kopyaları **yeniden kodlandı**, yazı boyutları büyüdü → ölçüm
tekrarlandı. Yöntem aynı: kare/video metin bölgesine cover-fit çizilir, satır yüksekliğinde
hücrelere bölünüp en karanlık hücrenin luminansı bulunur, üstüne **mobil** yıkama
gradyanının o y konumundaki alfası uygulanır (mobilde yıkama yatay değil tekdüze).

375×812'de, 60 mobil kare × 2 set ve 12 zaman noktası × 3 hero. **Hero değerleri her
turda yeniden hesaplandı** — metin yeri, punto ve yıkama değişti; aşağıdakiler
üçüncü turun (metin üstte, ters çevrilmiş perde) değerleri:

| Metin | 3 videonun en düşüğü | Video tamamen siyah olsaydı | Eşik |
|---|---|---|---|
| `.hero-kick` (hap zeminli) | 6.89:1 | 6.85:1 | 4.5 |
| `.hero h1` | 9.00:1 | 8.83:1 | 3.0 |
| `.hero-sub` | 7.71:1 | 7.34:1 | 4.5 |
| `.hero-link` (hap zeminli) | 7.08:1 | 7.06:1 | 4.5 |
| nav marka adı | 9.00:1 | 8.67:1 | 4.5 |
| nav marka alt satırı (`--hero-sub`) | 7.56:1 | 7.16:1 | 4.5 |
| `.stage-num` (hap zeminli) | 6.87:1 | — | 4.5 |
| sahne `h2` | 9.22:1 | — | 3.0 |
| sahne `p` | **7.79:1** | — | 4.5 |
| `.scrub-hint` | 11.67:1 | — | 4.5 |

`.hero-link` düz metin olarak ölçüldüğünde en kötü durumda **4.49:1** çıkıyordu
(eşik 4.5) — o yükseklikte yıkama bilerek düşük. Yıkamayı artırmak yerine
`.hero-kick` / `.scroll-cue` ile aynı kalıp uygulandı: `rgba(255,255,255,.72)` hap
zemin → videodan bağımsız 7.06:1.

**Nav marka alt satırı düzeltildi:** üst yıkama düşünce `--ink-dim` orada **2.63:1**'e
iniyordu (ölçüldü — ilk turda da sınırdaydı). Nav linklerindeki kuralın aynısı uygulandı:
şeffaf navda `--hero-sub`, sayfa kayıp zemin kâğıda dönünce `--ink-dim`. Ek yıkama yok.

**Dayanıklılık kontrolü:** video tamamen siyah olsaydı bile (L=0) hero metinleri
7.64–9.10, nav 6.34 / **4.78** çıkıyor — hepsi eşiğin üstünde. Yani düşürülen üst yıkama
yalnızca bu üç videoya göre değil, en kötü duruma göre de yeterli.

Sahne (scroll) değerleri masaüstünden yüksek: orada yıkama yatay gradyan (sağ taraf açık),
mobilde tüm genişlikte tekdüze 0.58–0.62. Yazı boyutları yalnızca **arttığı** için
hiçbir eşik yükselmedi.

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
2. **Demo anahtarını sil** — `<div class="demo-bar">` bloğu (içindeki `.demo-toggle` ve
   `.demo-body` dahil) + tüm `.demo-*` CSS kuralları (base ve mobil blok) + `demoBar()` JS.
3. **Senaryolar sekmesini sil** — nav ve footer'daki `.nav-demo` linkleri + `senaryolar.html`
   dosyası + kullanılmayan hero/frames setleri.
   **Mobil dosyaları unutma:** seçilen senaryonun `hero<N>m.mp4/.webm` ve `frames/<s>m/`
   kalır, diğerleri silinir. Mobil kopyalar silinirse site çalışır ama telefonda
   masaüstü dosyaları iner (hata değil, sadece 3–4 kat veri).
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
