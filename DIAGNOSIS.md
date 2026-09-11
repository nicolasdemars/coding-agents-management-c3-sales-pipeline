# Diagnosis: S-014 August revenue discrepancy

## Red command

From the repository root, with the package installed or `PYTHONPATH=src`:

```powershell
pytest -q tests/test_pipeline.py::test_partner_store_monthly_total_matches_finance
```

Before the fix, the regression test failed with:

```text
E       AssertionError: assert Decimal('50043.47') == Decimal('56232.09')
1 failed
```

After the fix, the same command reports:

```text
1 passed
```

## Cause

Five amounts in the partner export use a non-breaking space as a thousands separator, for example `1 321,49`. `parse_amount` searched the raw text with a pattern that did not accept this separator, so it silently parsed that value as `1` instead of `1321.49`; the five resulting losses total exactly `6,188.62 EUR`, which is the complete difference reported by Finance.

The clue was that S-014 is the only store using the regional partner export. Inspecting values that contained characters outside the ordinary digit/comma format exposed the five affected rows.

## Evidence and eliminated causes

| Suspect checked | Observed value before the fix | Conclusion |
|---|---:|---|
| Missing or duplicate partner transactions | 420 rows and 420 unique transaction IDs | Row loss or de-duplication did not explain the revenue gap. |
| Multiplying every current net amount by `quantity` | `125,415.28 EUR` | An omitted global quantity multiplication did not produce Finance's total. |
| Aggregate rounding instead of row-level rounding | `0.12 EUR` difference | Rounding was far too small to explain `6,188.62 EUR`. |
| Summing the un-discounted parsed amounts | `52,448.67 EUR` | Discount application alone did not explain the expected total. |

The five malformed parses were:

| Transaction | Raw amount | Parsed before | Correct net after discount | Revenue loss |
|---|---:|---:|---:|---:|
| P-00219 | `1 321,49` | 1.00 | 1,321.49 | 1,320.49 |
| P-00272 | `1 613,17` | 1.00 | 1,613.17 | 1,612.17 |
| P-00275 | `1 197,00` | 0.85 | 1,017.45 | 1,016.60 |
| P-00298 | `1 070,51` | 0.90 | 963.46 | 962.56 |
| P-00314 | `1 503,12` | 0.85 | 1,277.65 | 1,276.80 |

Together, these losses are `6,188.62 EUR`. Correcting them changes the S-014 total from `50,043.47 EUR` to `56,232.09 EUR`.

## Fix and regression coverage

`parse_amount` now removes non-breaking (`U+00A0`) and narrow non-breaking (`U+202F`) thousands separators before applying the existing decimal parser. The field-level test covers the exact regional number shape, and the pipeline-level regression test checks Finance's complete S-014 total using the real CSV fixtures.

The targeted regression test passes after the fix, and the complete suite reports `19 passed`.

Finding location: the structure of the code — regional field normalization belongs in `pipeline.parse`, the boundary already responsible for reconciling exporter formats.

More than 1h30: No
