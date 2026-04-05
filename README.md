# 🌐 Pi-hole: Güvenli Web Yazılımı ve Tehdit Modellemesi
**Öğrenci:** Ahmet Arda Sezer
**Bölüm:** Bilişim Güvenliği Teknolojisi
**Seçilen Senaryo:** Senaryo 1 - Pasaport Kontrolü / Authentication

## 📋 Proje Kapsamı
Bu rapor, Pi-hole yönetim (Dashboard) panelinin mimari yapısını, kimlik doğrulama süreçlerini, web zafiyetlerini ve yazılımın sürekli entegrasyon güvenliğini (DevSecOps) incelemektedir.

---

## 🛠️ Temel Sistem ve Dağıtım Güvenliği
* **Kurulum Güvenliği:** Sistemin bağımlılıkları çekerken uyguladığı SHA-1 bütünlük kontrolleri ve `/var/www/html/admin` web dizinini oluştururken kullandığı yetki kısıtlamaları incelenmiştir.
* **Sistem İzolasyonu:** Web sunucusu (Lighttpd) ve bağımlılıkların kaldırılma sürecinin kalitesi analiz edilmiş, sistemin temiz bir konfigürasyona dönme becerisi test edilmiştir.

## ⚙️ Otomasyon ve Ağ Katmanı İzolasyonu
* **CI/CD Süreçleri ve Webhook:** GitHub Actions üzerindeki CI/CD dosyaları incelenmiş; `on: pull_request:` mekanizmasıyla tetiklenen otomatik güvenlik testleri ve linter kontrollerinin (Güvenli SDLC) önemi vurgulanmıştır.
* **Docker Mimarisi:** Web arayüzünün ve arka plan servislerinin izole bir Docker konteyneri (minimal debian katmanı) içinde çalıştırılarak saldırı yüzeyinin nasıl daraltıldığı analiz edilmiştir.

## 🎯 Ana Tehdit Modeli: Authentication Analizi ve CSRF Zafiyeti
Projenin ana odağı, admin panelinin kaynak kodları (`login.lp`) üzerinden yapılan kimlik doğrulama testleridir.
* **Oturum Yönetimi Analizi:** Kaynak kod taraması (Reasoning) sonucunda sistemin **Stateful Cookie (Çerez)** mantığıyla çalıştığı belirlenmiştir. Ayrıca parola sıfırlama mekanizmasının web üzerinden tamamen engellenerek kaba kuvvet (Brute-Force) riskinin ortadan kaldırıldığı saptanmıştır.
* **Cross-Site Request Forgery (CSRF) Modellemesi:** Web form yapılarında yapılan analizde "Anti-CSRF" token eksikliği tespit edilmiştir. Çerez tabanlı oturum kullanan sistemlerde, oturumu açık bir kullanıcının zararlı bir siteye yönlendirilmesi durumunda arka planda istem dışı DNS konfigürasyonu değişiklikleri yapılmasına neden olabilecek CSRF saldırı vektörü modellenmiştir.

> `[BURAYA LOGIN.LP KODLARI SS]`

---

## 🛑 Tespit Edilen Zafiyetler ve Geliştirme Önerileri (Mitigation)
Web arayüzü ve oturum yönetimi üzerinde yapılan tehdit modellemesinde şu zafiyetler tespit edilmiştir:

1. **Anti-CSRF Token Eksikliği (Yüksek):**
   * **Bulgu:** `login.lp` kaynak kodları incelendiğinde, `<form id="loginform">` bloğunun içinde dışarıdan gelen sahte istekleri engelleyecek benzersiz bir CSRF token bulunmadığı görülmüştür. Bu eksiklik, oturumu açık olan kurbanın yetkileriyle arka planda gizlice ayarlarının değiştirilmesine yol açabilir.
   * **Geliştirme Önerisi:** Backend tarafında her oturum için dinamik bir token üretilmeli ve frontend formlarına `<input type="hidden" name="csrf_token" value="...">` şeklinde gömülerek isteklerin Origin doğrulaması zorunlu tutulmalıdır.

2. **Çerez (Cookie) Güvenlik Bayrakları (Orta):**
   * **Bulgu:** Sistem "uses cookie" mantığıyla çalışmasına rağmen, modern çerez güvenlik politikalarının katı uygulanmadığı durumlar tespit edilmiştir. 
   * **Geliştirme Önerisi:** Uygulamanın API'si, oturum çerezlerini oluştururken mutlaka `HttpOnly`, `Secure` ve `SameSite=Strict` bayraklarını (flags) kullanmaya zorlanmalıdır. Bu sayede XSS saldırılarında çerez hırsızlığı engellenmiş olur.
