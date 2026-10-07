# Path Traversal — Traversal Dizileri Engelli, Absolute Path ile Atlatma

**Lab:** File path traversal, traversal sequences blocked with absolute path bypass
**Seviye:** Practitioner
**Kategori:** Path Traversal (Directory Traversal)
**Araçlar:** Burp Suite (Proxy / Repeater)
**Lab linki:** https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass

---

## Zafiyetin Özeti

Bu lab yine ürün görsellerinin gösteriminde path traversal açığı içeriyor; ancak bir önceki "basit durum" lab'inden farkı var: uygulama **traversal dizilerini (`../`) engelliyor**, fakat verilen dosya adını **varsayılan bir çalışma dizinine göreli (relative)** kabul ediyor. Amaç yine `/etc/passwd` dosyasını okumak.

Bu iki özellik birlikte, "absolute path (mutlak yol) ile atlatma" denen basit bir bypass'a kapı açıyor.

## Lab Açıklaması

![Lab açıklaması](images/02-lab-aciklama.png)

## Çözüm Adımları

### 1. Normal istek

Görsel isteğini Burp Repeater'a gönderiyoruz. Normal istek görseli başarıyla döndürüyor (`200 OK`, `image/jpeg`):

```
GET /image?filename=41.jpg
```

![Normal görsel isteği](images/02-normal-istek.png)

### 2. Klasik traversal denemesi — engelleniyor

Önceki lab'deki payload'ı deniyoruz:

```
GET /image?filename=../../../etc/passwd
```

Bu kez yanıt `400 Bad Request` ve gövdede `"No such file"` mesajı dönüyor. Uygulama `../` dizilerini tespit edip engelliyor (ya da temizliyor); bu yüzden klasik traversal çalışmıyor.

![Traversal engellendi — 400](images/02-traversal-engellendi.png)

### 3. Absolute path ile atlatma

`../` engellendiğine göre, hedef dosyaya hiç `../` kullanmadan, doğrudan **mutlak yol** vererek ulaşmayı deniyoruz:

```
GET /image?filename=/etc/passwd
```

Yanıt `200 OK` dönüyor ve gövdede `/etc/passwd` içeriği görünüyor (`root:x:0:0:...`). Lab çözüldü.

![Absolute path ile /etc/passwd](images/02-absolute-path.png)

## Neden Çalışıyor? (İşin Mantığı)

Buradaki kilit nokta, uygulamanın savunmasındaki boşluktur. Uygulama iki şey yapıyor:

1. **Traversal dizilerini engelliyor:** Girdide `../` görürse reddediyor. Bu yüzden `../../../etc/passwd` denememiz `400` ile reddedildi.
2. **Dosya adını göreli (relative) kabul ediyor:** Normalde dosya adını bir temel dizine ekliyor; örneğin `/var/www/images/` + `41.jpg` → `/var/www/images/41.jpg`. "Göreli kabul etme" varsayımı tam da burada devreye giriyor.

Sorun şu: uygulama yalnızca `../` dizilerini kontrol ediyor; ama girdinin **mutlak yol olup olmadığını kontrol etmiyor.** İşletim sisteminde `/` ile başlayan bir yol (`/etc/passwd`) **mutlak yoldur** ve bir temel dizine eklense bile sonuç yine o mutlak yola çözümlenir:

```
/var/www/images/ + /etc/passwd   →   /etc/passwd
```

Çoğu dosya yolu birleştirme fonksiyonunda (ör. Java'da `new File(base, input)`, birçok dilde benzer davranış), ikinci argüman mutlak bir yolsa temel dizin tamamen yok sayılır ve mutlak yol doğrudan kullanılır. Yani:

- `../` kullanmıyoruz → traversal filtresi hiç tetiklenmiyor.
- `/etc/passwd` mutlak yol → temel dizini "ezip" doğrudan hedef dosyaya gidiyor.

Böylece saldırı, filtrenin hiç beklemediği bir yoldan (dizi engelleme yerine mutlak yol) başarıya ulaşıyor. Ders: `../` engellemek tek başına yeterli bir savunma değildir; mutlak yollar da ele alınmalıdır.

## Önlem

- **Hem traversal hem mutlak yol kontrolü:** Yalnızca `../` dizilerini değil, `/` (veya Windows'ta `C:\`, `\`) ile başlayan mutlak yolları da reddetmek gerekir. Kara liste (blacklist) yaklaşımı eksik kalmaya meyillidir.
- **Kanonikleştirme + sınır kontrolü (önerilen):** Girdi tam (canonical) yola çözümlendikten sonra, oluşan yolun beklenen temel dizinin (ör. `/var/www/images/`) altında kaldığı doğrulanmalı. Bu yöntem hem `../` hem mutlak yol hem de kodlama hilelerini tek seferde kapatır:
  ```
  canonical = realpath(base + input)
  if not canonical.startsWith(realpath(base)): reddet
  ```
- **Girdiyi dosya yoluna hiç koymamak:** Dosyaları kimlik/indeks üzerinden sunmak (ham dosya adı yerine) en sağlam çözümdür.
- **En az yetki prensibi:** Uygulama kullanıcısının sistem dosyalarına gereksiz okuma yetkisi olmamalı.
