# AXIS Nedir?

AXIS, production veritabanı operasyonlarını çalışmadan önce politika ile değerlendiren ve her kritik karar için doğrulanabilir audit evidence üreten deterministik bir kontrol katmanıdır.

Mevcut implementasyonda AXIS, Rust tabanlı bir HTTP servis olarak çalışır. Uygulama, internal tool, script, AI agent veya servisler SQL isteklerini AXIS'in `/query` endpoint'ine gönderir. AXIS isteği doğrular, SQL'i PostgreSQL odaklı parser/classifier ile sınıflandırır, aktif policy'ye göre karar verir ve sadece izin verilen işlemleri PostgreSQL adapter üzerinden yürütür.

## Hangi problemi çözer?

Production veritabanlarında asıl risk çoğu zaman "hiç yetkisi olmayan biri bağlandı" değildir. Daha sık görülen risk, yetkili bir servis veya operatörün yanlış, geniş kapsamlı veya geri dönüşü zor bir SQL çalıştırmasıdır.

Örnek riskler:

- `WHERE` olmayan `DELETE`
- geniş kapsamlı `UPDATE`
- yanlış tenant veya tablo üzerinde write
- `DROP TABLE`, `TRUNCATE`, şema değiştiren DDL
- tek istekte birden fazla statement
- prepared statement ile niyetin saklanması
- unsupported SQL shape'in yanlışlıkla güvenli sanılması

AXIS bu riskleri execution öncesinde görünür ve kontrol edilebilir hale getirir.

## Authorized execution neden risklidir?

Sadece identity veya bağlantı yetkisi yeterli değildir. Yetkili bir backend servisi, migration job'u, admin script'i veya AI agent yanlış SQL üretebilir. Veritabanı açısından bağlantı geçerli olabilir; fakat çalışacak operasyon production verisi için riskli olabilir.

AXIS bu nedenle sadece "kim bağlanıyor?" sorusuna bakmaz. "Ne çalışmak üzere?", "hangi tabloda?", "hangi kapsamda?", "hangi ortamda?", "hangi policy altında?" sorularını da sorar.

## WRITE, DELETE ve DDL neden kritiktir?

Read-only sorgular genellikle geri dönüşsüz veri değişimi yapmaz. Buna rağmen `SELECT FOR UPDATE`, `SELECT INTO` veya write-capable CTE gibi read görünümlü riskler ayrıca değerlendirilmelidir.

WRITE, DELETE ve DDL ise production durumunu doğrudan değiştirir:

| Operasyon | Risk |
|---|---|
| `INSERT` | yanlış tenant, eksik scope, kritik tabloya kayıt |
| `UPDATE` | geniş kapsamlı veri değişikliği |
| `DELETE` | geri dönüşü zor veri kaybı |
| DDL | tablo, kolon, index veya şema yapısının değişmesi |
| Multi-statement | güvenli görünen istekle tehlikeli ikinci statement'ın birlikte gelmesi |

AXIS özellikle bu write path üzerinde deterministik karar üretir.

## AXIS neyi korur?

AXIS, kendisinden geçen protected database operation akışını korur.

Koruduğu alanlar:

- SQL'in execution öncesi sınıflandırılması
- policy tabanlı `ALLOW`, `BLOCK`, `REQUIRE_APPROVAL` kararı
- approval gerektiren işlemlerin ilk istekte çalıştırılmaması
- onaylı retry için aynı request bağlamı, SQL fingerprint ve policy metadata kontrolü
- audit WAL üzerinde hash-chain evidence üretimi
- policy manifest ve SHA-256 doğrulaması
- fail-safe davranışla tehlikeli belirsizliklerin güvenli tarafa çekilmesi

## AXIS neyi korumaz?

AXIS bütün güvenlik problemlerini tek başına çözmez.

AXIS şunların yerine geçmez:

- IAM/RBAC
- network security
- database role separation
- backup ve restore planı
- monitoring ve alerting
- secrets management
- least privilege
- servis kimliği
- production deployment discipline
- harici tamper-proof ledger
- remote attestation

AXIS'ten tamamen bypass edilerek doğrudan PostgreSQL'e giden write trafiğini AXIS kontrol edemez. Production ortamda network, credential ve role separation ile protected write path'in AXIS'ten geçmesi zorunlu kılınmalıdır.

## AXIS ne değildir?

AXIS:

- sadece loglama aracı değildir
- sadece reverse proxy değildir
- WAF değildir
- IAM/RBAC yerine geçen tek başına bir sistem değildir
- AI agent değildir
- bütün güvenlik problemlerini otomatik çözen bir ürün değildir
- compliance certification kanıtı değildir
- native PostgreSQL wire protocol desteğini mevcut HTTP adapter modeliyle otomatik kanıtlamaz

AXIS'in değeri, production veritabanı operasyonu çalışmadan önce deterministik karar üretmesi ve kritik kararların audit evidence ile doğrulanabilir olmasındadır.

