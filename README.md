# MACROS

**MACROS** is an experimental quantitative intraday trading framework combining market structure, hourly price imbalances, statistical regime detection, historical analogue matching, and machine-learning-based feature analysis.

The system is currently under active research and development.

Current research performance objectives:

* **Sharpe Ratio:** 3.0–4.0
* **Minimum Hit-Rate Target:** approximately 60%
* **Stretch Hit-Rate Goal:** approximately 70%
* **Position Horizon:** intraday
* **End-of-Day Exposure:** flat

These figures are research objectives and are not claims of achieved or expected performance.

---

# 1. Structural Foundation

MACROS begins with two deterministic structural components:

1. **Market Structure Engine**
2. **Three-Candle Fair Value Gap / Imbalance Engine**

Both concepts currently exist as TradingView PineScript indicators and will be rewritten into the research environment.

The Python/C++ implementation must preserve the temporal behavior of the PineScript implementations and must not use future information that would not have been available at the original decision timestamp.

```text
MARKET DATA
     │
     ▼
MARKET STRUCTURE ENGINE
HH / HL / LH / LL
     │
     ▼
STRUCTURAL LEVEL
     │
     ▼
THREE-CANDLE FVG
     │
     ▼
Does imbalance form relative
to qualifying market structure?
     │
     ▼
MACROS TRADE CANDIDATE
```

---

# 2. Market Structure Engine

MACROS uses a stateful market-structure algorithm to identify:

```text
HH = Higher High
HL = Higher Low
LH = Lower High
LL = Lower Low
```

The current PineScript implementation uses confirmed pivots with:

```text
leftLen  = 20
rightLen = 2
```

A pivot high is generated conceptually from:

```text
ta.pivothigh(high, 20, 2)
```

and a pivot low from:

```text
ta.pivotlow(low, 20, 2)
```

This is an important temporal constraint.

A pivot occurring at bar \(t\) cannot be considered known at bar \(t\).

Because:

```text
rightLen = 2
```

the structural pivot only becomes confirmed after two subsequent bars have been observed.

Conceptually:

```text
               candidate pivot
                     │
                     ▼

t-20 ............... t ...... t+1 ...... t+2
                              │           │
                              │           ▼
                              │      PIVOT KNOWN
                              │
                        still unconfirmed
```

Therefore:

$$
KnowledgeTime(Pivot_t)=t+2
$$

for the current `rightLen = 2` configuration.

The rewritten research implementation must preserve this confirmation delay.

---

# 3. Structure Initialization

Before MACROS assigns a market trend, the engine waits until both:

* a confirmed pivot high has been observed;
* a confirmed pivot low has been observed.

The chronological order of those first pivots determines the initial structural interpretation.

If the first confirmed high precedes the first confirmed low:

```text
High first
    ↓
   LH
    \
     \
      LL

Initial interpretation:
DOWNTREND
```

The engine initializes:

```text
structLH
structLL
```

with:

```text
lastStructure = "LL"
```

If the first confirmed low precedes the first confirmed high:

```text
      HH
     /
    /
   HL
    ↑
Low first

Initial interpretation:
UPTREND
```

The engine initializes:

```text
structHL
structHH
```

with:

```text
lastStructure = "HH"
```

---

# 4. Uptrend Structure Logic

The market is treated as structurally bullish when:

```text
lastStructure == HH
```

or:

```text
lastStructure == HL
```

The active structural boundaries are:

```text
structHH = current structural high
structHL = current structural low
```

Conceptually:

```text
                     structHH
                        ●
                       / \
                      /   \
                     /     \
                    /       \
                   ●         \
              structHL        \
```

Two important events can occur.

## 4.1 Break Above Structural High

If:

$$
High_t > structHH
$$

a potential new **Higher High** is detected.

The engine does not immediately accept the new HH.

Instead:

```text
pendingType = HH
waitingForConfirmation = true
```

The system waits for a confirmed pivot high.

Once that pivot is confirmed, the structure becomes:

```text
previous HH
      │
      │            NEW HH
      │              ●
      │             / \
      ●            /   \
       \          /
        \        /
         ●──────
       NEW HL
```

The lowest low between the previous HH and the new high sequence is identified as the new:

```text
HL
```

The newly confirmed pivot becomes:

```text
HH
```

The market therefore continues its bullish structure:

$$
HL \rightarrow HH
$$

---

# 5. Bullish-to-Bearish Structure Transition

While in an uptrend, if:

$$
Low_t < structHL
$$

the active bullish structural low has been violated.

The engine enters:

```text
pendingType = LL
waitingForConfirmation = true
```

It waits for a confirmed pivot low.

Once confirmed:

```text
previous HH
     ●
      \
       \
        \
---------X--------- previous HL broken
          \
           \
            ●
           NEW LL
```

The previous structural HH becomes the new:

```text
LH
```

and the confirmed lower pivot becomes:

```text
LL
```

The state changes to:

```text
DOWNTREND
```

---

# 6. Downtrend Structure Logic

The market is treated as structurally bearish when:

```text
lastStructure == LL
```

or:

```text
lastStructure == LH
```

The active structural boundaries are:

```text
structLH = current structural high
structLL = current structural low
```

Conceptually:

```text
         structLH
             ●
              \
               \
                \
                 ●
              structLL
```

## 6.1 Break Below Structural Low

If:

$$
Low_t < structLL
$$

the system detects a potential new Lower Low.

It enters:

```text
pendingType = LL
```

and waits for a confirmed pivot low.

Once confirmed, the highest high between the previous LL and the new LL sequence becomes the new:

```text
LH
```

and the confirmed pivot becomes the new:

```text
LL
```

Conceptually:

```text
          NEW LH
             ●
            / \
           /   \
old LL   ●     \
                \
                 ●
               NEW LL
```

The bearish structure therefore continues:

$$
LH \rightarrow LL
$$

---

# 7. Bearish-to-Bullish Structure Transition

While bearish, if:

$$
High_t > structLH
$$

the active structural high has been violated.

The engine begins waiting for confirmation of a Higher High.

After a valid pivot-high confirmation:

```text
                 NEW HH
                    ●
                   /
                  /
---------X-------/
   previous LH broken
        /
       ●
 previous LL
```

The previous LL becomes the structural:

```text
HL
```

and the confirmed high becomes:

```text
HH
```

The state therefore transitions back into:

```text
UPTREND
```

---

# 8. Stateful Structure Machine

The market-structure component can therefore be represented as a state machine:

```text
                    break HH
          ┌────────────────────────┐
          │                        ▼
      ┌────────┐               ┌────────┐
      │   HH   │◄──────────────│   HL   │
      └────┬───┘               └────────┘
           │
           │ break HL
           ▼
      pending LL
           │
           │ pivot confirmation
           ▼
      ┌────────┐
      │   LL   │
      └────┬───┘
           │
           │ continuation
           ▼
      ┌────────┐
      │   LH   │
      └────┬───┘
           │
           │ break LH
           ▼
      pending HH
           │
           │ pivot confirmation
           └──────────────► HH
```

The production/research rewrite should preserve the state-machine behavior rather than implementing HH/HL/LH/LL as independent labels.

---

# 9. Structural Levels Used by MACROS

For MACROS research, the structure engine exposes two broad categories of levels:

### Structural High

A confirmed structural high may correspond to:

```text
HH
or
LH
```

### Structural Low

A confirmed structural low may correspond to:

```text
HL
or
LL
```

These levels become reference points for determining whether a qualifying FVG has formed in the required structural location.

The exact MACROS entry relationship between these structural levels and the FVG should remain explicit and testable.

Potential event fields include:

```text
structure_state
structure_level_type
structure_level_price
structure_level_timestamp
structure_confirmation_timestamp

previous_hh
previous_hl
previous_lh
previous_ll

fvg_distance_from_structure
structure_age_bars
structure_age_time
structure_swept
```

---

# 10. Fair Value Gap / Imbalance Engine

MACROS uses a three-candle Fair Value Gap.

For:

$$
C_{t-2},C_{t-1},C_t
$$

a bullish FVG exists when:

$$
High_{t-2}<Low_t
$$

and:

$$
Low_t-High_{t-2}>MinimumGap
$$

A bearish FVG exists when:

$$
Low_{t-2}>High_t
$$

and:

$$
Low_{t-2}-High_t>MinimumGap
$$

The existing PineScript optionally incorporates:

* ATR-adjusted minimum gap size
* volume filters
* buy/sell volume approximation
* EMA filtering
* FVG persistence
* FVG fill/invalidation detection

The original PineScript implementation should be treated as the behavioral reference when rewriting this component.

---

# 11. MACROS Structural Entry Event

The important distinction is:

```text
FVG != automatically a MACROS trade
```

Instead:

```text
Market Structure
      +
Qualifying FVG
      =
MACROS Candidate
```

Conceptually:

```text
             STRUCTURAL HIGH
                   ●
───────────────────┼──────────────────
                   │
                   │ price interaction
                   │
                   ▼
              displacement
                   │
              3-candle FVG
                   │
                   ▼
            MACROS CANDIDATE
```

and symmetrically around structural lows.

The event detector must preserve:

1. the structural level available at that timestamp;
2. when that level became known;
3. the FVG formation timestamp;
4. the spatial relationship between FVG and structure;
5. the direction of the setup.

This information must be stored rather than inferred retrospectively.

---

# 12. Persistent Hourly Imbalances

Qualifying imbalances are not necessarily discarded when the hour ends.

A valid imbalance may be carried forward as an active potential entry.

Example:

```text
10:00 FVG ────────────────────────────────►
               still active

11:00 FVG       ──────────────────────────►
                     still active

12:00 FVG               ──────────────────►
```

Each active imbalance should therefore have a lifecycle.

Potential state:

```text
CREATED
   ↓
ACTIVE
   ↓
├── TOUCHED
├── ENTERED
├── INVALIDATED
├── EXPIRED
└── EOD_CLOSED
```

The exact lifecycle rules remain part of the research specification.

---

# 13. Quantitative Context

Once a valid structural MACROS event exists, additional models provide context.

```text
                MACROS EVENT
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
        HMM      Weighted kNN   XGBoost
         │           │           │
         └───────────┼───────────┘
                     ▼
                TRADE / PASS
```

## HMM

30-minute observations estimate latent market regime probabilities.

Initial conceptual states:

$$
Z_t\in\{Bull,Bear,Range\}
$$

## Weighted kNN

Historical analogue analysis uses:

$$
xx{:}50\rightarrow xx{:}10
$$

to compare current top-of-hour behavior against historically similar observations.

## XGBoost

Initial core input feature families include:

* Volatility
* Volume
* Linear-regression-derived characteristics

Potential later features include outputs from:

* Market structure
* FVG engine
* HMM
* Weighted kNN

---

# 14. No-Look-Ahead Requirement

This is a core requirement of MACROS.

The system must distinguish between:

```text
EVENT TIME
```

and:

```text
KNOWLEDGE / CONFIRMATION TIME
```

For example, with:

```text
rightLen = 2
```

a pivot may geometrically occur at:

```text
10:00
```

but not become known until two bars later.

The research system cannot act as though the pivot was known at 10:00.

Instead it should store something equivalent to:

```text
pivot_time          = 10:00
confirmation_time   = 10:02
```

assuming a one-minute structural timeframe.

Any MACROS decision occurring before the confirmation timestamp cannot use that pivot.

The same principle applies throughout:

```text
NO future pivots
NO future normalization
NO future neighbors
NO future HMM observations
NO future regression values
NO future FVG information
NO target leakage
```

---

# 15. Event Dataset Additions

The Phase 1 event dataset should now include explicit market-structure information.

At minimum:

```text
event_id
timestamp
symbol
direction

fvg_upper
fvg_lower
fvg_size
fvg_formed_at

structure_state
structure_level_type
structure_level_price
structure_pivot_time
structure_confirmation_time
structure_age
structure_distance
structure_swept

volatility
volume

linreg_slope
linreg_r2
linreg_residual

window_high
window_low
window_return

entry_price

forward_return
mfe
mae

tp_hit
sl_hit
eod_return
```

The distinction between:

```text
structure_pivot_time
```

and:

```text
structure_confirmation_time
```

is particularly important for validating that the backtest contains no structural look-ahead bias.

---

# 16. Required Structure Tests

The Phase 1 implementation should include dedicated tests for:

* pivot-high detection
* pivot-low detection
* 20-left / 2-right pivot behavior
* delayed pivot confirmation
* initial trend construction
* HH continuation
* HL construction
* LL continuation
* LH construction
* bullish → bearish transition
* bearish → bullish transition
* pending structure confirmation
* structural level timestamps
* structure confirmation timestamps
* FVG relationship to structural highs
* FVG relationship to structural lows
* no-look-ahead behavior

A particularly important test should prove:

> A pivot cannot affect a MACROS event before its `rightLen` confirmation bars have completed.

---

# Research Objective

The structural component should first be reproduced faithfully from the PineScript implementation.

Only after behavioral equivalence has been established should alternative market-structure definitions or parameters be researched.

The initial objective is therefore:

```text
PineScript behavior
        ↓
deterministic reference tests
        ↓
Python/C++ implementation
        ↓
equivalent market-structure stream
        ↓
FVG + structure event dataset
        ↓
baseline MACROS research
```

Do not optimize or simplify the market-structure algorithm during the initial port.

First establish behavioral equivalence.
