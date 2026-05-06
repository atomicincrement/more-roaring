# Roaring Bitmap Research Findings

## Paper Summary
**Title:** "Consistently faster and smaller compressed bitmaps with Roaring"  
**Authors:** Daniel Lemire, Gregory Ssi-Yan-Kai, Owen Kaser  
**Published:** Software: Practice and Experience, Volume 46, Issue 11, November 2016  
**Paper URL:** https://arxiv.org/abs/1603.06549

## Key Insights

### 1. Multi-Level Hierarchical Structure

#### Two-Level Tree Architecture
- **First Level:** 16-bit prefix values (65,536 possible containers)
- **Second Level:** Containers holding 16-bit values with that prefix
- Each container is independently chosen for optimal compression at that density
- Enables adaptive compression based on data distribution

#### Benefits of Hierarchical Design
- Allows different compression strategies for different data regions
- Reduces memory footprint for sparse regions
- Maintains fast access times for dense regions
- Naturally partitions the problem space

### 2. Container Types (Adaptive Encoding)

Three container types are used based on cardinality (number of elements):

#### Bitmap Container
- **When:** Medium to high density (typically 100-40,000 elements)
- **Format:** Standard uncompressed 64KB bitmap (8,192 x 64-bit words)
- **Pros:** Fast operations (bitwise AND/OR/XOR), cache-friendly
- **Cons:** Fixed size overhead, wasteful for sparse data
- **SIMD Opportunity:** Vectorize bitwise operations across 64KB blocks

#### Array Container  
- **When:** Low density (typically 0-4,096 elements)
- **Format:** Sorted array of 16-bit values
- **Pros:** Compact for sparse data, minimal memory overhead
- **Cons:** Slower set operations (requires merge/intersect algorithms)
- **SIMD Opportunity:** Vectorize array element searches, merges with SIMD

#### Run-Length Encoding (RLE) Container
- **When:** Highly compressible patterns (long runs of consecutive values)
- **Format:** Sequence of (value, length) pairs
- **Pros:** Excellent for sorted/patterned data
- **Cons:** Decompression overhead, slower random access
- **SIMD Opportunity:** Vectorized RLE decompression and run scanning

### 3. Container Selection Logic (Density-Dependent)

Thresholds for switching between containers:
- **Array vs Bitmap:** ~4,096 elements (configurable)
- **Bitmap vs RLE:** Depends on run patterns and compression ratio
- Selection happens during construction and optimization passes

**Key Discovery:** Container choice is the primary driver of performance
- Different densities benefit from different strategies
- Hybrid approach beats single-strategy alternatives

### 4. Existing SIMD Opportunities in the Paper

#### Explicitly Mentioned
1. **Set Operations:** Intersection (AND), union (OR), difference (XOR/NOT)
   - Bitmap containers naturally support SIMD bitwise operations
   - Bulk operations on 64KB blocks
   - Modern CPUs: AVX-2 (256-bit) or AVX-512 (512-bit) registers

2. **Cardinality Computation**
   - Popcount operations can be parallelized
   - Modern CPUs have fast POPCNT instruction
   - Multiple popcounts can run in parallel across independent chunks

#### Implicitly Suggested (Not in Paper)
1. **Array Container Operations:**
   - Binary search vectorization using SIMD comparisons
   - Merge/intersect operations with SIMD element processing
   - Bulk value checking against array contents

2. **Container Conversion:**
   - Bitmap-to-array conversion: Find set bits in parallel
   - Array-to-bitmap conversion: Scatter 16-bit indices to bitmap

### 5. Data Structure Layout

#### Memory Organization
```
Root Level (High 16 bits):
  [Container 0] [Container 1] ... [Container 65535]
  
Each Container (for prefix value P):
  - Type flag (Bitmap/Array/RLE)
  - Cardinality metadata
  - Payload (8KB bitmap, or dynamic array, or RLE pairs)
```

#### Cache Efficiency
- Container-level granularity fits L3 cache (8-64MB typically)
- Individual containers usually < 16KB for bitmap type
- Array containers benefit from cache locality for sequential access
- RLE containers have better cache efficiency for compressed data

### 6. Performance Implications

#### Bitmap Containers
- Latency: O(1) for element lookup/insertion (bit operations)
- Throughput: Excellent with SIMD bitwise operations
- Memory: Fixed 8KB per container (wasteful if sparse)

#### Array Containers
- Latency: O(log N) for element lookup (binary search)
- Throughput: Variable, depends on merge/intersection cost
- Memory: O(N) where N = cardinality (efficient for sparse)

#### RLE Containers
- Latency: O(K) where K = number of runs
- Throughput: Good for bulk operations on runs
- Memory: O(K) where K = number of runs (best for patterned data)

## SIMD-Friendly Optimizations (AVX-512 BF Focus)

### AVX-512 BF (Bit Field) Instructions
Available on: Skylake-SP and newer, Ice Lake, etc.

1. **VPOPCNTD/Q:** Parallel population count
   - Count set bits in multiple 32/64-bit values simultaneously
   - Eliminates serialization of POPCNT instruction

2. **PDEP/PEXT:** Parallel deposit/extract
   - Extract selected bits in parallel
   - Useful for bitmap-to-array conversion

3. **VPSHLDD/Q:** Parallel shift + double precision
   - Useful for RLE encoding/decoding

### Recommended Optimizations

#### Priority 1: Bitmap-Bitmap Operations
- Implement AVX-512 OR/AND/XOR for bulk bitmap operations
- Parallelized popcount using VPOPCNTD/Q
- Expected speedup: 4-8x for large bitmaps

#### Priority 2: Array-Bitmap Conversion
- Use AVX-512 PDEP for array-to-bitmap scatter
- Use VPOPCNTD to find set bits in parallel
- Expected speedup: 3-5x for conversions

#### Priority 3: Container Selection
- Vectorize threshold checks across multiple containers
- Bulk container type decisions with SIMD comparisons
- Expected speedup: 1.5-2x for optimization passes

## Key Discoveries & Recommendations

1. **Hybrid Design is Critical:** No single container type dominates all scenarios
   - Different densities require different strategies
   - The container selection logic is crucial for performance

2. **Cache Locality Matters:** 
   - Container-level organization aligns with modern cache hierarchies
   - Memory access patterns dominate performance for large bitmaps

3. **SIMD is Most Effective for:**
   - Bulk bitwise operations (bitmap containers)
   - Popcount operations across multiple containers
   - Container conversion and selection logic

4. **Data Densities Matter:**
   - Sparse data (<5%): Arrays are 2-3x better
   - Medium (20-50%): Mixed overhead, both competitive
   - Dense (>80%): Bitmaps are 2-4x better

5. **AVX-512 BF Sweet Spots:**
   - Large number of containers to process simultaneously
   - High cardinality bitmaps with many bits to extract
   - Scenarios with many conversion operations

## Next Steps
- [ ] Implement baseline Rust version without SIMD
- [ ] Profile density distributions in real workloads
- [ ] Benchmark container selection heuristics
- [ ] Design AVX-512 integration points
