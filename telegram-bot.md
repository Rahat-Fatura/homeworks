### 1. Proje Tanımı ve Amaçları

#### Amaçlar:

-   Telegram bot aracılığıyla web sitelerinin durumunu kontrol etmek ve kullanıcılara bildirmek.
-   PostgreSQL veritabanı kullanarak domain ve durum bilgilerini depolamak.
-   Bull MQ kullanarak periyodik sorgulamaları ve hata yönetimini yapmak.
-   Kullanıcılara mail gönderme işlevselliği sunmak.
-   Express ile dışarıya API sunmak ve sorgulama geçmişini pagination mantığıyla sağlamak.

### 2. Teknoloji Yığını

-   Node.js
-   Express framework
-   PostgreSQL
-   Telegram Bot API
-   Bull MQ

### 3. Telegram Bot Tasarımı

##### Komutlar:

-   /domain_ekle: Kullanıcının veritabanına domain eklemesini sağlar.
-   /domain_sil: Kullanıcının veritabanından domain silmesini sağlar.
-   /domain_listele: Veritabanındaki domainleri listeler.

##### Örnek Senaryolar:

Kullanıcı /domain_ekle komutunu gönderdiğinde, bot kullanıcıdan domain bilgisini alır ve veritabanına kaydeder.
Kullanıcı /domain_sil komutunu gönderdiğinde, bot kullanıcıdan silmek istediği domaini alır ve veritabanından siler.
Kullanıcı /domain_listele komutunu gönderdiğinde, bot veritabanındaki tüm domainleri kullanıcıya listeler.

### 4. Uygulama Mimarisinin Tasarımı

##### Veritabanı Şeması:

-   users (id, username, email): Kullanıcı bilgilerini tutar.
-   domains (id, user_id, domain_name): Kullanıcıya ait domain bilgilerini tutar.
-   checks (id, domain_id, status_code, timestamp): Domainlerin kontrol edilme geçmişini tutar.

##### İlişkiler:

-   users ve domains tabloları arasında birçok kullanıcıya karşılık birçok domain ilişkisi vardır.
-   domains ve checks tabloları arasında birçok domainin birçok kontrol geçmişi ilişkisi vardır.

### 5. Temel Fonksiyonların Ayrıntılı Tanımı

##### 1. Domain Ekleme:

-   Kullanıcı /domain_ekle komutunu kullanarak bir domain ekleyebilir.
-   Telegram bot, kullanıcıdan domain bilgisini alır ve veritabanına kaydeder.
-   Eklenen domain, ilgili kullanıcıya bağlıdır.
-   Eğer veritabanında aynı isimde bir domain zaten varsa, hata mesajı gösterilir ve işlem iptal edilir.

##### 2. Domain Silme:

-   Kullanıcı /domain_sil komutunu kullanarak bir domain silebilir.
-   Telegram bot, kullanıcıdan silmek istediği domaini alır ve veritabanından siler.
-   Silinen domain, ilgili kullanıcıya bağlıdır.
-   Eğer veritabanında silinmek istenen domain bulunamazsa, hata mesajı gösterilir ve işlem iptal edilir.

##### 3. Domain Listeleme:

-   Kullanıcı /domain_listele komutunu kullanarak tüm domainleri listeleyebilir.
-   Telegram bot, veritabanındaki tüm domainleri ilgili kullanıcıya gösterir.
-   Eğer kullanıcının henüz eklenmiş bir domaini yoksa, uygun bir mesaj gösterilir.

##### Örnek Senaryolar:

-   Kullanıcı /domain_ekle komutunu kullanarak "example.com" domainini ekler.
-   Kullanıcı /domain_sil komutunu kullanarak "example.com" domainini siler.
-   Kullanıcı /domain_listele komutunu kullanarak tüm domainleri görüntüler.

### 6. Web Status Kontrolü

##### İş Akışı:

1. Belirli aralıklarla Bull MQ kuyruğuna eklenen sorgulama işlemleri, workerlar tarafından sıralı olarak işlenir.
2. Her sorgulama, ilgili domainin web durumunu kontrol eder.
3. Sorgulama periyotları:
    - 8:00 - 23:00 arası: 15 dakikada bir.
    - 23:00 - 8:00 arası: 30 dakikada bir.
4. Her sorgulama sonucu HTTP durum kodu değerlendirilir:
    - Durum kodu 400'den küçükse, veritabanına kaydedilir.
    - Durum kodu 400'den büyükse:
        - Sorgulama periyodu 1 dakikada bir olarak güncellenir.
        - Kullanıcıya hata bildirimi yapılır.
        - Durum kodu 400'den küçük olana kadar geçen süre takip edilir.
        - Durum kodu 400'den küçük olduğunda, kullanıcıya "x süredir kapalıydı, şimdi tekrar açıldı." gibi bir mesaj gönderilir.
        - Durum kodu 400'den büyük hata durumu 1 saati aşarsa, her saat başında "x saattir kapalı" mesajı gönderilir.
        - Tekrar açıldığında, toplam kapalı süre yeniden hesaplanır ve kullanıcıya "x saattir kapalıydı, şimdi tekrar açıldı." gibi bir mesaj gönderilir.

##### Örnek Senaryo:

-   Saat 10:00'da bir sorgulama işlemi başlatılır.
-   Domainin web durumu kontrol edilir ve 503 Service Unavailable durumu alınır.
-   Bu durum, sorgulama periyodu hızlandırılır ve kullanıcıya hata bildirimi yapılır.
-   Saat 10:01'de bir sonraki sorgulama işlemi başlatılır.
-   Domainin web durumu kontrol edilir ve 200 OK durumu alınır.
-   Bu durum, sorgulama periyodu tekrar normal hızına döner ve kullanıcıya "1 dakika boyunca kapalıydı, şimdi tekrar açıldı." gibi bir mesaj gönderilir.
-   Domain tekrar kapalı olduğunda, saat 11:00'de kullanıcıya "1 saat boyunca kapalıydı" gibi bir mesaj gönderilir.

### 7. Bull MQ Kuyruğu ve Workerları

##### Kuyruk ve İşleyiş:

-   Bull MQ, sorgulama işlemlerini düzenlemek ve sıraya almak için kullanılır.
-   Uygulama, sorgulama işlemlerini kuyruğa ekler.
-   Kuyruktaki işlemler, çalışan (worker) süreçler tarafından sırayla alınır ve işlenir.

##### Worker İşlevselliği:

-   Her worker, kuyruktan bir işi alır ve işler.
-   İşleme alınan her bir sorgulama, ilgili domainin web durumunu kontrol eder.
-   HTTP isteği ile domainin durumu belirlenir ve ilgili işlemler yapılır.
-   Workerlar, veritabanına HTTP durum kodu ve zaman damgası gibi bilgileri kaydeder.
-   Gerekli durumlarda kullanıcıya hata bildirimleri yapılır.

##### Örnek Senaryo:

-   Uygulama, Bull MQ kuyruğuna bir sorgulama işlemi ekler.
-   Bir worker, kuyruktaki işlemi alır ve ilgili domainin durumunu kontrol eder.
-   Domainin durumu 200 OK ise, durum veritabanına kaydedilir ve işlem tamamlanır.
-   Domainin durumu 404 Not Found ise, kullanıcıya hata bildirimi yapılır ve sorgulama periyodu hızlandırılır.
-   Eğer domainin durumu 400'den büyük hata durumunu 1 saat aşarsa, her saat başında kullanıcıya "x saattir kapalı" mesajı gönderilir.

##### Kuyruk ve Worker Ayarları:

-   Bull MQ, işçilerin eşzamanlı olarak kaç işi işleyebileceğini, işlemlerin ne kadar süreyle saklanacağını ve diğer ayarları yapılandırmanızı sağlar.
-   Bu ayarlar, uygulamanın ihtiyacına ve performans gereksinimlerine göre yapılandırılmalıdır.

### 8. Mail Gönderme

##### İşleyiş:

-   Telegram bot, gönderdiği tüm mesajların altına "Mail Gönder" butonu ekler.
-   Kullanıcı butona tıkladığında, bot tarafından kullanıcıdan bir mail adresi istenir.
-   Girilen mail adresi geçerli bir formatta olmalıdır.
-   Mail adresi doğrulandıktan sonra, butona basılan mesajın içeriği olduğu gibi mail olarak gönderilir.
-   Mail gönderme işlemi başarılı olduğunda, kullanıcıya bilgilendirme mesajı gönderilir.

##### Örnek Senaryo:

-   Telegram bot, bir mesaj gönderir ve mesajın altına "Mail Gönder" butonunu ekler.
-   Kullanıcı butona tıkladığında, bot tarafından kullanıcıdan bir mail adresi istenir.
-   Kullanıcı geçerli bir mail adresi girer ve onaylar.
-   Girilen mail adresi doğrulandıktan sonra, butona basılan mesajın içeriği olduğu gibi mail olarak gönderilir.
-   Mail gönderme işlemi başarılı bir şekilde tamamlanır ve kullanıcıya bilgilendirme mesajı gönderilir.

##### Mail Gönderme Ayarları:

-   Mail gönderme işlemi için kullanılacak SMTP sunucusu ve kimlik doğrulama bilgileri gibi ayarlar yapılandırılmalıdır.
-   Mail içeriği, kullanıcıya gelen mesajın içeriği ile aynı olmalıdır.
-   Mail gönderme işleminin başarılı veya başarısız olduğuna dair uygun günlüğe kayıt yapılmalıdır.

### 9. Express Endpoint Tasarımı

##### İşleyiş:

-   Uygulama, Express üzerinde belirli bir endpoint'i dinleyerek HTTP isteklerini karşılar.
-   Endpoint'e gelen POST isteğinin gövdesinde belirli parametreler bulunur.
-   İstek gövdesindeki parametrelere göre işlem yapılır ve uygun yanıt döndürülür.

##### Endpoint Detayları:

-   Endpoint: POST /api/domain-history
-   Header:
-   Content-Type: application/json
-   x-api-key: Sabit bir API anahtarı
-   İstek Gövdesi (JSON):

```json
{
    "page_index": 0,
    "page_size": 100
}
```

##### Yanıt:

-   page_index: İstenen sayfa indeksi
-   page_size: Sayfa boyutu
-   count: Toplam öğe sayısı
-   items: Sayfa içeriği

##### Örnek Senaryo:

-   Kullanıcı, /api/domain-history endpoint'ine bir POST isteği gönderir.
-   İstek başlığında Content-Type ve x-api-key ile ilgili değerler gönderilir.
-   İstek gövdesinde page_index ve page_size parametreleri belirtilir.
-   Uygulama, isteği doğrular, gerekli işlemi yapar ve uygun yanıtı oluşturur.
-   Oluşturulan yanıt, istemciye geri döndürülür.

##### Endpoint Güvenliği:

-   Endpoint'e erişim sadece yetkilendirilmiş kullanlar tarafından sağlanan doğru API anahtarı ile mümkündür.
-   API anahtarı, her istekte kontrol edilerek doğrulama yapılır.
-   Yetkilendirme başarısız olduğunda, HTTP 401 Unauthorized yanıtı gönderilir.
