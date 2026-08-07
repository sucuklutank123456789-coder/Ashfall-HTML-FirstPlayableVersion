# ASHFALL — The Last Warlord

`Ashfall_Mobil_Kontroller.html` — tek dosyalık, sıfır bağımlılıklı, **tamamen prosedürel**
bir dark-fantasy aksiyon-platform oyunu. Sprite, arazi, gökyüzü, müzik ve ses efektlerinin
hepsi çalışma anında koddan üretiliyor; repoda hiç asset yok.

Bu belge oyunun grafik dilini, oynanış hissini ve atmosferini tarif eder. **Bu dosyaya
herhangi bir ekleme yapmadan önce buradaki kuralları uygula** — dosyanın bütünlüğü
yazılı olmayan bir avuç kurala dayanıyor ve bunlardan habersiz eklenen tek bir sprite
veya efekt görsel tutarlılığı anında bozar.

---

## 1. Proje kimliği

- Tek HTML dosyası: `<style>` + `<script>` gömülü, harici istek yok.
- Kod **27 numaralı bölüme** ayrılmış (`/* --- N. BAŞLIK --- */` yorumları). Yeni kod
  konusuna ait numaralı bölüme girer, dosyanın sonuna eklenmez.
- Dahili çözünürlük `VW = 480 × VH = 270` (16:9). `resize()` mümkün olduğunda **tam sayı**
  ölçek kullanır (`s >= 1 ? fl(s) : ...`), böylece piksel ızgarası bozulmaz.
- Her yerde `imageSmoothingEnabled = false`, CSS'te `image-rendering: pixelated`,
  ana context `{ alpha: false }`.
- `#wrap` üzerinde global renk düzeltmesi: `filter: saturate(0.96) contrast(1.03)`.

---

## 2. Palet — `P` (bkz. bölüm 1, ~satır 290)

**Yeni renk sabiti ekleme.** Mevcut ailelerden seç, gerekirse `mixc(a, b, t)` veya
`shade(hex, faktör)` ile türet. Aileler:

| Aile | Sabitler |
|---|---|
| Yıpranmış çelik | `stlHi #c2cad4` → `stlLt` → `stl` → `stlM` → `stlD` → `stlS` → `stlX #1a1f27` |
| Pas | `rust #6d4a30`, `rust2`, `rust3` |
| Deri | `leaHi` → `lea` → `leaD` → `leaS` |
| Kızıl kumaş | `clHi #96322f` → `cl` → `clD` → `clS #2c0d10` |
| Taş duvar | `stnHi #666c75` → `stnLt` → `stn` → `stnM` → `stnD` → `stnS` → `stnX #0e1014` |
| Yosun | `mossHi`, `moss`, `mossD` |
| Goblin | `gobHi #93a355` → `gob` → `gobM` → `gobD` → `gobS`; kan `gobBl`, `gobBlD` |
| Altın / bronz | `gldHi #e6cd8b`, `gld #b8963f`, `gldD`, `gldS`, `bronze`, `bronzeD` |
| Ateş | `fireHi #ffd39a`, `fire #e07f2c`, `fireM`, `fireD`, `ember` |
| Gökyüzü | `sky0 #05060a` → `sky5` → `horiz #3b4356` |
| Ay | `moonHi`, `moon`, `moonD`, `moonS` |
| Sis | `fogA`, `fogB`, `fogC` |
| Kan / kemik / mürekkep | `bld`, `bldD`, `bldS`, `bone`, `boneD`, `ink`, `ink2` |

### Renk dili (en kritik kural)

- **Dünya** mavi-gri, taş ve küldür. Doygunluk düşük.
- **Kızıl** yalnızca şövalyenin kumaşı (pelerin, sorguç, taht minderi, yırtık sancaklar)
  ve kandır. Başka hiçbir yerde kullanılmaz.
- **Altın yalnızca ödül ve tehdit-çözümü içindir**: parry patlaması, stagger metresi,
  düşmanın "!" uyarısı, Grullk'un tacı, iksir şifa zerreleri, ASHFALL logosu, dokunmatik
  butonun `.on` durumu. Altını dekorasyon için harcama — oyuncu altını "iyi bir şey oldu"
  diye okumayı öğreniyor.
- **Turuncu/ateş** yalnızca mangallar, uzaktaki kale yangınları ve gökyüzünün sıcaklığı.

---

## 3. Çizim primitive'leri (bölüm 0, ~satır 160–260)

`r` (dikdörtgen), `px` (tek piksel), `ra` (alfa'lı dikdörtgen), `ell` (tarama satırıyla
doldurulmuş sert kenarlı elips — **anti-aliasing yok**), `ring`, `dith`.

İki iş atı:

- **`plate(g, cx, cy, len, wid, ang, c)`** — yönlü, piksel piksel rasterize edilen levha.
  Zırh, uzuv ve silahların tamamı bundan yapılır. `c` nesnesi: `{ hi, base, sh, edge, spec,
  taper, round }`. Üç tonlu gölgeleme otomatik (`vn < -0.42` → `hi`, `vn > 0.46` → `sh`).
  Yatay aynı-renk koşular tek `fillRect` olarak yazılır; piksel başına `fillStyle` atamak
  pişirmeyi yavaşlatan şeydir.
- **`dome(g, cx, cy, rx, ry, c, lightX, lightY)`** — yönlü ışıklı elips. Miğfer kubbeleri,
  topuzlar, kalkan göbeği, dirsek/diz eklemleri.
- **`seg(g, x0, y0, x1, y1, w, c)`** — iki nokta arasına `plate` sarar; iskelet çizerken bu kullanılır.

### Yasaklar

- **Gradyan yok, blur yok.** Ton geçişleri `dith()` ile deterministik dama deseni olarak
  yapılır (bkz. gökyüzü rampası, zırh altı zincir, sis kenarları).
- Işık havuzları ve haleler **üst üste bindirilmiş sert kenarlı yarı saydam elipslerle**
  yapılır — `drawBraziers()` ve ay halosu (`drawBackground`) örnek alınmalı.

### Rastgelelik

- **Çevre detayında her zaman `hn(x, y, seed)`** (kararlı hash gürültüsü). Aynı girdi her
  karede aynı sonucu verir. Tuğla lekesi, çatlak, yosun, dağ çentiği, bayrak yırtığı —
  hepsi bunu kullanır. Burada `Math.random()` kullanılırsa grain her karede titrer.
- `Math.random()` / `rnd` / `rint` yalnızca parçacık ve yapay zekâ içindir.
- Tohumlanmış diziler gerekirse `mulberry(seed)`.

---

## 4. Işık yasası (tek kural — her şey buna uyar)

**Ay sol üstte sabittir.** `drawBackground()` içinde `fl(74 - cx*0.022), fl(20 - cy*0.02)`
konumuna çizilir ve neredeyse hiç kaymaz (parallax 0.022).

Bunun iki sonucu var ve ikisi de zorunludur:

1. Her `dome`/`plate` çağrısının ışık vektörü bu yönle uyumlu: `lightX ≈ -0.5`,
   `lightY ≈ -0.75` … `-0.9`.
2. Her sprite `finish()` son geçişinden geçer. Bu geçiş:
   - üste bakan kenarlara soluk ay rim'i ekler (`rim` çarpanı + `rimAdd` sabiti),
   - alta bakan kenarları koyulaştırır (`bot` çarpanı),
   - sol kenara hafif bir yükseltme uygular (×1.13 + 6),
   - dışına 1 px **`#07080b`** siluet konturu çeker.

Kullanılan `opt` değerleri:

| Varlık | opt |
|---|---|
| Şövalye | `{ rim: 1.23, rimAdd: 12, bot: 0.58 }` |
| Goblin | `{ rim: 1.22, rimAdd: 11, bot: 0.58 }` |
| Grullk | `{ rim: 1.17, rimAdd: 8, bot: 0.56 }` |
| Pelerin | `{ rim: 1.16, rimAdd: 8, bot: 0.72 }` |
| Hançer | `{ rim: 1.2, bot: 0.7 }` |
| Kemer (arch) | `{ rim: 1.1, rimAdd: 4, bot: 0.8, noOutline: true }` |
| Ön plan siluetleri | `{ raw: 1 }` — hiç `finish()` yok, düz `#05070a` |

---

## 5. Karakter mimarisi

**Sprite sheet yok, el çizimi kare yok.** Her karakter:

iskelet + iki kemikli IK (`ik(ax, ay, bx, by, l1, l2, bend)`) + keyframe poz tablosu
→ `plate`/`dome`/`seg` ile rasterize → `finish()` → `SPR` önbelleğine bir kez pişirilir.

Pişirme `bakeInto()` üzerinden **paylaşılan tek bir `willReadFrequently` scratch canvas**'ta
yapılır; her sprite için GPU canvas'tan piksel okumak çizmekten çok daha pahalıdır.
Ayna (`flipOf`) ve hasar tint'i (`tintOf`) önceden hesaplanır. Hasar flash'ı sprite'ın
**üstüne** 0.58 alfa ile bindirilir (`blit`'in `flash` parametresi) — böylece zırh detayı
beyaz bir silüette kaybolmaz.

| Varlık | Canvas | Origin (ayak) | Not |
|---|---|---|---|
| Şövalye | 88 × 80 | (44, 66) | `KW/KH/KOX/KOY` |
| Goblin | 62 × 58 | (31, 48) | tip ölçeği: grunt 1.00, thrower 0.92, brute 1.30 |
| Grullk | 176 × 168 | (84, 138) | `BW/BH/BOX/BOY` |
| Pelerin | 44 × 52 | (22, 7) | 9 kare (`CAPE_N`), gövdeden ayrı |

Pelerin ayrı bir sprite'tır ve gövdenin arkasından **yay ile gecikir**: hedef değer
animasyonun `cape` alanı + yatay hız + düşüş hızından hesaplanır, `capeK`/`capeV` ile
yumuşatılır (`plUpdate` sonu). Bu, hareketin ağırlığını taşıyan tek detaydır.

### Yeni animasyon eklerken

`KA` (şövalye) / `GA` (goblin) / `BA` (boss) tablolarına giriş ekle:

```js
{ n: <kare sayısı>, dur: <saniye>, loop: 1?, cape: <-1..1>?, hit: [<kare indeksleri>]?,
  f(t, p) { /* p pozunu 0..1 arası t için doldur */ } }
```

`buildWarmQueue()` tabloları gezerek yeni kareleri otomatik ısıtır — ayrıca bir şey yapma.

---

## 6. Sekiz parallax katmanı (bölüm 10, ~satır 2215+)

Arkadan öne, parallax çarpanlarıyla:

1. **Gökyüzü** — dört duraklı dikey rampa, üzerine dama dithering'i (gradyan piksel gibi
   okunsun diye) ve pişmiş duman bantları. Kamerayla neredeyse hiç kaymaz.
2. **Ay + bulutlar** — 52×52 kraterli ay (0.022), 4 bulut bandı (0.03–0.09, kendi hızlarıyla
   sürüklenir), 3 katlı sert kenarlı hale.
3. **Dağlar** — iki asimetrik sırt silueti, 0.10 ve 0.17; tepeler üst üste yığılmış çentikli
   üçgenlerden, tepe çizgisi ay ışığı alır.
4. **Kuşatılmış kale** (0.26) — tekrar eden duvar değil, **landmark**. Gedik açılmış perde
   duvar, 8 kule (üçü yıkık), çöken şato çatısı, içeriden aydınlanan dar pencereler,
   yırtık bayraklar, harabede yanan 4 ateş ve yükselen duman sütunları.
5. **Uzaktaki ordu** (0.34) — 260 mızrak, 9 sancak, 300 miğfer; hepsi neredeyse siyah.
6. **Sis şeritleri** — 4 adet (plx 0.12 / 0.20 / 0.36 / 0.55), dither'lı kenar, sinüs
   sürüklenmesi. Biri kalenin arkasında, biri ordunun üstünde, biri oyun alanının önünde.
7. **Arka plan harabeleri** (0.60–0.74, alfa 0.78) — kemerler, sütunlar, sancaklar.
8. **Ön plan siluetleri** (1.24–1.34, alfa 0.94) — direkler, moloz, ot, zincir, mızrak.
   Düz `#05070a`, hiç detay yok.

Arenada bunların önüne ayrıca `arenaCv` (880×210 büyük blok duvar, demir kuşaklar,
taçlı kafatası frizi, yırtık kızıl perdeler) ve taht çizilir.

### Post-process (`drawScreenFx`)

- Pişmiş sert vinyet; **arenada 0.42 alfa ile bir kat daha** bindirilir (kapatılmışlık hissi).
- Tarama çizgileri: her 2 px'te `#000` @0.13, her 8 px'te `#0a0f18` @0.05, toplam 0.9 alfa.
- Hasar vinyeti: `rgba(120,16,22)` radyal, 0.32 s sönümlü.
- Parry: additive `#5c4718` ekran yıkaması, 0.24 s.
- Parry temas noktasında ekran uzayında genişleyen altın haç yıldızı (`GAME.starFx`).

### Duvar ve zemin dokusu

`brickFace()` (7 px sıra yüksekliği, kaydırmalı derz, tuğla başına ton varyasyonu, içeri
çökmüş tuğlalar, harç gölgesi, benek) + `addCracks()` (rastgele yürüyüş) + `addMoss()` +
`addStain()`. Yeni bir platform veya yapı için bunları kullan, elle tuğla çizme.

---

## 7. Oynanış tuning'i — `PLC` (bölüm 15, ~satır 3313)

Souls dili: her aksiyon stamina yer, her aksiyon taahhüt gerektirir, savunma aktiftir.

**Temel**
- Can 120, stamina 100. Rejenerasyon 30/s, son harcamadan **0.42 s sonra** başlar;
  blok halindeyken %35 hızda.
- Koşu 136 px/s, accel 940, sürtünme 1400, hava accel 560.
- Yerçekimi 980, zıplama −348, maks düşüş 460.
- Coyote 0.10 s. Girdi tamponu 0.13 s (`IN.down` içinde) — **her aksiyon `IN.consume()`
  üzerinden okunur**, `IN.held` sadece yön ve guard için. (`PLC.buffer = 0.14` tanımlı ama
  kullanılmıyor; gerçek değer 0.13.)

**Saldırı**
- Hafif: 11 stamina, 13 hasar, 0.34 s. İki vuruşluk kombo: `atk1` → `atk1b` (ters el),
  pencere 0.55 s. Her vuruşta `face` yönünde +34 anlık hız.
- Ağır: 25 stamina, 30 hasar, 0.82 s. `t` 0.52–0.66 arasında gövdeyi öne sürükler (+300/s) —
  ağır vuruşun "ağırlığı" bu sürüklemeden gelir.
- İsabet halinde stamina iadesi: hafif +3, ağır +6. Agresif oynamak ödüllendirilir.

**Savunma**
- Yuvarlanma: 22 stamina, 210 px/s (sönümlenerek), 0.46 s, **i-frame 0.06–0.34 s**.
  Hançerler yuvarlanmanın içinden geçer.
- Guard basılı tutulur; **taze basış 0.19 s parry penceresi açar** (`PL.parryT`).
- Blok: `13 + 0.55 × hasar` stamina yer ve hasarın **%12'sini yine de geçirir**. Stamina
  sıfırlanırsa guard break: hasarın %30'u + 0.62 s açık kalma.
- Parry: 0.16 s hitstop, altın patlama (22 parçacık + 2 genişleyen halka + ekran yıldızı),
  +26 stamina, 0.30 s i-frame, kamera kick 4.2. **Hançeri geri sektirir** (×1.5 hız,
  goblinlere 36, boss'a 30 hasar) ve **Grullk'u anında sersemletir**.
- İksir: 4 adet, +45 can, 0.62 s taahhüt (yerde ve tam can değilken).

**Düşme**
- Çukura düşmek ölüm değil: 34 hasar, son yere basılan noktaya ışınlanma, 1.1 s i-frame.

### Vuruş hissi katmanları — hepsi her temasta birlikte çalışır

| Katman | Değerler |
|---|---|
| Hitstop | hafif 0.055 · ağır 0.11 · oyuncu hasarı 0.07 · blok 0.045 · **parry 0.16** · boss ölümü 0.35 |
| Kamera | `CAM.kick(miktar, sönüm)`; sarsıntı **karesel** sönümlenir, kick 1.5 (hafif) → 5.2 (boss smash) |
| Parçacık | `FX` içindeki hazır reçeteler — `hitLight`, `hitHeavy`, `blockSparks`, `parryBurst`, `stoneBurst`, `dustColumn`, `dustRing` |
| Tint | 0.09 s düşman flash'ı, 0.32 s oyuncu flash'ı |
| Ses | `AUD` içindeki eşleşen efekt |
| Ekonomi | isabette stamina iadesi |

**Yeni bir saldırı eklerken bu altısının hepsi bağlanmalı.** Biri eksikse vuruş "boş" hissettirir.

---

## 8. Düşmanlar ve boss

`ETUNE` (bölüm 16) — spawn'da kullanılan gerçek değerler:

| Tip | Can | Hız | Menzil | Not |
|---|---|---|---|---|
| `grunt` | 42 | 58 | 27 | Paslı satır. Her vuruşta sendeler. |
| `thrower` | 30 | 48 | 150 | 82 px'ten yakına gelirse geri çekilir, 175 px'ten uzaksa yaklaşır; arada hançer atar. |
| `brute` | 88 | 46 | 34 | **poise 1** — hafif vuruşta sendelemez. %24 ihtimalle guard alır; guard hafif vuruşları savurur, ağır vuruş geçer. |

Düşmanlar uyandıklarında altın bir "!" gösterir ve ses çıkarır; uçurumdan yürüyerek düşmezler.

### Grullk, Goblin Kralı — 640 can

**Her saldırı tek bir savunma fiiline eşlenir ve oyun bunu ekranda açıkça söyler.**
Bu oyunun merkezî tasarım kararıdır — ezber değil, okuma oyunu:

| Saldırı | Cevap | Telegraph rengi |
|---|---|---|
| `smash` — dikey çakma, geniş yer sıçraması | **ROLL** | mavi `#8fb9dc` |
| `sweep` — ayak bileği hizasında yatay savurma | **JUMP** | yeşil `#8fc47f` |
| `diag` — taahhütlü çapraz balta | **PARRY** | altın `#dcb85f` |

`drawBossTell()` renk kodlu panel + **dolan zamanlama çubuğu** + son 0.30 s'de sert
yanıp sönme + aşağıyı gösteren küçük ok çizer. Yeni bir boss saldırısı eklenirse
`BTELL` girişi de eklenmelidir.

- **Stagger** 145 (faz 2'de 185). Dolduğunda 2.7 s `STAGGERED` + **1.75× hasar**.
  Parry anında doldurur.
- **Faz 2** %50 canda: kükreme, "THE CROWN BURNS", yürüyüş 34 → 46, cooldown
  0.8–1.35 → 0.42–0.9, %34 ihtimalle ikinci saldırıyı zincirler.
- **Arena kilidi**: `PL.x > LV.arenaX0 + 44` geçilince kapı 0.42 s'de düşer (ease-in),
  kamera kilitlenir, drone boss parçasına geçer, isim kartı gösterilir.

---

## 9. Atmosfer

**Kurgu:** Ay ışığında bir gece. Ufukta kale yanıyor, ötesinde mızrak ordusu toplanıyor.
Oyuncu **zaten kaybedilmiş** bir kuşatmadan uzaklaşıp goblin kralının salonuna yürüyor.
Kül hiç durmadan yağıyor — üç derinlik grubunda, ekran uzayında, oyun hiç
durmadığı sürece hiç kesilmiyor.

**Her şey yıpranmış olmalı.** Mevcut örnekler ölçü olarak alınmalı:
kılıç ağzındaki iki çentik; kalkandaki "neredeyse hiçliğe kadar kararmış" arma;
yırtık kızıl sorguç; bir ucu kırılmış, bir taşı eksik taç; Grullk'un kemerinde ganimet
olarak asılı bir şövalye miğferi; taht minderindeki yırtıklar; arena zemininde eski
yeşil goblin kanı lekeleri. Yeni hiçbir nesne yeni görünmemeli.

**Metin dili:** tek satır, ağır, kaderci, hepsi büyük harf.
"FORGING THE RUINS" · "THE ASH SETTLES OVER A KINGDOM WITH NO KING" ·
"THE HOST MARCHES ON WITHOUT YOU" · "THE CROWN BURNS" · "YOU DIED".

**Ortam yaşamı:** mangallarda kare kare üretilen alevler ve ışık havuzları, korlar, duman,
kemerlerin ve ateşin yanında toz zerreleri, zemine yapışan sis tutamları (arenada daha
yoğun), çukurların içinde süzülen kül.

**Ses — `AUD`, %100 Web Audio sentezi, örnek dosya yok:**
filtrelenmiş gürültü patlamaları (`nz`) + tonal blip'ler (`tn`). Müzik yavaş bir drone:
saha G1 (49 Hz), boss A1 (55 Hz); lowpass kesim frekansı LFO ile süpürülür (saha 0.07 Hz,
boss 0.16 Hz); üzerine minör gamda seyrek rastgele motif — **boss b2 ekler, yani Frigyen**.
Boss drone'u daha alçak (kök ×0.5 eklenir), daha kalın ve daha yüksek (0.30 vs 0.20).

---

## 10. Mobil kontroller (bölüm 26b, ~satır 4909)

Bu dosyanın klavye sürümünden ayrıldığı yer.

**Tespit sırası:** `#touch` / `#notouch` hash override → `matchMedia('(pointer:coarse)')`
→ `ontouchstart` / `navigator.maxTouchPoints`.

**Mimari kural:** Butonlar klavyeyle **aynı aksiyon adlarına** map'lenir (`left`, `right`,
`jump`, `roll`, `guard`, `light`, `heavy`, `flask`, `pause`, `mute`). Oyun mantığı
dokunmatikten hiç haberdar değil.

**Çoklu dokunuş:** `pointerId` defteri (`TC_PTR`) + aksiyon başına referans sayacı
(`TC_CNT`). Aynı butona iki parmak değerse aksiyon çift tetiklenmez; parmak kaydırıp
başka yerde bırakılırsa `setPointerCapture` + global `pointerup`/`pointercancel`/`blur`
temizliği devreye girer. Bu defter bozulursa butonlar "yapışır" — dikkat.

**Yerleşim:**
- Sol: FLASK (üstte, hafif içeri kaydırılmış) — altında ◀ ▶.
- Sağ: GUARD/PARRY (üstte) — altında LIGHT · HEAVY — altında ROLL · JUMP.
- Sağ üst: pause (`II`) + mute.
- Boyut `--u` değişkeniyle: 62 px, `max-height:430px` → 52 px, `max-height:340px` → 44 px.
  HEAVY ×1.2, LIGHT ×1.05, FLASK ×0.88 — önem sırası boyutla anlatılıyor.
- `env(safe-area-inset-*)` her kenarda uygulanır.
- Aksiyon başına kenarlık rengi (roll/guard mavi, jump yeşil, light/heavy bakır,
  flask kızıl); basılı `.on` durumu **altın** (`#b8963f` kenar, `#e9d49c` metin, ×0.93 ölçek).

**Oyun alanına dokunuş** başlık ve bitiş ekranlarında START/RETRY olarak çalışır.
Dokunmatikte klavye ipucu satırı gizlenir ve `resize()` dolgu payını sıfırlar.

**Yeni bir aksiyon eklenirse:** `KEYMAP`'e tuş + `#tc` içine `data-act` taşıyan bir
`.tb` butonu. Başka hiçbir yere dokunmak gerekmez.

---

## 11. Yeni içerik eklerken kontrol listesi

1. Renkler `P` paletinden mi geliyor (veya `mixc`/`shade` ile ondan türetildi mi)?
2. Sprite `finish()` geçişinden geçti mi? Işık sol üstten mi geliyor?
3. Çevre detayında `hn()` kullanıldı mı (`Math.random()` değil)?
4. Sprite `SPR` önbelleğine pişiriliyor mu? **Kare başına `plate`/`dome` çağrısı olmamalı.**
5. Yeni animasyon kareleri `buildWarmQueue()` tarafından ısıtılıyor mu?
6. Yeni saldırı varsa: telegraph (boss ise `BTELL`) + hitstop + kamera kick + parçacık +
   ses + stamina ekonomisi — **altısı birden**.
7. Yeni bir oyuncu aksiyonu varsa dokunmatik butonu eklendi mi?
8. Gradyan veya blur kullanıldı mı? Kullanıldıysa `dith()` veya katmanlı sert kenarlı
   elipslerle değiştir.
