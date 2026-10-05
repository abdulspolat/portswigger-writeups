# SQL Injection — UNION Attack: Sorgunun Döndürdüğü Sütun Sayısını Belirleme

**Lab:** SQL injection UNION attack, determining the number of columns returned by the query
**Seviye:** Practitioner
**Kategori:** SQL Injection (UNION)
**Araçlar:** Burp Suite (Proxy + Repeater), tarayıcı
**Lab linki:** https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns

---

## Zafiyetin Özeti

Uygulamadaki ürün kategorisi filtresi SQL enjeksiyonuna açık ve sorgunun sonuçları doğrudan sayfada gösteriliyor. Bu durumda `UNION` saldırısı kullanarak başka tablolardan veri çekebiliriz.

`UNION` ifadesinin çalışabilmesi için iki şart vardır:

1. Birleştirilen iki sorgu **aynı sayıda sütun** döndürmelidir.
2. Karşılıklı sütunların **veri tipleri uyumlu** olmalıdır.

Bu yüzden bir UNION saldırısının ilk adımı, orijinal sorgunun kaç sütun döndürdüğünü bulmaktır. Bu lab tam olarak bu adımı hedefliyor: doğru sütun sayısıyla `NULL` değerler içeren ek bir satır döndürmek.

## Lab Açıklaması

![Lab açıklaması](images/03-lab-aciklama.png)

## Yöntem

Sütun sayısını belirlemenin standart yolu, artan sayıda `NULL` içeren bir `UNION SELECT` denemesidir:

```
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```

`NULL` kullanılır çünkü her veri tipine dönüştürülebilir; böylece tip uyumsuzluğu sorunu yaşanmaz ve yalnızca sütun sayısına odaklanılır. Sütun sayısı yanlış olduğunda uygulama genellikle **hata** (ör. HTTP 500) döndürür; doğru sayıya ulaşıldığında ise **hatasız** (HTTP 200) bir yanıt gelir.

## Çözüm Adımları

### 1. Normal isteği incelemek

Önce kategori filtresinin normal halini görüntülüyoruz. "Gifts" kategorisi seçildiğinde ürünler listeleniyor ve lab henüz çözülmemiş durumda.

![Normal kategori sayfası](images/03-normal-sayfa.png)

### 2. İsteği Burp ile yakalayıp Repeater'a göndermek

İsteği Burp Proxy ile yakalayıp Repeater'a gönderiyoruz. Repeater'da denemeleri rahatça tekrarlayıp yanıtları karşılaştırabiliyoruz. Aşağıda normal `GET /filter?category=Gifts` isteği ve `200 OK` yanıtı görülüyor.

![Repeater — normal istek (200 OK)](images/03-repeater-normal.png)

### 3. Sütun sayısını denemek

`category` parametresine artan sayıda `NULL` içeren UNION payload'ları enjekte ediyoruz.

**Tek sütun denemesi** — `Gifts' UNION SELECT NULL--`

Bu istek `500 Internal Server Error` döndürüyor; yani orijinal sorgu tek sütun döndürmüyor.

![Repeater — tek NULL, 500 hatası](images/03-repeater-1sutun-500.png)

**Üç sütun denemesi** — `Gifts' UNION SELECT NULL,NULL,NULL--`

Sütun sayısını artırarak denemeye devam ettiğimizde, üç `NULL` kullanıldığında yanıt `200 OK` oluyor. Hata kaybolduğuna göre orijinal sorgu **3 sütun** döndürüyor.

![Repeater — üç NULL, 200 OK](images/03-repeater-3sutun-200.png)

> Yöntem gereği NULL sayısı 1'den başlayıp birer birer artırılır (1 → 2 → 3). Hata kaybolan ilk değer doğru sütun sayısıdır. Burada 3'te hata ortadan kalktı.

### 4. Tarayıcıda doğrulamak

Doğru payload'ı tarayıcıda URL üzerinden de çalıştırıyoruz:

```
/filter?category=Gifts'+UNION+SELECT+NULL,NULL,NULL--
```

Sayfa hatasız yükleniyor ve lab çözülmüş olarak işaretleniyor.

![Lab çözüldü](images/03-lab-cozuldu.png)

## Neden Çalışıyor?

`UNION`, iki `SELECT` sorgusunun sonuçlarını tek sonuç kümesinde birleştirir. Enjeksiyon noktasında ek bir `SELECT` çalıştırabildiğimiz için, sütun sayısı eşleştiği anda kendi satırımızı sonuçlara ekleyebiliyoruz. `NULL` değerleri tip uyumu derdini ortadan kaldırarak bu aşamada yalnızca sütun sayısını tespit etmemizi sağlar. Bu bilgi, sonraki adımda (veri sızdırma) hangi sütunların kullanılabilir metin sütunu olduğunu bulmak için temel oluşturur.

## Önlem

- **Parametreli sorgular (prepared statements):** Kullanıcı girdisi sorguya birleştirilmek yerine parametre olarak bağlanmalı; böylece `UNION` gibi ifadeler enjekte edilemez.
- **Girdi doğrulama (allow-list):** `category` gibi parametreler yalnızca bilinen, izin verilen değerlere karşı doğrulanmalı.
- **Ayrıntılı hata mesajlarını gizlemek:** Veritabanı hatalarının kullanıcıya yansıtılması saldırgana geri bildirim verir; üretimde genel hata sayfaları kullanılmalı. (Tek başına yeterli bir önlem değildir, asıl çözüm parametreli sorgulardır.)
