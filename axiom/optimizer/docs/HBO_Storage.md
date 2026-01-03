# Axiom HBO Storage: Three Separate Caches

Axiom maintains three distinct caches for history-based optimization, each with different key formats and purposes.

## 1. Filter Selectivity Cache

**Purpose**: Track what fraction of rows pass filter predicates on base tables.

**Key**: `HiveTableHandle::toString()` - includes the full table handle with filters

The table handle contains:
- Connector ID (e.g., `"hive"`)
- Table name (e.g., `"orders"`)
- Subfield filters (column-level filters like `total > 1000`)
- Remaining filter (complex expressions that couldn't be pushed down)
- Data columns being read
- SerDe parameters

**Value**: `float` between 0.0 and 1.0

**Example keys** (conceptual):
```
"hive.default.orders[total>1000]" → 0.15
"hive.default.orders[status='shipped']" → 0.32
"hive.default.orders[total>1000 AND status='shipped']" → 0.05
"hive.default.orders" → 1.0  (no filters)
```

**Storage**:
```cpp
folly::F14FastMap<std::string, float> leafSelectivities_;
```

---

## 2. Join Sample Cache

**Purpose**: Track empirical join fanout between base table pairs.

**Key**: Canonical `"table1 col1 col2   table2 col1 col2 "` string

Key generation:
```cpp
// QueryGraph.cpp:299-310
auto left = fmt::format("{} ", leftTable_->schemaTable->name());
auto right = fmt::format("{} ", rightTable_->schemaTable->name());

// Sort keys alphabetically for determinism
for (auto i : sortedIndices) {
  left += leftKeys_[i]->toString() + " ";
  right += rightKeys_[i]->toString() + " ";
}

// Lexicographic ordering - smaller string first
if (left < right) {
  return {left + "  " + right, false};  // false = not swapped
}
return {right + "  " + left, true};     // true = swapped
```

**Value**: `pair<float, float>` representing `(lr_fanout, rl_fanout)`
- `lr_fanout`: Average right rows matched per left row
- `rl_fanout`: Average left rows matched per right row

**Example keys**:
```
"customers id   orders cust_id " → (10.5, 0.95)
"lineitem l_orderkey   orders o_orderkey " → (1.0, 4.2)
```

**Storage**:
```cpp
folly::F14FastMap<std::string, std::pair<float, float>> joinSamples_;
```

**Important**: Join sample keys intentionally **exclude filters**. The filter effect is multiplied separately:
```cpp
lrFanout_ = samplePair.second * baseSelectivity(rightTable_);
rlFanout_ = samplePair.first * baseSelectivity(leftTable_);
```

---

## 3. Plan Node History Cache

**Purpose**: Track actual cardinalities for executed plan subtrees.

**Key**: `historyKey()` - recursive canonical plan representation

Key generation varies by node type:

**TableScan**:
```cpp
// RelationOp.cpp:289-314
out << "scan " << baseTable->schemaTable->name() << "(";
for (auto& key : keys) {
  out << "lookup " << key->toString() << ", ";
}
// Filters ARE included (sorted for determinism)
std::ranges::sort(filters);
for (auto& f : filters) {
  out << "f: " << f << ", ";
}
out << ")";
```

**Join** (recursive):
```cpp
// RelationOp.cpp:446-464
auto& leftTree = input_->historyKey();
auto& rightTree = right->historyKey();

// Canonicalize by ordering subtrees deterministically
if (leftTree < rightTree || joinType != kInner) {
  out << "join " << joinTypeLabel(joinType) << "("
      << leftTree << " keys " << leftText << " = " << rightText
      << rightTree << ")";
} else {
  // Swap for inner joins if right < left lexicographically
  out << "join " << joinTypeLabel(reverseJoinType(joinType)) << "("
      << rightTree << " keys " << rightText << " = " << leftText
      << leftTree << ")";
}
```

**Value**: `NodePrediction` struct
```cpp
struct NodePrediction {
  float cardinality;
  float peakMemory{0};
  float cpu{0};
};
```

**Example keys**:
```
"scan orders(f: status = 'shipped', f: total > 1000, )" → {card: 15000}
"join inner(scan customers() keys id = cust_id scan orders(f: total > 1000, ))" → {card: 8500}
```

**Storage**:
```cpp
folly::F14FastMap<std::string, NodePrediction> planHistory_;
```

---

## Summary: What's Canonicalized Where

| Cache | Includes Filters? | Includes Table Names? | Includes Columns? |
|-------|------------------|----------------------|-------------------|
| **Filter Selectivity** | ✅ Yes (full predicate) | ✅ Yes | ✅ Yes (filtered cols) |
| **Join Samples** | ❌ No | ✅ Yes | ✅ Yes (join keys only) |
| **Plan History** | ✅ Yes (in scan nodes) | ✅ Yes | ✅ Yes |

---

## Serialization Format

```json
{
  "leaves": [
    {"key": "hive.orders[total>1000]", "value": 0.15},
    {"key": "hive.orders[status='shipped']", "value": 0.32}
  ],
  "joins": [
    {"key": "customers id   orders cust_id ", "lr": 10.5, "rl": 0.95},
    {"key": "lineitem l_orderkey   orders o_orderkey ", "lr": 1.0, "rl": 4.2}
  ],
  "plans": [
    {"key": "scan orders(f: total > 1000, )", "card": 15000.0},
    {"key": "join inner(scan customers()...)", "card": 8500.0}
  ]
}
```

---

## How the Caches Combine

Final cardinality estimation:

```
Cardinality(A ⋈ B with filters) ≈
    Cardinality(A) × filterSelectivity(A) × joinFanout(A→B) × filterSelectivity(B)
```

This assumes independence between:
- Filter selectivity and join fanout
- Filter columns and join key columns

When this assumption breaks (correlated data), estimates may be inaccurate.

---

## Source File References

| Component | File | Lines |
|-----------|------|-------|
| Filter selectivity storage | `Cost.h` | 88-90 |
| Join sample storage | `VeloxHistory.h` | 56 |
| Plan history storage | `VeloxHistory.h` | 57 |
| Join sample key generation | `QueryGraph.cpp` | 283-311 |
| TableScan history key | `RelationOp.cpp` | 289-314 |
| Join history key | `RelationOp.cpp` | 446-464 |
| Serialization | `VeloxHistory.cpp` | 259-305 |
| Filter selectivity sampling | `VeloxHistory.cpp` | 85-144 |
| Join sampling | `JoinSample.cpp` | 211-256 |
