# Data

The notebook expects `data/pentathlon_nptb.parquet` (600,000 rows, 16 columns)
next to this folder. That file is **not included in this repo**.

Schema — see `pentathlon_nptb_description.md` for the full column definitions.
In short, one row per customer–email pair:

- `custid` — anonymized customer id
- `message` — endurance, strength, water, team, backcountry, racquet, or control
- `buyer` — yes/no: clicked the email and purchased within two days
- `total_os` — total order size in euros, conditional on buying
- `age`, `female`, `income`, `education`, `children` — demographics
- `freq_endurance` … `freq_racquet` — purchases per department in the last year
- `training` — 1 = train (420,000 rows), 0 = test (180,000 rows)

With the parquet file in place, the notebook runs top to bottom and reproduces
every number in the README.
