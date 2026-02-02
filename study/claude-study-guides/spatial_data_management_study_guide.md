# Spatial Data Management - Study Guide
**Author:** Nikos Mamoulis (University of Hong Kong)
**Source:** Morgan & Claypool Synthesis Lectures on Data Management #21

---

## Executive Summary

This paper presents core concepts in **Spatial Data Management**, focusing on how Database Management Systems (DBMS) are extended to handle spatial data. Key topics include:

- **Spatial data characteristics**: location and geometric extent
- **Spatial relationships**: topological, directional, and distance
- **Spatial queries**: selection, nearest neighbor, spatial joins
- **Spatial indexing**: R-trees and variants for efficient query processing
- **Challenges**: multidimensional data lacks total ordering, complex geometry requires special handling

---

## Chapter 1: Introduction to Spatial Data Management

### 1.1 What is Spatial Data?

**Spatial Data** = Data with location and/or geometric extent information

**Key Examples:**
- Restaurants on a city map (location + building footprint)
- Roads (represented as polylines)
- Geographic regions (polygons)

### 1.2 Why Spatial DBMS (SDBMS)?

**Problem**: Traditional relational DBMS cannot handle spatial queries efficiently
- Example: "Find nearest restaurant to my location" - can't be done with standard SQL operations
- Requires spatial attributes, spatial predicates, and spatial indexing

**Why not just use GIS?**
- Geographic Information Systems (GIS) focus on visualization and analysis of geographic data
- GIS ≠ DBMS: GIS doesn't support set operations, aggregation, ranking like DBMS
- GIS cannot replace DBMS functionality, but can be built on top of SDBMS

### 1.3 Spatial Data Types

**Location:**
- Expressed using coordinate systems (x, y) or (lat, lon)
- Can be stored directly in relation schema

**Extent (Geometric Shape):**
- **Vector approximation**: sequence of vertices (polygons, polylines)
  - Variable length data type
  - Example: Road = sequence of line segments
- **Field representation**: raster/pixels, quadtree (not focus of this paper)

**Trade-off:** Exact geometry vs. storage/computation cost

### 1.4 Spatial Relationships

Three main types define how objects relate in space:

#### 1.4.1 Topological Relationships
Based on boundaries and interiors of objects:
- **intersects** (general relationship)
- **disjoint** (no intersection)
- **contains** (one object fully inside another)
- **inside** (opposite of contains)
- **adjacent** (touching boundaries)
- **overlaps** (partial intersection)
- **equals** (same geometry)

**Key formula:** `intersects(o1,o2) = ¬disjoint(o1,o2)`

#### 1.4.2 Directional Relationships
Based on orientation relative to reference system:
- Cardinal: north, south, east, west, northeast, etc.
- Relative: left, right, above, below, front, behind

**Note:** Can be subjective or fuzzy (e.g., "20% north, 80% east")

#### 1.4.3 Distance Relationships
Based on geometric distance metrics (e.g., Euclidean distance):
- Can be explicit (actual distance value) or implicit (distance ranges)
- Example ranges: "near" (<100m), "reachable" (100m-1km), "far" (1-10km), "very far" (>10km)

### 1.5 Spatial Query Types

#### 1.5.1 Spatial Selection (Range Query)
- **Definition**: Find all objects satisfying a spatial predicate with a query region/object
- **Example**: "Find all cities within 100km of point F"
- **Special case**: Point query (find objects containing a point)

#### 1.5.2 Nearest Neighbor Query
- **Definition**: Find k-nearest objects to a reference object q
- **Example**: "Find the closest city to point F"

#### 1.5.3 Spatial Join
- **Definition**: Combine two relations using a spatial relationship
- **Formal**: `R ⨝θ S = {(r,s) : r ∈ R, s ∈ S, r θ s}`
- **Example**: "Find all pairs of cities and rivers that intersect"
- **Result**: Set of object pairs satisfying the predicate

**Advanced queries:**
- Spatial aggregation: "Count lakes in a region"
- Combined operations: "Find 5 nearest gas stations, then restaurants within 100m of each"

### 1.6 Extending DBMS to SDBMS

**Four key extensions:**

1. **Data Modeling:**
   - Support Abstract Data Types (ADTs) for spatial types
   - OGC (Open Geospatial Consortium) standards for spatial ADTs

2. **Query Language:**
   - Extend SQL with spatial functions
   - Example: `ST_Contains(m.the_geom, r.the_geom)`, `ST_Length(r.the_geom)`
   - Core SQL semantics unchanged

3. **Storage & Indexing:**
   - Use Minimum Bounding Rectangles (MBRs) as cheap approximations
   - Two-step processing: **filter** (MBR test) → **refinement** (exact geometry test)
   - Spatial indexes: R-tree (hierarchical), grid files, space-filling curves

4. **Query Evaluation:**
   - Cost/selectivity estimation models for spatial operations
   - Enhanced query optimizer with spatial-aware plans

### 1.7 Historical Evolution

**1980s:** Focus on spatial indexes (B-tree inadequate for spatial data)

**1990s:** Object-oriented and object-relational models for spatial data

**2000s:** Spatio-temporal data, continuous queries, data mining, geographic IR

**Systems:**
- Commercial: Oracle Spatial (1997), IBM DB2 Spatial Extender, Microsoft SQL Server (2008)
- Open-source: PostGIS (PostgreSQL), MySQL, SpatiaLite (SQLite)
- GIS: ArcGIS (ESRI), MapInfo, GRASS GIS

---

## Chapter 2: Spatial Data

### 2.1 Spatial Relations (Tables)

**Spatial relation** = Table where:
- Each row = spatial object
- Each column = attribute (spatial or non-spatial)
- Spatial attribute = vector of coordinates (polyline, polygon)

**Example:**
```
ID | Name      | Type    | Polyline
1  | Boulevard | avenue  | (10023,1094), (9034,1567), (9020,1610)
2  | Leeds     | highway | (4240,5910), (4129,6012), (3813,6129)
```

### 2.2 Defining Topological Relationships

Using **interiors** and **boundaries** of objects:

| Relationship | Definition |
|--------------|------------|
| **disjoint(o1, o2)** | `(interior(o1) ∩ interior(o2) = ∅) ∧ (boundary(o1) ∩ boundary(o2) = ∅)` |
| **intersects(o1, o2)** | `(interior(o1) ∩ interior(o2) ≠ ∅) ∨ (boundary(o1) ∩ boundary(o2) ≠ ∅)` |
| **equals(o1, o2)** | `(interior(o1) = interior(o2)) ∧ (boundary(o1) = boundary(o2))` |
| **inside(o1, o2)** | `interior(o1) ⊂ interior(o2)` |
| **contains(o1, o2)** | `interior(o2) ⊂ interior(o1)` |
| **adjacent(o1, o2)** | `(interior(o1) ∩ interior(o2) = ∅) ∧ (boundary(o1) ∩ boundary(o2) ≠ ∅)` |
| **overlaps(o1, o2)** | `(interior(o1) ∩ interior(o2) ≠ ∅) ∧` <br> `(∃p ∈ o1: p ∉ interior(o2) ∧ p ∉ boundary(o2)) ∧` <br> `(∃p ∈ o2: p ∉ interior(o1) ∧ p ∉ boundary(o1))` |

### 2.3 Issues in Spatial Query Processing

#### 2.3.1 No Total Ordering in Multidimensional Space

**Problem:** Cannot sort multidimensional points to preserve spatial proximity

**Space-filling curves** (e.g., Z-order, Hilbert curve):
- Map 2D space to 1D ordering
- Still cannot guarantee proximity preservation for arbitrary data

**Implication:**
- No theoretical bounds like B-tree (O(log n + k) for range queries)
- Cannot guarantee cost linearly depends on query selectivity
- Need specialized spatial access methods

#### 2.3.2 Complex Geometry is Expensive

**Problem:** Testing spatial predicates on exact polygons is expensive
- Polygon intersection test: O(n log n) where n = total edges

**Solution: Filter-Refine Framework**

1. **Filter Step:**
   - Use Minimum Bounding Rectangle (MBR) as cheap approximation
   - MBR = minimum rectangle enclosing the object
   - Test query against MBRs (cheap, fast pruning)
   - Produces candidate results

2. **Refinement Step:**
   - Test exact geometry of candidates against query
   - Produces actual results
   - Can sometimes be skipped (e.g., if MBR fully inside query window)

**Key insight:** For most topological relationships, MBRs must use "intersects" relationship

#### 2.3.3 When to Ignore Extent?

**Observation:** In many applications, object extents are negligible compared to query scale

**Examples:**
- "Find nearest restaurant" - restaurant building size irrelevant
- Cars/buildings on city map - tiny compared to city area

**Implication:**
- Can treat objects as points for many location-based queries
- Simplifies storage, indexing, search
- No refinement step needed

---

## Chapter 3: Spatial Indexing

### 3.1 Goal of Spatial Access Methods (SAMs)

**Objective:** Group objects into disk pages such that:
- Objects in same page are close in space
- Minimize disk I/O for spatial queries

**Types:**
- **Point Access Methods (PAMs)**: For points only
- **General SAMs**: For objects with extent

### 3.2 Point Access Methods

#### 3.2.1 Grid File

**Structure:**
- Divide space using axis-parallel hyperplanes into cells
- Group cells with enough points into disk blocks
- Directory maps cells → blocks

**Properties:**
- Good for uniform, static data
- Problems: skewed distributions → large directory, under-utilized blocks
- Dynamic updates expensive (splits affect many cells)

#### 3.2.2 Space-Filling Curves

**Concept:**
- Map multidimensional space → 1D space
- Continuous mapping preserving (some) locality
- Index 1D values with B+-tree

**Z-order (Peano) Curve:**
- Recursively divide each dimension into 2^n units
- Each cell ordered by 'N' pattern
- 1D value: interleave bits of x and y coordinates
- Example: (x=6, y=4) → binary 110, 100 → Z-order 111000 = 56

**Query Processing:**
- Query window → set of Z-order value intervals
- Issue range queries to B+-tree for each interval
- Expected small number of intervals (locality preservation)

**Advantages:**
- Logarithmic update time (B+-tree)
- Good for static domain and granularity

**Other curves:** Hilbert curve (also popular)

#### 3.2.3 Quadtree

**Structure:**
- Tree (not necessarily balanced), each non-leaf has exactly 4 children
- Root = entire 2D space
- Each node's children = 4 equal quadrants of parent
- Leaves = disk blocks (store points in corresponding regions)
- Partitioning follows data distribution (adaptive)

**Partitioning rule:** Split if region has > block capacity points

**Query Processing:**
- Recursively traverse nodes overlapping query
- Smart addressing schemes possible
- Can index addresses with B+-tree

**Issue:** Under-utilization of leaf nodes (regular partitioning requirement)
**Solution:** Store multiple leaf nodes per disk block

### 3.3 Indexing Objects with Extent

**Problem with PAMs:** Space decomposition into disjoint partitions
- Point → unique partition
- Extended object → may overlap multiple partitions

**Two solutions:**

1. **Object Clipping:**
   - Split objects crossing partition boundaries
   - Distribute components to different blocks
   - **Drawback:** Data replication, especially for large extents

2. **Overlapping Partitions (R-tree approach):**
   - Allow partition MBRs to overlap
   - No object replication
   - More flexible for balancing

### 3.4 R-tree

#### 3.4.1 Structure

**Properties:**
- Height-balanced tree (like B+-tree)
- Each node = disk page
- **Entry format:** `(MBR, ptr)`
  - Leaf: MBR = object MBR, ptr = object-id
  - Non-leaf: MBR = MBR of all objects in subtree, ptr = child node

**Constraints:**
- Each node (except root): [m, M] entries where m = 0.4M (40% full minimum)
- Guarantees space utilization

#### 3.4.2 Query Processing

**Range Query Algorithm:**
1. Start from root
2. Recursively traverse entries whose MBR intersects query window
3. Return objects in leaves that intersect window

**Observations:**
- Entry MBR intersects query ≠ object definitely qualifies (dead space, overlap)
- Dead space and overlap cause false positives
- Good insertion/split algorithms minimize this

#### 3.4.3 R-tree Optimization Goals

**Four key metrics to optimize:**

1. **Minimize Area:**
   - Minimize area covered by directory MBRs
   - Reduces dead space
   - Fewer false positives

2. **Minimize Overlap:**
   - Minimize overlap between MBRs at same level
   - Prevents multiple search paths for same query
   - Most critical for search performance

3. **Minimize Margin:**
   - Margin = sum of all four side lengths
   - Prefer square-shaped MBRs over long stripes
   - Square MBRs minimize expected touches by arbitrary query
   - Margin minimization > area minimization

4. **Maximize Occupancy:**
   - Keep nodes as full as possible
   - Fewer nodes → shorter search paths
   - Trade-off with optimal partitioning

**Note:** These goals can conflict - optimization is a balancing act

### 3.5 R*-tree (Optimized R-tree)

**Key difference from R-tree:** Improved insertion and splitting algorithms

#### 3.5.1 Insertion Algorithm

**choose_subtree(Node n, Entry e) → leaf node:**

```
1. If n is leaf: return n
2. If n's entries point to leaves:
   - Choose entry minimizing overlap increase if e added
   - Ties: choose minimum area enlargement
   - Ties: choose minimum area
3. If n's entries point to non-leaves:
   - Choose entry with minimum area enlargement if e added
   - Ties: choose minimum area
4. Recurse into chosen entry
```

**Key insight:** Minimize overlap for leaf-level entries (most critical)

#### 3.5.2 Split Algorithm

**Goals:** Minimize area, overlap, margin of resulting groups

**Algorithm:**
```
For each dimension i:
  Sort entries by lower bounds in dimension i
  For each valid split point k (k in [m, M+1-m]):
    Compute sum of margins for split
  Sort entries by upper bounds in dimension i
  For each valid split point k:
    Compute sum of margins for split
  Sum all margins → Si

Choose split dimension: dimension with minimum Si
  (Intuition: split along "longest" axis → more quadratic nodes)

Consider all valid splits along chosen dimension (both sortings):
  Choose split minimizing overlap
  Ties: choose split minimizing area
```

**Complexity:** O(M log M) per split

#### 3.5.3 Forced Reinsertion

**Technique to improve occupancy:**
- On node overflow, before splitting:
  - Remove 30% of entries farthest from node center
  - Reinsert them
  - If they return to same node → proceed with split

**Benefits:**
- Increases space utilization
- Makes tree less sensitive to insertion order
- Better overall structure

**Drawback:** May traverse multiple paths per insertion (complicates concurrency control)

#### 3.5.4 Deletion

**Algorithm:**
- If leaf underflows after deletion:
  - Remove all remaining entries
  - Reinsert them
- Insertion algorithm handles relocation
- Simpler than B+-tree deletion

### 3.6 Bulk-Loading R-trees

**Problem with iterative insertion:**
- Slow for large static datasets
- Low occupancy (~67% for R*-tree)

**Goal:** Build R-tree fast with:
- Maximum occupancy
- High node quality (min area, margin, overlap)
- Minimum height

**Sort-Based Approach:**
1. Choose sort axis (e.g., x-axis)
2. Sort object MBRs by projection on axis (e.g., center)
3. Build tree bottom-up:
   - Group consecutive sorted objects into leaf nodes
   - Recursively build directory levels

**Issues:**
- Simple 1D sorting may not be optimal for 2D queries
- Better algorithms use recursive partitioning (STR, Hilbert R-tree)

---

## Key Concepts & Definitions

### Terminology

| Term | Definition |
|------|------------|
| **SDBMS** | Spatial Database Management System |
| **MBR** | Minimum Bounding Rectangle - smallest rectangle enclosing an object |
| **SAM** | Spatial Access Method - index structure for spatial data |
| **PAM** | Point Access Method - index for point data only |
| **Dead Space** | Empty space within an MBR not covered by actual objects |
| **Filter-Refine** | Two-step query processing: fast MBR test + exact geometry test |
| **Vector Approximation** | Representing geometry as sequence of vertices/coordinates |
| **Raster Approximation** | Representing geometry as set of pixels/granules |
| **Spatial Relation** | Database table with at least one spatial attribute |
| **Spatial Predicate** | Condition using spatial relationships in queries |

### Important Formulas

**Polygon Intersection Cost:** O(n log n) where n = total edges

**R-tree Node Capacity:** [m, M] entries where m = 0.4M

**Rectangle Margin:** sum of lengths of all four sides

**Z-order Value:** Interleave bits of x and y coordinates

### Spatial Relationship Hierarchy

```
ANY
├── intersects
│   ├── equals
│   ├── inside
│   ├── contains
│   ├── adjacent
│   └── overlaps
└── disjoint
```

---

## Important Algorithms Summary

### 1. R-tree Range Query
```
RangeQuery(Node n, Rectangle W):
  For each entry e in n:
    If e.MBR intersects W:
      If n is leaf:
        Add e.object to results
      Else:
        RangeQuery(e.ptr, W)
```

### 2. R*-tree Insertion Path Selection
```
1. Leaf level: minimize overlap increase
2. Non-leaf level: minimize area enlargement
3. Break ties: choose minimum area
```

### 3. R*-tree Split
```
1. For each dimension, compute sum of margins for all valid splits
2. Choose dimension with minimum sum (split axis)
3. Among all valid splits on that axis, choose:
   - Minimum overlap
   - Ties: minimum area
```

### 4. Filter-Refine Query Processing
```
Step 1 (Filter): Test query against object MBRs (cheap, pruning)
  → Candidates

Step 2 (Refine): Test exact geometry of candidates (expensive, accurate)
  → Actual results
```

---

## Exam Preparation Tips

### High-Priority Topics

1. **Understand the "why":**
   - Why can't relational DBMS handle spatial data?
   - Why is there no total ordering in multidimensional space?
   - Why do we need MBRs and filter-refine?

2. **Spatial Relationships:**
   - Know the three types: topological, directional, distance
   - Understand topological relationships and their definitions
   - Be able to give examples

3. **Spatial Queries:**
   - Three main types: selection, nearest neighbor, spatial join
   - Know their definitions and examples
   - Understand how they differ from relational queries

4. **R-tree:**
   - Structure (hierarchical, MBR entries, leaf vs. non-leaf)
   - Optimization goals (area, overlap, margin, occupancy)
   - How range queries work (recursive traversal)
   - Issues: dead space, overlap

5. **R*-tree Improvements:**
   - Insertion: minimize overlap at leaf level, area at higher levels
   - Splitting: choose axis with min margin sum, then min overlap
   - Forced reinsertion to improve occupancy

### Common Question Types

**Conceptual:**
- "Why is spatial data management challenging?"
- "Explain the filter-refine framework"
- "What are the limitations of PAMs for extended objects?"

**Technical:**
- "How does an R-tree answer a range query?"
- "What are the optimization goals for R-tree nodes?"
- "Explain the R*-tree split algorithm"

**Applied:**
- Given a spatial query, write it in extended SQL
- Given objects and an R-tree, trace query execution
- Compare different spatial indexes (grid, quadtree, R-tree)

### Memory Aids

**Three Spatial Relationship Types:** **T**opological, **D**irectional, **D**istance (TDD)

**Four R-tree Optimization Goals:** **A**rea, **O**verlap, **M**argin, **O**ccupancy (AOMO)

**R*-tree Splitting:** "**M**argin for axis, **O**verlap for split" (MO)

### What to Memorize

- Three spatial query types
- Four DBMS extensions needed for spatial data
- Four R-tree optimization goals
- R*-tree insertion heuristics (overlap for leaves, area for non-leaves)
- Filter-refine framework steps
- Constraint: R-tree nodes 40% full (m = 0.4M)

### Common Pitfalls

- Don't confuse GIS with SDBMS (GIS ≠ SDBMS)
- Don't say "distance relationship" when you mean "topological"
- Remember: intersects is the most general topological relationship
- R*-tree optimizes for different metrics at different levels (overlap at leaves, area at non-leaves)
- Space-filling curves use B+-tree, not a new index structure

---

## Practice Questions

### Question 1: Concepts
**Q:** Why can't we use a B+-tree to index 2D spatial points?

**A:** Because there is no total ordering in multidimensional space that preserves spatial proximity. No matter which 1D ordering we choose (row-order, Z-order, Hilbert, etc.), we cannot guarantee that spatially close points remain close in the linear order for arbitrary data distributions. This means a B+-tree cannot provide the same theoretical bounds (O(log n + k)) as it does for 1D data.

### Question 2: Relationships
**Q:** For two polygons A and B, if inside(A,B) is true, what can we say about their MBRs?

**A:** The tightest relationship we can guarantee between MBR(A) and MBR(B) is intersects(MBR(A), MBR(B)). We cannot guarantee inside(MBR(A), MBR(B)) because the MBR approximation may include dead space that extends outside B, even though A itself is fully inside B.

### Question 3: R-tree
**Q:** An R-tree node has capacity M=10. After inserting a new entry, the node has 11 entries. Using R*-tree, describe the two options for handling this overflow.

**A:**
1. **Forced Reinsertion** (tried first): Remove 30% of entries (≈3 entries) farthest from the node's center and reinsert them into the tree. If these entries end up in different nodes, overflow is avoided.

2. **Split** (if reinsertion fails): If the reinserted entries return to the same node, perform a split:
   - For each dimension, compute sum of margins for all valid splits
   - Choose dimension with minimum margin sum as split axis
   - Among all splits on that axis, choose the one minimizing overlap (ties: minimum area)
   - Create two nodes with entries distributed according to the chosen split

### Question 4: Query Processing
**Q:** Explain why the filter-refine framework is necessary for spatial queries.

**A:** The filter-refine framework addresses two problems:
1. **Complex geometry**: Testing exact polygonal representations is expensive (O(n log n) for intersection)
2. **Large datasets**: Exhaustively checking all objects requires massive I/O

**Solution:**
- **Filter step**: Use cheap MBR approximations to quickly eliminate objects that definitely don't qualify, reducing candidates
- **Refine step**: Test exact geometry only for candidates

This dramatically reduces computation (MBR tests are cheap) and I/O (only fetch candidates for refinement).

---

## Additional Resources

**Standards:**
- Open Geospatial Consortium (OGC): http://www.opengeospatial.org/

**Systems:**
- PostGIS: http://www.postgis.org/
- Oracle Spatial: Oracle Database documentation
- ArcGIS (ESRI): http://www.esri.com/

**Conferences:**
- ACM SIGSPATIAL (ACM GIS)
- Symposium on Spatial and Temporal Databases (SSTD)
- IEEE Mobile Data Management (MDM)

**Further Reading:**
- Shekhar and Chawla [2003]: Spatial Databases: A Tour
- Rigaux et al. [2001]: Spatial Databases (GIS focus)
- Gaede and Günther [1998]: Multidimensional Access Methods

---

## Quick Reference Card

### Must-Know for Exam

| Category | Key Points |
|----------|------------|
| **Why Spatial DBMS?** | Relational DBMS can't handle spatial queries; need spatial predicates, indexes |
| **Data Types** | Location (coordinates), Extent (vector: polygons/polylines) |
| **Relationships** | Topological (intersects, contains, overlaps), Directional (north/south), Distance |
| **Queries** | Selection (range), Nearest Neighbor (k-NN), Spatial Join |
| **Filter-Refine** | Step 1: MBR test (fast), Step 2: exact geometry (accurate) |
| **R-tree** | Hierarchical, balanced, MBR entries, [0.4M, M] entries per node |
| **R*-tree** | Better insertion (min overlap/area), better split (min margin→axis, min overlap→split) |
| **No Total Order** | Cannot sort multidimensional points preserving proximity → no theoretical bounds |
| **Space-Filling** | Z-order, Hilbert → map 2D to 1D → index with B+-tree |

---

**Good luck with your exam!**
