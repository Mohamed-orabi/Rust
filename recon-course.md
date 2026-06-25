# Building a Payments Reconciliation Engine in Rust

*A step-by-step course — from a `Currency` newtype to a reconciliation engine running end to end in the browser.*

This course builds a payments settlement reconciliation engine bottom-up, in dependency order, because that's also conceptual order: each layer only depends on the ones below it, so you never have to hold unfinished things in your head.

**How to use this:** type each piece into your own editor and run it — typing (not pasting) is the reps. At the end of each module you'll have something that compiles and tests green before moving on.

## Syllabus

1. **The money core** — `Money` and `Currency`. Making illegal states unrepresentable.
2. **The canonical model** — `CanonicalTxn`, and the audit-trail principle.
3. **The matching engine** — the real IP: exact/tolerant matching, exhaustive exception handling.
4. **Ingestion** — parsing messy CSV behind a trait; the project grows into a workspace.
5. **Orchestration + synthetic data** — the engine crate.
6. **The HTTP API** — Axum, async, the request lifecycle.
7. **The web UI** — a single-file dashboard.

The seven modules get you to a working MVP. Beyond them sits the roadmap: real Visa/Mastercard/Meeza parsers, many-to-one matching, the settlement/payout layer, and persistence.

---

# Module 1 — The money core

We start at the very bottom: representing money. Everything else holds amounts, and this teaches the single most important habit in Rust domain modeling — *encoding rules in types so the compiler enforces them.*

## Step 1: create the crate

```powershell
cargo new --lib recon-core
cd recon-core
```

We start as one plain library crate — not the four-crate workspace yet. A workspace is a solution to a problem we don't have yet. When we hit the moment that *demands* a second crate (Module 4), we'll split, and the reason will be obvious because you'll have lived without it.

C# analogy: `cargo new --lib` is `dotnet new classlib`. `Cargo.toml` is your `.csproj`. The crate is the assembly.

In `Cargo.toml`:

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "1"
```

`serde` is serialization (think `System.Text.Json`, but compile-time via derive macros). `chrono` is dates (`NodaTime`, roughly). `thiserror` is a helper for defining error types.

## Step 2: the principle, before any code

**Never represent money as a floating-point number.** Ever.

`f64` can't exactly store most decimal fractions. In C# you've felt this:

```
0.1 + 0.2 == 0.30000000000000004   // not 0.3
```

For a UI that's a rounding annoyance. For a reconciliation engine whose entire job is deciding whether two amounts are *equal*, it's fatal — you'd flag real matches as mismatches because of invisible binary dust. So we store money as an **integer count of the smallest unit** — piastres, cents, fils. 150.00 EGP becomes `15000`. Addition and equality become exact integer operations. This is the same choice Stripe and every serious payments system makes.

## Step 3: the `Currency` type — our first newtype

Create `src/money.rs`:

```rust
use serde::{Deserialize, Serialize};
use std::fmt;

/// A currency code (e.g. "EGP", "USD"), stored as 3 fixed bytes rather than a
/// String: it's Copy, needs no heap allocation, and compares cheaply. We
/// validate the code once, at construction.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct Currency([u8; 3]);
```

Three decisions worth pausing on:

**The newtype pattern**: wrapping `[u8; 3]` in a struct named `Currency` instead of passing raw bytes or a string around. In C# you'd reach for a `readonly struct Currency` for the same reason — a distinct type the compiler can check, so you can never accidentally pass a merchant ID where a currency belongs.

**`[u8; 3]` instead of `String`**. A `String` is a heap allocation and a pointer; comparing two means chasing pointers. A fixed three-byte array lives inline, is `Copy` (cheap to pass by value, like a C# `struct`), and compares as fast as an integer. Currencies are always three letters, so we exploit that.

**The derives**. `#[derive(...)]` auto-generates trait implementations — roughly the compiler writing your `Equals`, `GetHashCode`, and `ToString`. `PartialEq, Eq, Hash` let us use `Currency` as a dictionary key later; `Copy` makes it pass-by-value; `Serialize, Deserialize` give JSON.

## Step 4: validation at construction — "parse, don't validate"

We never want a `Currency` holding garbage like `"x1!"`. So we don't expose the inner bytes — the only way to make one is through a constructor that *checks* and returns a `Result`:

```rust
#[derive(Debug, thiserror::Error)]
pub enum CurrencyError {
    #[error("currency code must be exactly 3 ASCII uppercase letters, got {0:?}")]
    Invalid(String),
}

impl Currency {
    pub fn new(code: &str) -> Result<Self, CurrencyError> {
        let bytes = code.as_bytes();
        if bytes.len() == 3 && bytes.iter().all(|b| b.is_ascii_uppercase()) {
            Ok(Currency([bytes[0], bytes[1], bytes[2]]))
        } else {
            Err(CurrencyError::Invalid(code.to_string()))
        }
    }

    pub fn code(&self) -> &str {
        // Safe: we only ever store 3 ASCII-uppercase bytes.
        std::str::from_utf8(&self.0).unwrap_or("???")
    }
}

impl fmt::Display for Currency {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(self.code())
    }
}
```

"Parse, don't validate": instead of scattering `if (IsValidCurrency(x))` everywhere, you validate **once**, at the boundary where the value is born, and from then on the *type itself* is the proof it's valid. A `Currency` value cannot exist in an invalid state. In C# terms it's a guard clause in a factory method — except the type system remembers the guard ran, so nobody downstream re-checks.

`thiserror` generates the boilerplate; `#[error("...")]` writes the `Display` implementation for you. `impl Display` teaches a type to print itself — like overriding `ToString()`.

## Why the field is private

The field in `struct Currency([u8; 3])` is private — only code inside the `money` module can name it. From anywhere else, `Currency::new` is the *only door in*, and that door validates. The guarantee isn't "we remembered to check everywhere," it's "there is no other way to make one." Encapsulation makes "illegal states unrepresentable" airtight rather than aspirational.

## Why `code()` returns `&str`, not `String`

`&str` is a borrowed *view* into the three bytes already living inside `self` — no allocation, no copy. Returning `String` would heap-allocate and copy on every call, for a caller who probably just wants to read. If they need ownership they can `.to_string()` themselves. You push the cost to whoever actually needs it.

## Part 2 — `Money` itself

Add to `money.rs`, below the `Currency` code:

```rust
/// A monetary amount in minor units. `minor` may be negative (refunds,
/// chargebacks). Two `Money` values are only meaningfully comparable or
/// addable when their currencies match.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub struct Money {
    pub minor: i64,
    pub currency: Currency,
}
```

`i64` holds about ±92 quadrillion minor units — far more than any real ledger needs, and it's exact. `Copy` because a `Money` is just 16 bytes of plain data.

**The interesting part: arithmetic across currencies.** The bug we design against: someone writes `egp_amount + usd_amount`. In a naive design those are both "numbers," so it compiles and silently produces nonsense. We want that mistake to be impossible to do silently.

Honest note: Rust *can* make currency mismatch a compile error using one type per currency (phantom types), but that's heavy machinery and overkill here. Instead we make it a **forced runtime decision**: arithmetic returns a `Result`, so the caller cannot get a `Money` back out without acknowledging the operation might have failed. That's the realistic, idiomatic 95% solution.

```rust
#[derive(Debug, thiserror::Error)]
pub enum MoneyError {
    #[error("currency mismatch: {0} vs {1}")]
    CurrencyMismatch(Currency, Currency),
    #[error("arithmetic overflow")]
    Overflow,
}

impl Money {
    pub fn new(minor: i64, currency: Currency) -> Self {
        Money { minor, currency }
    }

    pub fn checked_add(self, other: Money) -> Result<Money, MoneyError> {
        if self.currency != other.currency {
            return Err(MoneyError::CurrencyMismatch(self.currency, other.currency));
        }
        let minor = self
            .minor
            .checked_add(other.minor)
            .ok_or(MoneyError::Overflow)?;
        Ok(Money { minor, currency: self.currency })
    }

    /// Absolute difference in minor units, ignoring sign. Errors on currency
    /// mismatch. This is what the matching engine will lean on later.
    pub fn abs_diff_minor(self, other: Money) -> Result<i64, MoneyError> {
        if self.currency != other.currency {
            return Err(MoneyError::CurrencyMismatch(self.currency, other.currency));
        }
        Ok((self.minor - other.minor).abs())
    }
}
```

Three things to notice. We did **not** implement the `+` operator. `+` in Rust can't return a `Result` cleanly — it would force us to panic on mismatch, exactly the silent explosion we're avoiding. A named `checked_add` returning `Result` is honest that adding money can fail.

`i64::checked_add` returns `Option` — `Some(sum)` normally, `None` on overflow. The `.ok_or(...)?` converts `None` into `MoneyError::Overflow` and the `?` bails out early. That `?` is "if this failed, return the error up the call stack now."

`abs_diff_minor` is a forward investment: the matcher's whole job is "how far apart are these two amounts?", and that question should also refuse to compare across currencies.

**Teach `Money` to print itself:**

```rust
impl fmt::Display for Money {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let sign = if self.minor < 0 { "-" } else { "" };
        let abs = self.minor.unsigned_abs();
        write!(f, "{}{}.{:02} {}", sign, abs / 100, abs % 100, self.currency)
    }
}
```

Presentation only — `15000` displays as `150.00 EGP`. Storage stayed an integer; only rendering splits into major/minor with `/ 100` and `% 100`. (Real currencies have different minor-unit exponents — JPY has zero, KWD has three — handled later. For now, two decimals.)

## Checkpoint

`src/lib.rs`:

```rust
pub mod money;
```

Tests at the bottom of `money.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn egp() -> Currency { Currency::new("EGP").unwrap() }

    #[test]
    fn currency_validation() {
        assert!(Currency::new("EGP").is_ok());
        assert!(Currency::new("usd").is_err());   // lowercase rejected
        assert!(Currency::new("EG").is_err());    // too short
        assert!(Currency::new("EGPX").is_err());  // too long
    }

    #[test]
    fn add_same_currency() {
        let a = Money::new(15000, egp());
        let b = Money::new(2550, egp());
        assert_eq!(a.checked_add(b).unwrap().minor, 17550);
    }

    #[test]
    fn add_currency_mismatch_is_an_error() {
        let a = Money::new(100, egp());
        let b = Money::new(100, Currency::new("USD").unwrap());
        assert!(matches!(a.checked_add(b), Err(MoneyError::CurrencyMismatch(_, _))));
    }

    #[test]
    fn display_formats_as_decimal() {
        assert_eq!(Money::new(123456, egp()).to_string(), "1234.56 EGP");
        assert_eq!(Money::new(-50, egp()).to_string(), "-0.50 EGP");
    }
}
```

Run `cargo test`. Four green.

**Feel it:** annotate the result of `checked_add` across currencies as `: Money` and watch the compiler refuse — it hands you a `Result<Money, _>`, not a `Money`, and won't let you pretend otherwise. That refusal is the safety; the currency bug can't slip through silently.

---

# Module 2 — The canonical transaction

The most important architectural idea in the whole project lives here.

**The principle: the engine never sees a file format.** Your inputs will be wildly different shapes — a switch CSV, a Visa BASE II clearing file, a bank settlement statement, a Mastercard IPM binary. If the matching engine had to *know* each format, every new format would mean surgery on your core logic. That's how legacy systems rot.

So every input, no matter how ugly, gets translated into **one normalized shape** — the `CanonicalTxn` — *before* the engine runs. The engine only ever reasons about canonical transactions. Adding a new format later means writing one translator; the engine doesn't change at all.

C# instinct: this is the Adapter pattern, or a DTO that every importer maps *into*. The canonical model is your domain entity; file formats are external representations mapped in at the boundary.

## Step 1: the identifier newtypes

Create `src/txn.rs`:

```rust
use crate::money::Money;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use std::fmt;

/// Stable identifier for one canonical transaction within a recon run.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct TxnId(pub String);

/// Which source system a transaction came from (e.g. "switch", "bank").
/// Matching pairs records across *different* sources.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct SourceId(pub String);

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct MerchantId(pub String);

impl fmt::Display for TxnId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(&self.0)
    }
}
```

These are tuple structs — `TxnId(pub String)` wraps one value, reached via `.0`. We made the inner `String` `pub` this time (unlike `Currency`) because there's no validation rule to protect — any string is a valid ID. The newtype is purely about *type distinction*, not invariants. That's a real judgment call: **wrap-and-hide when there's a rule to enforce, wrap-and-expose when you only want type safety.** `Hash` is on all three because we'll use them as `HashMap` keys in the matcher.

## Step 2: transaction type as an enum

A transaction is a sale, refund, chargeback, or reversal. In Rust we use a real enum, and the payoff is *exhaustiveness*: later, when the matcher does `match txn.txn_type { ... }`, if you add a fifth variant and forget to handle it somewhere, **the code won't compile.**

```rust
/// Modeled as an enum so any code branching on it must handle every variant —
/// a forgotten case is a compile error, not a silent bug at runtime.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum TxnType {
    Sale,
    Refund,
    Chargeback,
    Reversal,
}

impl TxnType {
    /// Lenient parsing — real files spell things many ways.
    pub fn parse(s: &str) -> Option<Self> {
        match s.trim().to_ascii_lowercase().as_str() {
            "sale" | "purchase" | "payment" => Some(TxnType::Sale),
            "refund" | "credit" => Some(TxnType::Refund),
            "chargeback" | "cb" => Some(TxnType::Chargeback),
            "reversal" | "void" => Some(TxnType::Reversal),
            _ => None,
        }
    }
}
```

`parse` returns `Option<Self>`. We're being *lenient about input* ("purchase" and "payment" both map to `Sale`) but *strict about the internal model* (exactly four variants). Lenient at the boundary, strict in the core — the same shape as `Currency::new`.

## Step 3: the audit trail — the part that makes this sellable

A reconciliation result that says "these 4,000 transactions matched" is *worthless to a bank* if an auditor can't ask "prove it — show me exactly which line of which file each side came from." Without traceability, your output is an unverifiable assertion. With it, it's evidence.

```rust
/// A pointer back to the original source data, so every match and every
/// exception is auditable to the exact file and line it came from. Without
/// this, a reconciliation result is unverifiable — and therefore worthless to
/// an auditor.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct RawRef {
    pub file: String,
    pub line: u64,
}
```

Small struct, large consequence. This is the kind of detail that separates a toy from a product.

## Step 4: assemble the canonical transaction

```rust
/// The normalized transaction. This is the single shape the matching engine
/// operates on — no matter what file format it originally came from.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct CanonicalTxn {
    pub id: TxnId,
    pub source: SourceId,
    pub merchant_id: MerchantId,
    pub amount: Money,
    pub txn_type: TxnType,
    /// Authorization code, when present. A strong matching key.
    pub auth_code: Option<String>,
    /// Retrieval Reference Number — the most reliable cross-system key.
    pub rrn: Option<String>,
    /// Tokenized/masked PAN. NEVER store a raw card number here.
    pub pan_token: Option<String>,
    pub captured_at: DateTime<Utc>,
    pub raw_ref: RawRef,
}
```

`auth_code`, `rrn`, and `pan_token` are `Option<String>` — Rust's way of saying "this might be absent." There is no `null`; the *type* tells you it can be missing, and the compiler forces you to handle the empty case. A whole category of null-reference bugs simply can't occur. The comment on `pan_token` isn't decoration — storing a raw PAN is a PCI-DSS violation. The model encodes the compliance rule.

## Step 5: the method that feeds the matcher

RRN is the gold standard (designed to be unique across systems); auth code is the fallback.

```rust
impl CanonicalTxn {
    /// The best available natural key for matching: prefer RRN, then auth code.
    /// Returns `None` when neither exists (such records can't be matched on a
    /// strong key and will fall to exceptions).
    pub fn natural_key(&self) -> Option<&str> {
        self.rrn
            .as_deref()
            .or(self.auth_code.as_deref())
    }
}
```

Read the chain literally: "give me the RRN as a `&str`; if it's `None`, fall back to the auth code." `.as_deref()` converts `&Option<String>` into `Option<&str>` (a borrowed view — no allocation). `.or(...)` picks the first `Some`. This is the seam where Module 2 hands off to Module 3.

## Checkpoint

`src/lib.rs`:

```rust
pub mod money;
pub mod txn;
```

Tests at the bottom of `txn.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::money::{Currency, Money};

    #[test]
    fn txn_type_parsing_is_lenient() {
        assert_eq!(TxnType::parse("Sale"), Some(TxnType::Sale));
        assert_eq!(TxnType::parse("PURCHASE"), Some(TxnType::Sale));
        assert_eq!(TxnType::parse(" refund "), Some(TxnType::Refund));
        assert_eq!(TxnType::parse("nonsense"), None);
    }

    #[test]
    fn natural_key_prefers_rrn_then_auth() {
        let mut txn = CanonicalTxn {
            id: TxnId("t1".into()),
            source: SourceId("switch".into()),
            merchant_id: MerchantId("m1".into()),
            amount: Money::new(15000, Currency::new("EGP").unwrap()),
            txn_type: TxnType::Sale,
            auth_code: Some("AUTH9".into()),
            rrn: Some("RRN123".into()),
            pan_token: None,
            captured_at: Utc::now(),
            raw_ref: RawRef { file: "switch.csv".into(), line: 2 },
        };
        assert_eq!(txn.natural_key(), Some("RRN123"));   // RRN wins

        txn.rrn = None;
        assert_eq!(txn.natural_key(), Some("AUTH9"));     // falls back to auth

        txn.auth_code = None;
        assert_eq!(txn.natural_key(), None);              // neither -> None
    }
}
```

Run `cargo test`. Everything from Module 1 plus these two.

**Feel it:** write a throwaway `match` on `TxnType` that deliberately omits `Chargeback` and `Reversal`. The compiler refuses: `non-exhaustive patterns`. In a 14,000-line legacy system, that refusal is the difference between a caught bug and a production incident.

---

# Module 3 — The matching engine

The centerpiece — what you'd actually be selling. Split into two parts: **Part 1 is the vocabulary of outcomes** (what a reconciliation *produces*), **Part 2 is the algorithm** (how).

## Part 1 — Designing the outcomes

Define every possible outcome as a type, then the algorithm becomes almost mechanical — its job is just to route each transaction into one of the buckets we've named.

When you reconcile two piles of transactions, what can happen to any record? It pairs cleanly with the same amount (a **match**); it pairs but amounts differ slightly (a *softer* match to flag); it pairs by key but amounts are way off (an **exception** to investigate); or it has no partner at all (an **exception**). Plus two edge cases that *will* bite: the same key appearing twice on one side (ambiguous), and a record with no key. Both exceptions.

Two clean categories: **matches** (with a quality flag) and **exceptions** (with a reason).

### Step 1: the match types

Create `src/matching.rs`:

```rust
use crate::money::Money;
use crate::txn::{CanonicalTxn, SourceId, TxnId};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

/// How a pair was matched. Lets an auditor see how much of a run cleared on
/// strong evidence vs. tolerant heuristics.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum MatchKind {
    /// Same key, same amount, same currency.
    Exact,
    /// Same key, amount within configured tolerance.
    Tolerant,
}

/// A confirmed pairing: this transaction in one source corresponds to that one
/// in another.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct Match {
    pub left: TxnId,
    pub right: TxnId,
    pub kind: MatchKind,
    /// Minor-unit difference (0 for exact, within tolerance otherwise).
    pub amount_diff_minor: i64,
}
```

We store `amount_diff_minor` even when often zero because an auditor reviewing a *tolerant* match wants to see "off by 2 piastres" without recomputing it. The result should carry its own evidence — same instinct as the audit trail.

### Step 2: exceptions as a sum type — the important one

In C#, an `enum` is just named integers. To attach *different data* to each case you'd reach for a class hierarchy or a discriminated-union library. A Rust enum **is** a discriminated union — each variant can carry its own distinct payload:

```rust
/// Why a transaction could not be matched. Each variant carries exactly the
/// context needed to investigate that specific failure.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(tag = "reason", rename_all = "snake_case")]
pub enum Exception {
    /// Present in one source with no counterpart in the other.
    Unmatched { txn: TxnId, source: SourceId },

    /// Keys matched but the amount gap exceeded tolerance.
    AmountMismatch { left: TxnId, right: TxnId, diff_minor: i64 },

    /// One key appears more than once within a source — ambiguous, so we
    /// refuse to guess which record is the real pair.
    DuplicateKey { key: String, txns: Vec<TxnId> },

    /// A record had no usable natural key (no RRN and no auth code).
    NoMatchKey { txn: TxnId, source: SourceId },
}
```

Each variant has its own *shape*: `Unmatched` carries one txn and its source; `AmountMismatch` carries two txns and the gap; `DuplicateKey` carries the offending key and a *list* of colliding txns. One type, four shapes, each holding exactly what that situation needs. When you later `match` on an `Exception`, Rust forces you to destructure each variant correctly — you literally cannot read `diff_minor` off an `Unmatched`, because it isn't there.

The `#[serde(tag = "reason", ...)]` produces clean JSON — each exception serializes as `{"reason": "amount_mismatch", "left": ..., "right": ..., "diff_minor": ...}`.

### Step 3: the knobs and the container

```rust
/// Tunable parameters for a run.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MatchConfig {
    /// Max amount difference (minor units) still treated as a match. Absorbs
    /// rounding, FX timing, small fee deltas.
    pub amount_tolerance_minor: i64,
}

impl Default for MatchConfig {
    fn default() -> Self {
        MatchConfig { amount_tolerance_minor: 0 }
    }
}

/// The complete result of one reconciliation run.
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct ReconResult {
    pub matches: Vec<Match>,
    pub exceptions: Vec<Exception>,
}
```

Tolerance defaults to `0`, meaning *strict* (only exact matches) until you deliberately loosen it. Safe by default; you opt into leniency.

### Step 4: a summary

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub struct ReconSummary {
    pub matched_total: usize,
    pub matched_exact: usize,
    pub matched_tolerant: usize,
    pub exceptions_total: usize,
}

impl ReconResult {
    pub fn summary(&self) -> ReconSummary {
        let mut exact = 0;
        let mut tolerant = 0;
        for m in &self.matches {
            match m.kind {
                MatchKind::Exact => exact += 1,
                MatchKind::Tolerant => tolerant += 1,
            }
        }
        ReconSummary {
            matched_total: self.matches.len(),
            matched_exact: exact,
            matched_tolerant: tolerant,
            exceptions_total: self.exceptions.len(),
        }
    }
}
```

That inner `match m.kind` is your exhaustiveness safety net: add a third `MatchKind` next month and this won't compile until you decide how to count it.

Register in `src/lib.rs`: `pub mod matching;`

## Part 2 — The matching algorithm

**The strategy, on paper.** Imagine two stacks of transactions: `left` (switch records) and `right` (bank settlement). You would not compare every left against every right — that's N×M, a trillion comparisons for two million-row files. Instead you'd **sort both stacks by reference number, then walk down matching same-to-same.** Group by key, then compare within a key.

The algorithm:

1. **Bucket each side by natural key** (RRN, falling back to auth code). Lookups become instant.
2. While bucketing, catch the structural problems: records with **no key** (-> `NoMatchKey`) and keys appearing **more than once on one side** (-> `DuplicateKey`).
3. **Walk the left keys.** For each, look up the same key on the right: found + equal amounts -> `Exact`; found + within tolerance -> `Tolerant`; found + too far -> `AmountMismatch`; not found -> `Unmatched`.
4. **Walk the right keys** for any the left never had -> `Unmatched`.

The grouping turns the quadratic nightmare into roughly linear work — and it's just "use a hash map."

### Step 1: the bucketing helper

```rust
/// Bucket transactions by natural key. Emits `NoMatchKey` (no usable key) and
/// `DuplicateKey` (same key seen more than once on this side) into `result` as
/// it goes. Returns only the keys that map to exactly ONE transaction — the
/// only ones safe to match on.
fn bucket_by_key<'a>(
    txns: &'a [CanonicalTxn],
    result: &mut ReconResult,
) -> HashMap<String, &'a CanonicalTxn> {
    // First pass: group every txn under its key.
    let mut groups: HashMap<String, Vec<&CanonicalTxn>> = HashMap::new();
    for txn in txns {
        match txn.natural_key() {
            Some(key) => groups.entry(key.to_string()).or_default().push(txn),
            None => result.exceptions.push(Exception::NoMatchKey {
                txn: txn.id.clone(),
                source: txn.source.clone(),
            }),
        }
    }

    // Second pass: keep unique keys; flag collisions as ambiguous.
    let mut unique = HashMap::new();
    for (key, group) in groups {
        if group.len() == 1 {
            unique.insert(key, group[0]);
        } else {
            result.exceptions.push(Exception::DuplicateKey {
                key,
                txns: group.iter().map(|t| t.id.clone()).collect(),
            });
        }
    }
    unique
}
```

The `<'a>` and `&'a CanonicalTxn` are **lifetimes**. Read `&'a CanonicalTxn` as "a borrowed reference to a transaction that lives at least as long as `'a`." We store *references* in our map, not copies — the map points back into the original `txns` slice. The lifetime is your promise: "the original data outlives this map." In C# every reference type is silently a reference and the GC cleans up whenever; in Rust there's no GC, so you state *how long* a borrow is valid and the compiler verifies you never hold a dangling pointer. It's the GC's safety guarantee, enforced at compile time at zero runtime cost.

`groups.entry(key).or_default().push(txn)` is idiomatic get-or-create — like C#'s `GetValueOrAdd`.

Why `.clone()` on the IDs going *into* exceptions but *references* into the map? The map is short-lived scratch space, so borrowing is fine and fast. The exceptions are part of the result we *return* — they must own their data, because they outlive the borrowed `txns`. Borrow when transient, own when you escape.

### Step 2: the main `reconcile` function

```rust
/// Reconcile a `left` source against a `right` source. In short: bucket both
/// sides by key, then route every key into a match or a typed exception.
pub fn reconcile(
    left: &[CanonicalTxn],
    right: &[CanonicalTxn],
    config: &MatchConfig,
) -> ReconResult {
    let mut result = ReconResult::default();

    let left_buckets = bucket_by_key(left, &mut result);
    let right_buckets = bucket_by_key(right, &mut result);

    // Walk every key on the left, look for its partner on the right.
    for (key, left_txn) in &left_buckets {
        match right_buckets.get(key) {
            Some(right_txn) => {
                let diff = left_txn
                    .amount
                    .abs_diff_minor(right_txn.amount)
                    .unwrap_or(i64::MAX); // currency mismatch => never matches

                if diff == 0 {
                    result.matches.push(Match {
                        left: left_txn.id.clone(),
                        right: right_txn.id.clone(),
                        kind: MatchKind::Exact,
                        amount_diff_minor: 0,
                    });
                } else if diff <= config.amount_tolerance_minor {
                    result.matches.push(Match {
                        left: left_txn.id.clone(),
                        right: right_txn.id.clone(),
                        kind: MatchKind::Tolerant,
                        amount_diff_minor: diff,
                    });
                } else {
                    result.exceptions.push(Exception::AmountMismatch {
                        left: left_txn.id.clone(),
                        right: right_txn.id.clone(),
                        diff_minor: diff,
                    });
                }
            }
            None => result.exceptions.push(Exception::Unmatched {
                txn: left_txn.id.clone(),
                source: left_txn.source.clone(),
            }),
        }
    }

    // Any right-side key the left never had is also unmatched.
    for (key, right_txn) in &right_buckets {
        if !left_buckets.contains_key(key) {
            result.exceptions.push(Exception::Unmatched {
                txn: right_txn.id.clone(),
                source: right_txn.source.clone(),
            });
        }
    }

    result
}
```

`abs_diff_minor` — the method you planted in Module 1 "for later" — is the heart of the decision. It returns a `Result` (currency mismatch is an error), and `.unwrap_or(i64::MAX)` says "if the currencies don't even match, treat the gap as infinite so it can never be a match." A deliberate, safe interpretation, not a crash.

The three-way decision reads exactly like the business rule. The ordering matters — exact is checked first so a tolerance of zero still produces `Exact`, not `Tolerant`, for identical amounts. The second loop only sweeps up right-only leftovers; we check `!left_buckets.contains_key(key)` so we don't double-report.

### Step 3: re-exports

`src/lib.rs`:

```rust
pub mod matching;
pub mod money;
pub mod txn;

pub use matching::{
    reconcile, Exception, Match, MatchConfig, MatchKind, ReconResult, ReconSummary,
};
pub use money::{Currency, Money};
pub use txn::{CanonicalTxn, MerchantId, RawRef, SourceId, TxnId, TxnType};
```

`pub use` re-exports so the API crate writes `use recon_core::reconcile` instead of `use recon_core::matching::reconcile`. It's your crate's public façade.

## Checkpoint

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::money::{Currency, Money};
    use crate::txn::{MerchantId, RawRef, TxnType};
    use chrono::Utc;

    fn egp() -> Currency { Currency::new("EGP").unwrap() }

    fn txn(id: &str, source: &str, minor: i64, rrn: Option<&str>) -> CanonicalTxn {
        CanonicalTxn {
            id: TxnId(id.into()),
            source: SourceId(source.into()),
            merchant_id: MerchantId("m1".into()),
            amount: Money::new(minor, egp()),
            txn_type: TxnType::Sale,
            auth_code: None,
            rrn: rrn.map(|s| s.to_string()),
            pan_token: None,
            captured_at: Utc::now(),
            raw_ref: RawRef { file: "test".into(), line: 1 },
        }
    }

    #[test]
    fn exact_match() {
        let left = vec![txn("l1", "switch", 5000, Some("RRN1"))];
        let right = vec![txn("r1", "bank", 5000, Some("RRN1"))];
        let res = reconcile(&left, &right, &MatchConfig::default());
        assert_eq!(res.matches.len(), 1);
        assert_eq!(res.matches[0].kind, MatchKind::Exact);
        assert!(res.exceptions.is_empty());
    }

    #[test]
    fn tolerant_match_within_tolerance() {
        let left = vec![txn("l1", "switch", 5000, Some("RRN1"))];
        let right = vec![txn("r1", "bank", 5002, Some("RRN1"))];
        let cfg = MatchConfig { amount_tolerance_minor: 5 };
        let res = reconcile(&left, &right, &cfg);
        assert_eq!(res.matches[0].kind, MatchKind::Tolerant);
        assert_eq!(res.matches[0].amount_diff_minor, 2);
    }

    #[test]
    fn amount_mismatch_beyond_tolerance() {
        let left = vec![txn("l1", "switch", 5000, Some("RRN1"))];
        let right = vec![txn("r1", "bank", 9999, Some("RRN1"))];
        let res = reconcile(&left, &right, &MatchConfig::default());
        assert!(res.matches.is_empty());
        assert!(matches!(res.exceptions[0],
            Exception::AmountMismatch { diff_minor: 4999, .. }));
    }

    #[test]
    fn unmatched_on_both_sides() {
        let left = vec![txn("l1", "switch", 5000, Some("RRN1"))];
        let right = vec![txn("r1", "bank", 5000, Some("RRN2"))];
        let res = reconcile(&left, &right, &MatchConfig::default());
        assert!(res.matches.is_empty());
        assert_eq!(res.exceptions.len(), 2); // one each side
    }

    #[test]
    fn duplicate_key_is_ambiguous() {
        let left = vec![
            txn("l1", "switch", 5000, Some("RRN1")),
            txn("l2", "switch", 5000, Some("RRN1")),
        ];
        let right = vec![txn("r1", "bank", 5000, Some("RRN1"))];
        let res = reconcile(&left, &right, &MatchConfig::default());
        assert!(res.matches.is_empty());
        assert!(res.exceptions.iter()
            .any(|e| matches!(e, Exception::DuplicateKey { .. })));
    }

    #[test]
    fn no_key_becomes_exception() {
        let left = vec![txn("l1", "switch", 5000, None)];
        let right = vec![txn("r1", "bank", 5000, Some("RRN1"))];
        let res = reconcile(&left, &right, &MatchConfig::default());
        assert!(res.exceptions.iter()
            .any(|e| matches!(e, Exception::NoMatchKey { .. })));
    }
}
```

Run `cargo test`. Six new tests, each lighting up one path. Green means the engine is *correct*, not just compiling.

`matches!(value, Pattern { .. })` is "does this value match this shape?" returning a bool, with `..` meaning "don't care about the rest of the fields."

**The duplicate-key principle:** the matcher *refuses* to pair anything when `RRN1` appears twice, even though one is a perfect amount match. That's correct. *A reconciliation engine's worst failure is a confident wrong match, not an honest unmatched.* If you auto-pick the one whose amount lines up, you've made an unauditable guess — maybe the *other* one was the real pair and you just hid a genuine discrepancy. An exception says "a human must look." A wrong match says nothing and corrupts the books silently. In money systems, **silence is the expensive failure.**

---

# Module 4 — Ingestion (and the workspace split)

**The moment.** We add CSV parsing, which pulls in the `csv` crate. But `recon-core` is pure domain logic — the part you'd unit-test exhaustively and trust forever. If you bolt a file parser onto it, you weld your pristine domain to a specific input format. Tomorrow's Visa parser, Mastercard parser, each drag their own dependencies into the heart of your engine. *That's the pressure: a dependency that doesn't belong in the domain wants in.*

The relief is a second crate. `recon-ingest` depends on `recon-core` (it produces `CanonicalTxn`s), but `recon-core` knows nothing about `recon-ingest`. Dependencies point *inward*, toward the domain, never out. That's Clean Architecture as a compiler-enforced rule.

## Step 1: restructure into a workspace

From the folder *containing* `recon-core`:

```powershell
mkdir recon
mkdir recon\crates
move recon-core recon\crates\recon-core
cd recon
```

The workspace manifest at `recon\Cargo.toml` (no `[package]`, only members and shared versions):

```toml
[workspace]
resolver = "2"
members = ["crates/recon-core", "crates/recon-ingest"]

[workspace.package]
version = "0.1.0"
edition = "2021"

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "1"
csv = "1"
```

`[workspace.dependencies]` is version control for the whole repo: declare a version once, every crate says "use the workspace version." No drift.

Update `crates/recon-core/Cargo.toml` to inherit:

```toml
[package]
name = "recon-core"
version.workspace = true
edition.workspace = true

[dependencies]
serde = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
```

## Step 2: scaffold the ingest crate

```powershell
cd crates
cargo new --lib recon-ingest
cd ..
```

`crates/recon-ingest/Cargo.toml` — note the **path dependency** on core:

```toml
[package]
name = "recon-ingest"
version.workspace = true
edition.workspace = true

[dependencies]
recon-core = { path = "../recon-core" }
serde = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
csv = { workspace = true }
```

That `recon-core = { path = "../recon-core" }` line *is* the architecture. There is no reverse line, and never will be.

## Step 3: the `Source` trait — the seam

Open `crates/recon-ingest/src/lib.rs`:

```rust
//! `recon-ingest` — converts external file formats into `CanonicalTxn`.
//! This crate is the quarantine zone for messy real-world data: all the format
//! quirks and lenient parsing live HERE so the engine never has to know.

use recon_core::{CanonicalTxn, Currency, MerchantId, Money, RawRef, SourceId, TxnId, TxnType};
use chrono::{DateTime, NaiveDate, NaiveDateTime, TimeZone, Utc};
use serde::Deserialize;

#[derive(Debug, thiserror::Error)]
pub enum IngestError {
    #[error("csv parse error: {0}")]
    Csv(#[from] csv::Error),
    #[error("row {line}: {message}")]
    Row { line: u64, message: String },
}

/// Every source parser implements this. The rest of the system depends only on
/// this trait, not on any concrete format — so adding a new format never
/// touches the engine.
pub trait Source {
    fn parse(&self, data: &[u8]) -> Result<Vec<CanonicalTxn>, IngestError>;
}
```

A `trait` is an interface (C# `interface`). A function can accept `&dyn Source` and not care whether it's CSV or Visa. `#[from]` auto-generates the conversion from a `csv::Error` — so when a csv call fails and you write `?`, it becomes our `IngestError` for free.

## Step 4: the lesson — parsing money without floats

A CSV has `"150.00"`. The lazy parse is `"150.00".parse::<f64>()` then multiply by 100. But you spent Module 1 keeping money as exact integers precisely to avoid float dust — if you let a float in *at the door*, you've defeated the whole design. So we parse the decimal string to integer minor units **by hand, never touching `f64`**:

```rust
/// Parse a decimal-string amount like "150.05" into integer minor units
/// (15005) WITHOUT floating point. Supports an optional leading '-'.
fn parse_amount_minor(s: &str) -> Result<i64, String> {
    let s = s.trim();
    let (neg, digits) = match s.strip_prefix('-') {
        Some(rest) => (true, rest),
        None => (false, s),
    };
    let (whole, frac) = match digits.split_once('.') {
        Some((w, f)) => (w, f),
        None => (digits, ""),
    };
    // Normalize the fractional part to exactly 2 digits.
    let frac2 = match frac.len() {
        0 => "00".to_string(),
        1 => format!("{frac}0"),
        2 => frac.to_string(),
        _ => return Err(format!("more than 2 decimal places: {s:?}")),
    };
    let whole_num: i64 = if whole.is_empty() { 0 }
        else { whole.parse().map_err(|_| format!("bad whole part: {whole:?}"))? };
    let frac_num: i64 = frac2.parse().map_err(|_| format!("bad fraction: {frac2:?}"))?;
    let minor = whole_num.checked_mul(100)
        .and_then(|w| w.checked_add(frac_num))
        .ok_or_else(|| "amount overflow".to_string())?;
    Ok(if neg { -minor } else { minor })
}
```

Split off a leading minus, split on the dot, pad the fraction to two digits, parse *each part as an integer*, combine as `whole * 100 + frac`. `"150.5"` -> whole `150`, frac `"5"` -> padded `"50"` -> `15050`. No float ever exists. We reject three-decimal input rather than silently rounding — strict at the core.

## Step 5: supporting parsers

```rust
fn parse_datetime(s: &str) -> Result<DateTime<Utc>, String> {
    let s = s.trim();
    if let Ok(dt) = NaiveDateTime::parse_from_str(s, "%Y-%m-%dT%H:%M:%S") {
        return Ok(Utc.from_utc_datetime(&dt));
    }
    if let Ok(d) = NaiveDate::parse_from_str(s, "%Y-%m-%d") {
        return Ok(Utc.from_utc_datetime(&d.and_hms_opt(0, 0, 0).unwrap()));
    }
    Err(format!("unrecognized date: {s:?}"))
}

/// Empty string => None. This is where "" becomes a proper absent value.
fn opt(s: String) -> Option<String> {
    let t = s.trim();
    if t.is_empty() { None } else { Some(t.to_string()) }
}
```

`opt` is principled: a CSV's empty cell is `""`, but our domain wants `Option<String>` — *absence*, not an empty string. We convert at the boundary so the core only sees honest `None`.

## Step 6: the CSV row and the parser

The DTO mirrors the *file*, not the domain:

```rust
#[derive(Debug, Deserialize)]
struct CsvRow {
    txn_id: String,
    merchant_id: String,
    amount: String,        // kept as String — we parse it ourselves, no floats
    currency: String,
    txn_type: String,
    #[serde(default)] auth_code: String,
    #[serde(default)] rrn: String,
    #[serde(default)] pan_token: String,
    captured_at: String,
}
```

`#[serde(default)]` means a missing column becomes empty rather than an error. Now the parser:

```rust
/// A generic CSV source, tagged with the source-system name it represents.
pub struct CsvSource {
    pub source_id: SourceId,
    pub file_name: String,
}

impl CsvSource {
    pub fn new(source_id: impl Into<String>, file_name: impl Into<String>) -> Self {
        CsvSource {
            source_id: SourceId(source_id.into()),
            file_name: file_name.into(),
        }
    }
}

impl Source for CsvSource {
    fn parse(&self, data: &[u8]) -> Result<Vec<CanonicalTxn>, IngestError> {
        let mut reader = csv::ReaderBuilder::new()
            .trim(csv::Trim::All)
            .from_reader(data);
        let mut out = Vec::new();

        // CSV line numbers: header is line 1, so the first record is line 2.
        for (idx, record) in reader.deserialize::<CsvRow>().enumerate() {
            let line = idx as u64 + 2;
            let row = record?;

            let amount_minor = parse_amount_minor(&row.amount)
                .map_err(|m| IngestError::Row { line, message: m })?;
            let currency = Currency::new(row.currency.trim())
                .map_err(|e| IngestError::Row { line, message: e.to_string() })?;
            let txn_type = TxnType::parse(&row.txn_type)
                .ok_or_else(|| IngestError::Row {
                    line, message: format!("unknown txn_type: {:?}", row.txn_type) })?;
            let captured_at = parse_datetime(&row.captured_at)
                .map_err(|m| IngestError::Row { line, message: m })?;

            out.push(CanonicalTxn {
                id: TxnId(row.txn_id),
                source: self.source_id.clone(),
                merchant_id: MerchantId(row.merchant_id),
                amount: Money::new(amount_minor, currency),
                txn_type,
                auth_code: opt(row.auth_code),
                rrn: opt(row.rrn),
                pan_token: opt(row.pan_token),
                captured_at,
                raw_ref: RawRef { file: self.file_name.clone(), line },
            });
        }
        Ok(out)
    }
}
```

The thing to notice: `let line = idx as u64 + 2;` and how `line` rides along into *every* error and into `raw_ref`. That's the **audit trail from Module 2 being born right here.** Every canonical transaction remembers the exact file and line — so months later, when an auditor points at one disputed match, you can answer "row 4,118 of switch.csv." Every fallible step is wrapped with `.map_err(|...| IngestError::Row { line, ... })?` — so a bad cell says *which row and why*.

## Checkpoint

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_amount_never_uses_float() {
        assert_eq!(parse_amount_minor("150.00").unwrap(), 15000);
        assert_eq!(parse_amount_minor("150").unwrap(), 15000);
        assert_eq!(parse_amount_minor("150.5").unwrap(), 15050);
        assert_eq!(parse_amount_minor("0.99").unwrap(), 99);
        assert_eq!(parse_amount_minor("-12.34").unwrap(), -1234);
        assert!(parse_amount_minor("1.234").is_err());   // 3 decimals rejected
    }

    #[test]
    fn full_csv_round_trip() {
        let csv = "\
txn_id,merchant_id,amount,currency,txn_type,auth_code,rrn,pan_token,captured_at
T1,M1,150.00,EGP,sale,AUTH1,RRN1,tok_abc,2026-05-20
T2,M1,75.50,EGP,refund,,RRN2,,2026-05-20T10:15:00
";
        let src = CsvSource::new("switch", "switch.csv");
        let txns = src.parse(csv.as_bytes()).unwrap();
        assert_eq!(txns.len(), 2);
        assert_eq!(txns[0].amount.minor, 15000);
        assert_eq!(txns[0].rrn.as_deref(), Some("RRN1"));
        assert_eq!(txns[1].auth_code, None);          // empty cell => None
        assert_eq!(txns[1].raw_ref.line, 3);          // audit trail populated
    }

    #[test]
    fn bad_row_reports_its_line_number() {
        let csv = "\
txn_id,merchant_id,amount,currency,txn_type,auth_code,rrn,pan_token,captured_at
T1,M1,1.234,EGP,sale,,RRN1,,2026-05-20
";
        let err = CsvSource::new("switch", "switch.csv")
            .parse(csv.as_bytes()).unwrap_err();
        match err {
            IngestError::Row { line, .. } => assert_eq!(line, 2),
            other => panic!("expected Row error, got {other:?}"),
        }
    }
}
```

Run `cargo test` from the workspace root. Two crates now compile and test together.

**Feel it:** try to add `recon-ingest = { path = "../recon-ingest" }` to `recon-core/Cargo.toml` and `cargo build`. Cargo rejects it — a **circular dependency**. Core cannot depend on ingest, because ingest already depends on core. The compiler physically enforces the dependency direction. That refusal is Clean Architecture made of iron.

---

# Module 5 — Orchestration

**The principle: orchestration is the layer that knows the *order of operations*, and nothing else.**

`recon-ingest` knows how to turn bytes into `CanonicalTxn`s. `recon-core` knows how to match them. But nobody yet knows the *workflow*: "first parse the left file, then parse the right file, then feed both into the matcher." That sequencing belongs to neither — ingest shouldn't know matching exists, and core shouldn't know files exist.

So it gets its own home: `recon-engine`, the **application/use-case layer**. In your C# world this is a "service" or "handler": it doesn't contain business rules, it *orchestrates* the things that do. Engine depends on both core and ingest; neither depends back.

A second job: **synthetic data generation**, which turns your engine into something demonstrable.

## Step 1: the crate

```powershell
cd crates
cargo new --lib recon-engine
cd ..
```

Add to workspace `members`:

```toml
members = ["crates/recon-core", "crates/recon-ingest", "crates/recon-engine"]
```

`crates/recon-engine/Cargo.toml` — depends on **both** lower crates:

```toml
[package]
name = "recon-engine"
version.workspace = true
edition.workspace = true

[dependencies]
recon-core = { path = "../recon-core" }
recon-ingest = { path = "../recon-ingest" }
chrono = { workspace = true }
thiserror = { workspace = true }
```

## Step 2: the orchestration function

```rust
//! `recon-engine` — orchestration. Loads transactions from sources, runs the
//! matching engine, and produces a result. Also generates synthetic data so
//! the product is demonstrable without real (sensitive, hard-to-get) files.

use recon_core::{
    reconcile, CanonicalTxn, Currency, MatchConfig, MerchantId, Money, RawRef,
    ReconResult, SourceId, TxnId, TxnType,
};
use recon_ingest::{CsvSource, IngestError, Source};

#[derive(Debug, thiserror::Error)]
pub enum EngineError {
    #[error(transparent)]
    Ingest(#[from] IngestError),
}

/// Reconcile two CSV payloads end to end: parse each, then match.
pub fn reconcile_csv(
    left_name: &str,
    left_csv: &[u8],
    right_name: &str,
    right_csv: &[u8],
    config: &MatchConfig,
) -> Result<ReconResult, EngineError> {
    let left = CsvSource::new(left_name, format!("{left_name}.csv")).parse(left_csv)?;
    let right = CsvSource::new(right_name, format!("{right_name}.csv")).parse(right_csv)?;
    Ok(reconcile(&left, &right, config))
}
```

Read the body as a sentence: parse the left source, parse the right source, reconcile them. That's the entire workflow. Everything hard happens *inside* `parse` and `reconcile` — this layer just names the sequence.

`#[error(transparent)]` means "this error has no message of its own — forward the inner error's message." We wrap `IngestError` in `EngineError` so the engine has *one* error type to return, but we don't bury the useful "row 412: bad currency" detail.

## Step 3: why synthetic data is a feature

To show this working, you need data. But real switch logs and bank settlement files are *sensitive* and *hard to get* — chicken and egg: they won't give you data until they trust the product, but you can't demo without data.

The escape: generate realistic *fake* data with the **same kinds of discrepancies real reconciliation hits** — mostly-clean matches, a sprinkle of rounding gaps, some real mismatches, some transactions present on one side but missing on the other. If your generator injects the same problems the engine catches, a synthetic run *demonstrates the actual value*. It's also how you load-test.

## Step 4: a deterministic PRNG

We need randomness to scatter discrepancies, but **reproducible**: the same seed must always produce the same data set. A demo that looks different every run is hard to talk through, and a *test* on random data is worthless if it can't repeat.

```rust
/// Tiny deterministic PRNG (xorshift). Same seed => same sequence, every time.
/// Not cryptographic — we only need reproducible spread, not security.
struct Rng(u64);

impl Rng {
    fn next(&mut self) -> u64 {
        let mut x = self.0;
        x ^= x << 13;
        x ^= x >> 7;
        x ^= x << 17;
        self.0 = x;
        x
    }
    fn below(&mut self, n: u64) -> u64 {
        self.next() % n
    }
}
```

The three shift-and-XOR lines are the whole algorithm: mangle the state's bits, store it back, return it. It looks random because bit-mangling scrambles thoroughly, but it's *fully determined* by the starting `self.0`. The seed can't be zero (zero XOR-shifted stays zero forever). This is non-cryptographic — fine for test/demo data; for real randomness you'd reach for `rand`.

## Step 5: the generator

```rust
pub struct SyntheticData {
    pub switch_txns: Vec<CanonicalTxn>,
    pub bank_txns: Vec<CanonicalTxn>,
}

pub fn generate_synthetic(count: usize, seed: u64) -> SyntheticData {
    let egp = Currency::new("EGP").unwrap();
    let mut rng = Rng(seed | 1);            // | 1 guarantees nonzero
    let base = chrono::Utc::now();

    let mut switch_txns = Vec::with_capacity(count);
    let mut bank_txns = Vec::new();

    for i in 0..count {
        let rrn = format!("RRN{:08}", 10_000 + i);
        let amount_minor = 1_000 + rng.below(500_000) as i64;   // ~10..5010 EGP
        let merchant = format!("M{:03}", rng.below(20));

        let switch_txn = CanonicalTxn {
            id: TxnId(format!("S{i}")),
            source: SourceId("switch".into()),
            merchant_id: MerchantId(merchant.clone()),
            amount: Money::new(amount_minor, egp),
            txn_type: TxnType::Sale,
            auth_code: Some(format!("A{:06}", rng.below(1_000_000))),
            rrn: Some(rrn.clone()),
            pan_token: Some(format!("tok_{:x}", rng.next())),
            captured_at: base,
            raw_ref: RawRef { file: "switch.csv".into(), line: (i + 2) as u64 },
        };

        // Roll for this row's fate on the bank side.
        let roll = rng.below(100);
        let bank_amount = if roll < 70 {
            amount_minor                                   // 70%: exact
        } else if roll < 85 {
            amount_minor + 1 + rng.below(3) as i64         // 15%: tiny delta
        } else if roll < 92 {
            amount_minor + 500 + rng.below(2_000) as i64   // 7%: real mismatch
        } else {
            switch_txns.push(switch_txn);                  // 8%: missing on bank
            continue;
        };

        bank_txns.push(CanonicalTxn {
            id: TxnId(format!("B{i}")),
            source: SourceId("bank".into()),
            merchant_id: MerchantId(merchant),
            amount: Money::new(bank_amount, egp),
            txn_type: TxnType::Sale,
            auth_code: switch_txn.auth_code.clone(),
            rrn: Some(rrn),
            pan_token: switch_txn.pan_token.clone(),
            captured_at: base,
            raw_ref: RawRef { file: "bank.csv".into(), line: (i + 2) as u64 },
        });
        switch_txns.push(switch_txn);
    }

    // A few bank-only rows (settlement entries with no switch record).
    for j in 0..(count / 25).max(1) {
        bank_txns.push(CanonicalTxn {
            id: TxnId(format!("BX{j}")),
            source: SourceId("bank".into()),
            merchant_id: MerchantId(format!("M{:03}", rng.below(20))),
            amount: Money::new(2_000 + rng.below(50_000) as i64, egp),
            txn_type: TxnType::Sale,
            auth_code: Some(format!("A{:06}", rng.below(1_000_000))),
            rrn: Some(format!("RRN{:08}", 90_000 + j)),
            pan_token: None,
            captured_at: base,
            raw_ref: RawRef { file: "bank.csv".into(), line: (count + j + 2) as u64 },
        });
    }

    SyntheticData { switch_txns, bank_txns }
}
```

The percentages *are* the design. That `roll < 70 / 85 / 92 / else` ladder models a healthy reconciliation: ~70% clean, ~15% within-tolerance noise, ~7% genuine breaks, ~8% timing gaps. The bank row reuses the switch row's `rrn` and `auth_code`, so the matcher pairs them on the natural key from Module 2.

One Rust note: see `switch_txn.auth_code.clone()` *before* `switch_txns.push(switch_txn)`. We clone the fields we need first, because `push` *moves* `switch_txn` — gives away ownership — and after that we can't read from it. Flip those lines and it won't compile: "use of moved value." That's ownership doing its job.

## Step 6: convenience wrapper

```rust
pub fn run_synthetic(count: usize, seed: u64, config: &MatchConfig)
    -> (SyntheticData, ReconResult)
{
    let data = generate_synthetic(count, seed);
    let result = reconcile(&data.switch_txns, &data.bank_txns, config);
    (data, result)
}
```

Returns *both* the data and the result so a caller can show the run *and* its inputs.

## Checkpoint

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn same_seed_is_reproducible() {
        let cfg = MatchConfig { amount_tolerance_minor: 5 };
        let (_, a) = run_synthetic(200, 42, &cfg);
        let (_, b) = run_synthetic(200, 42, &cfg);
        assert_eq!(a.summary(), b.summary());   // identical, always
    }

    #[test]
    fn produces_a_realistic_mix() {
        let cfg = MatchConfig { amount_tolerance_minor: 5 };
        let (_, res) = run_synthetic(500, 7, &cfg);
        let s = res.summary();
        assert!(s.matched_exact > 0,    "should have exact matches");
        assert!(s.matched_tolerant > 0, "should have tolerant matches");
        assert!(s.exceptions_total > 0, "should have exceptions");
    }

    #[test]
    fn reconcile_csv_end_to_end() {
        let left = "\
txn_id,merchant_id,amount,currency,txn_type,auth_code,rrn,pan_token,captured_at
T1,M1,150.00,EGP,sale,A1,RRN1,,2026-05-20
T2,M1,75.00,EGP,sale,A2,RRN2,,2026-05-20
";
        let right = "\
txn_id,merchant_id,amount,currency,txn_type,auth_code,rrn,pan_token,captured_at
B1,M1,150.00,EGP,sale,A1,RRN1,,2026-05-20
B2,M1,80.00,EGP,sale,A2,RRN2,,2026-05-20
";
        let res = reconcile_csv("switch", left.as_bytes(),
                                "bank", right.as_bytes(),
                                &MatchConfig::default()).unwrap();
        let s = res.summary();
        assert_eq!(s.matched_exact, 1);     // T1/B1 line up
        assert_eq!(s.exceptions_total, 1);  // T2/B2 differ by 5.00 EGP
    }
}
```

Run `cargo test`. Three crates compile together, and `reconcile_csv_end_to_end` exercises the *entire stack* — parse, normalize, match, summarize — in one call.

**Feel it:** add `println!("{:#?}", s);` in `produces_a_realistic_mix`, run `cargo test -- --nocapture produces_a_realistic_mix`, then change the seed. The exact counts shift, but the *proportions* stay roughly 70/15/7/8, because that's baked into the ladder, not the seed. That's the difference between **what's random (the specific rows) and what's structural (the distribution).**

---

# Module 6 — The HTTP API

**The principle: the API layer is a translator, and it must stay thin.** Everything valuable already exists. `reconcile_csv` and `run_synthetic` are the product. The API's *only* job is to translate between HTTP/JSON/files and your engine's function calls, then translate results back. No business logic here.

This is the outermost ring — the **delivery mechanism**. You could throw away `recon-api` and replace it with a CLI or a gRPC server, and `recon-engine` wouldn't change by a line. Delivery is interchangeable; the core is not.

## Step 1: a binary, not a library

```powershell
cd crates
cargo new recon-api          # no --lib: this is a binary
cd ..
```

Add to workspace `members`, then `crates/recon-api/Cargo.toml`:

```toml
[package]
name = "recon-api"
version.workspace = true
edition.workspace = true

[dependencies]
recon-core = { path = "../recon-core" }
recon-engine = { path = "../recon-engine" }
serde = { workspace = true }
serde_json = { workspace = true }
axum = { workspace = true }
tokio = { workspace = true }
tower-http = { workspace = true }
tracing = { workspace = true }
tracing-subscriber = { workspace = true }
```

Add to root `[workspace.dependencies]`:

```toml
serde_json = "1"
axum = { version = "0.7", features = ["multipart"] }
tokio = { version = "1", features = ["full"] }
tower-http = { version = "0.5", features = ["cors"] }
tracing = "0.1"
tracing-subscriber = "0.3"
```

In C# terms: `axum` is your web framework (ASP.NET Core), `tokio` is the async *runtime* (.NET bakes this in; Rust makes you bring your own), `tower-http` is middleware, `tracing` is `ILogger`, `serde_json` is the JSON serializer.

## Step 2: async — the one new concept

A web server spends almost all its time *waiting*. If each wait blocked a whole OS thread, you'd need a thread per connection and the server would fall over. `async` lets a function *pause* at each await point and hand the thread to other work, resuming when ready. One thread serves thousands of connections by never sitting idle.

Same idea as C#'s `async`/`await`. Two differences: Rust's async values are **futures** (C#'s `Task`) and they're *lazy* — a future does nothing until something "drives" it. That something is the runtime you bring — Tokio — which is why `main` gets `#[tokio::main]`.

## Step 3: the skeleton

`crates/recon-api/src/main.rs`:

```rust
//! `recon-api` — the HTTP delivery layer. Deliberately thin: parse the request,
//! call into `recon-engine`, serialize the result. No business logic here.

use axum::{routing::get, Router};
use std::net::SocketAddr;

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .init();

    let app = Router::new()
        .route("/api/health", get(health));

    // Host/port configurable so we can dodge OS-reserved port ranges (the
    // Windows 10013 trap). Default to loopback — all we need for local dev.
    let host = std::env::var("HOST").unwrap_or_else(|_| "127.0.0.1".to_string());
    let port: u16 = std::env::var("PORT").ok()
        .and_then(|p| p.parse().ok())
        .unwrap_or(3000);
    let addr: SocketAddr = format!("{host}:{port}").parse().expect("bad HOST/PORT");

    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    tracing::info!("recon-api listening on http://{addr}");
    axum::serve(listener, app).await.unwrap();
}

async fn health() -> &'static str {
    "ok"
}
```

The host/port default of `127.0.0.1:3000` is the fix for the Windows error 10013 (a reserved-port-range / `0.0.0.0`-permission problem); override with `set PORT=5005` if 3000 is taken. `Router::new().route("/api/health", get(health))` says "a GET to `/api/health` runs `health`." That `async fn` returning a string is a **handler**.

Run `cargo run -p recon-api`, then `curl http://localhost:3000/api/health` -> `ok`.

## Step 4: the synthetic endpoint

The response shape solves a real problem: a million-row run would crush the browser. Cap the rows, keep the summary honest:

```rust
use recon_core::{Exception, Match, MatchConfig, ReconResult, ReconSummary};
use recon_engine::run_synthetic;
use serde::{Deserialize, Serialize};

const MAX_ROWS: usize = 500;

/// What every endpoint returns. Summary always reflects the FULL run; the
/// match/exception lists are capped so a huge run doesn't overwhelm the client.
#[derive(Serialize)]
struct ReconResponse {
    summary: ReconSummary,
    matches: Vec<Match>,
    exceptions: Vec<Exception>,
    truncated: bool,
}

fn build_response(result: ReconResult) -> ReconResponse {
    let summary = result.summary();             // computed BEFORE truncating
    let truncated =
        result.matches.len() > MAX_ROWS || result.exceptions.len() > MAX_ROWS;
    let mut matches = result.matches;
    let mut exceptions = result.exceptions;
    matches.truncate(MAX_ROWS);
    exceptions.truncate(MAX_ROWS);
    ReconResponse { summary, matches, exceptions, truncated }
}
```

Compute the summary from the *complete* result, *then* truncate. The headline numbers stay truthful even though you only ship 500 rows. Now the request type and handler:

```rust
#[derive(Deserialize)]
struct SyntheticRequest {
    #[serde(default = "default_count")]   count: usize,
    #[serde(default = "default_seed")]    seed: u64,
    #[serde(default)]                     tolerance_minor: i64,
}
fn default_count() -> usize { 1_000 }
fn default_seed() -> u64 { 42 }

async fn synthetic(
    axum::Json(req): axum::Json<SyntheticRequest>,
) -> axum::Json<ReconResponse> {
    let count = req.count.min(1_000_000);     // cap to protect the server
    let cfg = MatchConfig { amount_tolerance_minor: req.tolerance_minor.max(0) };
    let (_, result) = run_synthetic(count, req.seed, &cfg);
    axum::Json(build_response(result))
}
```

The `axum::Json(req): axum::Json<SyntheticRequest>` parameter is an **extractor** — "read the request body as JSON, deserialize it, and if that fails, automatically return a 400 before my code runs." You declare *what you need* as a typed parameter, and Axum produces it or rejects the request. Returning `axum::Json<ReconResponse>` does the reverse. The `.min(1_000_000)` / `.max(0)` guards stop a hostile request from asking for a billion rows or a negative tolerance.

Wire it in:

```rust
use axum::routing::post;

let app = Router::new()
    .route("/api/health", get(health))
    .route("/api/synthetic", post(synthetic));
```

## Step 5: the file-upload endpoint

File uploads arrive as `multipart/form-data`:

```rust
use axum::{extract::Multipart, http::StatusCode};

async fn reconcile_upload(
    mut multipart: Multipart,
) -> Result<axum::Json<ReconResponse>, (StatusCode, String)> {
    let mut left: Option<Vec<u8>> = None;
    let mut right: Option<Vec<u8>> = None;
    let mut tolerance_minor: i64 = 0;

    // A multipart body is a stream of named fields; walk them one at a time.
    while let Some(field) = multipart.next_field().await
        .map_err(|e| (StatusCode::BAD_REQUEST, format!("multipart error: {e}")))?
    {
        match field.name().unwrap_or("") {
            "left" => left = Some(field.bytes().await
                .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?.to_vec()),
            "right" => right = Some(field.bytes().await
                .map_err(|e| (StatusCode::BAD_REQUEST, e.to_string()))?.to_vec()),
            "tolerance_minor" => {
                let txt = field.text().await.unwrap_or_default();
                tolerance_minor = txt.trim().parse().unwrap_or(0);
            }
            _ => {}   // ignore unexpected fields
        }
    }

    let left = left.ok_or((StatusCode::BAD_REQUEST, "missing 'left' file".into()))?;
    let right = right.ok_or((StatusCode::BAD_REQUEST, "missing 'right' file".into()))?;

    let cfg = MatchConfig { amount_tolerance_minor: tolerance_minor.max(0) };
    let result = recon_engine::reconcile_csv("left", &left, "right", &right, &cfg)
        .map_err(|e| (StatusCode::UNPROCESSABLE_ENTITY, e.to_string()))?;

    Ok(axum::Json(build_response(result)))
}
```

The return type is `Result<Json<...>, (StatusCode, String)>` — success serializes to JSON; failure becomes an HTTP status plus message. This is **HTTP error handling as a return value**, not exceptions. A missing file isn't a crash; it's a 400. The status codes carry meaning: `BAD_REQUEST` (400) for a malformed upload, `UNPROCESSABLE_ENTITY` (422) when files arrived fine but the *CSV content* was bad — that 422 is your `IngestError` from Module 4 (with its row number) surfacing to the client.

Add the route: `.route("/api/reconcile", post(reconcile_upload))`.

## Step 6: serve the UI and allow CORS

```rust
use axum::response::Html;
use tower_http::cors::CorsLayer;

// Pulls web/index.html into the binary at compile time. We build it next module.
const INDEX_HTML: &str = include_str!("../../../web/index.html");

async fn index() -> Html<&'static str> {
    Html(INDEX_HTML)
}
```

Final router:

```rust
let app = Router::new()
    .route("/", get(index))
    .route("/api/health", get(health))
    .route("/api/synthetic", post(synthetic))
    .route("/api/reconcile", post(reconcile_upload))
    .layer(CorsLayer::permissive());
```

`include_str!` reads a file *at compile time* and bakes its contents into the executable — so your finished server is a single binary with the UI inside it. It needs `web/index.html` to exist; create a placeholder for now:

```powershell
mkdir web
echo "<!doctype html><title>recon</title><h1>UI coming in Module 7</h1>" > web\index.html
```

## Checkpoint

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use recon_engine::run_synthetic;

    #[test]
    fn summary_is_honest_even_when_rows_are_truncated() {
        let cfg = MatchConfig { amount_tolerance_minor: 5 };
        let (_, result) = run_synthetic(5_000, 42, &cfg);
        let full_summary = result.summary();

        let resp = build_response(result);

        // Detail is capped...
        assert!(resp.matches.len() <= MAX_ROWS);
        assert!(resp.exceptions.len() <= MAX_ROWS);
        assert!(resp.truncated);
        // ...but the summary still reflects the FULL run.
        assert_eq!(resp.summary.matched_total, full_summary.matched_total);
        assert_eq!(resp.summary, full_summary);
    }
}
```

Run `cargo test` — all four crates compile and test together.

**Feel it:** start the server and hit it three ways. (1) The happy path — the synthetic `curl`. (2) A malformed request — `curl -X POST http://localhost:3000/api/synthetic -H "Content-Type: application/json" -d "garbage"` — you'll get a **400**, generated by the `Json` extractor *before your handler ran a single line*. (3) The upload missing a file — `curl -X POST http://localhost:3000/api/reconcile -F left=@web/index.html` — your **400 "missing 'right' file"**, the error path you *did* write. One error you got for free from a typed extractor, one you authored as a `Result`. Both became correct HTTP responses without a single `try`/`catch`.

---

# Module 7 — The web UI

**The principle: the frontend is a thin skin over an API that already does everything.** Every hard decision is *already made and enforced* on the server. Money is integer minor units. Exceptions are typed. The summary is honest even when truncated. The browser's only jobs: collect input, POST to one of your two endpoints, paint the JSON that comes back. The UI *displays*; it does not *decide*.

This is why it's one self-contained `index.html` — no build step, no framework. Reaching for React here would be carrying workspace-sized machinery for a genuinely simple job. We go **dark, dense, monospace-for-numbers** — the aesthetic of a trading terminal, not a consumer app. The audience is an ops analyst staring at discrepancies.

## Step 1: structure and design tokens

Replace the placeholder `web/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>recon · settlement reconciliation</title>
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link href="https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;500;600&family=IBM+Plex+Sans:wght@400;500;600;700&display=swap" rel="stylesheet" />
<style>
  :root {
    --bg: #0c0e12;      --panel: #14171d;   --panel-2: #1a1e26;
    --line: #262b35;    --ink: #e6e9ef;     --ink-dim: #8a93a3;
    --ink-faint: #5b6373;
    --accent: #4ade80;  /* exact   — green  */
    --blue:   #60a5fa;  /* tolerant — blue  */
    --bad:    #f87171;  /* exception — red  */
    --warn:   #fbbf24;  /* mismatch — amber */
    --mono: "IBM Plex Mono", monospace;
    --sans: "IBM Plex Sans", system-ui, sans-serif;
  }
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body {
    background: var(--bg); color: var(--ink);
    font-family: var(--sans); font-size: 14px; line-height: 1.5;
    -webkit-font-smoothing: antialiased; min-height: 100vh;
    background-image:
      radial-gradient(circle at 12% -10%, rgba(74,222,128,0.06), transparent 40%),
      radial-gradient(circle at 90% 0%, rgba(96,165,250,0.05), transparent 35%);
  }
  .wrap { max-width: 1180px; margin: 0 auto; padding: 32px 28px 80px; }
</style>
</head>
<body>
<div class="wrap">
  <!-- content -->
</div>
<script>
  // logic
</script>
</body>
</html>
```

CSS custom properties (`:root { --accent: ... }`) are the color system declared once — the CSS equivalent of `[workspace.dependencies]`. And the colors aren't decorative: `--accent` *is* "exact match," `--bad` *is* "exception." The palette encodes the domain. IBM Plex Mono for numbers is non-negotiable: monetary columns must align digit-under-digit.

## Step 2: the controls

The app talks to exactly two endpoints, so the UI has exactly two control panels:

```html
<header style="display:flex;align-items:baseline;gap:16px;border-bottom:1px solid var(--line);padding-bottom:18px;margin-bottom:28px;">
  <span style="font-family:var(--mono);font-weight:600;font-size:20px;">re<span style="color:var(--accent)">·</span>con</span>
  <span style="color:var(--ink-faint);font-family:var(--mono);font-size:12px;">settlement reconciliation engine</span>
</header>

<div style="display:grid;grid-template-columns:1fr 1fr;gap:16px;margin-bottom:28px;">
  <!-- Panel A -> POST /api/synthetic -->
  <div class="card">
    <h2>A · synthetic run</h2>
    <div class="row">
      <div class="field"><label>transactions</label>
        <input type="number" id="synCount" value="1000" min="1" max="1000000" /></div>
      <div class="field"><label>seed</label>
        <input type="number" id="synSeed" value="42" /></div>
      <div class="field"><label>tolerance (minor)</label>
        <input type="number" id="synTol" value="3" min="0" /></div>
    </div>
    <button id="runSyn">generate &amp; reconcile -></button>
  </div>

  <!-- Panel B -> POST /api/reconcile -->
  <div class="card">
    <h2>B · reconcile CSV files</h2>
    <div class="field"><label>left source (switch.csv)</label>
      <input type="file" id="fileLeft" accept=".csv" /></div>
    <div class="field"><label>right source (bank.csv)</label>
      <input type="file" id="fileRight" accept=".csv" /></div>
    <div class="row" style="align-items:flex-end;">
      <div class="field" style="margin-bottom:0;"><label>tolerance (minor)</label>
        <input type="number" id="upTol" value="0" min="0" /></div>
      <button id="runUp" style="flex:2;">reconcile files -></button>
    </div>
    <div class="err hidden" id="upErr"></div>
  </div>
</div>
```

Panel A's three inputs are *exactly* `{count, seed, tolerance_minor}` of your `SyntheticRequest`. Panel B's two file inputs are *exactly* the `left` and `right` multipart fields. The UI's shape is dictated by the API's shape — you're mirroring a contract.

## Step 3: the results region

```html
<div id="results" class="hidden">
  <div style="display:grid;grid-template-columns:repeat(4,1fr);gap:14px;margin-bottom:26px;">
    <div class="metric"><div class="label">total matched</div><div class="value" id="mTotal">0</div><div class="sub" id="mRate">—</div></div>
    <div class="metric ok"><div class="label">exact</div><div class="value" id="mExact">0</div></div>
    <div class="metric tol"><div class="label">tolerant</div><div class="value" id="mTol">0</div></div>
    <div class="metric exc"><div class="label">exceptions</div><div class="value" id="mExc">0</div></div>
  </div>

  <div style="margin-bottom:26px;">
    <div class="bar">
      <span class="b-ok"  id="barOk"></span>
      <span class="b-tol" id="barTol"></span>
      <span class="b-exc" id="barExc"></span>
    </div>
  </div>

  <div class="tabs">
    <div class="tab active" data-tab="exc">exceptions <span id="tabExcCount"></span></div>
    <div class="tab" data-tab="match">matches <span id="tabMatchCount"></span></div>
  </div>
  <div class="panel-body" id="panelExc"></div>
  <div class="panel-body hidden" id="panelMatch"></div>
</div>
```

**Exceptions are the default tab, not matches.** An analyst opens this to find the 30 things that broke, not admire the 1,400 that worked. Lead with the problems — the same instinct as Module 3's "silence is the expensive failure," expressed in layout.

## Step 4: the logic — fetch, then paint

```javascript
const $ = id => document.getElementById(id);

$("runSyn").addEventListener("click", async () => {
  $("runSyn").disabled = true;
  try {
    const res = await fetch("/api/synthetic", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        count: parseInt($("synCount").value) || 1000,
        seed: parseInt($("synSeed").value) || 42,
        tolerance_minor: parseInt($("synTol").value) || 0,
      }),
    });
    render(await res.json());
  } catch (e) {
    console.error(e);
  }
  $("runSyn").disabled = false;
});
```

`JSON.stringify({ count, seed, tolerance_minor })` is the JavaScript twin of your `SyntheticRequest` struct. You serialize here, serde deserializes there. JS `async`/`await` is where C#'s syntax was *borrowed from*, so this feels like home. The upload handler uses `FormData` instead of JSON:

```javascript
$("runUp").addEventListener("click", async () => {
  const l = $("fileLeft").files[0], r = $("fileRight").files[0];
  const err = $("upErr");
  err.classList.add("hidden");
  if (!l || !r) { err.textContent = "select both files"; err.classList.remove("hidden"); return; }

  const fd = new FormData();
  fd.append("left", l);
  fd.append("right", r);
  fd.append("tolerance_minor", $("upTol").value || "0");

  $("runUp").disabled = true;
  try {
    const res = await fetch("/api/reconcile", { method: "POST", body: fd });
    if (!res.ok) { throw new Error(await res.text()); }  // surfaces your 400/422
    render(await res.json());
  } catch (e) {
    err.textContent = e.message; err.classList.remove("hidden");
  }
  $("runUp").disabled = false;
});
```

The `FormData` fields — `"left"`, `"right"`, `"tolerance_minor"` — are *exactly* the strings `reconcile_upload` matches. And `if (!res.ok) throw new Error(await res.text())` catches your 422 with "row 412: bad currency" and shows the analyst that exact message. The audit-trail line number from Module 4 completes its journey: disk -> `IngestError` -> HTTP 422 -> a red message on screen.

## Step 5: the render functions

`render` trusts the server's `summary` completely — it never recomputes a total:

```javascript
let current = null;

function render(data) {
  current = data;
  $("results").classList.remove("hidden");
  const s = data.summary;

  $("mTotal").textContent = s.matched_total.toLocaleString();
  $("mExact").textContent = s.matched_exact.toLocaleString();
  $("mTol").textContent   = s.matched_tolerant.toLocaleString();
  $("mExc").textContent   = s.exceptions_total.toLocaleString();

  const denom = s.matched_total + s.exceptions_total;
  $("mRate").textContent = (denom ? (100*s.matched_total/denom).toFixed(1) : "0.0") + "% match rate";

  const total = Math.max(1, s.matched_exact + s.matched_tolerant + s.exceptions_total);
  $("barOk").style.width  = (100*s.matched_exact/total) + "%";
  $("barTol").style.width = (100*s.matched_tolerant/total) + "%";
  $("barExc").style.width = (100*s.exceptions_total/total) + "%";

  $("tabExcCount").textContent   = "(" + s.exceptions_total.toLocaleString() + ")";
  $("tabMatchCount").textContent = "(" + s.matched_total.toLocaleString() + ")";

  renderExceptions(data.exceptions);
  renderMatches(data.matches);
}
```

The subtle, important bit: the bar's widths come from the **summary** (the full honest counts), not from `data.matches.length` (capped at 500). If you'd drawn the bar from the truncated arrays, a million-row run would render a *lie*. Because you separated honest-summary from capped-detail in Module 6, the visualization stays truthful for free.

```javascript
function renderExceptions(exc) {
  const el = $("panelExc");
  if (!exc.length) { el.innerHTML = '<div class="empty">no exceptions — every record reconciled</div>'; return; }
  const rows = exc.map(e => {
    let detail = "";
    if (e.reason === "unmatched")            detail = `txn <b>${e.txn}</b> · source ${e.source}`;
    else if (e.reason === "amount_mismatch") detail = `<b>${e.left}</b> &harr; <b>${e.right}</b> · &Delta; ${(e.diff_minor/100).toFixed(2)}`;
    else if (e.reason === "duplicate_key")   detail = `key <b>${e.key}</b> · ${e.txns.length} txns`;
    else if (e.reason === "no_match_key")    detail = `txn <b>${e.txn}</b> · source ${e.source}`;
    return `<tr><td><span class="pill ${e.reason}">${e.reason.replace(/_/g,' ')}</span></td><td>${detail}</td></tr>`;
  }).join("");
  el.innerHTML = `<table><thead><tr><th>reason</th><th>detail</th></tr></thead><tbody>${rows}</tbody></table>`;
}

function renderMatches(matches) {
  const el = $("panelMatch");
  if (!matches.length) { el.innerHTML = '<div class="empty">no matches</div>'; return; }
  const rows = matches.map(m =>
    `<tr><td><span class="pill ${m.kind}">${m.kind}</span></td><td><b>${m.left}</b></td><td><b>${m.right}</b></td><td>${m.amount_diff_minor===0?"—":(m.amount_diff_minor/100).toFixed(2)}</td></tr>`
  ).join("");
  el.innerHTML = `<table><thead><tr><th>kind</th><th>left</th><th>right</th><th>&Delta; amount</th></tr></thead><tbody>${rows}</tbody></table>`;
}
```

That `if (e.reason === ...)` chain is the JavaScript echo of a Rust `match` on your `Exception` enum — each variant's distinct fields (`e.diff_minor` only on `amount_mismatch`, `e.txns` only on `duplicate_key`) come straight from `#[serde(tag = "reason")]`. One caveat: JS has no compiler forcing you to handle every variant, so if you add a fifth exception type in Rust, *this* code won't warn you — it'll silently render a blank row. **That's precisely the safety you lose crossing from Rust into JavaScript.** And `(e.diff_minor/100).toFixed(2)` — dividing minor units by 100 for *display only*. The integer crossed the wire intact; the browser splits it purely to show a human. The Module 1 philosophy, honored to the last.

## Step 6: tabs and remaining CSS

```javascript
document.querySelectorAll(".tab").forEach(t => t.addEventListener("click", () => {
  document.querySelectorAll(".tab").forEach(x => x.classList.remove("active"));
  t.classList.add("active");
  $("panelExc").classList.toggle("hidden", t.dataset.tab !== "exc");
  $("panelMatch").classList.toggle("hidden", t.dataset.tab !== "match");
}));
```

```css
.card { background: var(--panel); border: 1px solid var(--line); border-radius: 10px; padding: 20px; }
.card h2 { font-family: var(--mono); font-size: 11px; letter-spacing: 1.5px; text-transform: uppercase; color: var(--ink-dim); margin-bottom: 16px; }
.row { display: flex; gap: 12px; } .row > * { flex: 1; }
.field { margin-bottom: 14px; }
.field label { display: block; font-size: 11px; color: var(--ink-faint); font-family: var(--mono); margin-bottom: 6px; text-transform: uppercase; }
input { width: 100%; background: var(--bg); border: 1px solid var(--line); color: var(--ink); padding: 9px 11px; border-radius: 6px; font-family: var(--mono); font-size: 13px; }
input:focus { outline: none; border-color: var(--accent); }
button { background: var(--accent); color: #07210f; border: none; border-radius: 6px; padding: 10px 18px; font-family: var(--mono); font-weight: 600; cursor: pointer; width: 100%; }
button:disabled { opacity: .5; cursor: not-allowed; }
.metric { background: var(--panel); border: 1px solid var(--line); border-radius: 10px; padding: 18px; border-left: 3px solid var(--ink-faint); }
.metric.ok { border-left-color: var(--accent); } .metric.tol { border-left-color: var(--blue); } .metric.exc { border-left-color: var(--bad); }
.metric .label { font-family: var(--mono); font-size: 10px; letter-spacing: 1px; text-transform: uppercase; color: var(--ink-faint); margin-bottom: 10px; }
.metric .value { font-family: var(--mono); font-size: 30px; font-weight: 600; }
.metric .sub { font-size: 11px; color: var(--ink-dim); margin-top: 8px; font-family: var(--mono); }
.bar { display: flex; height: 10px; border-radius: 5px; overflow: hidden; border: 1px solid var(--line); }
.bar span { transition: width .5s ease; } .b-ok { background: var(--accent); } .b-tol { background: var(--blue); } .b-exc { background: var(--bad); }
.tabs { display: flex; gap: 4px; border-bottom: 1px solid var(--line); }
.tab { font-family: var(--mono); font-size: 12px; padding: 10px 16px; cursor: pointer; color: var(--ink-faint); border-bottom: 2px solid transparent; margin-bottom: -1px; }
.tab.active { color: var(--ink); border-bottom-color: var(--accent); }
table { width: 100%; border-collapse: collapse; font-family: var(--mono); font-size: 12.5px; }
th { text-align: left; padding: 11px 12px; color: var(--ink-faint); font-size: 10px; letter-spacing: 1px; text-transform: uppercase; border-bottom: 1px solid var(--line); }
td { padding: 9px 12px; border-bottom: 1px solid var(--panel-2); color: var(--ink-dim); }
tr:hover td { background: var(--panel); color: var(--ink); }
.pill { padding: 2px 8px; border-radius: 20px; font-size: 10px; text-transform: uppercase; }
.pill.exact { background: rgba(74,222,128,.14); color: var(--accent); } .pill.tolerant { background: rgba(96,165,250,.14); color: var(--blue); }
.pill.unmatched, .pill.duplicate_key { background: rgba(248,113,113,.14); color: var(--bad); }
.pill.amount_mismatch { background: rgba(251,191,36,.16); color: var(--warn); } .pill.no_match_key { background: rgba(138,147,163,.16); color: var(--ink-dim); }
.panel-body { max-height: 460px; overflow-y: auto; border: 1px solid var(--line); border-top: none; }
.empty { padding: 50px; text-align: center; color: var(--ink-faint); font-family: var(--mono); }
.hidden { display: none; } .err { color: var(--bad); font-family: var(--mono); font-size: 12px; margin-top: 10px; }
```

## Checkpoint — the whole machine, live

Because you `include_str!`'d this file, rebuild so the new HTML is baked in:

```powershell
cargo run -p recon-api
```

Open `http://localhost:3000`. Then: (1) hit **generate & reconcile** with 2,000 transactions — watch the metrics populate, the bar animate to ~70/15/7, the exceptions tab fill. (2) Make two tiny CSVs by hand and upload them in Panel B. (3) Upload only the left file — watch your **422/400 message** appear in red.

**The final experiment:** open dev tools, Network tab, run a synthetic batch of `1000000`. The `summary` says a million, but the `matches` array holds 500 and `truncated` is `true`. The bar and metrics are still *correct* — because they read the summary, not the array. You're watching a decision you made on the server in Module 6 keep the UI honest in Module 7.

---

# The through-line

Correctness pushed down into the lowest layer that can own it, so every layer above inherits it for free:

- Integer money in **Module 1** made the matcher exact in **Module 3**.
- The audit trail in **Module 2** became a real error message in **Module 7**.
- The layering the compiler enforced in **Module 4** is why none of this tangled.
- The honest-summary decision in **Module 6** kept the visualization truthful in **Module 7**.

You started with a `Currency` newtype and finished with a reconciliation engine running end to end in a browser — and you can explain *why* every piece is shaped the way it is, which is the thing that actually makes it yours.

## Where to go next

**The roadmap** (each a clean addition because of how this is structured):

1. Scheme-specific `Source` parsers: Visa BASE II/VSS, Mastercard IPM, Meeza.
2. Many-to-one matching (partial captures, split settlements).
3. Settlement layer: compute merchant payouts after interchange/fees/commission.
4. Persistence (Postgres) + an exception-resolution workflow with audit trail.
5. Per-source column mapping so any CSV layout works without code changes.

**Or the move I'd suggest first:** rebuild one module from memory, without looking. Pick Module 3, the matcher. If you can reconstruct it cold — the bucketing, the routing, the exception variants — you'll know it's truly yours and not just typed.
