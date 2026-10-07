# Path Traversal — Gereğinden Fazla (Superfluous) URL-Decode ile Atlatma

**Lab:** File path traversal, traversal sequences stripped with superfluous URL-decode
**Seviye:** Practitioner
**Kategori:** Path Traversal (Directory Traversal)
**Araçlar:** Burp Suite (Proxy / Repeater / Decoder)
**Lab linki:** https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-with-superfluous-url-decode

---

## Zafiyetin Özeti

Bu lab yine ürün görsellerinde path traversal açığı içeriyor. Bu seferki savunma iki aşamalı ve işte açığın kök sebebi tam da bu iki aşamanın sırasında gizli:

1. Uygulama, **path traversal dizisi içeren girdiyi engelliyor** (örน. `../` görürse reddediyor).
2. Ardından, girdiyi kullanmadan **önce bir kez daha URL-decode (çözme) yapıyor.**

Bu ikinci, "gereğinden fazla" (superfluous) URL-decode adımı güvenlik açığının kaynağıdır. Amaç yine `/etc/passwd` okumak.

## Lab Açıklaması

![Lab açıklaması](images/04-lab-aciklama.png)

## Çözüm Adımları

### 1. Normal istek

Görsel isteğini Burp Repeater'a gönderiyoruz; normal istek görseli döndürüyor (`200 OK`):

```
GET /image?filename=1.jpg
```

![Normal görsel isteği](images/04-normal-istek.png)

### 2. Klasik traversal — engelleniyor

```
GET /image?filename=../../../etc/passwd
```

Yanıt `400 Bad Request`, `"No such file"`. `../` dizisi doğrudan tespit edilip engelleniyor.

![Klasik traversal engellendi](images/04-traversal-engellendi.png)

### 3. İç içe (nested) deneme — bu lab'de de engelleniyor

Bir önceki lab'deki `....//` hilesini deniyoruz:

```
GET /image?filename=....//....//....//etc/passwd
```

Yine `400 Bad Request`, `"No such file"`. Bu lab'in filtresi bu varyasyonu da yakalıyor; dolayısıyla farklı bir yol gerekiyor.

![Nested deneme engellendi](images/04-nested-engellendi.png)

### 4. Çift URL-encode ile atlatma

Birkaç denemenin ardından, yalnızca **`/` karakterini iki kez (çift) URL-encode** etmenin yeterli olduğunu gördük. Burp Decoder ile `/` karakterinin çift kodlamasını üretiyoruz:

```
/        (ham karakter)
%2f      (bir kez URL-encode)
%252f    (iki kez URL-encode — ya da tam biçimiyle %25%32%66)
```

![Burp Decoder — / karakterinin çift kodlaması](images/04-decoder-double-encode.png)

Nihai payload — yalnızca `/` çift kodlanmış, `..` ve geri kalanı düz:

```
GET /image?filename=..%252f..%252f..%252fetc/passwd
```

Bu istek `/etc/passwd` içeriğini döndürüyor (`root:x:0:0:...`) ve lab çözülüyor.

## Neden Çalışıyor? (İşin Mantığı)

Anahtar, isteğin yolculuğu boyunca **kaç kez URL-decode edildiğidir.** Payload'ımızdaki tek bir `/`'yi (`%252f` olarak kodlanmış) adım adım izleyelim:

**1. Adım — Sunucu/çatı (framework) otomatik decode'u.**
Web sunucusu, gelen istekteki sorgu parametrelerini standart olarak bir kez URL-decode eder. Yani `%252f` otomatik olarak bir kez çözülür:

```
%252f   →   %2f
```

Bu aşamadan sonra girdimiz `..%2f..%2f..%2fetc/passwd` halindedir.

**2. Adım — Uygulamanın traversal filtresi.**
Uygulama şimdi girdide `../` (veya `..\`) arar. Ama elimizdeki değer `..%2f...`; içinde **literal `../` yok**, sadece `..%2f` var. Filtre bu yüzden hiçbir traversal dizisi göremez ve girdiyi **temiz sanıp geçirir.** İşte savunmanın kör noktası burasıdır.

**3. Adım — Gereğinden fazla (superfluous) URL-decode.**
Uygulama, dosyayı açmadan hemen önce girdiyi **bir kez daha** URL-decode eder. Bu ikinci çözme, `%2f`'yi gerçek `/`'ye dönüştürür:

```
..%2f..%2f..%2fetc/passwd   →   ../../../etc/passwd
```

Ve bu dönüşüm **filtre kontrolünden sonra** gerçekleştiği için artık kontrol edilmez. Sonuçta dosya açma fonksiyonuna tam da istediğimiz `../../../etc/passwd` gider ve `/etc/passwd` okunur.

**Özetle:** Girdi toplam **iki kez** çözülüyor (biri çatının otomatik decode'u, biri uygulamanın fazladan decode'u). Çift kodladığımız `/` ilk decode'dan sonra `%2f` olarak "zararsız" görünüp filtreyi atlatıyor; ikinci (fazladan) decode ise onu filtre devreye girdikten sonra gerçek `/`'ye çevirerek traversal'ı tamamlıyor. Zafiyetin kök sebebi, **doğrulamanın tüm decode işlemleri bittikten sonra değil, aradaki bir aşamada yapılmasıdır** — yani uygulama kendi yaptığı fazladan decode'un filtreyi geçersiz kıldığının farkında değildir.

> Not: Neden sadece `/`'yi kodladık? Çünkü filtre esasen `/` içeren traversal dizisini (`../`) yakalıyor. `/`'yi gizlediğimizde filtre `..%2f`'i bir dizi olarak göremiyor; `..` noktalarını ayrıca kodlamaya gerek kalmıyor. (İstersek `%252e` ile noktaları da kodlayabilirdik, ama gerekmedi.)

## Önlem

- **Decode işlemlerini doğrulamadan önce bitirmek:** Girdi, kullanılmadan önce ne kadar decode edilecekse hepsi tamamlanmalı; doğrulama/temizleme **en son**, nihai değer üzerinde yapılmalı. Araya veya sonraya ek bir decode koymak tüm filtreyi geçersiz kılar.
- **Kara liste yerine kanonikleştirme + sınır kontrolü (önerilen):** Girdi nihai, tam (canonical) yola çözümlendikten sonra, oluşan yolun beklenen temel dizinin altında kaldığı doğrulanmalı. Bu yöntem kodlama hilelerinden etkilenmez:
  ```
  canonical = realpath(base + input)
  if not canonical.startsWith(realpath(base)): reddet
  ```
- **Girdiyi dosya yoluna hiç koymamak:** Dosyaları kimlik/indeks üzerinden sunmak en sağlam çözümdür.
- **En az yetki prensibi:** Uygulama kullanıcısının sistem dosyalarına gereksiz okuma yetkisi olmamalı.
