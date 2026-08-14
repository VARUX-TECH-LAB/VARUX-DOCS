# Kontrol Listesi

Bu checklist, açıklama paketinin amacına ulaşıp ulaşmadığını kontrol etmek için kullanılabilir.

## Anlama kontrolü

- [ ] Okuyucu AXIS'i tek cümlede anlatabiliyor mu?
- [ ] Okuyucu AXIS'in ne olmadığını açıkça görebiliyor mu?
- [ ] Okuyucu mimariyi kabaca çizebiliyor mu?
- [ ] Okuyucu policy-before-execution modelini anlayabiliyor mu?
- [ ] Okuyucu traditional execution sonrası log modeli ile AXIS modelini ayırabiliyor mu?
- [ ] Okuyucu `ALLOW`, `BLOCK`, `REQUIRE_APPROVAL` davranışlarını ayırabiliyor mu?
- [ ] Okuyucu `APPROVAL_REQUIRED` kavramsal adı ile API'deki `REQUIRE_APPROVAL` adını karıştırmadan anlayabiliyor mu?

## Evidence kontrolü

- [ ] Okuyucu audit evidence modelini anlayabiliyor mu?
- [ ] Okuyucu WAL'ın source of truth olduğunu görebiliyor mu?
- [ ] Okuyucu JSONL projection ve runtime logs'un sınırlı rolünü anlayabiliyor mu?
- [ ] Okuyucu `event_hash` ve `previous_hash` ilişkisini açıklayabiliyor mu?
- [ ] Okuyucu restart continuity'nin neden önemli olduğunu anlayabiliyor mu?
- [ ] Okuyucu evidence export'un ne işe yaradığını ve neyi kanıtlamadığını görebiliyor mu?

## Approval kontrolü

- [ ] Okuyucu approval flow'u anlayabiliyor mu?
- [ ] Pending, approved, rejected ve retry ayrımı net mi?
- [ ] Reject durumunda bile evidence gerekliliği açıklanmış mı?
- [ ] Final approval state'in immutable olması anlaşılır mı?

## Fail-safe kontrolü

- [ ] Okuyucu fail-safe davranışı anlayabiliyor mu?
- [ ] Policy invalid olduğunda startup fail-fast gerektiği açık mı?
- [ ] Audit write failure durumunda protected execution'ın durması gerektiği açık mı?
- [ ] Result evidence commit failure'ın dürüstçe raporlanması gerektiği açık mı?

## Reviewer kontrolü

- [ ] Reviewer claim/evidence ayrımını görebiliyor mu?
- [ ] Hangi claim'in hangi kaynak veya endpoint ile doğrulanacağı yazılmış mı?
- [ ] Red flag ve good sign listeleri yeterince somut mu?
- [ ] Failure-mode matrix ciddi ve reviewer-friendly mi?

## Limit kontrolü

- [ ] Limitler açıkça yazılmış mı?
- [ ] Direct DB bypass sınırı saklanmamış mı?
- [ ] Native wire protocol limitleri açık mı?
- [ ] Local manifest SHA-256'ın remote attestation olmadığı yazılmış mı?
- [ ] IAM, backup, monitoring, network security ve least privilege gerekliliği korunmuş mu?
- [ ] Marketing veya abartılı iddia var mı?

