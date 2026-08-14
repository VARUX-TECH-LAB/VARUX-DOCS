# Fail-Safe Davranış

AXIS'in güvenlik modeli riskli belirsizliklerde fail-closed davranmayı hedefler. Yani sistem bir işlemi güvenli şekilde değerlendiremiyorsa, production write path'i açık bırakmamalıdır.

## Fail-safe ve fail-closed

Fail-safe genel prensiptir: hata durumunda güvenli tarafa geç.

Fail-closed ise bu prensibin write path karşılığıdır: belirsizlik veya güvenlik kritik hata varsa execution kapalı kalır.

## Başlangıç fail-fast

AXIS başlangıçta bazı koşulları doğrular:

- audit WAL açılabilir ve zincir devam ettirilebilir olmalı,
- policy manifest okunabilir olmalı,
- active policy path policy dizini dışına çıkmamalı,
- SHA-256 manifest ile eşleşmeli,
- policy schema valid olmalı,
- activation dry-run corpus başarısız olmamalı,
- approval SQLite store bozuk olmamalı,
- production runtime profile zayıf operator token ile başlamamalı.

Bu kontroller başarısızsa sistemin trafik kabul etmemesi beklenir.

## Runtime protected execution safeguards

Protected write için önemli kural şudur:

```text
Durable audit decision evidence yazılamıyorsa protected execution ilerlememeli.
```

Mevcut enforcer, protected write öncesi decision evidence commit başarısız olursa DB call yapmadan block döndürür.

## Audit write failure behavior

Audit WAL yazılamıyorsa:

- protected write execution öncesi durdurulmalıdır,
- block veya service unavailable gibi güvenli response dönmelidir,
- runtime log güvenli özet yazabilir ama audit evidence yerine geçmez.

Execution gerçekleştikten sonra result evidence commit başarısız olursa durum farklıdır: PostgreSQL execution confirmed olabilir, fakat durable result evidence yoktur. Mevcut implementasyon bunu kritik integrity state olarak raporlar; sessiz başarı gibi göstermemelidir.

## Policy invalid behavior

Policy manifest eksik, checksum mismatch veya policy validation hatalıysa:

- başlangıç fail-fast olmalıdır,
- permissive default policy'ye sessiz fallback yapılmamalıdır,
- activation failure evidence yazılabiliyorsa yazılmalıdır.

## Approval corruption behavior

Approval store bozuksa unsafe resolve yapılmamalıdır. Mevcut store SQLite integrity check ve schema doğrulaması yapar; corrupt DB startup veya store açılışında hata üretir.

## Hata davranışı tablosu

| Durum | Beklenen davranış | Neden |
|---|---|---|
| Policy yüklenemiyor | Startup fail-fast | Yanlış policy ile write path açılmamalı |
| Policy checksum mismatch | Startup fail-fast | Local policy bütünlüğü bozulmuş olabilir |
| Audit WAL corrupt | Startup fail-fast | Evidence zinciri güvenilmez |
| Audit WAL unwritable | Protected execution durmalı | Evidence olmadan kritik write yapılmamalı |
| Approval store corrupt | Resolve/retry durmalı | Onay durumu güvenilmez |
| Parser failure | BLOCK | Güvenli sınıflandırılamayan SQL çalışmamalı |
| Multi-statement | BLOCK | Gizli ikinci operasyon riski |
| DB timeout | Execution state unknown raporlanmalı | Protected write otomatik retry edilmemeli |
| Result evidence commit failure | Kritik integrity failure | Execution olmuş ama evidence eksik |

## Dürüst raporlama

Fail-safe davranışın önemli parçası, belirsizliği saklamamaktır. AXIS "başarılı" diyemiyorsa bunu açıkça raporlamalıdır:

- `execution_state: unknown`
- `result_evidence_commit_failed`
- `audit_storage_unavailable`
- `approval_requires_manual_review`

Bu alanlar operatöre incident veya manuel inceleme gerekip gerekmediğini gösterir.

