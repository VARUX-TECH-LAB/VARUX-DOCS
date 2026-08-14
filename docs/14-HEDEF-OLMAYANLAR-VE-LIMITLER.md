# Hedef Olmayanlar ve Limitler

Bu dosya AXIS'in ne iddia etmediğini açıkça yazar. Bu sınırlar zayıflık saklamak için değil, doğru teknik değerlendirme için gereklidir.

## AXIS'in iddia etmediği şeyler

AXIS:

- bütün database security kontrollerinin yerine geçmez,
- IAM/RBAC yerine tek başına geçmez,
- backup, monitoring, network security veya least privilege yerine geçmez,
- WAF değildir,
- AI agent değildir,
- compliance certification sağlamaz,
- her SQL dialect için tam coverage iddia etmez,
- native PostgreSQL wire protocol desteğini mevcut HTTP adapter ile kanıtlamaz,
- direct DB bypass trafiğini kendi içinde engelleyemez,
- remote attestation veya hardware-backed trust sağlamaz,
- production-ready HA mimarisi iddia etmez.

## Local manifest SHA-256 limiti

Policy manifest içindeki SHA-256, local policy dosyasının manifest ile uyumlu olduğunu gösterir. Bu önemlidir ama sınırı açıktır:

- asymmetric signing değildir,
- remote attestation değildir,
- KMS/HSM backed key trust değildir,
- multi-instance consensus değildir,
- host compromise riskini ortadan kaldırmaz.

## Native database wire protocol limiti

Mevcut repo, HTTP `/query` adapter modelinin yanında native PostgreSQL wire protocol entegrasyonunu da içerir; wire coverage ayrı evidence paketiyle doğrulanmıştır (Prisma OID-less bind inference; Hibernate 6 + pgjdbc explicit-OID binds, batch, savepoint/rollback; mTLS dahil). Bu, transparent drop-in proxy veya her driver/ORM kombinasyonu için tam coverage iddiası değildir; protokol-edge uyumluluk farkları (ör. unknown portal/statement handling, SQLSTATE redaction, named-portal re-Bind, DECLARE CURSOR) aktif hardening altındadır.

Bu özellikle müşteri entegrasyonlarında net söylenmelidir.

## Direct DB bypass limiti

AXIS sadece kendisinden geçen operasyonları kontrol eder. Eğer bir servis doğrudan PostgreSQL'e write-capable credential ile bağlanabiliyorsa AXIS bu write'ı göremez.

Production deployment'ta şunlar gerekir:

- private DB network,
- firewall/security group,
- DB role separation,
- credential discipline,
- uygulamaların write-capable direct credential almaması,
- bypass monitoring.

## Policy kalitesi limiti

AXIS policy kadar iyi karar verir. Kötü yazılmış veya fazla permissive policy, AXIS'in güvenlik değerini azaltır.

Bu yüzden:

- policy review gerekir,
- candidate diff gerekir,
- dry-run gerekir,
- unsafe default allow risklidir,
- policy değişikliği audit edilmelidir.

## Deployment trust limiti

AXIS runtime host, audit storage ve policy dosyaları güven sınırı içindedir. Bu host veya storage kontrol dışına çıkarsa local evidence güveni zayıflar.

Bu nedenle production planı:

- hardened host,
- restricted filesystem permissions,
- backup ve retention,
- secret management,
- monitored audit path,
- incident runbook içermelidir.

## Approval limiti

Approval, ilk request'i bekletir ve execution yapmaz. Approved durumunda execution için aynı request context ve `approval_id` ile retry gerekir. Bu model uzun süre açık database transaction tutmaz.

Bu iyi bir güvenlik/operasyon tradeoff'u olabilir, fakat uygulama entegrasyonunun retry semantics'i doğru uygulaması gerekir.

## Prepared statement limiti

Mevcut model AXIS-side prepared intent tracking yapar. Database-side PostgreSQL prepared statement connection affinity iddiası yapmaz. In-memory session state restart sonrası kaybolur; eski `EXECUTE` fail-safe block olur.

## Evidence signing limiti

Repo evidence bundle signing için Ed25519 desteği içerir. Ancak signing etkin değilse evidence export bunu imzasız olarak raporlamalıdır. İmzalı bundle bile host ve key management güvenlik varsayımlarını ortadan kaldırmaz.

## Performans limiti

AXIS execution path'e ek kontrol ve audit maliyeti getirir. Benchmark sonuçları müşteri workload'u için otomatik garanti değildir. Pilotlar gerçek workload, latency toleransı ve failure mode gereksinimleriyle test edilmelidir.

