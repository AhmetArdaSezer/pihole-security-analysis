# 🌐 Pi-hole: Güvenli Web Yazılımı ve Tehdit Modellemesi
**Öğrenci:** Ahmet Arda Sezer
**Bölüm:** Bilişim Güvenliği Teknolojisi
**Seçilen Senaryo:** Senaryo 1 - Pasaport Kontrolü / Authentication

## 📋 Proje Kapsamı
Bu rapor, Pi-hole yönetim (Dashboard) panelinin mimari yapısını, kimlik doğrulama süreçlerini, web zafiyetlerini ve yazılımın sürekli entegrasyon güvenliğini (DevSecOps) incelemektedir.

---

## 🚀 Analiz Aşamaları

### Adım 1: Kurulum Scripti ve Web Dizin Güvenliği
Uygulamanın kullandığı `basic-install.sh` betiği web mimarisi açısından statik olarak incelenmiştir. Scriptin dışarıdan paket çekerken uyguladığı SHA-1 bütünlük kontrolleri teyit edilmiştir. Kurulum esnasında admin arayüzünün barındırılacağı `/var/www/html/admin` web dizininin root yetkileriyle izole bir şekilde oluşturulduğu ve web sunucusu (Lighttpd) yetkilendirme yapıları analiz edilmiştir.


![vize](https://github.com/user-attachments/assets/550c07f5-146a-4749-8753-e4a82f6ba203)
![vize 2](https://github.com/user-attachments/assets/e752e6d4-e094-4b8c-a607-e0a6bb7e1e17)

### Adım 2: İzolasyon ve Web Servislerinin Temizliği (Clean-up)
Uygulamanın kaldırılma sürecinin "Clean-up" kalitesi sanal makine (Sandbox) üzerinde test edilmiştir. `sudo pihole uninstall` komutu sonrası sistemde web sunucusu (Lighttpd) yapılandırmalarının ve PHP/Lua kalıntılarının tamamen izole edildiği adli komutlarla doğrulanmıştır.


![gene](https://github.com/user-attachments/assets/9fb99f11-a39a-42d7-8c3b-83760b051053)

### Adım 3: CI/CD Pipeline ve Webhook Güvenliği
GitHub Actions üzerindeki `test.yml` dosyası incelenmiştir. Kodun her PR (Pull Request) anında Webhook üzerinden tetiklenerek otomatik güvenlik testlerinden ve linter kontrollerinden geçirilmesi, güvenli yazılım yaşam döngüsü (SDLC) açısından değerlendirilmiştir.

### Adım 4: Docker Mimarisi ve Ağ Güvenliği
Docker imajının katmanlı yapısı incelenmiştir. Web arayüzünün (Port 80/443) ve arka plan servislerinin izole bir Docker konteyneri içinde çalıştırılarak host sistemden nasıl ayrıştırıldığı ve saldırı yüzeyinin nasıl daraltıldığı analiz edilmiştir.

### Adım 5: Kaynak Kod Analizi (Pasaport Kontrolü & CSRF)
Admin panelindeki `login.lp` (Lua Pages) dosyası "Reasoning" tekniği ile taranmıştır.
* **Authentication:** Oturum yönetiminin **Stateful Cookie (Çerez)** tabanlı olduğu tespit edilmiştir. Şifre sıfırlama mekanizmasının web üzerinden tamamen engellenerek (`pihole setpassword` terminal komutuna zorlanarak) Brute-Force riskinin ortadan kaldırıldığı saptanmıştır.
* **Tehdit Modelleme (CSRF):** Form yapısında "Anti-CSRF" token eksikliği saptanmış ve oturumu açık kullanıcılar üzerinden yapılabilecek "Sahte İstek Gönderimi" (CSRF) saldırı senaryosu modellenmiştir.



---![vizee](https://github.com/user-attachments/assets/e48303a8-3fa3-4f12-a397-5e825a415bee)


## 🛑 Tespit Edilen Zafiyetler ve Geliştirme Önerileri (Mitigation)
Web arayüzü ve oturum yönetimi üzerinde yapılan tehdit modellemesinde şu zafiyetler tespit edilmiştir:

1. **Anti-CSRF Token Eksikliği (Yüksek):**
   * **Bulgu:** `login.lp` kaynak kodları incelendiğinde, `<form id="loginform">` bloğunun içinde dışarıdan gelen sahte istekleri engelleyecek benzersiz bir CSRF token bulunmadığı görülmüştür. Bu eksiklik, oturumu açık olan kurbanın yetkileriyle arka planda gizlice ayarlarının değiştirilmesine yol açabilir.
   * **Geliştirme Önerisi:** Backend tarafında her oturum için dinamik bir token üretilmeli ve frontend formlarına `<input type="hidden" name="csrf_token" value="...">` şeklinde gömülerek isteklerin Origin doğrulaması zorunlu tutulmalıdır.

2. **Çerez (Cookie) Güvenlik Bayrakları (Orta):**
   * **Bulgu:** Sistem "uses cookie" mantığıyla çalışmasına rağmen, modern çerez güvenlik politikalarının katı uygulanmadığı durumlar tespit edilmiştir. 
   * **Geliştirme Önerisi:** Uygulamanın API'si, oturum çerezlerini oluştururken mutlaka `HttpOnly`, `Secure` ve `SameSite=Strict` bayraklarını (flags) kullanmaya zorlanmalıdır. Bu sayede XSS saldırılarında çerez hırsızlığı engellenmiş olur.





### 📊 CSRF Saldırı Akış Diyagramı (Data Flow)
Aşağıdaki şema, tespit edilen CSRF zafiyetinin nasıl istismar edilebileceğini modellemektedir:

```mermaid
sequenceDiagram
    participant Kurban as Kurban (Admin)
    participant Hacker as Zararlı Site
    participant Pihole as Pi-hole API
    
    Kurban->>Pihole: 1. Sisteme Giriş Yapar (Session Cookie Alır)
    Hacker->>Kurban: 2. Sosyal Mühendislik ile Link Gönderir
    Kurban->>Hacker: 3. Zararlı Sayfayı (Kedi Videosu) Açar
    Hacker->>Pihole: 4. Arka planda gizli POST isteği yollar (Cookie ile)
    Pihole-->>Pihole: 5. CSRF Token YK! İstek güvenilir sanılır.
    Pihole-->>Hacker: 6. DNS Ayarları Değiştirilir.
