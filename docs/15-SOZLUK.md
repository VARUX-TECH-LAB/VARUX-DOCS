# Sözlük

| Terim | Türkçe açıklama |
|---|---|
| deterministic | Deterministik. Aynı input ve aynı policy ile aynı kararın üretilmesi. AI tahmini veya rastlantısal yorum değildir. |
| policy | Politika. Hangi SQL operasyonunun allow, block veya approval gerektirdiğini belirleyen güvenlik kuralları. |
| execution | Çalıştırma. SQL'in PostgreSQL'e gönderilip veritabanında uygulanması veya sonuç üretmesi. |
| audit evidence | Denetlenebilir kanıt. Karar, context, policy metadata ve hash-chain bilgisi taşıyan güvenlik kaydı. |
| WAL | Write-ahead log. AXIS'te audit evidence için source of truth olarak kullanılan append-only kayıt dosyası. |
| JSONL projection | JSON Lines projection. WAL commit sonrası operator kolaylığı için yazılan türetilmiş JSON satırları. WAL'ın yerine geçmez. |
| hash chain | Hash zinciri. Her event'in bir önceki event hash'ine bağlanması. |
| event_hash | Event içeriğinden hesaplanan SHA-256 hash. Event değişirse hash doğrulaması bozulur. |
| previous_hash | Önceki event'in `event_hash` değeri. Zincir sürekliliğini sağlar. |
| approval | Onay. Riskli işlemin ilk istekte çalışmaması ve operatör kararı gerektirmesi. |
| fail-safe | Hata durumunda güvenli tarafa geçme prensibi. |
| fail-closed | Belirsizlik veya kritik hata durumunda execution yolunun kapalı kalması. |
| manifest | Active policy dosyasını, policy kimliğini, versiyonunu ve SHA-256 değerini tanımlayan dosya. |
| SHA-256 | Kriptografik hash algoritması. AXIS policy integrity ve event hash hesaplarında kullanılır. |
| parser/classifier | SQL'i parse edip read/write/DDL, target, scope ve risk sinyallerini çıkaran katman. |
| control plane | Operatör arayüzü. AXIS backend'e server-side proxy üzerinden bağlanan Next.js UI. |
| operator token | Operatör yetkilendirme token'ı. Approval resolve ve policy mutation gibi endpoint'lerde kullanılır; browser'a verilmemelidir. |
| trust boundary | Güven sınırı. Hangi bileşenin trusted, hangi girdinin untrusted olduğunu ayıran mimari çizgi. |
| production write path | Production verisini değiştiren write/delete/DDL operasyonlarının geçtiği kritik execution yolu. |
| `ALLOW` | Policy'nin execution'a izin verdiği karar. |
| `BLOCK` | Policy, parser veya fail-safe mekanizmanın execution'ı durdurduğu karar. |
| `APPROVAL_REQUIRED` | Kavramsal approval kararı. Mevcut API'de çoğunlukla `REQUIRE_APPROVAL` adıyla görünür. |
| `SUSPEND` | Kavramsal askıya alma durumu. Mevcut policy enum'u değildir; approval bekleme veya manual review gibi durumları anlatır. |
| `REJECT` | Approval resolve kararında reddetme. Store durumunda `REJECTED`, final execution kararında block olarak yansır. |

