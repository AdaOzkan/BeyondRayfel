# Marswalk HUD – Arcadia Planitia

Mars'ta, **Arcadia Planitia** bölgesindeki bir kratere inen mürettebat için hazırlanmış, tarayıcıda çalışan bir **EVA (yüzey yürüyüşü) planlama arayüzü**. Tek bir HTML dosyasından (`projects_patched9.html`) oluşur ve iki sekme içerir:

1. **Rota planlayıcı (Leaflet):** NASA Mars Trek katmanları üzerinde rota çizin; mesafe, yol süresi, O₂ marjı ve radyasyon dozunu anlık görün.
2. **3B yükseklik görünümü:** Aynı krateri, görüntü parlaklığından türetilmiş yaklaşık bir kabartmayla 3B inceleyin; aynı rotayı orada da düzenleyin.

İki sekme **canlı eşleşir**: birinde çizdiğiniz rota diğerine aktarılır.

> Bu README yeni sürüme (`projects_patched9.html`) göre yazılmıştır. Arayüz TR/EN dil düğmesini destekler; aşağıdaki isimler İngilizce arayüz adlarıdır, Türkçe karşılıkları parantez içinde verilmiştir.

---

## İçindekiler

- [Hızlı başlangıç](#hızlı-başlangıç)
- [Nasıl çalışır?](#nasıl-çalışır)
- [Sekme 1: Rota planlayıcı](#sekme-1-rota-planlayıcı)
  - [Neden düz rota değil de daireler?](#neden-düz-rota-değil-de-daireler)
  - [Rota oluşturma](#rota-oluşturma)
  - [Mission info paneli ve hesaplar](#mission-info-paneli-ve-hesaplar)
  - [Harita katmanları](#harita-katmanları)
  - [Kaskta pusula şeridi](#kaskta-pusula-şeridi)
  - [Mission plan (Dünya → Mars → Dünya)](#mission-plan-dünya--mars--dünya)
  - [Barınak (habitat) şeması](#barınak-habitat-şeması)
- [Sekme 2: 3B yükseklik görünümü](#sekme-2-3b-yükseklik-görünümü)
- [Sabitler ve varsayımlar](#sabitler-ve-varsayımlar)
- [Dışa aktarma (JSON)](#dışa-aktarma-json)
- [Bilinen sınırlar](#bilinen-sınırlar)
- [Veri kaynakları](#veri-kaynakları)

---

## Hızlı başlangıç

1. `projects_patched9.html` dosyasını bir tarayıcıda açın (derleme veya sunucu gerekmez).
2. **İnternet bağlantısı gerekir.** Leaflet, geotiff.js, three.js, yazı tipleri ve Mars Trek harita karoları (tiles) CDN/NASA üzerinden yüklenir.
3. İlk açılışta **Route planner (Leaflet)** sekmesi gelir. Haritaya tıklayarak rota oluşturun veya alt çubuktaki **Ice EVA** düğmesiyle hazır bir buz sondajı rotası yükleyin.
4. Üstteki **3D elevation view** sekmesine geçtiğinizde kısa bir bilgi kutusu çıkar: 3B kabartmanın ölçülmüş yükseklik değil, görüntüdeki gölgelenmeden çıkarılan **yaklaşık** bir kabartma olduğunu söyler.

Mars Trek'in varsayılan GeoTIFF dosyaları bu krater için 3B görünüm sunmadığından kabartma, projede geliştirilen kendi yöntemle üretilir (bkz. [3B bölümü](#sekme-2-3b-yükseklik-görünümü)).

---

## Nasıl çalışır?

![Mimari diyagram](docs/images/arch-diagram.png)

- Dosya bir **kabuk sayfa** ve onun içindeki **iki `iframe`** (`f1`: Rota planlayıcı, `f2`: 3B görünüm) olarak yapılandırılmıştır. İki sayfanın kodu kabuğun içine gömülüdür; her sekme ilk açıldığında yüklenir.
- Sekmeler arası eşleşme `window.postMessage` ile yapılır. Bir sekmede rota değiştiğinde durum (`n`, `HP`, `items`, hız, durak süresi, SPE, O₂ tabanı) kabuğa gönderilir, kabuk de diğer sekmeye iletir. Böylece iki sekme aynı rotayı gösterir.
- Aynı hesap modeli iki sekmede de kullanılır:

![Hesap hattı](docs/images/calc-pipeline.png)

---

## Sekme 1: Rota planlayıcı

Harita, **Leaflet 1.9.4** ve **EPSG:4326** (enlem/boylam) koordinat sistemiyle çalışır. Başlangıç görünümü Arcadia Planitia'dadır (46,6°K, 173°B).

![D1–D10 daireleri](docs/images/map-d1-d10-rings.png)

Haritadaki ana noktalar:

| Nokta | Konum | Not |
|---|---|---|
| Ana sığınak (üs) ⌂ | 46,560°K 173,005°B | Rotanın her zaman başladığı yer; tıklayınca [habitat şeması](#barınak-habitat-şeması) açılır |
| İniş bölgesi (LZ) | 46,56126°K 173,03432°B | D1–D10 dairelerinin merkezi |
| İniş noktası | 46,560°K 173,00468°B | Üssün hemen yanında |
| Buz sondaj noktası (BZ) | 46,695°K 173,005°B | Üsten 8 km halkası üzerinde |
| Acil sığınak 2 (S2) | 46,620°K 172,620°B | İleri mesafeli acil sığınak |
| Hidratlı mineral / krater (KR) | 46,510°K 172,515°B | CRISM işareti, HiRISE kaya haritası, ~3 km yarıçaplı dairesel alan |

Her noktanın üzerine tıklayınca açıklama ve **Add to route** (rotaya ekle) düğmesi çıkar. Sol paneldeki **Locations** listesinden bir satıra tıklarsanız harita o noktaya uçar ve gidiş-dönüş yürüme süresi (hız ve durak süresine göre) görünür.

### Neden düz rota değil de daireler?

![Neden daireler](docs/images/ui-why-circles-note.png)

Tek bir sabit rota tanımlanmadı. Bunun yerine iniş bölgesi (LZ) merkezli, yarıçapları kabaca eşit adımlarla büyüyen **10 daire (D1–D10)** çizildi. Mars'ta koşullar anlık değişir: toz ve kum fırtınaları bir yolu kapatabilir veya değiştirebilir. Daireler sayesinde astronotun her zaman seçeneği olur; nokta dairenin kenarında veya içinde serbestçe kayabilir, yol bir engelin etrafından dolanıp LZ'den bilinen bir uzaklıkta kalabilir. Kendi yolunu çizmek isteyenler için ayrıca rota planlayıcı vardır.

Daire yarıçapları LZ'den (hesaplanan değerler):

| Daire | D1 | D2 | D3 | D4 | D5 | D6 | D7 | D8 | D9 | D10 |
|---|---|---|---|---|---|---|---|---|---|---|
| Yarıçap | 1,2 km | 2,4 km | 4,2 km | 6,2 km | 8,0 km | 9,9 km | 11,8 km | 13,7 km | 15,6 km | 17,5 km |

Her dairenin üzerindeki **D** etiketine tıklayınca o halkada yapılacak işler açılır:

| Daire | Konu |
|---|---|
| D1 | Ana sığınak (hab) çevresi: kurulum, yakın çevre keşfi, skafandr testi |
| D2 | Toprak ve kaya analizi: regolit numuneleri, XRF/Raman, blok sayımı |
| D3 | Krater morfolojisi ve topografyası: duvar, teras, eğim profili |
| D4 | Mineral ve spektral analiz: hidratlı mineral doğrulaması |
| D5 | **Buz sondajı:** GPR ile SHARAD yansımalarının doğrulanması, karot sondajı (ana ISRU hedefi) |
| D6 | Buz–regolit tabakalaşması: ikinci GPR/sondaj hattı |
| D7 | Volkanik birimler ve jeoloji: Arcadia Formasyonu / Amazonian lav birimleri |
| D8 | Çevre ve radyasyon istasyonu: hava durumu istasyonu, dozimetre |
| D9 | Acil sığınak 2 ve ileri depo: yedek O₂, su, gıda, haberleşme rölesi |
| D10 | Dış sınır ve referans noktası: uzun menzilli rover/dron taraması, karşılaştırma numunesi |

### Rota oluşturma

Alt çubuktaki düğmeler:

| Düğme | Ne yapar? |
|---|---|
| **Add breakpoint** (Kırılma noktası ekle) | Aç/kapa düğmesidir (varsayılan açık). Açıkken haritaya tıklamak, tıklanan yere en yakın rota bölümüne bir kırılma noktası (◆) ekler |
| **Undo** (Geri al) | Son eklenen noktayı veya dönüşü geri alır |
| **Return to base** (Üsse dön) | Rotanın sonuna ⌂ noktasını ekler |
| **Reset route** (Rotayı sıfırla) | Rotayı sıfırlar; daire sayısı **Route circles** alanındaki değerdir |
| **Download JSON** | Rotayı JSON olarak indirir (bkz. [Dışa aktarma](#dışa-aktarma-json)) |
| **Ice EVA** | 5 daireli, üsse dönüşlü örnek bir buz sondajı rotası kurar ve haritayı sığınak bölgesine odaklar |
| **Shelter area** | Haritayı tüm noktaları kapsayacak şekilde ortalar |
| **Mission plan** | Dünya → Mars → Dünya görev planını açar |

Kurallar:

- Rota her zaman **ana sığınaktan (⌂)** başlar.
- **Route circles (1–10)** kaç daireden geçileceğini belirler. Rotadaki daire noktaları **C1, C2…** diye adlanır. Her C noktası yalnızca kendi dairesi boyunca sürüklenir.
- Rotada olmayan daireler için de soluk (hayalet) C noktaları gösterilir; daire sayısını artırınca rota bu konumları kullanır.
- Haritaya tıklamak iki daire arasına bir **◆ kırılma noktası** ekler (**B1, B2…**). Kırılma noktası serbestçe taşınır; ◆'ye tıklayarak silebilirsiniz.
- Bölümler (legs) **Legs** listesinde gösterilir: bölümün adı, pusula yönü, mesafe ve süre. Bir bölüme tıklarsanız o bölüm seçilir ve pusula o bölümün yönünü gösterir.

![Bölüm listesi](docs/images/ui-legs-list.png)

Örnekte `⌂ → C1` bölümü 983 m uzunluğunda, yönü 339° KKB'dir ve 3 km/h hızda 20 dk sürer.

### Mission info paneli ve hesaplar

Sağ paneldeki değerler rota değiştikçe anında güncellenir. Her hesabın yanındaki **i** düğmesi kullanılan formülü canlı değerlerle açar.

![Mesafe modeli](docs/images/ui-mission-info-distance.png)

**Mesafe.** Her bölümün uzunluğu Mars'ı küre kabul eden büyük daire (haversine) formülüyle hesaplanır ve toplanır:

```
a  = sin²(Δφ/2) + cos φ₁ · cos φ₂ · sin²(Δλ/2)
dᵢ = 2R · asin(√a),   R = 3389,5 km
D  = Σ dᵢ
```

Eğim hesaba katılmaz; gerçek yol biraz daha uzundur.

**Yol süresi ve bilimsel duruş.** Yol süresi `L / v`'dir (`v` hız kaydırıcısından). **Science stop**, rotadaki her daire noktasında (C ve D noktaları) **Stop duration** kadar (varsayılan 25 dk) durulduğunu varsayar: `N × tₛ`. Kırılma noktalarında durulmaz.

**O₂ marjı.** Kullanılabilir oksijen, 8 saatlik EVA'dan %30 yedek düşülerek bulunur ve yedek hiçbir zaman harcanmaz:

```
O₂_kullanılabilir = 8 sa × (1 − 0,30) = 336 dk
Marj = 336 − (yol süresi + durak süresi)
```

![O₂ marjı modeli](docs/images/ui-o2-margin-model.png)

Bu örnekte rota 8088 m, 3 km/h hızda yol süresi 161,8 dk, 5 durak × 25 dk = 125 dk olduğundan marj **+49,2 dk ✓** çıkar.

**O₂ margin basis** seçimi:

- **One way (straight out)**: yol süresi çizilen rota uzunluğu (`D`) üzerinden alınır.
- **Round trip (same route back)**: son durağa kadar olan mesafe iki katına çıkarılır (`2 × son durağa mesafe`); aynı yoldan dönüş varsayılır.

Marj negatifse kırmızı ✗ görünür ve "O₂ yetmiyor, … dk açık var" uyarısı çıkar.

![Negatif marj örneği](docs/images/ui-mission-info-negative-margin.png)

Bu örnekte rota 22 420 m'dir (6 bölüm); 3 km/h'da yol süresi 448 dk, duruşlar 125 dk, marj **−238 dk ✗** olur. Aynı rota daha hızlı bir araçla veya kısaltılarak denenebilir.

Hız etkisini görmek için, her dairenin ucuna gidip dönmenin (tek 25 dk duruşla) O₂ bütçesine sığıp sığmadığına bakın:

![Daire erişim grafiği](docs/images/ring-reach-o2.png)

3 km/h'da D5 ve ötesi, 5 km/h'da D8 ve ötesi tek EVA'ya sığmaz; 10 km/h'lik gezginle D10'a bile ulaşılır (bu grafik yukarıdaki formüllerle hesaplanmış, eğimsiz, düz çizgi varsayımlı bir tahmindir).

**Diğer değerler:**

| Değer | Hesap |
|---|---|
| **Dose (EVA)** | `0,67 mSv/gün ÷ 24 × toplam süre (saat)`. **Solar particle event (×5 dose)** işaretliyse ×5; ayrıca "EVA'yı kısaltın, sığınağa dönün" uyarısı çıkar |
| **Emergency return** | İlerleme noktasından üsse kuş uçuşu mesafe ÷ güncel hız (dk) |
| **Progress along route** | Rota boyunca bir "astronot" noktası gezdirir; acil dönüş süresi ve pusula bu noktaya göre güncellenir |
| **Speed** | 1–15 km/h kaydırıcı. **Rover** düğmesi 10 km/h'ı (4 kişilik mürettebat), **Human** düğmesi 5 km/h'ı (ortalama yürüme) ayarlar; kaydırıcı sonra elle değiştirilebilir |

### Harita katmanları

![Harita katmanları](docs/images/ui-map-layers.png)

**Base** (altlık): Viking renkli mozaik (232 m) veya MOLA renkli topografya (463 m).

**Overlay** (üst katman), saydamlığı **Opacity** ile ayarlanır:

| Katman | Açıklama |
|---|---|
| None | Üst katman yok |
| Viking, MOLA | Altlık seçenekleriyle aynı |
| THEMIS IR day / night (100 m) | Mars Trek'ten kızılötesi mozaikler |
| Thermal, Layer 3, Dust, Slope | Ekibin hazırladığı GeoTIFF katmanları (`thermal.tif`, `dust.tif`, `slope.tif` ve Google Drive'dan çekilen "Layer 3"). Dosya okunamazsa **Choose file** ile elle seçilebilir |
| Exploration Zones – NASA EMC (100 km circles) | NASA keşif bölgesi önerileri, 100 km çaplı daireler (konumlar yaklaşık) |

Ayrıca **Custom layer ID** alanına bir Mars Trek katman kimliği veya `…/z/y/x.png` biçiminde karo adresi girerek kendi katmanınızı ekleyebilirsiniz. Katman karoları yüklenmezse bir uyarı çıkar.

**Mission layers** ile şunlar açılıp kapatılır: sığınaklar ve iniş noktası, bilim noktaları, menzil halkaları (üsten 2/4/8/16/20 km; 8 km halkası vurgulu), bölge daireleri, D1–D10 daireleri, etiketler, enlem/boylam ızgarası ve rota. **Hide all / Show all** hepsini tek tuşla gizler.

### Kaskta pusula şeridi

![Pusula şeridi](docs/images/ui-compass-tape.png)

Ekranın üstündeki şerit, kask HUD'u gibi çalışır. Seçili bölümün (yoksa ilerleme noktasındaki bölümün) yönünü derece ve 16 yönlü pusula adıyla gösterir (ör. `339° NNW`). Rota yoksa "Add a route for heading" yazar.

### Mission plan (Dünya → Mars → Dünya)

**Mission plan** düğmesi, görev takvimini hesaplayan tam ekran bir pencere açar:

- **Profile:** düşük enerjili uzun kalış (259 gün geçiş), hızlı geçiş (~180 gün) veya kısa kalış (30 gün, Venüs geçişli, ~600 gün).
- **Opposition:** Mars'ın Dünya'ya en yakın olduğu tarih (2029, 2031, 2033, 2035, 2037).
- Çıktı: Kennedy Uzay Merkezi'nden kalkış, Mars'a varış, yüzeyde kalış, Mars'tan kalkış, Dünya'ya dönüş, toplam süre, karşıtlık mesafesi ve tahmini doz (600 mSv sınırı kontrol edilir).
- **Mission day** kaydırıcısıyla, Dünya, Mars ve uzay aracının konumu bir yörünge animasyonunda izlenir.
- Altta; kalkış, iniş yeri, yaşam yeri (artı/eksi), araştırma, hayatta kalma ve yarışma uyumu maddeleri listelenir.

Model basit dairesel yörünge modelidir; gerçek tarihler birkaç hafta kayabilir ve Venüs geçişi şematiktir. Kesin değerler için JPL Horizons ile doğrulayın.

### Barınak (habitat) şeması

Haritada **ana sığınağa (⌂)** tıklayınca, yeraltı Mars sığınağının yeni giriş modülünü ve **hidrojenle ısıtma sistemini** anlatan bir şema açılır: yeraltı buzundan su → elektroliz (H₂ ve O₂) → hidrojen yakıt hücresi → ısıtma devresi, ayrıca hava kilidi, dekontaminasyon bölgesi ve iç basınç kapısı.

---

## Sekme 2: 3B yükseklik görünümü

Kraterin yörünge görüntüsünü (yaklaşık 43,4 km K–G × 39,0 km D–B) three.js (r128) ile 3B bir yüzey olarak gösterir. Görüntü dosyanın içine gömülüdür.

![Eğik görünümde daireler](docs/images/3d-tilted-circles.png)

**Kabartma nasıl üretilir?** Mars Trek'in GeoTIFF'leri bu krater için 3B vermediğinden, görüntüdeki **parlaklık** yükseklik gibi kullanılır: gölgeler alçak, aydınlık yüzeyler yüksek sayılır. **Illustrative relief** kaydırıcısı bu etkiyi büyütür. Sonuç gerçek yükseklik **değildir**, yalnızca açıklayıcıdır.

![Kabartma](docs/images/3d-relief.png)

**Kontroller**

| Kontrol | İşlev |
|---|---|
| **Tilt** | Kamerayı 0° (tepeden) – 90° arasında eğer |
| **Illustrative relief** | Kabartma şiddeti (0 = düz) |
| Fare | Sürükle: kaydır (eğikken döndür) · sağ tuşla sürükle: döndür · tekerlek: yakınlaştır |
| **Markers and layers** | Krater merkezi, ızgara, D1–D10 daireleri, menzil halkaları, bölge dairesi, noktalar, rota ve etiketleri aç/kapat |
| **Route planner** | Rota planlayıcı sekmesindeki aynı daireler ve rota; hız, durak süresi, O₂ tabanı ve SPE seçenekleri ve aynı O₂/doz hesabı |
| **Click mode** (alt çubuk) | **Plan route**: araziye tıklayarak ◆ ekle (çift tıklayınca sil). **Measure**: iki nokta arasında ölçüm |
| **Reset view** | Kamerayı sıfırlar |

**Ölçüm aracı.** **Measure** modunda araziye iki nokta tıklayınca çizgi kabartmayı izler; mesafe, yön (kuzeyden), göreli yükseklik farkı, ortalama ve en yüksek eğim hesaplanır. Eğim, açıklayıcı kabartmadan türetilir, ölçülmüş yükseklikten değil.

![Ölçüm profili](docs/images/3d-measure-profile.png)

---

## Sabitler ve varsayımlar

| Parametre | Değer |
|---|---|
| Mars yarıçapı (R) | 3389,5 km |
| EVA süresi / O₂ yedeği | 8 sa / %30 (kullanılabilir 336 dk) |
| Varsayılan hız | 3 km/h (Rover: 10 km/h, Human: 5 km/h) |
| Varsayılan duruş süresi | 25 dk / daire noktası |
| Radyasyon | RAD ≈ 0,67 mSv/gün; SPE'de ×5 |
| Uçuş dozu (mission plan) | 1,84 mSv/gün (geçişte), 0,67 mSv/gün (yüzeyde) |
| Rota daire sayısı | 1–10 (varsayılan 5) |
| Menzil halkaları | 2, 4, 8, 16, 20 km (üsten) |

---

## Dışa aktarma (JSON)

**Download JSON** düğmesi `marswalk-rota.json` dosyasını indirir. Dosyada iniş bölgesi (`landing_zone`), rota noktaları (`waypoints`: kimlik, tür, enlem, boylam), toplam mesafe (`distance_m`) ve kaynak (`source`) alanları bulunur.

---

## Bilinen sınırlar

- Mesafeler eğimi hesaba katmaz; gerçek yol biraz daha uzundur.
- 3B kabartma görüntü parlaklığından türetilir; ölçülmüş yükseklik veya gerçek eğim değildir.
- O₂ ve doz hesapları basit modellerdir (sabit hız, sabit tüketim). Planlama amaçlıdır, operasyonel karar için kullanılmamalıdır.
- Görev planı basit dairesel yörünge modeline dayanır.
- Harita karoları ve bazı GeoTIFF dosyaları ağdan yüklenir; çevrimdışı çalışmaz. "Layer 3" Google Drive'dan alınır, erişilemezse elle dosya seçilmelidir.
- Dosya, Google Drive erişimi için bir **API anahtarı** içeriyor. Depoyu herkese açık yayınlamadan önce bu anahtarı kısıtlayın veya kaldırın.

---

## Veri kaynakları

- [NASA Mars Trek](https://trek.nasa.gov/mars/): Viking, MOLA, THEMIS katmanları
- Arcadia Planitia verileri: HiRISE kaya haritası, CRISM, SHARAD
- [api.nasa.gov](https://api.nasa.gov/), [code.nasa.gov](https://code.nasa.gov/), [NASA Learning Resources](https://www.nasa.gov/learning-resources/), [NASA Resources](https://www.nasa.gov/resources/)
- Kütüphaneler: [Leaflet](https://leafletjs.com/) 1.9.4, [geotiff.js](https://geotiffjs.github.io/) 2.1.3, [three.js](https://threejs.org/) r128
- Yazı tipleri: Inter, Fraunces, IBM Plex Mono
