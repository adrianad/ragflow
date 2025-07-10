# GraphRAG Data Loss Analysis - Root Cause Identified

## Problem Summary
RAGFlow's knowledge graph system experiences significant data loss during multi-document processing. Users report seeing only a fraction of expected nodes/edges after processing multiple documents simultaneously.

## Root Cause: Delete-Rebuild Race Condition

### The Issue
The fundamental problem is in the `set_graph()` function's **delete-all-then-rebuild** pattern combined with a race condition in the graph merging process.

### Detailed Race Condition Flow

1. **Process A** builds up graph to substantial size (e.g., 18 nodes, 10 edges, 5 sources)
2. **Process B** starts `merge_subgraph()` for a new document
3. **Process B** calls `get_graph()` and successfully reads the large graph (18 nodes)
4. **⚠️ CRITICAL WINDOW:** Between Process B's `get_graph()` call and actual merge execution
5. **Process C** (another document) acquires the Redis lock and:
   - Calls `set_graph()` 
   - **DELETES ALL GRAPH DATA** (`settings.docStoreConn.delete({"knowledge_graph_kwd": ["graph", "subgraph"]}, ...)`)
   - Rebuilds with only its own data (e.g., 5 nodes, 3 edges, 1 source)
6. **Process B** continues with merge, but now `get_graph()` returns the smaller graph
7. **Result:** Final graph contains only recent documents instead of accumulated data

### Evidence from Logs

**Before race condition:**
```
set_graph FINAL STATE: Graph should now have 18 nodes, 10 edges from 5 sources
```

**During race condition (Process B gets smaller graph):**
```
get_graph SUCCESS: Parsed graph with 5 nodes, 3 edges
merge_subgraph EXISTING GRAPH: Found existing graph with 5 nodes, 3 edges from 1 sources
merge_subgraph EXISTING SOURCES: ['6aa092f45dd211f08185d6128b385681']
```

**After race condition:**
```
set_graph FINAL STATE: Graph should now have 8 nodes, 5 edges from 2 sources
```

**Data loss:** 18 → 8 nodes (10 nodes lost), 5 → 2 sources (3 sources lost)

## Why Redis Locks Don't Prevent This

The Redis distributed locks work correctly and prevent corruption, but they **cannot prevent this race condition** because:

1. **Lock Scope**: Locks only protect the `set_graph()` operation itself
2. **Read-Modify Gap**: The race occurs between `get_graph()` (read) and the actual merge execution
3. **Lock Timing**: Process B reads the graph **before** acquiring the lock for its own `set_graph()` call

### Lock Flow (Working as Intended)
```
Process B: get_graph() -> reads large graph (unlocked operation)
Process C: acquires lock -> deletes all data -> rebuilds with small graph -> releases lock  
Process B: acquires lock -> merges with now-small graph -> set_graph() -> releases lock
```

## Technical Details

### Key Functions Involved

1. **`set_graph()`** - Always deletes ALL graph data before rebuilding:
   ```python
   # DELETE PHASE: This deletes ALL existing graph and subgraph data
   await settings.docStoreConn.delete({"knowledge_graph_kwd": ["graph", "subgraph"]}, ...)
   ```

2. **`get_graph()`** - Reads current graph state (not lock-protected):
   ```python
   # Called during merge_subgraph() before lock acquisition
   old_graph = await get_graph(tenant_id, kb_id, subgraph.graph["source_id"])
   ```

3. **`merge_subgraph()`** - Merges new document with existing graph:
   ```python
   # Race condition window exists here:
   old_graph = await get_graph(...)  # ← Reads graph state
   # ... other processing ...
   new_graph = graph_merge(old_graph, subgraph, change)  # ← May use stale data
   await set_graph(...)  # ← Lock acquired here, but too late
   ```

### Files Involved
- `/ragflow/graphrag/utils.py` - Contains `set_graph()`, `get_graph()`, `graph_merge()`
- `/ragflow/graphrag/general/index.py` - Contains `merge_subgraph()`, `run_graphrag()`
- `/ragflow/rag/utils/redis_conn.py` - Redis lock implementation (working correctly)

## Impact Assessment

### Data Loss Patterns
- **Partial Loss**: Graph size randomly fluctuates during processing
- **Complete Loss**: Graph may reset to just the most recent document(s)  
- **Source Loss**: Number of source documents decreases unexpectedly
- **Non-Deterministic**: Same input produces different graph sizes

### User-Visible Symptoms
- "Graph is still deleting halfway through"
- "Not all nodes and edges are there in the end"
- Graph shows only 3 nodes instead of expected 50+
- Knowledge graph appears to randomly reset during multi-document processing

## Solution Requirements

The fix must address the fundamental architectural issue while maintaining performance:

1. **Eliminate delete-all pattern** in `set_graph()` 
2. **Implement incremental updates** instead of full rebuilds
3. **Ensure atomic read-modify-write** operations
4. **Maintain consistency** across concurrent document processing

### Potential Solutions

1. **Incremental Updates**: Modify `set_graph()` to update only changed nodes/edges
2. **Copy-on-Write**: Create new graph version, then atomically swap
3. **Lock Scope Expansion**: Extend lock to cover `get_graph()` + merge + `set_graph()`
4. **Document Queuing**: Serialize document processing to eliminate concurrency

## Testing Methodology

To reproduce the issue:
1. Upload 10+ documents simultaneously to trigger concurrent processing
2. Monitor logs for `get_graph()` and `set_graph()` operations
3. Look for decreasing node/edge counts between processes
4. Verify final graph size vs expected cumulative size

## Historical Context

- **Original Issue**: Redis lock `blocking_timeout=1` causing immediate failures
- **First Fix**: Fixed Redis lock parameters and acquisition logic  
- **Current Issue**: Delete-rebuild race condition (architectural)
- **Status**: Root cause identified, solution design needed

## Logging Added for Diagnosis

Comprehensive logging was added to track the issue:
- `set_graph()`: Node/edge counts during delete-rebuild phases
- `merge_subgraph()`: Graph sizes during merging operations  
- `run_graphrag()`: Lock acquisition/release timing
- `graph_merge()`: Detailed merge statistics
- `get_graph()`: What data is actually retrieved from storage

This logging successfully captured the race condition in action and confirmed the root cause.