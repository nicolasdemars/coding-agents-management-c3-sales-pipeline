# Diagnosis: S-014 August revenue discrepancy

## Red reproduction

The original application was exercised without creating a repository database:

```powershell
python -c "from pathlib import Path; from decimal import Decimal; from pipeline import ingest, load, report, transform; rows=ingest.read_all(Path('data/raw')); conn=load.connect(':memory:'); load.load(transform.transform_all(rows), conn); actual=dict(report.all_store_totals(conn))['S-014']; print(f'S-014 August total: {actual:.2f} EUR; expected: 56232.09 EUR'); assert actual == Decimal('56232.09'), f'finance mismatch: got {actual:.2f}, expected 56232.09'"
```

Observed result:

```text
S-014 August total: 50043.47 EUR; expected: 56232.09 EUR
AssertionError: finance mismatch: got 50043.47, expected 56232.09
```

The application reports **50,043.47 EUR**; Finance expects **56,232.09 EUR**. The gap is **6,188.62 EUR**.

## Current documented behavior

The README says that transformation “applies the discount and computes the net amount.” The implemented formula is:

```text
gross_amount = parse_amount(amount)
discount_pct = parse_amount(discount_pct), or 0 when blank
net_amount = round_to_cent(gross_amount × (100 − discount_pct) / 100)
```

`quantity` is parsed and stored, but is not used in this calculation. Reporting sums the stored `net_amount` values for the store.

For S-014, the 420 partner rows produce:

```text
gross total: 52,448.67 EUR
discounted net total: 50,043.47 EUR
```

## Evidence against other explanations

- **Quantity:** multiplying every stored net amount by quantity produces **125,415.28 EUR**, not 56,232.09 EUR. The repository does not document whether `amount` is a unit price or a line total.
- **Rounding:** row-level cent rounding versus aggregate rounding changes the result by only a few cents, not 6,188.62 EUR.
- **Duplicates:** all 2,080 transaction IDs across the three CSV files are unique. The partner file has 420 unique rows.
- **Missing rows:** the partner export contains 420 rows for S-014, matching the transaction count stated in the bug report.
- **Tax, fees, commission, VAT, and markup:** no such fields, rules, comments, tests, or documentation exist in the repository. No calculation can be justified from them.
- **Gross-versus-net selection:** the S-014 gross total is 52,448.67 EUR, which also does not equal Finance’s 56,232.09 EUR.

## Concrete row trace

The first partner rows trace as follows:

| Transaction | Raw amount | Quantity | Discount | Current net |
|---|---:|---:|---:|---:|
| P-00000 | 91,83 | 4 | 10% | 82.65 |
| P-00001 | 32,02 | 2 | 15% | 27.22 |
| P-00002 | 153,13 | 3 | 10% | 137.82 |
| P-00003 | 109,46 | 2 | blank | 109.46 |
| P-00004 | 169,70 | 1 | blank | 169.70 |

For example, P-00000 is currently calculated as:

```text
91.83 × (100 − 10) / 100 = 82.647 → 82.65 EUR
```

The report does not say whether that row should instead be treated as a unit price, a line total, a net amount, or something else. Therefore the row trace demonstrates the current behavior but does not establish a different correct value.

## Cause and conclusion

`BUG-REPORT.md` supplies the two aggregate values and confirms that the transaction count is 420, but it does not define the partner field contract for `amount`, `quantity`, or `discount_pct`. It also does not define any additional revenue component.

The cause is therefore an under-specified business/data contract, not a demonstrable reporting formula defect. The finding belongs in the specification/data contract.

No honest production fix can be selected without clarification from Finance or the partner-export owner. In particular, no formula should be invented from the target total alone.

No application code was modified while producing this diagnosis.

More than 1h30: No
