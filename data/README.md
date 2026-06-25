# data/README.md

# Data documentation

## Source

The original live dataset is the English volunteer CSV from the public Ukrainian air-raid sirens dataset repository:

```text
https://raw.githubusercontent.com/Vadimkin/ukrainian-air-raid-sirens-dataset/main/datasets/volunteer_data_en.csv
```

The live source is updated over time. Results based directly on that URL may therefore change between notebook runs.

## Frozen snapshot

This repository uses a fixed snapshot:

```text
volunteer_data_en_snapshot_2026-06-25.csv
```

Snapshot properties:

* extraction date: June 25, 2026;
* rows: 101,969;
* columns: 4;
* columns: `region`, `started_at`, `finished_at`, `naive`;
* timestamp timezone in the source: UTC.

SHA-256:

```text
0354ac29bf8086fa8ca0e51c5bd63041bdb4d8671da93dc189a40cf684f62a38
```

The snapshot is used instead of the live URL for all reported analysis, validation and test results.

## Verification

The checksum can be verified in Python:

```python
import hashlib

filename = "volunteer_data_en_snapshot_2026-06-25.csv"

with open(filename, "rb") as file:
    checksum = hashlib.sha256(file.read()).hexdigest()

print(checksum)
```

Expected output:

```text
0354ac29bf8086fa8ca0e51c5bd63041bdb4d8671da93dc189a40cf684f62a38
```

On macOS or Linux, it may also be checked with:

```bash
shasum -a 256 volunteer_data_en_snapshot_2026-06-25.csv
```

## Relevant fields

* `region`: name of the Ukrainian region;
* `started_at`: alert start timestamp;
* `finished_at`: alert finish timestamp;
* `naive`: indicates that the ending was estimated.

## Data-quality findings

The verified snapshot contained:

* no failed timestamp conversions;
* no exact duplicate rows;
* no negative durations;
* five zero-duration records;
* 5,023 `naive=True` records;
* approximately 4.93% estimated endings.

All `naive=True` records had an exact duration of 30 minutes.

For daily occurrence modelling, alert starts were used regardless of whether the ending was estimated.

For duration analysis, estimated endings and non-positive durations were excluded.

## Project subset

The final modelling task uses:

* region: Khmelnytska oblast;
* local timezone: `Europe/Kyiv`;
* daily window: February 26, 2022 through June 24, 2026;
* target: whether at least one alert starts on the following local calendar day.

## Limitations

The dataset is volunteer-collected rather than a guaranteed complete operational record.

A missing alert record may mean either:

* no alert occurred;
* an alert was not captured by the source.

Alert records should not be interpreted as confirmed attacks, damage or casualties.

The data must not be used as a substitute for official Ukrainian warning systems.


# They are intentionally included in this repository.
