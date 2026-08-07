# CLAUDE.md — Ashfall: The Last Warlord

Bu repo tek bir oyun dosyası barındırıyor: **`Ashfall_Mobil_Kontroller.html`** (~4995 satır, ~220 KB).
Bu dosya kendi kendine yeten (self-contained) bir HTML5 canvas oyunudur: **hiçbir dış asset yok** —
sprite'lar, arka planlar, yazı tipi, ses, müzik dahil **her şey çalışma anında prosedürel olarak üretilir**.

Bu belge kodun mimarisini, grafik dilini, oynanış tasarımını ve atmosferini kaydeder.
Bu dosyaya dokunacak her değişiklik burada tanımlanan kurallara uymalıdır.

---

## 1. Kimlik: Oyun Nedir?

**ASHFALL — THE LAST WARLORD**

Yan görünüş (2D side-scroller), pixel-art, **karanlık fantezi aksiyon** oyunu.
Tür olarak "Souls-like 2D": tek hayat, stamina yönetimi, telegraph'lı düşman saldırıları,
i-frame'li yuvarlanma, parry ödülü, dolu şişe (flask) ile iyileşme ve tek bir büyük boss.

**Anlatı çerçevesi:** Kuşatılmış, düşmüş bir krallığın külleri. Oyuncu son şövalyedir;
kalenin kalıntıları boyunca soldan sağa ilerler, goblin sürüsünü keser ve sonunda
kapalı bir arenada **GRULLK, THE GOBLIN KING** ile hesaplaşır.
Zafer metni tonu belirler: *"THE ASH SETTLES OVER A KINGDOM WITH NO KING"*.
Ölüm metni: *"THE HOST MARCHES ON WITHOUT YOU"*.

Dosya adındaki "Mobil Kontroller" (bölüm 26b), oyuna sonradan eklenen **çok parmaklı
dokunmatik kontrol katmanını** işaret eder; klavye/fare mantığını hiç değiştirmeden
aynı aksiyon adlarına bağlanır.

---

## 2. Dosya Yapısı — Numaralı Bölümler

Kod, kaynak içinde numaralandırılmış banner yorumlarıyla ayrılmıştır. Haritası:

| # | Bölüm | Satır | İçerik |
|---|---|---|---|
| 0 | CANVAS + LOW LEVEL PIXEL PLUMBING | 139 | `VW/VH`, çizim primitifleri (`r`, `px`, `ra`, `ell`, `ring`, `plate`, `dome`, `dith`), renk yardımcıları, deterministik rastgelelik |
| 1 | PALETTE | 287 | `P` — tüm oyunun tek renk sözlüğü |
| 2 | INPUT | 324 | `KEYMAP`, `IN` (held/hit/rel + input buffer) |
| 3 | AUDIO | 361 | `AUD` — saf Web Audio sentezi, SFX + drone müzik |
| 4 | SPRITE FINISHING PASS | 519 | `finish()` rim-light/outline, `tintOf`, `flipOf`, `bakeInto`, `SPR` cache, `blit` |
| 5 | PARTICLE SYSTEM | 619 | `PS` havuzu (pooled), `FX` parçacık reçeteleri |
| 6 | THE KNIGHT | 966 | IK iskeleti, kılıç/kalkan/miğfer/gövde çizimi, pelerin |
| 7 | KNIGHT ANIMATION TABLE | 1315 | `KA` — poz fonksiyonlu animasyon sözlüğü |
| 8 | GOBLINS | 1490 | `GTYPE` 3 varyant, `drawGoblin`, `GA` animasyon tablosu, fırlatma hançeri |
| 9 | GRULLK, THE GOBLIN KING | 1824 | Boss çizimi (topuz, taç, kral kafası), `BA` animasyon tablosu |
| 10 | THE WORLD | 2215 | 8 paralaks katman + props + sis + vignette/scanline |
| 11 | LEVEL LAYOUT | 2872 | `LV` — platformlar, çukurlar, mangal, dekor, spawn'lar |
| 12 | CAMERA | 2970 | `CAM` — takip, look-ahead, kilit, shake |
| 13 | WORLD RENDERING | 2997 | `drawBackground`, `drawPits`, `drawPlatforms`, `drawBraziers`, arena, ambient parçacık yatakları |
| 14 | EXTRA KNIGHT ANIMATIONS | 3279 | `KA.atk1b` (kombo ters vuruş), `KA.drink` |
| 15 | THE PLAYER | 3310 | `PLC` tuning, `PL` durum, `plUpdate`, `plStrike`, `damagePlayer` |
| 16 | GOBLINS (mantık) | 3686 | `ETUNE`, `EN`, `enemyUpdate` AI, `enemyHurt` |
| 17 | THROWN DAGGERS | 3873 | `DAG` — parry ile sahibine geri döner |
| 18 | GRULLK — boss logic | 3935 | `BATK`, `BOSS`, `bossUpdate`, `bossImpact`, stagger/stun/faz |
| 19 | THE GATE | 4220 | Arena kilidi ve düşen demir kapı |
| 20 | PIXEL TYPE | 4257 | `FONT` 5×7 bitmap, `text/textC/textSh` |
| 21 | GAME STATE | 4307 | `GAME`, sprite ısıtma kuyruğu (`warm`), `resetRun` |
| 22 | ENTITY RENDERING | 4354 | `drawPlayer/drawEnemies/drawBoss/drawBossTell/drawDaggers/drawGate` |
| 23 | HUD | 4524 | Dövme demir paneller, can/stamina bar, flask, boss plakası |
| 24 | SCREEN-SPACE EFFECTS | 4611 | Hasar vinyeti, parry flaşı, yıldız FX, vignette, scanline |
| 25 | TITLE / VICTORY / DEFEAT | 4658 | Logo, başlık ekranı, bitiş ekranları, pause |
| 26 | MAIN LOOP | 4759 | `ambientTick`, `updatePlay`, `renderPlay`, `frame` |
| 26b | DEVICE PROFILE + ON-SCREEN CONTROLS | ~4915 | `TOUCH` tespiti, `DEV` cihaz profili, `viewW/viewH`, dokunmatik pad'ler, çok parmak sayacı |
| 27 | BOOT | ~4995 | `resize` (cihaz-pikseli ölçekleme), `sizeTouchPads`, `updateOrientationGate`, props/arena/warm inşası, ilk `requestAnimationFrame` |

> Satır numaraları yaklaşıktır ve düzenlemelerle kayar; bölüm banner'ları otoritedir.

---

## 3. Grafik Tarzı — "32-bit el yapımı pixel art"

### 3.1 Temel kurallar
- **Dahili çözünürlük sabit: `VW=480 × VH=270`.** Canvas CSS ile ölçeklenir; ölçekleme
  kuralları bölüm 8'de. CSS'te `image-rendering: pixelated` ve
  `ctx.imageSmoothingEnabled = false` her yerde zorunludur.
- **Anti-aliasing yok.** Her koordinat `fl()` ile ızgaraya oturtulur. Yumuşak geçişler
  blur ile değil, **dither** (`dith`) ve katmanlı yarı-saydam sert kenarlı şekillerle yapılır
  (ör. ay halesi = iç içe 3 `ell` çağrısı, `globalAlpha` 0.02·i).
- **Tek ışık kaynağı: sol-üstteki ay.** `drawBackground`'da ay ekranın sol üstünde sabitlenir
  (`74 - cx*0.022`, `20 - cy*0.02`) ve oyundaki *bütün* rim-light bu yönle uyumludur.
  `dome()` varsayılan ışık vektörü `(-0.5, -0.75)`, `plate()` üst kenarı `hi`, alt kenarı `sh` yapar.

### 3.2 Sprite üretim hattı (çok önemli)
Karakterler el ile çizilmiş PNG değildir; **küçük bir iskeletten prosedürel olarak pişirilir**:

1. **`ik(ax,ay,bx,by,l1,l2,bend)`** — iki kemikli ters kinematik; kalça→ayak, omuz→el.
2. **`seg(g, x0,y0, x1,y1, w, mat)`** → **`plate()`** — herhangi bir açıda piksel-piksel
   rasterize edilmiş yönlü plaka. Zırh, uzuv, silah hep budur. `taper`, `round`, `edge`,
   `spec` opsiyonlarıyla konikleşme/yuvarlanma/keskin ağız/parlama verilir.
3. **`dome()`** — yönlü gölgeli elips (miğfer kubbesi, dizlik, topuz, kalkan göbeği).
4. **Malzeme setleri** (`{hi, base, sh, edge}`): `M_NEAR` (öndeki uzuvlar, açık çelik),
   `M_FAR` (arkadaki uzuvlar, koyu çelik — derinlik ayrımı bu ikiliyle yapılır),
   `M_DARK`, `M_LEA` (deri), `M_CLOTH` (kızıl kumaş), `M_GOLD`, goblin için `M_SKIN`/`M_IRON`/`M_RUST`.
5. **`finish()` bitirme geçişi** — *her* sprite aynı muameleyi görür:
   üst kenar piksellerine ay rim'i (`×1.30 + 20`), alt kenara gölge (`×0.60`),
   sol kenara hafif ışık (`×1.13 + 6`) ve **1px koyu siluet konturu** (`#07080b`).
   Bu, tüm karakterlerin tek bir ışık mantığını paylaşmasını sağlar.
6. **`bakeInto()`** tek bir paylaşılan `willReadFrequently` scratch canvas kullanır
   (GPU canvas'tan piksel okumak pahalı olduğu için), sonuç `SPR.cache`'e girer.
   `flipOf()` ile ayna kopyası, `SPR.flash()` ile hasar flaşı tint'i tembel üretilir.

**Kural:** `plate`/`dome`/`finish` sadece **pişirme sırasında** çağrılır, asla frame içinde değil.
Kare başına yol sadece önbelleklenmiş `drawImage` (`blit`) ve `fillRect` içerir.

### 3.3 Sprite ölçüleri (ayak/taban orijinli)
| Varlık | Tuval | Orijin (ox, oy) |
|---|---|---|
| Şövalye | `KW×KH = 88×80` | `KOX,KOY = 44,66` |
| Pelerin (ayrı sprite, 9 kare) | `CW×CH = 44×52` | `COX,COY = 22,7` |
| Goblin | `GW×GH = 62×58` | `GOX,GOY = 31,48` |
| Grullk (boss) | `BW×BH = 176×168` | `BOX,BOY = 84,138` |
| Hançer (8 dönüş karesi) | `20×20` | merkez |

Pelerin gövdeden **ayrı** sprite'tır; böylece hareketi gecikmeli (lag) takip edebilir
(`PL.capeK`/`capeV` yaylı sönümleme, `KA[anim].cape` hedefi + hız/düşüş katkısı).

### 3.4 Dünya: 8 paralaks katman
`drawBackground()` sırasıyla:
1. **Gökyüzü** (`skyCv`) — dikey gradyan + bantlı dither + pişmiş duman şeritleri.
2. **Ay + bulutlar** — `moonCv` kraterli; 4 bulut bandı kendi `sp`/`plx` değerleriyle kayar.
3. **Dağlar** — `mtFar` (plx 0.10), `mtNear` (plx 0.17); çentikli, asimetrik zirveler.
4. **Kuşatılmış kale** (`castleCv`, plx 0.26) — yıkık kuleler, gedikler, yanan ateşler,
   yükselen duman, yırtık bayraklar. Tekrarlayan duvar değil, **landmark**.
5. **Uzaktaki ordu** (`armyCv`, plx 0.34) — mızrak ormanı, sancaklar, miğfer kütlesi.
6. **Savaş alanı sis şeritleri** (`fogs[0..3]`) — farklı paralaks/hız/salınımla katmanlar arasına serpilir.
7. **Orta zemin harabeleri** (`LV.bg`: `arch`, `pillar`, `banner`, plx 0.60–0.74).
8. **Ön plan siluetleri** (`LV.fg`: `fgPost`, `fgRubble`, `fgVeg`, `fgChain`, `fgSpear`, plx 1.24–1.34) —
   neredeyse tamamen siyah (`#05070a`), `{raw:1}` ile `finish()` uygulanmadan pişer.

Üstüne **vignette** (`vigCv`; arenada ikinci kez %42 daha) ve **scanline** (`scanCv`) geçirilir.

### 3.5 Masonluk / doku yardımcıları
`brickFace()` (kaydırmalı tuğla sıraları, harç gölgesi, benek), `addCracks()`,
`addMoss()`, `addStain()`. Arena duvarı (`buildArena`) daha iri bloklar, demir kuşaklar,
taçlı kafatası frizi, yırtık kızıl perdeler ve tabanda yeşil goblin lekeleri kullanır.

### 3.6 Yazı
Gömülü **5×7 bitmap font** (`FONT`, her karakter 7 satırlık bit maskesi).
`text/textC/textSh` ölçek (`sc`) ve harf aralığı (`sp`) alır. Sadece büyük harf.
Logo, üst yarısı ay ışığını yakalayan (`clip` ile iki tonlu) altın harflerle çizilir.

---

## 4. Palet (`P`) — Kasvetli ve Dar

Tek bir sözlükte toplanmıştır; **yeni renk eklemeden önce buradan biri kullanılmalı**.

- **Yıpranmış çelik:** `stlHi #c2cad4` → `stlX #1a1f27` (7 kademe)
- **Pas:** `rust #6d4a30`, `rust2`, `rust3`
- **Deri:** `leaHi #8a6440` → `leaS #261a10`
- **Kızıl kumaş (pelerin, sancak, perde):** `clHi #96322f`, `cl #6f2224`, `clD`, `clS`
- **Taş:** `stnHi #666c75` → `stnX #0e1014`; yosun `moss/mossD/mossHi`
- **Goblin teni:** `gobHi #93a355` → `gobS #2b3416`; goblin kanı `gobBl #42591f`
- **Altın/bronz:** `gldHi #e6cd8b`, `gld #b8963f`, `gldD`, `gldS` — **parry ve kraliyet rengi**
- **Ateş:** `fireHi #ffd39a`, `fire #e07f2c`, `fireM`, `fireD`, `ember`
- **Gökyüzü/atmosfer:** `sky0 #05060a` → `horiz #3b4356`, `moon*`, `fogA/B/C`
- **Kan:** `bld #7d1520`, `bldD`, `bldS`
- **UI:** `ink #07080b`, `ink2`, `bone`, `boneD`

**Renk anlamları (sözleşme):** altın = parry/ödül/kraliyet · kızıl = oyuncu kimliği ve can ·
yeşil = goblin · turuncu = ateş/mangal · mavi-gri = çelik ve soğuk ay ışığı.

---

## 5. Atmosfer — Nasıl Bir His Yaratılıyor?

Atmosfer tek bir efektten değil, **sürekli çalışan küçük sistemlerin toplamından** doğar:

- **Sürekli düşen kül.** `spawnAsh()` üç derinlik grubu üretir (deep/mid/fore),
  farklı hız, renk, saydamlık ve katmanla. Ekran uzayında (`world:false`) yaşar, hep hareket eder.
- **Toz zerrecikleri** (`spawnMote`) mangal ve kemerlerin çevresinde belirir; arenada yoğunlaşır.
- **Yerde sürünen sis** (`spawnFogWisp`) zemin yüksekliğini takip eder; arena kilitliyken sıklaşır.
- **Mangallar** (`LV.braziers`, 11 adet) — titreyen alev sütunu (`sin` çift frekans),
  4 katmanlı sıcak ışık havuzu, kıvılcım (`FX.ember`) ve ince duman (`FX.smoke`).
- **Çukurlar** (`drawPits`) — katman katman koyulaşan karanlık, ıslak çıkıntılar,
  dipte demir kazıklar, içine süzülen kül.
- **Müzik = drone.** Melodi yoktur; kök nota üzerine detune'lu sinüs/testere katmanları,
  LFO ile süpürülen alçak geçiren filtre. Sahada `root = 49 Hz` (G1), boss'ta `55 Hz` (A1)
  + bir oktav alt takviye. Üstüne **seyrek motif**: `setInterval` ile rastgele skala
  notaları (saha 3400 ms / boss 2100 ms; boss skalası yarım-ton içerir → daha tehditkâr).
- **Ekran efektleri:** hasarda kırmızı kenar vinyeti (`dmgCv`), parry'de altın additive flaş
  ve temas noktasında piksel yıldız, sürekli vignette + scanline.

**Ton kuralı:** hiçbir şey parlak, doygun veya "temiz" değil. Metal paslı, taş çatlak ve yosunlu,
bayraklar yırtık, kale gedikli, ordu uzakta ve ilgisiz. Oyuncu bu dünyada **son kalan**dır.

---

## 6. Oynanış Tarzı

### 6.1 Kontroller
| Aksiyon | Klavye | Fare | Dokunmatik |
|---|---|---|---|
| Hareket | `A/D` veya `←/→` | — | `◀ ▶` |
| Zıplama | `W` / `↑` / `Space` | — | `JUMP` |
| Yuvarlanma | `S` / `↓` | — | `ROLL` |
| Hafif saldırı | — | Sol tık | `LIGHT` |
| Ağır saldırı | — | Sağ tık | `HEAVY` |
| Guard / Parry | `Q` (basılı tut; **taze basış** = parry) | — | `GUARD` |
| Flask | `E` | — | `FLASK` |
| Başlat / Pause / Mute / Restart | `Enter` / `Esc`,`P` / `M` / `R` | — | ekrana dokun / `II` / `MUTE` / `QUIT` (yalnızca pause'da görünür) |

`IN` nesnesi `held` (basılı), `hit` (bu karede kenar) ve **`buf` input buffer'ı** (0.13 s) tutar.
Aksiyonlar `IN.consume(act)` ile tüketilir → bir tık, kısa bir animasyon kilidinden hemen sonra da işler.

### 6.2 Oyuncu tuning (`PLC`) — otorite değerler
```
runSpeed 136   accel 940    friction 1400   airAccel 560
gravity  980   jumpV -348   maxFall 460     gövde w13 h44
maxHp 120      maxStam 100  stamRegen 30/s  stamDelay 0.42 s
roll: speed 210, dur 0.46 s, cost 22, i-frame penceresi 0.06–0.34 s
maliyetler: light 11, heavy 25, block 13 + gelen hasar×0.55
parryWindow 0.19 s   coyote 0.10 s
flask: 4 adet × 45 can
```
Hasar: **light 13**, **heavy 30**. İsabetli vuruş stamina geri verir (light +3, heavy +6).

### 6.3 Savaş döngüsü
- **Kombo:** `light` → `atk1`; 0.55 s içinde ikinci `light` → `atk1b` (ters el savurma).
  Üçüncüsü tekrar `atk1`'e döner (2'li zincir).
- **Ağır saldırı** (`atk2`, 0.82 s) uzun, taahhütlü; darbe anında gövdeyi öne sürükler
  (`t 0.52–0.66` arası `+300·dt` ivme). Menzili daha uzun (`x1 40` vs `32`).
- **Yuvarlanma** yönü kilitler, hız eğrisi sönümlenir, ortada i-frame penceresi vardır.
- **Guard (`block`)** basılı tutulur; hareket hızını 42'ye düşürür, stamina rejenerasyonunu %35'e indirir.
  Bloklanan darbe: stamina yer + can'ın **%12'si** sızar. Stamina biterse **guard break**
  (`amount×0.30` can, 0.62 s savunmasızlık).
- **Parry:** guard'a *taze basış* 0.19 s'lik pencere açar. Başarılıysa:
  altın patlama + yıldız, 0.16 s hitstop, ekran sarsıntısı, **+26 stamina**, 0.30 s i-frame.
  - Goblin saldırısını parry → goblin sendeler ve 6 hasar alır.
  - **Fırlatılan hançeri parry → hançer düşmana geri döner** (`friendly`, 1.5× hız,
    goblinlere 36, boss'a 30 hasar). `GAME.reflectKills` sayacı bunu ödüllendirir.
  - **Boss saldırısını parry → Grullk anında STUN olur.** Oyunun en büyük ödülü budur.
- **Hasar akışı** tek kapıdan geçer: `damagePlayer(amount, srcX, kind, contactY)` →
  `'dodge' | 'parry' | 'block' | 'break' | 'hit'` döner. Sadece **önden** gelen darbeler bloklanabilir.
- **Hitstop** vurgu aracıdır: light 0.055 s, heavy 0.11 s, parry 0.16 s, oyuncu hasarı 0.07 s,
  boss ölümü 0.35 s. Hitstop sırasında `WT` (dünya saati) çalışmaya devam eder — ambiyans donmaz.

### 6.4 Düşmanlar (`ETUNE` otoritedir)
| Tip | HP | Hız | Menzil | Hasar | Poise | Karakter |
|---|---|---|---|---|---|---|
| `grunt` | 42 | 58 | 27 | 12 | 0 | Kapüşonlu leş toplayıcı, paslı satır |
| `thrower` | 30 | 48 | 150 | 10 | 0 | Sıska, hançer bandolyeri; **mesafe korur** (82'den yakınsa geri çekilir, 175'ten uzaksa yaklaşır) |
| `brute` | 88 | 46 | 34 | 20 | 1 | Boynuzlu demir miğfer, ağır balta; %24 ihtimalle **guard** alır, poise sayesinde hafif vuruşlarda sendelemez |

AI durum makinesi: `idle` (devriye / kovalama) → `wind` (telegraph) → `atk` → cooldown.
`thrower` için `throwWind` → hançer. Düşmanlar **çukura veya kenardan düşmez**
(`groundAt(ahead)` kontrolü ile dururlar). Fark edildiklerinde `!` işareti ve `gobAlert` sesi çıkar.
Brute'un guard'ı **sadece hafif** vuruşları savurur; ağır vuruş deler.

### 6.5 Boss: Grullk

```
hp 640   stagger eşiği 145 (faz 2'de 185)   stun süresi 2.7 s
stun sırasında alınan hasar ×1.75
faz 2 tetikleyici: hp ≤ %50 → roar + "THE CROWN BURNS"
```

Üç saldırı ve **her birinin tek doğru cevabı** vardır — `BTELL` tablosu bunu ekranda söyler:

| Saldırı | Süre | Hasar / Knockback | Telegraph metni | Doğru cevap |
|---|---|---|---|---|
| `smash` (tepeden çakma) | 1.55 s | 30 / 190 | **ROLL** (mavi) | içinden yuvarlan |
| `sweep` (alçak yatay savurma) | 1.30 s | 24 / 240 | **JUMP** (yeşil) | üzerinden atla |
| `diag` (sıçramalı çapraz) | 1.25 s | 26 / 210 | **PARRY** (altın) | savuştur |

`drawBossTell()` başın üstüne bir plaka çizer: aksiyon adı + **darbe anına dolan zamanlama çubuğu**;
son 0.30 s'de yanıp söner. Mesafeye göre saldırı seçimi (`<50` yakın, `<96` orta, `<190` uzak);
faz 2'de %34 ihtimalle **zincir saldırı** yapar ve cooldown kısalır (0.42–0.9 s vs 0.8–1.35 s).

Boss dövüşü tetikleyicisi: `PL.x > LV.arenaX0 + 44` → `startBossFight()` →
kamera kilitlenir (`CAM.lock`), **demir kapı düşer** (`GATE`), boss müziği başlar,
"THE LAST WARLORD / GRULLK" başlığı belirir. Oyuncu artık arenaya hapsolmuştur.

### 6.6 Seviye
Tek, kesintisiz bir seviye: **3640 px genişlik**, zemin `GY = 200`.
- 13 platform: `ground` (ana zemin), `ledge` (dar çıkıntı, altı yosunlu), `arena` (daha koyu, iri bloklu).
- 3 çukur (`pits`) — düşmek **34 can** götürür, oyuncuyu `lastGround`'a 1.1 s i-frame ile geri koyar.
- 15 goblin spawn'ı (`LV.spawns`), 11 mangal, 25 dekor, 21 arka plan yapısı, 15 ön plan silueti.
- Zorluk eğrisi: grunt → ledge'den thrower → ilk brute (x≈1420) → karışık gruplar → kapı → boss.

### 6.7 Akış / mod makinesi
`GAME.mode`: `boot` → `title` → `play` → (`paused`) → `victory` | `defeat` → `title`/`play`.
Başlık ekranında sprite'lar **artımlı olarak pişirilir** (`warmTick(7)` = kare başına 7 sprite),
"FORGING THE RUINS %" göstergesiyle; hazır olunca `PRESS ENTER`.
Zafer ekranı istatistik gösterir: öldürülen goblin, **çevrilen hançer**, kalan flask.

---

## 7. Mobil / Dokunmatik Katman (Bölüm 26b)

- **Tespit (`TOUCH`):** `location.hash === '#touch'` (zorla aç) / `'#notouch'` (zorla kapat) →
  `matchMedia('(pointer:coarse)')` → *hiç ince işaretçi yoksa* (`!(any-pointer:fine)`)
  `ontouchstart`/`maxTouchPoints`. İkinci adım kasıtlıdır: **dokunmatik ekranlı bir dizüstü**
  `maxTouchPoints > 0` bildirir ama klavyesi ve faresi vardır — ona başparmak pad'i verilmez.
- Pad'ler HTML'de `<div id="tc">` içinde durur, `data-act` özniteliğiyle **klavyeyle aynı
  aksiyon adlarına** bağlanır (`left/right/jump/roll/light/heavy/guard/flask/pause/mute`).
  Bu yüzden oyun mantığında dokunmatiğe özel tek bir satır bile yoktur.
- **Çok parmak:** `TC_PTR` (pointerId → aksiyon) ve `TC_CNT` (aksiyon → kaç parmak) ile
  referans sayımı yapılır; aynı aksiyona ikinci parmak `IN.down`'ı tekrar tetiklemez,
  ilk parmağın kalkması aksiyonu bırakmaz. `setPointerCapture` + global `pointerup/cancel/blur`
  temizliği ile "yapışan tuş" engellenir.
- **Pad boyutu JS'ten gelir.** `sizeTouchPads()` `--u`'yu **kısa ekran kenarının %17'si**,
  `[42, 84] px` arasına kırpılmış olarak verir; `--gap` de `u`'nun %18'i. Sabit 62px bir buton
  küçük telefonda başparmağı yutar, tablette pul gibi kalır. CSS'teki değerler yalnızca
  ilk `resize()` gelene kadar geçerli yedeklerdir.
  Buton boyutu önemi yansıtır: `heavy` 1.2×, `light` 1.05×, `flask` 0.88×.
- `env(safe-area-inset-*)` çentik/ev düğmesi alanına saygı duyar — bu **yalnızca** viewport
  meta etiketindeki `viewport-fit=cover` sayesinde çalışır; o kaldırılırsa insetler sessizce
  sıfırlanır.
- Başlık ve bitiş ekranlarında **oyun alanına dokunmak** `start` sayılır.
- **Ekran metinleri cihaza duyarlıdır.** Başlık/pause/bitiş ekranları ve ilk-oyun ipucu
  `TOUCH` durumuna göre dallanır: dokunmatikte "TAP TO BEGIN" / "TAP TO RISE AGAIN" /
  "TAP II TO CONTINUE" ve klavye tuş listesi yerine tek satırlık parry ipucu gösterilir.
  **Yeni bir istem metni eklerken bu dallanmayı koru** — mobilde "PRESS ENTER" yazmak yalandır.
- **`QUIT` pad'i (`restart`) bağlamsaldır.** `tcSyncQuit()` onu yalnızca `GAME.mode === 'paused'`
  iken gösterir ve `.hidden` yazmasını yalnızca durum *değiştiğinde* yapar (kare başına DOM
  yazımı yok). Gerekçe klavyeyle birebir aynı: `R` de sadece `paused` içinde okunur, oyun
  sırasında hiçbir şey yapmaz. Sahanın üzerinde sürekli duran ve yanlış dokunuşta boss
  dövüşünü çöpe atan bir buton olmamalı. Rengi kan kırmızısıdır (`#a8635c`), yıkıcı olduğu
  okunsun diye.

---

## 8. Sunum: Ölçekleme ve Yönlendirme (Bölüm 27)

### 8.1 Cihaz profili (`DEV`)
Her `resize()` çağrısında `profileDevice()` tazelenir:
`{ touch, kind, portrait, dpr, vw, vh, scale, unit, integer }`.
- `kind`: `'desktop'` (dokunmatik değilse) · aksi hâlde **kısa kenar** `>= 480` CSS px ise
  `'tablet'`, değilse `'phone'`. Kısa kenar yönden bağımsızdır, yani telefon çevrilince
  telefon kalır.
- Ölçüler `visualViewport`'tan okunur (`viewW/viewH`), `innerWidth/Height`'tan değil:
  mobilde adres çubuğu girip çıkarken dürüst olan tek kaynak odur.

### 8.2 Ölçek seçimi — **cihaz pikselinde tam sayı**
Bir oyun pikseli **tam sayıda fiziksel piksel** kaplamalıdır; yoksa kamera kaydıkça
sütun genişlikleri değişir ("kaynayan piksel") ve bu, bu sanat tarzının gizleyemediği
tek bozulmadır. Bu yüzden yuvarlama CSS pikselinde değil, `fit × dpr` üzerinde yapılır.

- **Masaüstü: her zaman tam sayı.** Ekran geniştir, letterbox ucuzdur.
- **El cihazı: tam sayı yalnızca ucuzsa** — `INT_SCALE_FILL = 0.86`, yani tam sayı adım
  sığdırılmış boyutun %86'sından azını kaplayacaksa kesirli ölçek alınır ve ekran doldurulur.
  Gerekçe: dpr 2–3'te bir cihaz pikselliği fark ~0.1 mm'dir, gözle görünmez; küçük bir
  telefonda ekranın beşte birini kaybetmek ise görünür.

Doğrulanmış çıktılar:

| Cihaz | Sonuç | Cihaz px / oyun px |
|---|---|---|
| PC 1920×1080 dpr1 | 1440×810 | 3 (tam) |
| PC retina 1280×800 dpr2 | 1200×675 | 5 (tam) |
| PC 4K dpr1 | 3360×1890 | 7 (tam) |
| Telefon 844×390 dpr3 | 640×360 | 4 (tam) |
| Telefon 640×360 dpr2 | 640×360 | 2.67 (kesirli, tam ekran) |
| Tablet 1024×768 dpr2 | 960×540 | 4 (tam) |

`document.body.style.height` de `DEV.vh`'ye sabitlenir, böylece canvas tarayıcı çubuğu
geri geldiğinde onun arkasında kalmaz.

### 8.3 Dikey mod kapısı
`updateOrientationGate()` — **yalnızca dokunmatik cihazlarda** ve yalnızca `vh > vw` iken:
- `#rot` katmanı açılır: dönen telefon SVG'si + **"PLEASE ROTATE YOUR DEVICE"** ve
  **"ASHFALL IS PLAYED IN LANDSCAPE"**. Oyunun paletiyle aynı (altın `#b8963f`, zemin
  `#05060a`) ve aynı scanline'ı taşır. `prefers-reduced-motion` animasyonu kapatır.
- Pad'ler gizlenir, tutulan bütün parmaklar `tcUp` ile bırakılır, `IN.held`/`IN.buf` temizlenir.
- Oyun **oynanıyorsa `paused`'a alınır** — şövalye görünmeyen bir katmanın ardında dayak yemez.
- `GAME.blocked = true` olur; `frame()` bu bayrakta **hiçbir şey tiklemeden** çıkar
  (`lastT` yine de ilerletilir, böylece geri dönüşte birikmiş bir kare işlenmez).

Kapı `resize`, `orientationchange`, `visualViewport.resize/scroll` ve
`matchMedia('(orientation:portrait)')` olaylarının hepsinden tazelenir.

---

## 9. Performans Sözleşmeleri

Bunlar ihlal edilirse oyun mobilde çöker — değişiklik yaparken korunmalıdır:

1. **Kare içinde piksel okuma yok.** `getImageData`/`finish()` sadece pişirmede.
2. **Kare içinde prosedürel çizim yok.** Karakterler yalnızca `SPR.cache`'ten `drawImage`.
3. **Parçacıklar havuzlanmış ve tavanlı:** toplam 350, ambiyans 110, savaş 180.
   `PS.spawn` tavan aşılırsa sessizce `null` döner — çağıranlar bunu tolere etmelidir.
4. **Ekran dışı kırpma** her çizim döngüsünde yapılır (`if (sx > VW || sx + w < 0) continue`).
5. **`plate()` içinde yatay renk koşuları tek `fillRect` olarak yazılır** — piksel başına
   `fillStyle` atamak pişirmeyi yavaşlatan asıl şeydir.
6. **Additive mod toplu yönetilir:** `PS.draw` `globalCompositeOperation`'ı parçacık başına
   değil, gruplar halinde değiştirir.
7. `dt` **0.06 s'de kırpılır**; sekmeye geri dönüşte fizik patlamaz.
8. Deterministik gürültü `hn(x,y,seed)` kullanılır → doku her karede aynı kalır, "kaynamaz".

---

## 10. Kod Sözleşmeleri

- **Tek dosya, `'use strict'`, ES5+ düz JS.** Modül, build adımı, bağımlılık yok.
- Kısaltmalar bilinçlidir: `fl/ce/ab/mn/mx` = `Math.floor/ceil/abs/min/max`, `r/px/ra` = dikdörtgen çizim.
- **Animasyon tabloları veri odaklıdır.** `KA`/`GA`/`BA` içindeki her giriş:
  `{ n: kare sayısı, dur: saniye, loop?: 1, cape?: k, hit?: [kare indeksleri], f(t, p) }`.
  `f` normalize zamanı (`t ∈ [0,1)`) alır ve poz nesnesini (`p`) mutasyona uğratır.
  Yeni animasyon eklemek = tabloya bir giriş eklemek; `buildWarmQueue()` onu otomatik pişirir.
- **Vuruş kutuları elle yazılmış dikdörtgenlerdir** (`plStrike`, `BATK[].box`), yerel
  koordinatlarda `[x0, y0, x1, y1]` ve yön (`face`) ile aynalanır.
- Durum geçişleri yardımcılarla yapılır: `plSet(state, anim)`, `eSet(e, s, a)`, `bSet(s, a)`.
- İngilizce yorumlar, tasarım niyetini açıklar ("*roll through it*", "*long anticipation,
  heavy recovery*"). Bu üslup korunmalı.

---

## 11. Bilinen Tuhaflıklar / Ölü Kod

Değiştirmeden önce bilinmesi gerekenler:

- `PLC.buffer` (0.14) **hiç kullanılmıyor**; gerçek buffer süresi `IN.down` içinde sabit **0.13**'tür.
- `GAME.slow`, `GAME.fps`, `GAME.prompt` tanımlı ama kullanılmıyor.
- `GTYPE[t].hp` (grunt 42 / thrower 30 / **brute 86**) sadece görsel yapılandırmada durur;
  **savaş için otorite `ETUNE[t].hp`'dir** (brute burada **88**). İkisi brute'ta uyuşmaz.
- `FX.swingLight` içinde `cos(a)*rnd(...)*dir*0` ifadesi etkisiz (sıfırla çarpım) —
  yalnızca `dir * rnd(20,60)` terimi çalışır.
- `plUpdate` içinde `if (!landed) { }` boş bir dal olarak durur.
- **`BOSS.state = 'sleep'` gerçekten çalışan bir durumdur, ölü kod değil.** `resetRun()` →
  `BOSS.reset()` boss'u `active = true` yapar, yani Grullk oyunun başından itibaren arenada
  durur ve çizilir. `bossUpdate`'in `switch`'inde `sleep` için **case yoktur** — bu kasıtlıdır:
  boss yalnızca fizik + `idle` animasyon döngüsü çalıştırır, saldırmaz, hedef seçmez.
  `startBossFight()` onu `intro`'ya çekene kadar bekler. Yan etki: `sleep` hâlindeyken bile
  vurulabilir/hasar alabilir (`plStrike` yalnızca `BOSS.active`'e bakar), ancak kapı
  kapanmadan oyuncu ona yaklaşamadığı için pratikte sorun çıkmaz — arena tetikleyicisi
  (`arenaX0 + 44`) boss'un konumundan (`x = 3150`) çok önce devreye girer.
- `KA.roll` tablo girişi animasyon fonksiyonu içermez (`drawKnightRoll` ayrı yol kullanır),
  bu yüzden `buildWarmQueue` içindeki `A.n || 6` yedeği önemlidir.

---

## 12. Değişiklik Yaparken Kontrol Listesi

1. Yeni renk mi gerekiyor? Önce `P` paletine bak — muhtemelen zaten var.
2. Yeni sprite mi? `bakeInto` + `SPR.get`/`SPR.cache` üzerinden geç, `finish()` seçeneklerini
   komşu varlıklarla tutarlı tut (şövalye `rim 1.23`, goblin `1.22`, boss `1.17`).
3. Yeni animasyon mu? İlgili tabloya (`KA`/`GA`/`BA`) ekle; `buildWarmQueue` gerisini halleder.
4. Yeni aksiyon mu? `KEYMAP`'e **ve** `#tc` içindeki bir butonun `data-act`'ine ekle,
   yoksa mobilde erişilemez olur.
5. Yeni boss saldırısı mı? `BA` (poz) + `BATK` (kutu/hasar) + `BTELL` (telegraph) —
   **üçü birden** olmadan saldırı okunamaz hâle gelir; okunabilirlik bu oyunun sözleşmesidir.
6. Dengeleme değeri mi? `PLC` / `ETUNE` / `BATK` / `BOSS` içindeki tek noktadan değiştir;
   sihirli sayıyı mantığın içine gömme.
7. Ekrana yeni bir istem/ipucu metni mi yazıyorsun? `TOUCH` üzerinden dallandır —
   dokunmatikte tuş adı ("PRESS ENTER", "LMB") yazmak yalandır (bkz. bölüm 7).
8. Ana döngüye yeni bir iş mi ekliyorsun? `frame()` içindeki `GAME.blocked` erken çıkışının
   **üstünde** değil altında olduğundan emin ol; dikey moddayken hiçbir şey tiklememelidir.
9. Her değişiklikten sonra hem masaüstünde (fare + klavye) hem `#touch` hash'iyle test et;
   dokunmatikte **her iki yönü de** dene — dikey mod kapısı ayrı bir kod yoludur.
