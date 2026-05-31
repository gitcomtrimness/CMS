# MRF Extractor

A streaming extractor for healthcare Transparency in Coverage (TiC) Machine-Readable Files. Pulls a payer's in-network rates file from a URL, decompresses on the fly, filters to a behavioral-health code whitelist, and emits normalized JSONL rows.

The whole point: a single payer MRF can be 30+ GB decompressed. This tool never materializes the full file. It streams from the source, decompresses on the fly, parses incrementally, matches against the code whitelist, and discards everything else. Peak memory stays in the hundreds of MB regardless of input size.

This is a Phase 1 prototype validating the architectural thesis for a larger behavioral-health reimbursement-benchmarking platform. Later phases move this output into S3 + Parquet with a DuckDB query layer on top.

## Phase 1 results

Validated against two payers with zero code changes between them.

| Payer | File | Compressed | Rows emitted | Runtime |
|---|---|---|---|---|
| UHC IND-00 (national) | `IND-00_N4_in-network-rates.json.gz` | 2.3 GB | 228,934 | ~15 min |
| Cigna Denver Connect | `denver-co-connect-network_in-network-rates.json.gz` | 47.5 MB | 11,841 | 15 sec |

Cross-payer 90837 (60-min psychotherapy) benchmark, produced directly from the data above:

```
90837 (60-min psychotherapy), professional, in-network

                          UHC IND-00 (national)     Cigna Denver Connect
  count                              13,366                       673
  median                            $142.81                   $180.65
  p10 - p90                    $106 - $247               $139 - $248
```

That cross-payer cross-market distribution is the product proposition in miniature: a real reimbursement benchmark for a real code in a real market, produced from federal public data, on a laptop.

## Setup

Requires Python 3.10+.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Typical workflow

```bash
# 1. Download a payer's TOC manually from their TiC portal (browser).

# 2. See what in-network files it lists, with sizes:
python toc_explorer.py <toc_file>.json

# 3. Pick a target file by name. Get its full URL:
python find_url.py <partial-filename-match>

# 4. Extract (caffeinate prevents macOS from sleeping mid-stream):
caffeinate -i python extractor.py "$(python find_url.py <match>)" output.jsonl
```

The URL passed to `extractor.py` must be the direct location of a `.json.gz` in-network rates file (not a TOC).

## Project files

- **`extractor.py`** — the streaming extractor.
- **`codes.py`** — behavioral-health code whitelists (CPT/HCPCS and revenue codes), plus the `is_target_code` matcher.
- **`toc_explorer.py`** — reads a downloaded TOC, lists in-network files with sizes.
- **`find_url.py`** — finds a specific file's URL inside a TOC by filename match.
- **`peek.py`** — diagnostic; prints code types and top codes inside an MRF.
- **`diag.py`** — diagnostic; confirms gzip-over-HTTP streaming works against a given host.
- **`analyze.py`**, **`analyze2.py`**, **`analyze_cigna.py`** — quick statistics scripts over the extracted JSONL.

## Output format

One JSON object per line:

```json
{
  "billing_code": "90837",
  "billing_code_type": "CPT",
  "billing_code_type_version": "2026",
  "billing_code_modifier": [],
  "name": "Psychotherapy, 60 minutes",
  "description": "Psychotherapy, 60 minutes with patient",
  "negotiation_arrangement": "ffs",
  "negotiated_rate": 142.18,
  "negotiated_type": "negotiated",
  "billing_class": "professional",
  "setting": "outpatient",
  "service_codes": ["11"],
  "expiration_date": "9999-12-31",
  "additional_information": null,
  "provider_reference_ids": [16316]
}
```

Fields map directly to the CMS v2.0 in-network-rates schema: <https://github.com/CMSgov/price-transparency-guide/tree/master/schemas/in-network-rates>.

## Things to know about the data

Real findings from running this against production payer files.

### The `negotiated_type` trap

`negotiated_rate` is **not** always a dollar amount. Per the spec, `negotiated_type` can be:

- `negotiated` — dollar amount.
- `fee schedule` — also a dollar amount, semantically distinct (used for cost-sharing).
- `percentage` — percent of billed charges. Value `65` means 65%, not $65.
- `per diem` — dollars per day, not per service.
- `derived` — internal accounting figure.

Mixing these in one statistic produces nonsense. Any query that aggregates `negotiated_rate` must filter or stratify by `negotiated_type`. For dollar-rate benchmarks, filter `negotiated_type IN ('negotiated', 'fee schedule')`.

### Payers tag the same data differently

UHC uses `negotiated_type: "negotiated"` almost exclusively. Cigna uses `negotiated_type: "fee schedule"` almost exclusively. Both represent contracted per-service rates. Filtering to only one and assuming the other is missing data is a real and common error.

### `CSTM-00` and `CSTM-ALL` wildcards

A row with `billing_code: "CSTM-00"` means "this rate applies to all codes under this `billing_code_type`." `CSTM-ALL` extends across all code types. This extractor currently skips them and tracks them in `wildcard_codes_seen`. UHC doesn't appear to use them; Cigna does. A future version should expand them against the whitelist.

### Out-of-spec `.zip` files

Cigna's TOC includes some `.zip` files where the spec requires `.json.gz`. This extractor doesn't handle `.zip`. Skip or special-case.

### Cross-payer file references

A payer's TOC sometimes points at files hosted by other payers (e.g., Cigna's TOC references MVP Health Care files hosted on Kyruus). One TOC's `in_network_files[]` is not necessarily one payer's data.

### Many CDNs and signed URLs

UHC hosts on Azure Blob (`mrfstore.uhc.com`). Cigna hosts on CloudFront (`d25kgz5rikkq4n.cloudfront.net`) and via partners like `mrf.healthsparq.com`. URLs are typically signed and time-limited; they must be re-fetched from the TOC monthly.

### Place-of-service matters

The same provider can be contracted at different rates for the same code based on `service_codes` (POS codes). A rate row's primary key includes `(code, modifier, provider_reference, billing_class, setting, service_code)` — not just `(code, provider_reference)`.

### Placeholder rates exist

Some payers publish near-zero rates (e.g., $0.13) as placeholders for codes a provider technically can bill but isn't actually paid for. For clean rate distributions, filter `negotiated_rate < $20`.

### `provider_reference_ids` are unresolved

The extractor captures integer IDs that point into the file's `provider_references` array but doesn't resolve them to NPIs. Provider identity resolution (NPI / TIN / parent organization linking) is its own significant chunk of work.

## File size expectations

Bimodal. Small regional HMO/Connect networks: 1–100 MB compressed. Large national PPO networks: 1–2 GB compressed. A few extreme cases (some UHC PPO size-band files) push 14+ GB. The streaming approach handles all of them on a laptop.

## Known limitations

- No retry-with-resume on dropped connections during long streams. A 14 GB file that fails 80% in starts over.
- No parallelism. One file at a time.
- `toc_explorer.py` runs HEAD requests serially — slow on large TOCs.
- Signed URLs expire and cannot be regenerated; the TOC must be re-fetched to get fresh ones.
- Only handles `.json.gz` (not `.zip`).
- No handling of `CSTM-00` / `CSTM-ALL` wildcard codes.

All of these are deliberate Phase 1 deferrals.

## References

- [CMS Transparency in Coverage technical guide](https://github.com/CMSgov/price-transparency-guide) — the canonical schema specification.
- [CMS schema validator](https://github.com/CMSgov/price-transparency-guide-validator) — useful for verifying that a given file conforms to the spec.

## License

TBD.
