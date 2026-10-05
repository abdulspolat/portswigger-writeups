# SQL Injection — UNION Attack: Diğer Tablolardan Veri Çekme

**Lab:** SQL injection UNION attack, retrieving data from other tables
**Seviye:** Practitioner
**Kategori:** SQL Injection (UNION)
**Araçlar:** Burp Suite (Proxy + Repeater), tarayıcı
**Lab linki:** https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables

---

## Zafiyetin Özeti

Kategori filtresi SQL enjeksiyonuna açık ve sorgu sonuçları sayfada gösteriliyor. Bu lab, önceki adımları birleştirerek gerçek bir `UNION` saldırısı kurmayı hedefliyor: başka bir tablodan veri çekmek.

Veritabanında `users` adında ayrı bir tablo var; sütunları `username` ve `password`. Amaç: tüm kullanıcı adı ve parolaları çekip bu bilgilerle `administrator` olarak giriş yapmak.

## Lab Açıklaması

![Lab açıklaması](images/05-lab-aciklama.png)

## Çözüm Adımları

### 1. Normal sayfa

"Pets" kategorisi görüntülendiğinde ürünler listeleniyor ve lab henüz çözülmemiş.

![Normal kategori sayfası](images/05-normal-sayfa.png)

### 2. Sütun sayısı ve tiplerini belirlemek

İsteği Burp Repeater'a gönderip sütun sayısını ve tiplerini tek adımda test ediyoruz. İki metin değeri deniyoruz:

```
Pets' UNION SELECT 'a','b'--
```

Yanıt `200 OK`. Buradan iki şeyi anlıyoruz: sorgu **2 sütun** döndürüyor ve **her iki sütun da** metin verisiyle uyumlu.

![Repeater — iki sütun, 'a','b', 200 OK](images/05-repeater-2sutun-200.png)

### 3. Veriyi çekmek

Tablo adını (`users`) ve sütun adlarını (`username`, `password`) bildiğimiz için, kullanıcı adı ve parolaları doğrudan çekiyoruz:

```
Pets' UNION SELECT username,password FROM users--
```

İstek `200 OK` dönüyor; `users` tablosundaki satırlar ürün listesine ek olarak sonuçlara ekleniyor.

![Repeater — username,password FROM users](images/05-repeater-veri-cekme.png)

### 4. Sonuçları tarayıcıda görüntülemek

Aynı payload'ı tarayıcıda URL üzerinden çalıştırıyoruz:

```
/filter?category=Pets'+UNION+SELECT+username,password+FROM+users--
```

Kullanıcı adı ve parolalar sayfada listeleniyor.

![Tarayıcıda sonuçlar](images/05-tarayici-sonuclar.png)

Çekilen kimlik bilgileri arasında `administrator` kullanıcısının parolası da yer alıyor:

![Kimlik bilgileri — carlos, administrator, wiener](images/05-kimlik-bilgileri.png)

### 5. administrator olarak giriş yapmak

Elde ettiğimiz `administrator` kullanıcı adı ve parolasıyla giriş formundan oturum açıyoruz. `administrator` hesabına erişildiğinde lab çözülmüş olarak işaretleniyor.

![Lab çözüldü — administrator olarak giriş](images/05-lab-cozuldu.png)

## Neden Çalışıyor?

Bu saldırı önceki lab’lerdeki tekniklerin birleşimidir: önce sütun sayısı ve metin uyumlu sütunlar tespit edilir, sonra `UNION SELECT` ile orijinal sorgunun sonuçlarına tamamen farklı bir tablodan (`users`) satırlar eklenir. Uygulama bu birleşik sonuç kümesini olduğu gibi ekranda gösterdiği için, hassas veriler (kullanıcı adı/parola) doğrudan sızdırılmış olur. Sütun adları ve tablo adı bu lab'de verilmişti; gerçek senaryolarda bunlar da veritabanının meta verisinden (ör. `information_schema`) enjeksiyonla öğrenilebilir.

## Önlem

- **Parametreli sorgular (prepared statements):** Kullanıcı girdisi sorguya birleştirilmemeli; böylece `UNION SELECT ... FROM users` gibi ifadeler enjekte edilemez.
- **Girdi doğrulama (allow-list):** `category` yalnızca bilinen değerlere karşı doğrulanmalı.
- **Parolaların güvenli saklanması:** Parolalar düz metin yerine güçlü bir hash algoritmasıyla (ör. bcrypt) saklanmalı; böylece tablo sızsa bile parolalar doğrudan kullanılamaz.
- **En az yetki prensibi:** Uygulamanın veritabanı kullanıcısı yalnızca gereken tablolara erişebilmeli.
