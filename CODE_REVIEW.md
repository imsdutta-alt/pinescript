# Code Review: Smart Money Concepts [LuxAlgo] (Pine Script v5)

## Summary
The script is feature-rich and organized, but there are a few correctness and robustness issues that can produce misleading signals or unstable behavior on early bars and when arrays are modified during iteration.

## Findings

### 1) `higherTimeframe()` compares chart timeframe to itself (logic bug)
**Severity:** High

```pine
higherTimeframe(string timeframe) => timeframe.in_seconds() > timeframe.in_seconds(timeframe)
```

Because the parameter name (`timeframe`) shadows Pine's built-in `timeframe` namespace, `timeframe.in_seconds()` in this context does **not** refer to the chart timeframe. This comparison can evaluate incorrectly and break MTF level gating (`D/W/M` checks).

**Recommendation:** Rename the parameter (e.g., `tf`) and compare chart TF vs provided TF explicitly:

```pine
higherTimeframe(string tf) => timeframe.in_seconds() > timeframe.in_seconds(tf)
```

---

### 2) Removing elements from arrays while iterating forward can skip items
**Severity:** Medium

In both `deleteOrderBlocks()` and `deleteFairValueGaps()`, elements are removed during a forward `for [index, item] in array` traversal. When an item is removed, subsequent elements shift left and the next element may be skipped.

**Recommendation:** Iterate indexes in reverse when deleting:

```pine
for i = array.size(arr) - 1 to 0
    if shouldDelete
        array.remove(arr, i)
```

---

### 3) `bar_index` used as a divisor without early-bar guard
**Severity:** Medium

In multiple places, cumulative values are divided by `bar_index`:

- `volatilityMeasure = ... ta.cum(ta.tr)/bar_index`
- FVG threshold uses cumulative sum divided by `bar_index`

At the first bar (`bar_index == 0`), this can create invalid values (`na`/`inf`) and pollute downstream logic.

**Recommendation:** Use `math.max(bar_index, 1)` as denominator.

---

### 4) Potential typo in confluence filter math for candle body/wick checks
**Severity:** Low

```pine
bullishBar := high - math.max(close, open) > math.min(close, open - low)
bearishBar := high - math.max(close, open) < math.min(close, open - low)
```

`math.min(close, open - low)` mixes absolute price (`close`) with distance (`open - low`), which is dimensionally inconsistent.

**Recommendation:** Revisit intent; likely one side should be `math.min(close, open) - low`.

---

### 5) `storeOrdeBlock` naming typo reduces readability
**Severity:** Low

Function name appears misspelled (`storeOrdeBlock` vs `storeOrderBlock`). Not a runtime issue, but it hurts maintainability.

## What is working well
- Good modularization into UDTs and helper functions.
- Clear user input grouping/tooltips and strong configurability.
- Reasonable separation between detection, storage, drawing, and alert logic.
- Thoughtful use of lookahead settings and explicit trend-source selection for candles/dashboard.

## Suggested priority
1. Fix `higherTimeframe()` (correctness).
2. Fix reverse-deletion loops (data integrity).
3. Add safe denominator guards (stability).
4. Re-validate confluence filter formula (signal quality).
5. Clean naming typo (maintainability).
