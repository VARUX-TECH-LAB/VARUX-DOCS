# Hata Modu Matrisi

Bu matris, reviewer'ın AXIS davranışını hata senaryoları altında değerlendirmesi için hazırlanmıştır. Amaç sadece "hata oldu" demek değil; hatanın write-path güvenliği, evidence ve operatör aksiyonu açısından ne anlama geldiğini göstermektir.

| Hata senaryosu | Risk | AXIS'in beklenen davranışı | Üretilen evidence | Neden önemli? |
|---|---|---|---|---|
| Postgres down | Allowed query çalışmayabilir; write sonucu belirsiz olabilir | Startup connection check başarısızsa başlama hatası; runtime'da structured DB error | Execution failed veya DB error evidence, mümkünse response evidence | DB hatası policy allow gibi gösterilmemeli |
| Audit unwritable | Kritik karar kanıtlanamaz | Protected write execution öncesi fail-closed | `audit_storage_unavailable` reason; WAL yazılamıyorsa event olmayabilir, runtime safe summary olabilir | Evidence olmadan production write güvenilir değildir |
| Invalid policy | Yanlış enforcement | Startup fail-fast veya candidate rejection | Policy activation failed event yazılabiliyorsa yazılır | Permissive fallback tehlikelidir |
| Policy checksum mismatch | Policy dosyası manifest ile uyumsuz | Startup fail-fast | Activation failure evidence mümkünse | Local policy bütünlüğü bozulmuş olabilir |
| Corrupt audit WAL/log | Evidence zinciri güvenilmez | Startup fail-fast; verification tampered/unverifiable raporlar | Verification error veya startup hata | Bozuk evidence üzerine güvenli runtime inşa edilmemeli |
| Corrupt approval store | Yanlış approve/reject veya replay riski | Store açılışı/resolve fail-safe hata üretmeli | Approval resolve error; varsa runtime safe summary | Approval state güvenilir değilse execution yapılmamalı |
| Malformed request | Parser veya validator bypass riski | Request rejection, structured error | Request rejected / decision made / blocked evidence mümkünse | Kötü input generic panic'e dönüşmemeli |
| Huge payload | Memory/latency baskısı | Body veya SQL size limit ile reddet | `request_body_too_large` veya `sql_too_large`; oversized SQL raw yazılmaz | Resource abuse ve secret leak riski azalır |
| Parser failure | Tehlikeli SQL güvenli sanılabilir | `BLOCK` / fail-closed | `parser_error`, `unsupported_sql_shape` veya ilgili reason | Unsupported şekil allow olmamalı |
| Concurrent approval race | Aynı approval birden fazla execution için kullanılabilir | Tek reservation başarılı olmalı; diğerleri block | Approval retry blocked veya execution state evidence | Single-use approval garantisi kritik |
| Restart during traffic | Prepared state veya audit continuation kaybolabilir | Audit chain son hash'ten devam etmeli; in-memory prepared state kaybolunca unresolved execute block olmalı | Restart sonrası previous_hash continuity; unresolved prepared evidence | Restart güvenlik modelini zayıflatmamalı |
| DB pool pressure | Request'ler bekler veya panic riski | Bounded acquire timeout, structured `db_pool_exhausted` | Execution failed / response evidence mümkünse | Saturation güvenli hata olmalı |
| Evidence commit failure | Execution olmuş ama sonuç evidence yazılamamış olabilir | Execution öncesiyse block; execution sonrasıysa critical integrity failure raporla | `result_evidence_commit_failed`, integrity state | Sessiz başarı yanlış güven yaratır |
| DB timeout | PostgreSQL query sonucu unknown olabilir | `execution_state: unknown`; protected write otomatik retry edilmemeli | `execution_unknown` evidence mümkünse | Timeout sonrası yeniden deneme veri tutarsızlığı yaratabilir |
| Policy reload failure | Yanlış veya yarım policy state | Eski active policy korunmalı | Reload failure evidence mümkünse | Policy değişimi atomik ve güvenli olmalı |
| Direct DB bypass | AXIS hiç karar veremez | AXIS bunu runtime içinde engelleyemez; deployment kapatmalı | AXIS evidence üretmez çünkü trafik AXIS'e gelmez | En önemli deployment boundary budur |

## Reviewer notu

Bir hata senaryosu güvenli sayılabilmek için şu üç soruya cevap vermelidir:

- Production write PostgreSQL'e ulaştı mı?
- Ulaştıysa sonucu kesin mi, belirsiz mi?
- Bu durum audit evidence ile dürüstçe raporlandı mı?

