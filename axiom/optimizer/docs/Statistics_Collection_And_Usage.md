# Axiom Statistics Collection and Usage

This document describes how Axiom collects cardinality statistics and uses them during query optimization. This is intended as a guide for implementing similar functionality in other systems (e.g., Presto with a native sidecar process).

## Overview

Axiom collects two types of statistics via sampling:
1. **Join fanout** - empirical measurement of how many rows match between table pairs
2. **Filter selectivity** - fraction of rows passing filter predicates

These are collected **during planning** (before query execution) and cached for reuse.

---

## 1. Join Fanout Sampling

### When It's Triggered

Join sampling happens once per JoinEdge during optimizer initialization:

```cpp
// Optimization.cpp:51-57
Optimization::Optimization(...) {
  root_ = toGraph_.makeQueryGraph(*logicalPlan_);
  root_->distributeConjuncts();
  root_->addImpliedJoins();
  root_->linkTablesToJoins();
  for (auto* join : root_->joins) {
    join->guessFanout();  // Triggers sampling for each join edge
  }
}
```

### The Sampling Algorithm

**Step 1: Check cache first**

```cpp
// VeloxHistory.cpp:32-52
std::pair<float, float> VeloxHistory::sampleJoin(JoinEdge* edge) {
  auto keyPair = edge->sampleKey();  // Generate canonical key

  {
    std::lock_guard<std::mutex> l(mutex_);
    auto it = joinSamples_.find(keyPair.first);
    if (it != joinSamples_.end()) {
      return it->second;  // Cache hit - return immediately
    }
  }

  // Cache miss - execute sample query...
}
```

**Step 2: Determine sample size**

```cpp
// JoinSample.cpp:211-235
std::pair<float, float> sampleJoin(
    SchemaTableCP left, const ExprVector& leftKeys,
    SchemaTableCP right, const ExprVector& rightKeys) {

  const auto leftRows = left->cardinality;
  const auto rightRows = right->cardinality;
  const auto leftCard = keyCardinality(leftKeys);   // Product of key cardinalities
  const auto rightCard = keyCardinality(rightKeys);

  static const auto kMaxCardinality = 10'000;  // Target sample size

  int32_t fraction = kMaxCardinality;

  if (leftRows < kMaxCardinality && rightRows < kMaxCardinality) {
    // Small tables: sample everything
    fraction = kMaxCardinality;
  } else if (leftCard > kMaxCardinality && rightCard > kMaxCardinality) {
    // Large tables with high-cardinality keys: sample a fraction
    const auto smaller = std::min(leftRows, rightRows);
    const float ratio = smaller / kMaxCardinality;
    fraction = std::max(2, kMaxCardinality / ratio);
  } else {
    // Low-cardinality keys: skip sampling (not enough diversity)
    return {0, 0};
  }
}
```

**Step 3: Build and execute sample queries**

```cpp
// JoinSample.cpp:100-158
std::shared_ptr<Runner> prepareSampleRunner(
    SchemaTableCP table, const ExprVector& keys, int64_t mod, int64_t lim) {

  // 1. Create base table scan
  auto base = make<BaseTable>();
  base->schemaTable = table;
  auto* scan = make<TableScan>(base, index, columns);

  // 2. Hash the join keys
  ExprVector hashes;
  for (const auto& key : keys) {
    hashes.push_back(makeCall("$internal$hash", BIGINT(), key));
  }
  ExprCP hash = makeCall("$internal$hash_mix", BIGINT(), hashes...);

  // 3. Apply deterministic sample filter: (hash % mod) < lim
  ExprCP filterExpr = makeCall(
      "$internal$sample", BOOLEAN(), hashColumn, bigintLit(mod), bigintLit(lim));

  // 4. Execute via LocalRunner
  auto plan = queryCtx()->optimization()->toVeloxPlan(filter);
  return std::make_shared<runner::LocalRunner>(plan, ...);
}
```

**Step 4: Build frequency maps and compute fanout**

```cpp
// JoinSample.cpp:163-200
// Run sample query and build hash -> frequency map
std::unique_ptr<KeyFreq> runJoinSample(Runner& runner) {
  auto result = std::make_unique<F14FastMap<uint32_t, uint32_t>>();

  while (auto rows = runner.next()) {
    auto hashes = rows->childAt(0)->as<FlatVector<int64_t>>();
    for (auto i = 0; i < hashes->size(); ++i) {
      if (!hashes->isNullAt(i)) {
        ++(*result)[static_cast<uint32_t>(hashes->valueAt(i))];
      }
    }
  }
  return result;
}

// Compute average matches per key
float freqs(const KeyFreq& left, const KeyFreq& right) {
  float hits = 0;
  for (const auto& [hash, _] : left) {
    auto it = right.find(hash);
    if (it != right.end()) {
      hits += it->second;  // Count matching rows in right
    }
  }
  return hits / left.size();  // Average matches per left key
}
```

**Step 5: Return bidirectional fanouts**

```cpp
// JoinSample.cpp:254-255
return std::make_pair(
    freqs(*rightFreq, *leftFreq),  // Right-to-left fanout
    freqs(*leftFreq, *rightFreq)); // Left-to-right fanout
```

### Sample Query Shape

The sample query executed is equivalent to:

```sql
SELECT hash_mix(hash(key1), hash(key2), ...) as h
FROM table
WHERE (hash_mix(...) % 10000) < fraction
```

This produces a deterministic sample of ~10K rows (or fewer for small tables).

---

## 2. Filter Selectivity Sampling

### When It's Triggered

Filter selectivity is sampled when building the table handle for a scan:

```cpp
// ToVelox.cpp:193-196
setLeafHandle(table->id(), handle, std::move(rejectedFilters));
if (updateSelectivity) {
  optimization->setLeafSelectivity(*const_cast<BaseTable*>(table), scanType);
}
```

### The Sampling Algorithm

```cpp
// VeloxHistory.cpp:85-144
bool VeloxHistory::setLeafSelectivity(BaseTable& table, const RowTypePtr& scanType) {
  auto [tableHandle, filters] = queryCtx()->optimization()->leafHandle(table.id());
  const auto string = tableHandle->toString();  // Key includes filters

  // Check cache
  auto it = leafSelectivities_.find(string);
  if (it != leafSelectivities_.end()) {
    table.filterSelectivity = it->second;
    return true;
  }

  // Execute sample via connector's layout
  auto sample = runnerTable->layouts()[0]->sample(
      tableHandle,
      1,        // 1% sample
      filters,
      scanType);

  // sample.first = rows scanned, sample.second = rows passing filters
  table.filterSelectivity =
      std::max(0.9f, sample.second) / static_cast<float>(sample.first);

  recordLeafSelectivity(string, table.filterSelectivity, false);
  return true;
}
```

### Connector-Level Sampling

The actual sampling is delegated to the connector's TableLayout:

```cpp
// LocalHiveConnectorMetadata.cpp:256-334
std::pair<int64_t, int64_t> LocalHiveTableLayout::sample(
    const ConnectorTableHandlePtr& tableHandle,
    float pct,
    RowTypePtr scanType, ...) {

  const auto maxRowsToScan = table().numRows() * (pct / 100);

  int64_t passingRows = 0;
  int64_t scannedRows = 0;

  for (const auto& file : files_) {
    auto dataSource = connector()->createDataSource(...);
    dataSource->addSplit(split);

    for (;;) {
      auto data = dataSource->next(kBatchSize, ignore).value();
      if (data == nullptr) break;

      passingRows += data->size();  // Rows passing filters

      if (scannedRows > maxRowsToScan) break;  // Early termination
    }
  }

  return {scannedRows, passingRows};
}
```

---

## 3. How Statistics Are Used in the Optimizer

### Join Fanout Usage

```cpp
// QueryGraph.cpp:493-529
void JoinEdge::guessFanout() {
  auto samplePair = opt->history().sampleJoin(this);
  auto left = joinCardinality(leftTable_, leftKeys_);
  auto right = joinCardinality(rightTable_, rightKeys_);

  if (samplePair.first == 0 && samplePair.second == 0) {
    // No sample available: use statistical estimation
    lrFanout_ = right.joinCardinality * baseSelectivity(rightTable_);
    rlFanout_ = left.joinCardinality * baseSelectivity(leftTable_);
  } else {
    // Sample available: use empirical fanout × filter selectivity
    lrFanout_ = samplePair.second * baseSelectivity(rightTable_);
    rlFanout_ = samplePair.first * baseSelectivity(leftTable_);
  }

  // Special case: unique keys (FK→PK joins)
  if (rightUnique_) {
    lrFanout_ = baseSelectivity(rightTable_);
    rlFanout_ = tableCardinality(leftTable_) / tableCardinality(rightTable_)
                * baseSelectivity(leftTable_);
  }
}
```

### Join Ordering

The fanouts drive join ordering decisions:

```cpp
// Optimization.cpp:372-418
std::vector<JoinCandidate> Optimization::nextJoins(PlanState& state) {
  std::vector<JoinCandidate> candidates;

  forJoinedTables(state, [&](JoinEdgeP join, PlanObjectCP joined, float fanout) {
    candidates.emplace_back(join, joined, fanout);
  });

  // Sort by fanout - lowest first (greedy algorithm)
  std::ranges::sort(candidates, [](auto& left, auto& right) {
    return left.fanout < right.fanout;
  });

  return candidates;
}
```

### Join Cost Calculation

```cpp
// RelationOp.cpp:388-420
Join::Join(..., float fanout, ...) {
  cost_.fanout = fanout;

  const float buildSize = right->resultCardinality();
  const auto numKeys = leftKeys.size();

  // Probe cost: hash table access + key comparisons
  const auto probeCost = Costs::hashTableCost(buildSize) +
      (Costs::kKeyCompareCost * numKeys * std::min<float>(1, fanout)) +
      numKeys * Costs::kHashColumnCost;

  // Row retrieval cost
  const auto rowBytes = byteSize(right->input()->columns());
  const auto rowCost = Costs::hashRowCost(buildSize, rowBytes);

  cost_.unitCost = probeCost + cost_.fanout * rowCost;
}
```

---

## 4. Key Assumptions and Limitations

### Independence Assumption

Axiom assumes filter selectivity is independent of join fanout:

```
Effective fanout = (sampled join fanout) × (filter selectivity)
```

This breaks down when:
- Filter columns are correlated with join keys
- Filtered rows have different join characteristics than average

### Base Tables Only

- Join sampling only works for base table pairs
- Derived tables (subqueries, CTEs) fall back to statistical estimation
- Filter selectivity only sampled at leaf scans

### Single-Process Sampling

- All sampling runs on the coordinator process
- Scans all partitions but filters to ~10K rows
- No distributed sampling

---

## 5. Mapping to Presto + Native Sidecar

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Presto Coordinator                        │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐  │
│  │   Parser    │───▶│  Analyzer   │───▶│    Optimizer    │  │
│  └─────────────┘    └─────────────┘    └────────┬────────┘  │
│                                                  │           │
│                                          Need stats?         │
│                                                  │           │
│                                                  ▼           │
│                                        ┌─────────────────┐  │
│                                        │  HBO Provider   │  │
│                                        │  (Redis/Cache)  │  │
│                                        └────────┬────────┘  │
└──────────────────────────────────────────────────┼──────────┘
                                                   │
                                            Cache miss?
                                                   │
                                                   ▼
┌─────────────────────────────────────────────────────────────┐
│                   Native Sidecar Process                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Sample Query Executor                   │    │
│  │                                                      │    │
│  │  1. Receive sample request (table, keys, filters)   │    │
│  │  2. Build sample query with hash-based filter       │    │
│  │  3. Execute against data (Velox engine)             │    │
│  │  4. Build frequency map, compute fanout             │    │
│  │  5. Return statistics to coordinator                │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Sample Query Templates

**Join Fanout Sample:**

```sql
-- Execute on sidecar for each table
SELECT
  hash_combine(hash(key1), hash(key2)) as h,
  count(*) as freq
FROM {table}
WHERE mod(hash_combine(hash(key1), hash(key2)), 10000) < {fraction}
GROUP BY 1
```

Then compute fanout by joining the two frequency maps.

**Filter Selectivity Sample:**

```sql
SELECT
  count(*) as total,
  count_if({filter_predicate}) as passing
FROM {table}
TABLESAMPLE BERNOULLI(1)  -- Or use hash-based sampling
```

### Sidecar API

```protobuf
service StatisticsSampler {
  // Sample join fanout between two tables
  rpc SampleJoinFanout(JoinSampleRequest) returns (JoinSampleResponse);

  // Sample filter selectivity for a table with predicates
  rpc SampleFilterSelectivity(FilterSampleRequest) returns (FilterSampleResponse);
}

message JoinSampleRequest {
  string left_table = 1;
  repeated string left_keys = 2;
  string right_table = 3;
  repeated string right_keys = 4;
  int32 target_sample_size = 5;  // Default 10000
}

message JoinSampleResponse {
  double lr_fanout = 1;  // Left-to-right fanout
  double rl_fanout = 2;  // Right-to-left fanout
  int64 sample_timestamp = 3;
}

message FilterSampleRequest {
  string table = 1;
  string filter_expression = 2;  // SQL predicate
  double sample_percentage = 3;  // Default 1.0
}

message FilterSampleResponse {
  double selectivity = 1;  // 0.0 to 1.0
  int64 rows_sampled = 2;
  int64 rows_passed = 3;
}
```

### Integration with Presto CBO

```java
public class SidecarStatisticsProvider implements HistoryBasedPlanStatisticsProvider {

    private final StatisticsSamplerClient sidecar;
    private final Cache<String, JoinSampleStatistics> joinCache;
    private final Cache<String, Double> filterCache;

    @Override
    public Optional<JoinSampleStatistics> getJoinSample(String joinKey) {
        // Check local cache first
        JoinSampleStatistics cached = joinCache.getIfPresent(joinKey);
        if (cached != null) {
            return Optional.of(cached);
        }

        // Check Redis (shared cache)
        cached = redisProvider.getJoinSample(joinKey);
        if (cached != null) {
            joinCache.put(joinKey, cached);
            return Optional.of(cached);
        }

        // Cache miss - trigger async sampling via sidecar
        // (or block if synchronous sampling is acceptable)
        return Optional.empty();
    }

    public void triggerJoinSampling(
            QualifiedObjectName leftTable, List<String> leftKeys,
            QualifiedObjectName rightTable, List<String> rightKeys) {

        String key = CanonicalJoinKeyGenerator.generateKey(
            leftTable, leftKeys, rightTable, rightKeys).getLeft();

        JoinSampleRequest request = JoinSampleRequest.newBuilder()
            .setLeftTable(leftTable.toString())
            .addAllLeftKeys(leftKeys)
            .setRightTable(rightTable.toString())
            .addAllRightKeys(rightKeys)
            .setTargetSampleSize(10000)
            .build();

        // Execute via sidecar
        JoinSampleResponse response = sidecar.sampleJoinFanout(request);

        JoinSampleStatistics stats = new JoinSampleStatistics(
            response.getLrFanout(),
            response.getRlFanout(),
            response.getSampleTimestamp());

        // Store in Redis and local cache
        redisProvider.putJoinSample(key, stats);
        joinCache.put(key, stats);
    }
}
```

### Sampling Execution Strategies

| Strategy | When to Use | Trade-offs |
|----------|-------------|------------|
| **Synchronous** | First query, planning time acceptable | Adds latency to first query |
| **Async background** | Warm cache proactively | Stats may be stale for first query |
| **On-demand + fallback** | Cache miss uses heuristics | First query may have suboptimal plan |
| **Periodic refresh** | Batch jobs overnight | Good for stable schemas |

### Key Implementation Considerations

1. **Sidecar placement**: Run on nodes with fast access to data (e.g., worker nodes)
2. **Partition sampling**: For large tables, sample across representative partitions
3. **Concurrency**: Handle multiple concurrent sample requests
4. **Timeout handling**: Return partial results or fall back to heuristics
5. **Cache invalidation**: Invalidate when table data changes significantly

---

## 6. Source File References

| Component | File | Lines |
|-----------|------|-------|
| Join sampling trigger | `Optimization.cpp` | 51-57 |
| Join sample cache lookup | `VeloxHistory.cpp` | 32-52 |
| Sample size calculation | `JoinSample.cpp` | 211-235 |
| Sample query construction | `JoinSample.cpp` | 100-158 |
| Frequency map building | `JoinSample.cpp` | 163-200 |
| Fanout usage in optimizer | `QueryGraph.cpp` | 493-529 |
| Join ordering | `Optimization.cpp` | 372-418 |
| Filter selectivity sampling | `VeloxHistory.cpp` | 85-144 |
| Connector-level sampling | `LocalHiveConnectorMetadata.cpp` | 256-334 |
