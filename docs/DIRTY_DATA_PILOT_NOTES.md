# Dirty Data Pilot Notes

Dirty data exists to avoid toy-data review. Dirty data mode is optional and must not slow down quickstart by default.

The generator in `scripts/generate_dirty_pilot_data.py` creates deterministic synthetic data for:

- users
- customers
- accounts
- transactions
- admin_actions

It includes nulls where the schema allows them, JSONB metadata columns, unicode and edge-case strings, old timestamps, soft-deleted records, multiple tenants, suspicious admin actions, and bulk-operation candidate records.

100k rows are not an enterprise-scale performance claim. This is realism for behavior and integration testing, not a final benchmark.

Benchmark claims require separate controlled benchmark work with defined hardware, dataset shape, workload, warmup, repetitions, and measurement methodology.

No real people, companies, or secrets should be generated.

Example:

Windows:

```powershell
python scripts\generate_dirty_pilot_data.py --rows 10000 --output demo-evidence-output\dirty-pilot-data.sql
```

macOS/Linux:

```bash
python3 scripts/generate_dirty_pilot_data.py --rows 10000 --output demo-evidence-output/dirty-pilot-data.sql
```

Load manually into the pilot database only when you want dirty-data behavior review:

```bash
docker compose -f demo/docker-compose.pilot.yml exec -T pilot-postgres psql -U varux -d prod_main < demo-evidence-output/dirty-pilot-data.sql
```
