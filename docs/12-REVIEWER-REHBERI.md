# Reviewer Rehberi

Bu rehber, dış teknik reviewer'ın AXIS'i iddia, kaynak ve evidence ayrımıyla incelemesi için hazırlanmıştır.

## İlk neye bakmalı?

1. Protected write path gerçekten AXIS'ten geçiyor mu?
2. `/query` akışı execution öncesi parser/classifier ve policy evaluate yapıyor mu?
3. `ALLOW`, `BLOCK`, `REQUIRE_APPROVAL` davranışları response ve audit event'lerde tutarlı mı?
4. Approval gereken request ilk istekte PostgreSQL'e gitmiyor mu?
5. Audit WAL hash-chain verification çalışıyor mu?
6. Policy manifest SHA-256 mismatch startup'ı durduruyor mu?
7. Control Plane browser'a backend URL veya operator token sızdırıyor mu?
8. Limitler açıkça yazılmış mı?

## Hangi claim nasıl doğrulanır?

| Claim | Doğrulama yolu |
|---|---|
| SQL execution öncesi policy kararı var | `src/gate/listener.rs`, `src/gate/enforcer.rs`, `/query` testleri |
| Dangerous delete block olur | Policy fixture ve regression testleri, `/query` örnekleri |
| Approval ilk request'te execution yapmaz | Approval response `NOT_EXECUTED`, audit event, DB state kontrolü |
| Approved retry exact context ister | `ApprovalRetryProof`, SQL fingerprint ve policy metadata eşleşme kontrolleri |
| WAL hash chain doğrulanır | `/evidence/verify`, `/audit/verify`, `src/audit/verifier.rs` |
| Policy integrity startup'ta kontrol edilir | `src/policy/manifest.rs`, checksum mismatch testi |
| Browser token görmez | `control-plane/src/app/api/axis/[...path]/route.ts`, client proxy çağrıları |

## AXIS nasıl zorlanır?

Reviewer şu inputları denemelidir:

- `DELETE FROM users`
- `DELETE FROM orders WHERE status='old'`
- `UPDATE orders SET status='x'`
- `UPDATE orders SET status='x' WHERE id=1 AND tenant_id='acme'`
- `DROP TABLE users`
- `SELECT 1; DELETE FROM users`
- `VACUUM`
- malformed SQL
- oversized SQL
- unknown prepared `EXECUTE`
- missing `session_id` ile `PREPARE` veya `EXECUTE`
- aynı approval id ile paralel retry
- policy checksum mismatch
- audit WAL tamper

## Kritik davranışlar

- Block edilen SQL PostgreSQL'e ulaşmamalı.
- Approval-required SQL ilk request'te çalışmamalı.
- Reject edilen approval execution'a dönüşmemeli.
- Approved retry sadece aynı bağlamla çalışmalı.
- Audit write failure protected write'ı durdurmalı.
- Result evidence commit failure sessiz başarı olmamalı.
- Unsupported SQL güvenli tarafta kalmalı.

## Anlamlı testler

- `cargo test`
- parser bypass case corpus
- policy case corpus
- approval race testleri
- audit restart continuity testleri
- chaos/failure-mode scriptleri
- `/evidence/verify` ve offline evidence verification
- Control Plane build/typecheck/e2e real mode testleri

## Sorulması gereken sorular

- Production deployment'ta direct DB bypass nasıl engellenecek?
- Request identity gerçekten nereden geliyor?
- Operator token nasıl saklanıyor ve rotate ediliyor?
- Policy değişikliklerini kim review ediyor?
- Audit WAL nerede tutuluyor, nasıl yedekleniyor?
- Evidence export imzalı mı, imzasızsa bu açıkça raporlanıyor mu?
- Native wire protocol gerekli mi, yoksa HTTP adapter yeterli mi?
- Failure durumunda operatör runbook'u ne diyor?

## Red flag'ler

- "AXIS her şeyi çözer" gibi genel iddia.
- Direct DB bypass limitinin saklanması.
- Runtime logların audit evidence gibi sunulması.
- Approval reject için evidence olmaması.
- Policy checksum mismatch'e rağmen servis başlaması.
- Browser'ın `AXIS_OPERATOR_TOKEN` veya backend URL görmesi.
- Unsupported SQL'in allow edilmesi.
- Evidence chain bozulduğunda sessiz repair yapılması.

## İyi işaretler

- Claim'lerin kaynak dosya ve testlerle bağlanması.
- `policy_id`, `policy_version`, `policy_sha256` alanlarının karar ve evidence içinde bulunması.
- Approval resolve sonrası immutable final state.
- Audit WAL'ın source of truth olarak ele alınması.
- JSONL projection ve runtime logs için sınırlı rol tanımı.
- Limitlerin açık ve görünür yazılması.
- Fail-closed davranışın testlerle desteklenmesi.

## Limitler nerede?

Limitlerin merkezi özeti [14-HEDEF-OLMAYANLAR-VE-LIMITLER.md](14-HEDEF-OLMAYANLAR-VE-LIMITLER.md) dosyasındadır. Reviewer, herhangi bir güçlü claim gördüğünde bu limitlerle karşılaştırmalıdır.

