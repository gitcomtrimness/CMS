# MRF Extractor

A streaming extractor for healthcare Transparency in Coverage (TiC) Machine-Readable Files. Pulls a payer's in-network rates file from a URL, decompresses on the fly, filters to a behavioral-health code whitelist, and emits normalized JSONL rows.

The whole point: a single payer MRF can be 30+ GB decompressed. This tool never materializes the full file. It streams from the source, decompresses on the fly, parses incrementally, matches against the code whitelist, and discards everything else. Peak memory stays in the hundreds of MB regardless of input size.

This is the Phase 1 prototype validating the architectural thesis for a Massachusetts behavioral-health reimbursement-benchmarking platform. Later phases move this output into S3 + Parquet with a DuckDB query layer on top.

## Phase 1 results

Validated against three payers (UHC, Cigna, BCBS-MA) with zero code changes between them.

| Payer | File | Compressed | Rows emitted | Runtime |
|---|---|---|---|---|
| UHC IND-00 (national) | `IND-00_N4_in-network-rates.json.gz` | 2.3 GB | 228,934 | ~15 min |
| Cigna Denver Connect | `denver-co-connect-network_in-network-rates.json.gz` | 47.5 MB | 11,841 | 15 sec |
| BCBS-MA HMO Blue Fully-Insured | `HMO-Blue-Fully-Insured_in-network-rates.json.gz` | 107.3 MB | 12,237 | 12 sec |
| BCBS-MA PAR Providers | `PAR-Providers_in-network-rates.json.gz` | 104.0 MB | 11,213 | 13 sec |

Cross-payer 90837 (60-min psychotherapy) benchmark, produced directly from the data:

```
90837 (60-min psychotherapy), professional

                          UHC IND-00 (national)     Cigna Denver Connect     BCBS-MA HMO Blue
  count                              13,366                       673                   799
  median                            $142.81                   $180.65               $188.19
  p10–p90                       $106 – $247               $139 – $248          ~$108 – ~$252
```

That cross-payer cross-market distribution is the product proposition in miniature: a real reimbursement benchmark for a real code in a real market, produced from federal public data, on a laptop.

## Massachusetts behavioral health rate sheet (Phase 1 capstone)

BCBS-MA HMO Blue rates for the Mass SUD / behavioral health code stack the platform targets:

```
                                  n       p25       median    p75
DETOX
  H0010 sub-acute residential     3     $220.00   $505.00   $515.00
  H0011 acute residential         3     $258.58   $525.00   $579.45
  H0014 ambulatory                2     $305.00   $345.00   $385.00

RESIDENTIAL
  H0017 residential w/ R&B        4     $675.00   $701.00   $931.00
  H0018 short-term residential   10     $267.80   $472.50   $655.00

PARTIAL HOSPITALIZATION / IOP
  H0035 mental health PHP        23     $255.00   $450.00   $535.50
  S0201 PHP (alt)                14     $459.00   $476.79   $503.93
  H0015 SUD IOP                  32     $150.00   $256.25   $330.00

PROFESSIONAL THERAPY (selection)
  90837 60-min psychotherapy    799     $108.05   $188.19   $251.67
  90834 45-min psychotherapy    738     $ 90.76   $141.18   $191.01
  90853 group psychotherapy     357     $ 31.69   $ 41.88   $ 53.58
  99214 E&M established       1,015     $ 53.35   $129.17   $177.22
```

Key structural finding: facility-side SUD codes (H0010, H0011, H0017, H0018) have very small sample sizes in every BCBS-MA commercial product file (HMO Blue, PAR Providers). This is the **behavioral-health carve-out** showing up in the data — most Mass SUD residential and detox contracts are held by carve-out managers (MBHP for Medicaid; Carelon, Optum, or Magellan for commercial), not BCBS-MA directly. Comprehensive Mass behavioral-health rate intelligence requires ingesting those carve-out networks' MRFs separately, not just the insurer files.

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
python find_url.py <toc_file>.json <partial-filename-match>

# 4. Extract (caffeinate prevents macOS from sleeping mid-stream):
caffeinate -i python extractor.py \
  "$(python find_url.py <toc_file>.json <match>)" \
  output.jsonl 2>&1 | tee run.log

# 5. Analyze:
python analyze_bcbsma.py output.jsonl
```

The URL passed to `extractor.py` must be the direct location of a `.json.gz` in-network rates file (not a TOC).

## Project files

- **`extractor.py`** — the streaming extractor.
- **`codes.py`** — Massachusetts behavioral-health code whitelists (CPT/HCPCS and revenue codes) organized by clinical category, plus the `is_target_code` matcher and `category_for` helper.
- **`toc_explorer.py`** — reads a downloaded TOC, lists in-network files with sizes. Falls back to unverified HTTPS on cert errors.
- **`find_url.py`** — finds a specific file's URL inside a TOC by case-insensitive substring match.
- **`peek.py`** — diagnostic; prints code types and top codes inside an MRF.
- **`diag.py`** — diagnostic; confirms gzip-over-HTTP streaming works against a given host.
- **`analyze.py`**, **`analyze2.py`**, **`analyze_cigna.py`**, **`analyze_bcbsma.py`** — statistics scripts over the extracted JSONL outputs.

## Output format

One JSON object per line:

```json
{
  "billing_code": "H0011",
  "billing_code_type": "HCPCS",
  "billing_code_type_version": "2026",
  "billing_code_modifier": [],
  "name": "Alcohol/drug acute detoxification",
  "description": "Alcohol and/or drug services; acute detoxification (residential addiction program inpatient)",
  "negotiation_arrangement": "ffs",
  "negotiated_rate": 525.00,
  "negotiated_type": "fee schedule",
  "billing_class": "institutional",
  "setting": "inpatient",
  "service_codes": ["21"],
  "expiration_date": "9999-12-31",
  "additional_information": null,
  "provider_reference_ids": [4218]
}
```

Fields map directly to the CMS v2.0 in-network-rates schema: <https://github.com/CMSgov/price-transparency-guide/tree/master/schemas/in-network-rates>.

## Code whitelist

Phase 1's whitelist is organized by clinical service category (see `codes.py`):

- **Detox / withdrawal management** (HCPCS): H0010, H0011, H0012, H0013, H0014
- **Residential treatment** (HCPCS): H0017, H0018, H0019, H2036
- **Partial hospitalization** (HCPCS): S0201, H0035
- **Intensive outpatient** (HCPCS): H0015
- **Professional therapy and E&M** (CPT): 90791, 90792, 90832, 90834, 90837, 90846, 90847, 90853, 99202–99205, 99211–99215
- **Revenue codes (UB-04 facility billing)**: 124, 134 (psych inpatient); 911, 912, 913 (BH treatment / PHP); 905, 906 (IOP)

The matcher in `codes.py` respects `billing_code_type` so revenue code "913" never accidentally matches CPT code "913" or vice versa.

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

UHC uses `negotiated_type: "negotiated"` almost exclusively. Cigna and BCBS-MA use `negotiated_type: "fee schedule"` almost exclusively. Both represent contracted per-service rates. Filtering to only one and assuming the other is missing data is a real and common error.

### Behavioral health carve-outs hide rates from insurer MRFs

For Massachusetts (and likely many other states), the major commercial insurers don't directly contract most SUD residential, detox, and PHP/IOP services — they delegate that to carve-out managers (MBHP / Carelon / Optum / Magellan). The insurer's own MRF will show very small sample sizes for these codes (n=2–10 typically), which can be misleading. Complete behavioral health rate intelligence requires identifying and ingesting the carve-out managers' MRFs separately.

### `CSTM-00` and `CSTM-ALL` wildcards

A row with `billing_code: "CSTM-00"` means "this rate applies to all codes under this `billing_code_type`." `CSTM-ALL` extends across all code types. This extractor currently skips them and tracks them in `wildcard_codes_seen`. Cigna uses them; UHC and BCBS-MA don't. A future version should expand them against the whitelist.

### Out-of-spec `.zip` files

Cigna's TOC includes some `.zip` files where the spec requires `.json.gz`. This extractor doesn't handle `.zip`. Skip or special-case.

### Cross-payer file references

A payer's TOC sometimes points at files hosted by other payers (e.g., Cigna's TOC references MVP Health Care files hosted on Kyruus). One TOC's `in_network_files[]` is not necessarily one payer's data.

### Many CDNs, signed URLs, and cert chain issues

Each payer uses different hosting. UHC: Azure Blob (`mrfstore.uhc.com`). Cigna: CloudFront (`d25kgz5rikkq4n.cloudfront.net`) plus partner hosts. BCBS-MA: a host whose TLS chain Python's strict validation rejects but browsers and curl accept. Production must handle arbitrary hosts and gracefully fall back when individual certs misbehave. The Phase 1 extractor includes a diagnostic-quality `verify=False` fallback for this; Phase 2 needs a proper cert configuration.

### Place-of-service matters

The same provider can be contracted at different rates for the same code based on `service_codes` (POS codes). A rate row's primary key includes `(code, modifier, provider_reference, billing_class, setting, service_code)` — not just `(code, provider_reference)`.

### Placeholder rates exist

Some payers publish near-zero rates (e.g., $0.13) as placeholders for codes a provider technically can bill but isn't actually paid for. For clean rate distributions, filter `negotiated_rate < $20`.

### `provider_reference_ids` are unresolved

The extractor captures integer IDs that point into the file's `provider_references` array but doesn't resolve them to NPIs. Provider identity resolution (NPI / TIN / parent organization linking, and entity graphs across facility / clinician / parent corporation) is the Phase 3 deliverable. Without it, the platform can answer "what does this network pay for this code at the market level" but not "what does this specific facility get paid."

### BCBS-MA publishes two corporate entities separately

Massachusetts law requires HMOs to be incorporated as separate legal entities. BCBS-MA accordingly publishes two TOCs: one under "Blue Cross and Blue Shield of Massachusetts HMO Blue, Inc." and one under "Blue Cross and Blue Shield of Massachusetts, Inc." For our purposes, the HMO Blue TOC also references the parent entity's named files (HMO Blue Fully-Insured, Blue Care Elect, PAR Providers, etc.). For complete coverage of BCBS-MA's commercial book, both corporate-entity TOCs should be checked.

## File size expectations

Bimodal. Small regional HMO / Connect / Insurer-named networks: 1–200 MB compressed. Large national PPO networks: 1–2 GB compressed. Large single-employer files (BCBS-MA self-insured): can exceed 40 GB compressed each. The streaming approach handles all of them on a laptop.

## Known limitations

- No retry-with-resume on dropped connections during long streams. A 14 GB file that fails 80% in starts over.
- No parallelism. One file at a time.
- `toc_explorer.py` runs HEAD requests serially — slow on large TOCs.
- Signed URLs expire and cannot be regenerated; the TOC must be re-fetched to get fresh ones.
- Only handles `.json.gz` (not `.zip`).
- No handling of `CSTM-00` / `CSTM-ALL` wildcard codes.
- Cert handling falls back to unverified HTTPS on validation errors. Acceptable for Phase 1 prototyping; not for production.
- `provider_reference_ids` are captured but not resolved to NPIs.

All of these are deliberate Phase 1 deferrals.

## References

- [CMS Transparency in Coverage technical guide](https://github.com/CMSgov/price-transparency-guide) — the canonical schema specification.
- [CMS schema validator](https://github.com/CMSgov/price-transparency-guide-validator) — useful for verifying that a given file conforms to the spec.

## Next phases

- **Phase 2:** S3 + Parquet output, DuckDB query layer, basic API. Properly configured cert handling. Parameterized whitelists.
- **Phase 3:** Provider identity resolution (NPI / TIN / parent-org linking, NPPES integration, entity graphs).
- **Phase 4:** Precomputed benchmark tables, percentile/percentile-rank API.
- **Phase 5:** Frontend, auth, subscriptions.
- **Phase 6:** AI insight layer (Claude API over query results).

## License

TBD.
