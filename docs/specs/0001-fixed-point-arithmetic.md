# 0001 — Fixed-Point Arithmetic

| | |
|---|---|
| Status | Accepted |
| Crate | `hypersentinel-core` (`src/fixed.rs`) |
| Related | ADR-001 (to be written) |

## Summary

All prices, sizes, positions, and monetary values are represented as scaled 64-bit integers with a
single global scale of 10⁸. Floating-point arithmetic is forbidden in the domain crates.

## Motivation

- `f64` cannot represent most decimal fractions exactly (`0.1 + 0.2 != 0.3`). Errors accumulate across
  fills, PnL, and margin computations and make results non-reproducible between live and replay runs.
- Hyperliquid transmits all numeric values as decimal strings (`"92458.3"`, `"0.00125"`), which makes
  exact parsing into integers possible.
- Integer arithmetic is deterministic across platforms, which the replay harness requires.

## Design

### Scale

`SCALE = 100_000_000` (8 decimal places). This covers Hyperliquid's maximum precision: spot sizes have up
to 8 decimals, and perp prices have at most `6 − szDecimals` decimals.

A fixed decimal scale is used rather than per-asset tick units because Hyperliquid's effective tick size
varies with price magnitude (5-significant-figure rule).

### Types

| Type | Meaning | Invariant |
|---|---|---|
| `Price` | Price of one unit of the base asset, in USD | `> 0` |
| `Size` | Unsigned quantity of the base asset | `>= 0` |
| `Position` | Signed quantity (long > 0, short < 0) | any |
| `Usd` | Signed USD amount (notional, PnL, equity) | any |

All four wrap an `i64` holding `value × 10⁸`. The inner field is private. Values are constructed only
through validating constructors, so the invariants hold everywhere a value exists.

A single signed representation is used for every type so that mixed arithmetic never needs `as` casts
between `u64` and `i64`, which silently wrap.

Range: `i64::MAX / 10⁸ ≈ 92,233,720,368`, i.e. about 92 billion units. That is sufficient for any single
price, size, or account value.

### Parsing

`Price`, `Size`, and `Usd` implement `std::str::FromStr`. Grammar (ASCII only):

```
value    := ["-"] digits ["." digits]      ; "-" accepted only by signed types
digits   := [0-9]+
```

| Input | Result |
|---|---|
| `"92458.3"` | `Price(9_245_830_000_000)` |
| `"0.00125"` | `Size(125_000)` |
| `"100"` | `100 × 10⁸` |
| `""`, `"."`, `".5"`, `"5."`, `"1.2.3"`, `" 1"`, `"1e5"`, `"+1"` | `Err` |
| more than 8 fractional digits | `Err` (rejected, never rounded) |
| integer part exceeding range | `Err` (overflow, never wraps) |
| `"0"` for `Price` | `Err` (violates invariant) |
| `"-1"` for `Price` or `Size` | `Err` |

Requirements:
- **No heap allocation** during parsing.
- Errors are a `thiserror` enum that distinguishes at least: empty input, invalid character (with byte
  position), too many fractional digits, overflow, and invariant violation.

### Formatting

`Display` produces the shortest exact decimal form: no trailing fractional zeros, and no `.` when the value
is an integer (`9_245_830_000_000` → `"92458.3"`).

**Round-trip property:** `x.to_string().parse() == Ok(x)` for every valid `x`.

### Arithmetic

- Same-type addition and subtraction are **checked** and return `Option` or `Result`. There is no
  implicit overflow and no `std::ops::Add` impl that could panic or wrap.
- `notional(price: Price, size: Size) -> Result<Usd, ArithmeticError>`:
  - The product of two scaled values is scaled by 10¹⁶. The multiplication is performed in **`i128`**,
    then divided by `SCALE` and narrowed back to `i64` with a checked conversion.
  - Rounding mode: **round half to even** (banker's rounding), so rounding introduces no systematic bias
    across many fills.
  - Narrowing overflow returns `Err`. It never truncates.

Signed `Position` × `Price` notional is out of scope for this spec. It is introduced with the risk
evaluator.

## Testing

| Kind | Coverage |
|---|---|
| Unit (table-driven) | Every row of the parsing table above; `Display` examples; `notional` exact cases, including a tie that rounds to even |
| Property (`proptest`) | Display/parse round-trip for all valid values; `notional` result equals an independent `i128` reference computation; `checked_add` agrees with `i128` addition whenever the result is in range |
| Boundary | `i64::MAX`-scale inputs; the largest valid `notional`; the smallest overflowing `notional` |

## Out of scope

- Hyperliquid order precision validation (`szDecimals`, 5 significant figures). See spec 0002.
- Performance benchmarks (introduced with the benchmark suite).
