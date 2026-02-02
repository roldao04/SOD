# Multidimensional Databases and Data Warehousing - Study Guide
**Authors:** Christian S. Jensen (Aarhus University), Torben Bach Pedersen (Aalborg University), Christian Thomsen (Aalborg University)
**Source:** Morgan & Claypool Synthesis Lectures on Data Management #9

---

## Executive Summary

This paper presents core concepts in **Multidimensional Databases and Data Warehousing**, focusing on how data is modeled and analyzed for business intelligence. Key topics include:

- **Multidimensional model**: Categorizes data as facts (with measures) and dimensions (context)
- **OLAP (On-Line Analytical Processing)**: Fast, interactive analysis of multidimensional data
- **Data Warehousing**: Integrated, subject-oriented, time-variant, non-volatile data repositories
- **Data modeling**: Cubes, dimensions (with hierarchies), facts, and measures
- **Relational representations**: Star schemas and snowflake schemas
- **Analysis operations**: Roll-up, drill-down, slicing, dicing, drill-across, ranking
- **Query languages**: MDX and SQL extensions (CUBE, window functions)

---

## Chapter 1: Introduction

### 1.1 What is Multidimensional Data Management?

**Historical Context:**
- **1970**: Codd introduces relational model → Turing Award
- **1990s**: Multidimensional data model emerges for data analysis
- Both underpin multi-billion-dollar industries (database vs. business intelligence)

**Key Insight:** Multidimensional models are designed expressly for **data analysis**, not transactions

### 1.2 Core Concept

**Multidimensional Model categorizes data into two types:**

1. **Facts**: Objects representing the subjects of analysis (events/processes)
   - Associated with numerical **measures**
   - Example: A book purchase

2. **Dimensions**: Textual characteristics that provide context for facts
   - Example: Location, Book (genre, author), Time

**Example Query:**
"Find total sales per month and author" → aggregates measure values over ranges of dimension values

### 1.3 Three Main Application Areas

#### 1.3.1 Data Warehousing
- Large repository of integrated data from multiple sources
- Specific purpose: data analysis
- Multidimensional modeling provides good analysis support

#### 1.3.2 OLAP (On-Line Analytical Processing)
**FASMI Definition:** "Fast Analysis of Shared Multidimensional Information"
- **Fast**: Response times within seconds (interactive analysis)
- **Analysis**: Support for business logic and statistical analyses
- **Shared**: Security mechanisms, concurrency control for multiple users
- **Multidimensional**: Hierarchical dimensions required
- **Information**: Manage all required data and derived information

**Three Types of OLAP Systems:**
- **ROLAP**: Relational OLAP (uses relational DBMS)
- **MOLAP**: Multidimensional OLAP (uses array-type technologies)
- **HOLAP**: Hybrid OLAP (combines both approaches)

#### 1.3.3 Data Mining
- (Semi-)automatic discovery of unknown knowledge
- Increasingly applied to multidimensional data
- Multidimensional organization well suited for data mining queries

### 1.4 Historical Evolution

**Late 1960s:** IRI and Comshare begin development
- IRI Express → marketing analysis tool (late 1970s/early 1980s) → acquired by Oracle
- Comshare → System W (financial planning, 1980s)

**1991:** Arbor formed → Essbase system ("multiuser, multidimensional database server")
- Later Hyperion
- Licensed basic version to IBM for DB2 integration

**1993:** Arbor and Codd coin term "OLAP"

**Early 1990s:** Large data warehouses emerge
- Based on relational star/snowflake schemas
- Implement multidimensional databases using relational technology

**1998:** Microsoft ships MS OLAP Server
- First multidimensional system for mass market
- Led to commodity products shipped with relational DBMSs

### 1.5 Key Terminology

| Term | Definition |
|------|------------|
| **OLAP** | On-Line Analytical Processing (vs. OLTP: On-Line Transaction Processing). Focus on analysis, not transactions. Fast, interactive response. Always employs multidimensional view. |
| **ROLAP** | Relational OLAP - systems based on relational DBMS technology |
| **MOLAP** | Multidimensional OLAP - systems using non-relational, array-type technologies |
| **HOLAP** | Hybrid OLAP - combines ROLAP and MOLAP |
| **Data Warehouse** | Repository of integrated enterprise data for decision support. Typically one per enterprise. Contains data from many sources. |
| **Data Mart** | Subset of a data warehouse |
| **ETL** | Extract-Transform-Load: Three-step process that feeds data warehouse:<br>1. Extract from operational sources<br>2. Transform (change format, combine, cleanse)<br>3. Load into data warehouse<br>Also called ETT (Extract-Transform-Transport) |
| **Business Intelligence (BI)** | Process of making intelligent decisions based on data analysis. Technology: data warehousing, reporting, OLAP, data mining, visualization, what-if analysis |

### 1.6 Shortcomings of Alternatives

#### 1.6.1 Spreadsheets

**Problems:**

1. **Two dimensions only**: Can use separate worksheets for 3rd dimension, but:
   - Only works for few dimension values
   - Analyses across multiple values are cumbersome
   - Many thousands of values (e.g., time) = infeasible
   - Four+ dimensions = very complex setup

2. **Grouping/hierarchies require duplication**:
   - Example: Group books into "Cooking" and "Fiction"
   - Must duplicate grouping info across all worksheets
   - Uses extra space, cumbersome to maintain

3. **Data storage tied to presentation**:
   - Structure and desired views not separated
   - Lack of flexibility

**What spreadsheets ARE good for:**
- Viewing and querying multidimensional data
- Pivot tables (2D tables with subtotals/totals)

#### 1.6.2 Relational Databases (SQL)

**Problems:**

1. **Desirable computations are hard/impossible in standard SQL:**
   - Cumulative aggregates (sales in year-to-date)
   - Totals and subtotals
   - Rankings (top 10 selling books)

2. **Interrow computations difficult:**
   - Only intercolumn computations are easy
   - Transpositions of rows/columns not easy (requires manual views)

3. **Hierarchical dimensions not handled satisfactorily:**
   - SQL extensions (CUBE operator, query windows) help some problems
   - But hierarchical dimensions remain challenging

**When are alternatives adequate?**
- Few dimensions
- No hierarchical dimensions needed
- Small data volume
→ Spreadsheets may be adequate

**Robust solution:** Data models and database technology with inherent support for full range of multidimensional concepts

### 1.7 OLAP vs OLTP

| Aspect | OLTP (Transaction Processing) | OLAP (Analytical Processing) |
|--------|-------------------------------|------------------------------|
| **Focus** | Single transactions | Large amounts of detail data |
| **Example** | Book retailer: track books in stock | Analyze each individual sale |
| **Data scope** | Most current data only | Historical data, each event |
| **Typical query** | "Decrease stock of book B by 1" | "Find average monthly children's books sold in last 3 years" |
| **Data organization** | Normalized relational database | Multidimensional database |
| **Optimization** | Retrieve/update small # of tuples | Retrieve/summarize very large amounts |
| **Updates** | Frequent updates and additions | No updates/deletes, only periodic additions |
| **Customer info** | May not be stored for analysis | Detailed info (city, date, salesperson, etc.) |
| **Design goal** | Avoid update anomalies | Easy analysis and aggregation |

**Key Differences:**
- OLTP: operational, current, transactional, small queries, normalized
- OLAP: analytical, historical, read-only, large queries, denormalized

---

## Chapter 2: Fundamental Concepts

### 2.1 Cubes

**Definition:** Truly multidimensional data structure for capturing and analyzing data

**Key Properties:**
- Generalizes tabular spreadsheet to any number of dimensions
- Hierarchies and formulas are first-class, built-in concepts
- No duplication of definitions
- Term "cube" used even for non-3D structures
- Alternative term: **hypercube**
- Collection of related cubes = **multidimensional database** or **multidimensional data warehouse**

#### 2.1.1 Cube Structure

**Cells:**
- Uniquely identifiable at each dimension intersection
- **Non-empty cell** = **fact**
- **Empty cell** = no information for that combination

**Facts:**
- Each fact has **measures** associated with it
- Measures = numerical values within cells
- Empty cell = no sale for that book/city/year combination

**Sparsity:**
- Percentage of non-empty cells varies widely
- Cubes range from sparse to dense
- Tend to become sparser with:
  - Increasing dimensionality
  - Increasingly fine granularities

**Typical Dimensionality:**
- Most real-world cubes: **4-12 dimensions**
- Performance problems possible with >10-15 dimensions
- **Viewing limitation**: Only 2-3 dimensions at a time
- Low-cardinality dimensions: up to 4 by nesting

**Projection and Aggregation:**
- Reduce cube dimensionality at query time
- Project down to 2-3 dimensions
- Aggregate measure values across projected-out dimensions
- Example: View sales by City and Time → aggregate over Book dimension

#### 2.1.2 Design Philosophy

**Key Goal:** "Provide as much context as possible for the facts"

**Implication:** Different view on redundancy than relational databases
- **Controlled redundancy** is appropriate if it increases information value
- Reason: Multidimensional DBs often derived from other sources
- Updates are rare, easier to handle
- **No redundancy in facts**, only in dimensions

### 2.2 Dimensions

**Definition:** Essential and distinguishing concept for multidimensional databases

**Two Purposes:**
1. **Selection of data** (filter)
2. **Grouping of data** at desired level of detail (aggregate)

#### 2.2.1 Dimension Structure

**Hierarchy:**
- Containment-like hierarchy composed of **levels**
- Each level = level of detail of interest to analyses
- **Instances** = **dimension values** or **dimension members**
- Each value belongs to a particular level

**Example: Location Dimension**
```
T (top)
├── State level (Virginia, Florida, Massachusetts)
    ├── City level (Arlington, Miami, Boston, Springfield)
```

**T Level:**
- Represents "all" of the dimension
- Every dimension value is part of T value
- No column needed in relational representation (always same value)

#### 2.2.2 Level Properties

**Definition:** Non-hierarchical information associated with a level

**Example:** Population size of city (level property of lowest level in Location dimension)

#### 2.2.3 Ordering and Metrics

**General case:**
- No ordering or distance metric on dimension values
- Only ordering: containment of lower-level in higher-level values

**Special cases:**
- Time dimension: ordering available
- Enables cumulative calculations (e.g., "total sales in year to date")

#### 2.2.4 Multiple Hierarchies

**Definition:** A dimension can have multiple hierarchies

**Example: Time Dimension**
- Fiscal Year hierarchy: Day → Month → Fiscal Quarter → Fiscal Year → T
- Calendar Year hierarchy: Day → Month → Calendar Quarter → Calendar Year → T
- **Common lowest levels**: Day and Month
- **Different grouping above**: Fiscal vs. Calendar Quarter/Year

**Benefits:** Easy reference to several ways of grouping

**Support:** Most multidimensional models allow multiple hierarchies

#### 2.2.5 Balanced Tree Requirement

**Most models require:**
1. **Uniform height**: All paths from bottom to top have same length
   - Example: If some shops in departments, ALL shops must be in departments

2. **No level jumping**: Direct links only between immediate parent-child levels
   - Can't jump two or more levels

3. **Single parent**: Each non-top value has precisely one parent
   - Example: Product belongs to exactly one product group

**Problem Example:** Washington, D.C. belongs to no state
- Violates requirement that all cities belong to states

### 2.3 Facts

**Definition:** Objects representing the subjects of desired analyses

**Characteristics:**
- Interesting "things," events, or processes to be analyzed
- Example: Sales (in book retailer)
- Implicitly defined by combination of dimension values
- Non-empty cell for combination → fact exists

**Granularity:**
- Each fact mapped to precisely one dimension value at lowest level in each dimension
- Determined by levels from which dimension values are drawn
- Example: "Year by Book by City"
- **Coarser granularity**: Higher-level dimension levels (e.g., "Year by Genre by City")
- **Finer granularity**: Lower-level dimension levels (e.g., "Day by Book by City")

#### 2.3.1 Types of Facts

**1. Event Facts:**
- Model events in real world
- Exactly one fact for each unique event in overall process
- Fine granularity: one fact per sale of a book
- Coarser granularity: one fact per day when one or more sales occurred
- **Examples**: sales, web page clicks, warehouse goods flow
- **Come into existence**: Unpredictably, as events happen

**2. Snapshot Facts (State Facts):**
- Model state of process at given point in time
- Same object may occur in facts at different time points
- **Examples**: inventory levels in stores/warehouses, website user count
- **Come into existence**: Predefined times (e.g., first day of each month)

**3. Measureless Facts (Factless Facts):**
- Event facts without measures
- Record that event took place
- Event has no numerical property
- **Example**: Company contacts customer at given time for marketing campaign

**Coexistence:** Different types often coexist in data warehouse for complementary analyses
- Same base data may feed three cube types: flow, inventory, flow year-to-date

### 2.4 Measures

**Definition:** Two components:
1. **Numerical property** of a fact (e.g., sales price, profit)
2. **Formula** to combine multiple measure values into one (often SUM)

**Purpose:** Properties of facts that users want to study/optimize

**Designer's Role:** Define what numerical measure represents
- Example: Snapshot fact (sales on day) → amount sold during that day
- Alternative: Cumulative (growing) → total sales on that day and all days before

**Mathematical View:** Partial function from Cartesian product of dimensions to numbers (N or R)

**Value variation:** Different values for different dimension value combinations

**Aggregation:** Meaningful for all combinations of aggregation levels (including T level)

**Metadata:** Formula defined once, not replicated (unlike spreadsheets)

**Optional:** Some models don't have measures; use dimension values for computations (less user-friendly)

#### 2.4.1 Types of Measures

**1. Additive Measures:**
- Can be combined meaningfully along ANY dimension
- No overlap among real-world phenomena causing individual values
- Example: Total sales over Book, Location, Time
- **Occur in**: Any kind of fact

**2. Semi-Additive Measures:**
- Cannot be combined along one or more dimensions (usually Time)
- Example: Inventory levels
  - Can sum across books and stores
  - Cannot sum across time (same item counted multiple times)
- **Occur in**: Often snapshot facts

**3. Non-Additive Measures:**
- Cannot be combined along any dimension
- Usually because of chosen formula
- Example: Averages for lower-level values cannot combine into higher-level averages
- **Occur in**: Any kind of fact

### 2.5 Relational Representations

Two principal ways to represent dimensions in relational database:

#### 2.5.1 Star Schema

**Structure:**

1. **Dimension Tables** (one per dimension):
   - Key column (surrogate key - "dumb" integer, no semantics)
   - One column per level (except T)
   - Textual descriptions of dimension values
   - One column per level property

2. **Fact Table**:
   - One row per multidimensional fact
   - One column per measure (holds measure values)
   - One foreign key column per dimension (references dimension table PKs)

**Visual:** Drawing looks like a star (fact table = center, dimension tables = points)

**Example:**
```
Book (BookID, Book, Genre)
Location (CityID, City, State)
Time (DayID, Day, Month, Year)
Sales (BookID, CityID, DayID, Sale)
```

**Key Choice: Surrogate Keys**
- "Dumb" integer key without semantics
- Advantages over information-bearing keys:
  - Better storage use
  - Prevents key re-use problems
  - Better support for dimension updates
  - More efficient query processing

**Redundancy:**
- Higher-level data duplicated
- Example: "March 2009" present for each day in month (31 rows)
- **Not a problem because**:
  - Dimensions typically 1-5% of total storage
  - No update-performance problems
  - Updates handled centrally → consistency ensured
  - Supports simpler, better-performing queries than normalized alternatives

#### 2.5.2 Snowflake Schema

**Structure:**

1. **Fact Table**: Same as star schema

2. **Multiple Dimension Tables per Dimension**:
   - One table per level (except T)
   - Key column
   - Column holding textual descriptions
   - Columns for level properties
   - Lower levels: foreign key to containing level

**Benefits:**
- Avoids redundancy
- Renders hierarchies explicit

**Drawback:**
- Harder to query (many joins required)
- More time-consuming for DBMS to compute results

**Visual:** Looks like snowflake when drawn

**Example:**
```
Day (DayID, Day, MonthID)
Month (MonthID, Month, YearID)
Year (YearID, Year)
```

#### 2.5.3 Starflake Schema

**Definition:** Hybrid between star and snowflake

**Options:**
- Apply snowflaking fully to only selected dimensions
- Apply snowflaking partially within a dimension

**Trade-off:** Depends on desired system properties

### 2.6 Data Warehouses and Data Marts

#### 2.6.1 Inmon's Data Warehouse Definition

**"A subject oriented, integrated, time variant, non-volatile collection of data in support of management's decision making process"**

Let's break down each element:

**1. Decision Support Purpose:**
- Very different from operational systems
- Operational: support daily business processes (orders, deliveries, bills)
- Data Warehouse: strategic analysis and decision making
- Different focus impacts WHAT data to store and HOW to store it

**2. Subject Oriented:**
- Designed around important business subjects
- Example: "sales" for book retailer
- Enables easy analysis of subjects
- Sales described by entities: "books," "shops"

**Contrast with Operational DBs:**
- Modeled to support daily operations
- Heavily influenced by workflows in business
- Influenced by how operational applications work
- Example: Separate apps for ordering, distribution, marketing, accounting
- Analyzing profit during campaign requires data from several systems with different organizations

**3. Integrated:**
- Data originates from several different operational systems
- Each system may use different formats
  - Example: Paperback coded as "P", "paperback", or "3"
  - Different units: cm vs. inches
- **Data warehouse**: Unique code/description for everything
- **Best practice**: Use understandable, textual descriptions (not cryptic codes)
- **ETL process responsibility**: Extract, clean, transform to integrated format, load

**4. Time Variant:**
- Shows evolution over time
- Not just most recent data (like operational systems)
- Possible to see what modeled world looked like at certain point in time
- Examples:
  - How many "Winnie the Pooh" sold up to now
  - How many sold by same time last year
- Captures changes in modeled world (e.g., shop expansion)
- Analyst can see size at different times, investigate expansion effects
- **Virtually any data warehouse has a time dimension**

**5. Non-Volatile:**
- Neither deletions nor updates applied to existing data
- Only changes: loading of new data
- **Contrast with operational systems:**
  - Designed for concurrent updates
  - Advanced transaction handling required
  - Normalized designs for efficient updates, avoid anomalies
- **Data warehouse:**
  - Normalization not needed (star schema)
  - No complex transaction handling
  - Includes historical data (not just recent)

#### 2.6.2 Data Mart

**Definition:** Subset of a data warehouse

**Characteristics:**
- Data warehouse: shared throughout organization, multiple subjects
- Data mart: single subject (e.g., sales)
- Often specialized towards (sub-)department needs

**Example:**
- DW: cubes for shop sales, Internet sales, stock keeping, staffing
- Sales DM: only sales data (for sales department)
- HR DM: only staffing data (for HR department)

**Important:** Departments should NOT have private data marts
- If multiple departments need sales data, use same sales data mart

**Data Flow:**
- Data warehouse: loaded from operational source systems
- Data mart: loaded from data warehouse (already cleaned/integrated) OR provides logical view

#### 2.6.3 Top-Down vs. Bottom-Up Approaches

**Top-Down (Inmon):**
- Build data warehouse first
- Derive data marts from data warehouse
- **Advantage**: Important integration
- **Problem**: Difficult development process for large organizations, lengthy, late ROI

**Bottom-Up:**
- Build data marts first
- Merge to obtain data warehouse
- Data warehouse = collection of data marts
- **Problem**: Lack of integration

**Middle Ground (Kimball): Conformed Dimensions and Facts**
- Organization standardizes facts and dimensions to be shared
- Enables process with both top-down and bottom-up elements
- Possible to combine several marts/cubes coherently

### 2.7 Multidimensional Modeling Process

**Key Differences from "Ordinary" Data Modeling:**

1. **Don't include everything:**
   - Only essential "drivers" of business
   - Not all available data
   - Not all existing relationships

2. **Redundancy acceptable:**
   - In well-chosen places (mostly dimensions)
   - If makes model more intuitive
   - Examples:
     - Month name repeated for each day
     - Customer info in both Customer and Demographics dimensions

**Kimball's Four-Step Process:**

#### Step 1: Choose Business Process(es) to Model

**Rationale:** Not all processes equally important

**Example:** Book retailer
- Sales of books to customers
- Purchases from publishers
- **Sales** has largest profit potential → prioritize

#### Step 2: Choose Granularity

**Importance:** Use granularity matching analysis needs

**Trade-offs:**
- Finer grain: more flexibility, more storage, worse performance
- Coarser grain: less flexibility, less storage, better performance

**Example:**
- "Individual sales of books" (finest)
- "Total sales per book per store per day" (coarser, performance/storage gains)

**Grain decides what a fact means**

#### Step 3: Design the Dimensions

**Process:** Refine each part of grain into complete dimension

**Components:**
- Levels
- Attributes

**Example:** Grain "sales per book per store per day"
- → Store dimension, Book dimension, Time dimension

#### Step 4: Choose the Measures

**Examples:**
- Dollar sales
- Unit sales
- Dollar cost
- Profit

**Consideration:** Numerical measures to capture for each dimension value combination

#### 2.7.1 Conformed Dimensions and Measures

**Purpose:** Ensure compatibility between cubes for comparable analysis results

**Conformed Dimensions:**
- Two dimensions are conformed if:
  - They are identical, OR
  - One is subset of other (dimension values AND attributes)

**Conformed Measures:**
- Equivalently defined
- Example: "dollar sales" defined same way (include tax, not rounded)

**Benefits:**
- Meaningfully combine data from different cubes
- Single definition used across all cubes
- Example: Compare planned sales (Budget cube) with actual sales (Sales cube)

**Challenge:**
- Sounds easier than it is
- Requires interactions, mediation, negotiations across departments
- Large organizations: 20+ source systems, each defining "customer" differently
- Requires strong management sponsorship and commitment

### 2.8 Analysis and Querying

**Objective:** Support data analyses of large amounts of data

#### 2.8.1 Roll-Up

**Definition:** Go from some level to a **higher level** in same hierarchy

**Effect:** Less detail shown

**Process:**
1. Start with cells at lowest levels
2. Apply aggregation formula to combine measure values
3. Result: cells at higher level

**Example:**
- From individual books → to Genre level
- From Genre level → to T level (projects out dimension)
- "Roll-up to T" = show totals without that dimension

**Measure Formula:** Typically SUM, combines multiple values into one

**Example Calculation:**
- Cookbooks sold in Arlington = 2 + 1 = 3
- (2 copies "Italian Cooking" + 1 copy "Tropical Food")

#### 2.8.2 Drill-Down

**Definition:** Opposite of roll-up; go to **lower level** for more details

**Effect:** More detail shown

**Usage Pattern:**
- Analyst starts with higher-level data
- Performs drill-downs to understand details

**Example Scenario:**
1. Analyst sees: 2005 children's books sales much higher than 2004
2. Drill-down Time dimension: Discovers July 2005 particularly high
3. Drill-down Book dimension: Sees "Harry Potter and the Half-Blood Prince" caused increase

#### 2.8.3 Drill-Out

**Definition:** Include additional dimension (at non-T level)

**Effect:** Measure values "spread out" among more cells

**Difference from Drill-Down:**
- Drill-down: expand within dimension
- Drill-out: add new dimension

**Example:**
- Start: Sales by Genre only (1D)
- Drill-out: Add Time dimension → Sales by Genre by Year (2D)

#### 2.8.4 Slicing and Dicing

**Metaphor:** Like slicing/dicing an onion for cooking

**Slicing:**
- Select specific value for a dimension
- Example: Select Year 2009 from Time dimension
- Result: Subset of cube (a "slice")

**Dicing:**
- Apply more complex conditions
- Further subset the data
- Example: Consider years 2008 AND 2009 AND books by Jane Austen
- Result: Smaller subset (a "dice")

**Process:** Creating these subsets = slicing and dicing

#### 2.8.5 Drill-Across

**Purpose:** Combine data from two cubes

**Requirement:** Shared dimensions (must be conformed)

**Process:**
1. Roll-up non-shared dimensions to top levels
2. Combine cubes via shared dimensions
3. Result inherits all measures from both cubes

**Relational Equivalent:** Full outer join

**Granularity Handling:**
- Cubes may have different granularities
- Resulting cube has levels that original cubes share
- Example: Budget (by genre by year) + Sales (by book by day)
  - Result: Sales rolled up to Year and Genre levels before combining

**Example Usage:**
- Shop Sales cube: dimensions Book, Location, Time; measure: sales
- Internet Sales cube: dimensions Book, Customer, Time; measure: sales
- Drill-across on Book and Time: Compare total sales across channels
- Find best-selling books considering both shop and Internet

**Why Necessary:**
- Can't just find top sellers in each cube separately
- Example:
  - Shop: Book A (10K), Book B (9K), Book C (8K)
  - Internet: Book D (5K), Book E (5K), Book C (4K)
  - Top 2 per cube: A and B (wrong!)
  - Correct answer: C (12K total) is best seller

#### 2.8.6 Pivot Tables

**Definition:** 2-dimensional table with subtotals and totals

**Capabilities:**

1. **Selection:**
   - Pick which data shown on axes
   - Example: Select genres Cooking and Fiction, states Florida, Massachusetts, Virginia
   - See only relevant subset and its subtotals/totals

2. **Drill-Down/Roll-Up:**
   - Use hierarchies to change detail level
   - Example: "Expand" Cooking row to see individual books

3. **Nesting:**
   - Multiple dimensions on one axis
   - Example: Add Year to x axis → rows for each genre/year combination

4. **Pivoting:**
   - "Rotate" dimensions
   - Swap x and y axes
   - Move dimensions between axes

**Example:**
```
                Florida  Massachusetts  Virginia  Total
Children's books  2042        8096        4031   14169
Cooking          20764       19174         573   40511
Fiction          50145       68944       12493  131582
Total            72951       96214       17097  186262
```

#### 2.8.7 Ranking

**Purpose:** Order data by measure values

**Operations:**
- Complete ranking: All items assigned numbers 1, 2, 3, ...
- **Top-k query**: Find top set (e.g., top-5 best selling books)
- Nested ranking: Top-k within top-k (e.g., 5 best books in 3 best genres)

**Basis:** Number of copies sold, profit, or other measures

**Support:** MDX query language offers ranking and sorting functions

### 2.9 Querying Languages

#### 2.9.1 SQL Extensions

**CUBE Operator (Gray et al., SQL:1999):**

**Purpose:** Generalizes GROUP BY, crosstabs, subtotals

**Special "ALL" value:**
- Denotes aggregation over all values for attributes
- Generates subtotals or grand total
- Similar to T-level in dimension
- Implemented as NULL (with special function to distinguish from real NULLs)

**Example Query:**
```sql
SELECT City, Book, SUM(Sales) AS Sales
FROM SalesTable
GROUP BY CUBE (City, Book)
```

**Result:**
- Sales by city and book (detailed)
- Subtotals by city (Book = ALL)
- Subtotals by book (City = ALL)
- Grand total (City = ALL, Book = ALL)

**Window Functions (SQL:2003):**

**Concept:**
- Regular aggregate: one result per group
- Window function: each row remains, aggregate over related rows

**Example Query:**
```sql
SELECT City, Book, Sales, SUM(Sales) as Total
OVER (PARTITION BY City)
FROM SalesTable
```

**Result:** Each row shows its sales + total for its city

**Ranking with Windows:**
```sql
SELECT City, Book, Sales, RANK()
OVER (PARTITION BY City ORDER BY Sales DESC)
FROM SalesTable
```

**Other SQL Features:**
- Data cube operators adopted by SQL standard
- Ranking, percentiles, windowing
- Mathematical and statistical functions
- **Iceberg cube**: Only returns rows where measure exceeds threshold

#### 2.9.2 MDX (MultiDimensional eXpressions)

**Origin:** Microsoft, now used in other OLAP products

**Key Difference from SQL:** Statements directly produce pivot tables

**Benefits:** Easy integration with OLAP client tools

**Terminology:**
- Dimension values = **members**
- Special **Measures dimension**: values = names of cube measures
- Any measure value referenced by combination of members

**Structure:**

**Dimensions and Hierarchies:**
- Dimensions have one or more hierarchies
- Hierarchies have levels
- Top level: **(All)** (like T value)
- Dot notation: `[Time].[Calendar].[Month].[May 2009]`

**Query Components:**

1. **FROM clause:** Specify input cube

2. **SELECT clause:** Specify axes
   - Up to 128 query axes (typically only few)
   - Five "standard" axes: COLUMNS, ROWS, PAGES, SECTIONS, CHAPTERS

3. **WHERE clause:** Slicer axis
   - Retrieve data for single member from hierarchies

**Example Query:**
```
SELECT
  { [City].Members, [City].[All Cities] } ON COLUMNS,
  { [Book].Members, [Book].[All Books] } ON ROWS
FROM [SalesCube]
WHERE ( [Measures].[Sales] )
```

**Notation:**
- `{...}`: Set of members
- `.Members`: All members in dimension
- `[All ...]`: Top of hierarchy

**Functions:**
- Navigate hierarchies (some understand time semantics)
- Perform calculations
- Rank data
- Much more

#### 2.9.3 Graphical Querying

**Problem:** MDX is powerful but complex
- Only IT specialists can write MDX queries
- Business analysts need user-friendly system

**Solution: OLAP Clients with GUIs**

**Features:**
- Easy-to-use components (tables, graphs)
- Click/menu-driven operations
- Drill-down, slicing, pivoting by clicking
- Behind scenes: generate MDX or SQL queries

**Dashboards:**
- Give overview of summarized data
- Show key performance indicators (KPIs)
- Limited number of components
- Pre-defined analysis results
- High-level overview
- Like car dashboard: readable at glance
- May support further analyses (drill-down)

---

## Key Concepts & Definitions

### Core Terminology

| Term | Definition |
|------|------------|
| **Cube** | Multidimensional data structure with any number of dimensions |
| **Hypercube** | Alternative term for cube (emphasizes n-dimensions) |
| **Fact** | Non-empty cell in cube representing subject of analysis |
| **Measure** | Numerical property of fact + aggregation formula |
| **Dimension** | Characteristic providing context for facts (usually textual) |
| **Level** | Degree of detail in dimension hierarchy |
| **Dimension Value/Member** | Instance of dimension at particular level |
| **Granularity** | Level of detail determined by dimension levels of facts |
| **Event Fact** | Models real-world events (e.g., sales) |
| **Snapshot Fact** | Models state at point in time (e.g., inventory) |
| **Measureless Fact** | Event fact without measures |
| **Additive Measure** | Can aggregate along any dimension |
| **Semi-Additive Measure** | Cannot aggregate along some dimensions (usually Time) |
| **Non-Additive Measure** | Cannot aggregate along any dimension |
| **Star Schema** | One dimension table per dimension, one fact table |
| **Snowflake Schema** | Multiple dimension tables per dimension (one per level) |
| **Surrogate Key** | "Dumb" integer key without semantics |
| **Level Property** | Non-hierarchical information for a level |
| **Conformed Dimension** | Identical dimensions or one subset of other |
| **Conformed Measure** | Equivalently defined measures across cubes |

### Important Acronyms

| Acronym | Full Form | Meaning |
|---------|-----------|---------|
| **OLAP** | On-Line Analytical Processing | Fast, interactive analysis with multidimensional view |
| **OLTP** | On-Line Transaction Processing | Traditional transactional systems |
| **ROLAP** | Relational OLAP | Uses relational DBMS technology |
| **MOLAP** | Multidimensional OLAP | Uses array-type technologies |
| **HOLAP** | Hybrid OLAP | Combines ROLAP and MOLAP |
| **ETL** | Extract-Transform-Load | Process feeding data warehouse |
| **BI** | Business Intelligence | Intelligent business decisions via analysis |
| **KPI** | Key Performance Indicator | Important business metric |
| **MDX** | MultiDimensional eXpressions | Multidimensional query language |

---

## Comparison Tables

### OLAP vs OLTP

| Characteristic | OLAP | OLTP |
|----------------|------|------|
| **Purpose** | Analytical processing | Transaction processing |
| **Focus** | Large data analyses | Single transactions |
| **Users** | Analysts, managers | Operational staff |
| **Data scope** | Historical, aggregated | Current, detailed |
| **Query type** | Complex, aggregating | Simple, targeted |
| **Query size** | Read many records | Read/write few records |
| **Updates** | Periodic batch loads | Frequent, concurrent |
| **Schema** | Denormalized (star/snowflake) | Normalized |
| **Response time** | Seconds (interactive) | Milliseconds |
| **Data volume** | Large (GB-TB) | Moderate (MB-GB) |
| **Optimization** | Query performance | Transaction throughput |

### Star Schema vs Snowflake Schema

| Aspect | Star Schema | Snowflake Schema |
|--------|-------------|------------------|
| **Structure** | One table per dimension | Multiple tables per dimension (one per level) |
| **Normalization** | Denormalized | Normalized |
| **Redundancy** | Present (higher levels duplicated) | Avoided |
| **Storage** | More space (1-5% overhead) | Less space |
| **Query complexity** | Simpler (fewer joins) | More complex (many joins) |
| **Query performance** | Faster | Slower |
| **Hierarchy visibility** | Implicit | Explicit |
| **Maintenance** | Easier (fewer tables) | Harder (more tables) |
| **Best for** | Query performance | Storage efficiency |

### Types of Facts

| Type | Description | Examples | Existence Pattern |
|------|-------------|----------|-------------------|
| **Event Fact** | Models real-world events | Sales, clicks, warehouse flow | Unpredictable (as events occur) |
| **Snapshot Fact** | Models state at point in time | Inventory levels, user count | Predefined times (e.g., monthly) |
| **Measureless Fact** | Event without numerical measure | Customer contacts, attendance | Unpredictable (as events occur) |

### Types of Measures

| Type | Aggregation Behavior | Example | Typical Fact Type |
|------|---------------------|---------|-------------------|
| **Additive** | Can sum along ALL dimensions | Sales count, revenue | Any |
| **Semi-Additive** | Can sum along SOME dimensions (not Time) | Inventory level | Snapshot |
| **Non-Additive** | Cannot sum along ANY dimension | Average price, ratios | Any |

**Why Semi-Additive Fails on Time:**
- Same inventory item counted multiple times across time
- Example: 10 books on shelf counted on Day 1 and Day 2 ≠ 20 books total

### Dimension Design Approaches

| Approach | Definition | Build Order | Key Concept | Advantage | Disadvantage |
|----------|------------|-------------|-------------|-----------|--------------|
| **Top-Down (Inmon)** | Build DW first, derive data marts | DW → DMs | Integration first | Strong integration | Lengthy, late ROI |
| **Bottom-Up** | Build data marts first, merge to DW | DMs → DW | Quick wins | Fast ROI | Poor integration |
| **Conformed (Kimball)** | Standardize shared facts/dimensions | Both | Compatibility | Best of both | Requires coordination |

---

## Important Concepts Summary

### The Multidimensional Model

**Core Principle:** Separate facts (what happened) from dimensions (context)

**Why It Works:**
1. Natural way to think about business problems
2. Easy to add context without changing facts
3. Supports flexible analysis at multiple granularities
4. Optimized for read-heavy analytical workloads

### Kimball's Four-Step Process

```
1. Choose Business Process → What to analyze?
2. Choose Granularity → At what level of detail?
3. Design Dimensions → What context is needed?
4. Choose Measures → What numbers to track?
```

**Key Insight:** Granularity decision (Step 2) is critical
- Affects storage, performance, and analytical flexibility
- General rule: Use finest granularity unless there's compelling reason not to

### FASMI Requirements for OLAP

```
F - Fast: Response in seconds
A - Analysis: Business logic + statistical support
S - Shared: Multi-user with security
M - Multidimensional: Hierarchical dimensions required
I - Information: Manage data + derived info
```

### Inmon's Data Warehouse Characteristics

```
S - Subject oriented: Designed around business subjects
I - Integrated: Unified from multiple sources
T - Time variant: Historical, shows evolution
N - Non-volatile: No updates/deletes, only additions
```

### Analysis Operations Hierarchy

```
Dimension Level Operations:
├── Roll-up (less detail, go up hierarchy)
├── Drill-down (more detail, go down hierarchy)
└── Drill-out (add dimension)

Cube Operations:
├── Slice (select specific dimension value)
├── Dice (complex subsetting)
└── Drill-across (combine cubes)

Presentation:
├── Pivot (rotate dimensions)
└── Rank (order by measure)
```

### Best Practices

**1. Dimension Design:**
- Include as much relevant context as possible
- Use surrogate keys (not business keys)
- Denormalize for query performance (star schema)
- Use textual descriptions (not cryptic codes)

**2. Fact Design:**
- Choose appropriate granularity (usually finest possible)
- Distinguish event vs. snapshot facts
- Classify measures (additive/semi-additive/non-additive)

**3. Integration:**
- Use conformed dimensions across cubes
- Use conformed measures for comparable results
- Requires organizational coordination

**4. ETL Process:**
- Critical for data quality
- Major part of any DW project
- Extract, cleanse, transform, integrate, load

---

## Exam Preparation Tips

### High-Priority Topics

1. **Core Concepts:**
   - What is multidimensional model? (facts + dimensions)
   - OLAP vs. OLTP differences
   - Data warehouse characteristics (SITN)
   - Cube structure (cells, facts, measures, dimensions)

2. **Dimensions:**
   - Hierarchies and levels
   - Multiple hierarchies (Fiscal vs. Calendar)
   - Balanced tree requirement
   - Level properties

3. **Facts and Measures:**
   - Event vs. snapshot facts
   - Additive vs. semi-additive vs. non-additive measures
   - Granularity concept
   - Why measures matter

4. **Relational Representations:**
   - Star schema structure and benefits
   - Snowflake schema structure and trade-offs
   - Surrogate keys advantages
   - When to use each schema type

5. **Analysis Operations:**
   - Roll-up vs. drill-down
   - Drill-out vs. drill-across
   - Slicing and dicing
   - Pivot tables

6. **Modeling Process:**
   - Kimball's 4-step process
   - Conformed dimensions and measures
   - Top-down vs. bottom-up approaches

7. **Query Languages:**
   - SQL CUBE operator
   - Window functions
   - MDX basics (axes, members, hierarchies)

### Common Question Types

**Conceptual:**
- "Explain the difference between OLAP and OLTP"
- "Why are spreadsheets inadequate for multidimensional data?"
- "What are Inmon's four characteristics of a data warehouse?"
- "Explain the concept of granularity"

**Technical:**
- "Design a star schema for [business scenario]"
- "Identify whether a measure is additive, semi-additive, or non-additive"
- "Classify a fact as event or snapshot"
- "Draw a dimension hierarchy for [domain]"

**Applied:**
- "Write a SQL CUBE query for [analysis]"
- "Explain how to perform [operation] on a cube"
- "Compare star and snowflake schemas for [scenario]"
- "Design dimensions for [business process]"

**Analysis:**
- "Given cube, show result after roll-up to [level]"
- "Given two cubes, explain drill-across operation"
- "Show pivot table for [data]"

### Memory Aids

**FASMI (OLAP Requirements):** **F**ast **A**nalysis of **S**hared **M**ultidimensional **I**nformation

**SITN (Data Warehouse):** **S**ubject-oriented, **I**ntegrated, **T**ime-variant, **N**on-volatile

**Four-Step Process:** **B**usiness → **G**ranularity → **D**imensions → **M**easures (BGDM)

**ETL:** **E**xtract → **T**ransform → **L**oad

**Measure Types:** **A**dditive (all), **S**emi-additive (some), **N**on-additive (none) (ASN)

**ROLAP, MOLAP, HOLAP:** **R**elational, **M**ultidimensional, **H**ybrid

### What to Memorize

**Must Know:**
- OLAP vs. OLTP differences (at least 4-5 key differences)
- FASMI definition
- Inmon's 4 DW characteristics
- Kimball's 4-step modeling process
- Types of measures (additive, semi-additive, non-additive) + examples
- Types of facts (event, snapshot) + examples
- Star vs. snowflake schema differences
- Main analysis operations (roll-up, drill-down, slice, dice)
- ETL process

**Good to Know:**
- Multiple hierarchies in dimensions
- Surrogate key advantages (at least 3)
- Conformed dimensions/measures
- Top-down vs. bottom-up approaches
- Window functions concept
- MDX basic structure
- Dashboard purpose

### Common Pitfalls

- Don't confuse OLAP with OLTP
- Don't say "factless facts" - say "measureless facts"
- Remember: Semi-additive usually fails on Time dimension
- Don't forget: Dimensions typically 1-5% of storage
- Star schema has redundancy (it's a feature, not a bug!)
- T level = top of every dimension, represents "all"
- Drill-across ≠ join (it's more like full outer join)
- Conformed dimensions must match on values AND attributes
- Event facts: unpredictable timing; Snapshot facts: predefined timing

---

## Practice Questions

### Question 1: OLAP vs OLTP
**Q:** A book retailer has two systems: one tracks inventory (updates when books sold/received), another analyzes sales trends over time. Classify each as OLAP or OLTP and explain why.

**A:**
- **Inventory system = OLTP**:
  - Focuses on individual transactions (single book sale/receipt)
  - Frequent updates to current data
  - Normalized schema for avoiding update anomalies
  - Small, fast queries ("decrease book B stock by 1")

- **Sales analysis = OLAP**:
  - Focuses on large-scale trend analysis
  - Historical data over time
  - Denormalized schema (multidimensional)
  - Complex queries aggregating many records ("average monthly sales by genre")

### Question 2: Measure Classification
**Q:** Classify each measure as additive, semi-additive, or non-additive:
- a) Total sales revenue per store per day
- b) Inventory level per product per warehouse per day
- c) Average price per product per region per month
- d) Number of transactions per customer per year

**A:**
- **a) Additive**: Can sum across stores (total company revenue), across days (total for period), across all dimensions meaningfully. Each sale is independent.

- **b) Semi-additive**: Can sum across products (total warehouse inventory) and across warehouses (total product inventory), but NOT across time. Same inventory item would be counted multiple times.

- **c) Non-additive**: Cannot sum averages meaningfully. Average of averages ≠ overall average (need weights/counts).

- **d) Additive**: Can sum across customers (total company transactions), across years (total customer lifetime transactions), across all dimensions.

### Question 3: Star vs Snowflake
**Q:** A company is deciding between star and snowflake schemas. They have:
- 10 million fact records
- 5 dimensions with 3-4 levels each
- Queries run every few minutes
- Monthly batch loads
- Storage cost is not primary concern

Which schema should they choose and why?

**A:** **Star schema** is better because:

1. **Query performance priority**: Queries run frequently (every few minutes), so fast response is critical. Star schema has fewer joins.

2. **Redundancy cost is low**:
   - Dimensions typically 1-5% of storage
   - With 10M facts, dimension redundancy is negligible
   - Storage cost not primary concern

3. **Simple queries**: Fewer joins = easier to write, optimize, and maintain

4. **Batch updates**: Monthly loads mean update performance is not critical, so normalization benefit of snowflake is minimal

5. **ETL manages consistency**: Central load process ensures redundant data stays consistent

**When snowflake would be better**: If storage was critical, or if dimension updates were very frequent and required normalized form.

### Question 4: Drill-Across
**Q:** A retailer has two cubes:
- **Store Sales**: dimensions (Product, Store, Date), measure (units sold)
- **Online Sales**: dimensions (Product, Customer, Date), measure (units sold)

Product and Date are conformed dimensions. Explain how to find the top 3 best-selling products across both channels.

**A:**

1. **Identify shared dimensions**: Product and Date (both conformed)

2. **Non-shared dimensions**: Store (Store Sales), Customer (Online Sales)
   - Roll both up to T level before combining

3. **Drill-across operation**:
   - Combine cubes via Product and Date
   - Result has dimensions: Product and Date
   - Measures: StoreUnitsSold, OnlineUnitsSold

4. **Calculate total**: Create derived measure
   - TotalUnits = StoreUnitsSold + OnlineUnitsSold

5. **Roll-up Date to T**: Aggregate across all time
   - Now have: Product dimension only with TotalUnits measure

6. **Rank and select top 3**: Order by TotalUnits DESC, take first 3

**Why drill-across is necessary**: Can't just find top 3 in each cube separately - a product selling well in both channels might be moderate in each individually but best overall.

### Question 5: Granularity Decision
**Q:** A retail chain (200 stores) wants to analyze sales. Each store sells ~1000 books/day, with ~250 different titles sold per store per day. Compare two granularity options:

- **Option A**: Individual sale level (Book, Store, Timestamp, Customer)
- **Option B**: Aggregated (Book, Store, Date)

Discuss trade-offs.

**A:**

**Option A (Individual Sales):**
- **Facts per day**: 200 stores × 1000 sales = 200,000 facts/day
- **Advantages**:
  - Maximum analytical flexibility
  - Can analyze by hour, minute, second
  - Can do customer-level analysis
  - Can discover purchasing patterns (data mining)
  - Can aggregate to any coarser grain later
- **Disadvantages**:
  - 200K facts/day = ~73M facts/year
  - Much more storage required
  - Slower query performance (more data to scan)

**Option B (Aggregated):**
- **Facts per day**: 200 stores × 250 titles = 50,000 facts/day (75% reduction!)
- **Advantages**:
  - Better query performance (less data)
  - 75% less storage needed
  - Sufficient for many analyses (daily trends, store comparisons)
- **Disadvantages**:
  - Cannot analyze by time of day
  - Cannot do customer-level analysis
  - Cannot discover purchasing patterns
  - Cannot drill down to individual sales
  - Lost information is unrecoverable

**Recommendation**: Use **Option A** (individual sales) unless:
- Storage/performance constraints are severe
- Time-of-day and customer analysis are definitely not needed
- Data mining is not a goal

**General principle**: "In general, the finest possible granularity should be used"

### Question 6: SQL CUBE Query
**Q:** Given table:
```
Sales (City, Book, Revenue)
```

Write a SQL CUBE query to show:
- Revenue per city and book
- Subtotals per city
- Subtotals per book
- Grand total

**A:**

```sql
SELECT City, Book, SUM(Revenue) AS TotalRevenue
FROM Sales
GROUP BY CUBE (City, Book)
ORDER BY City, Book
```

**Result format:**
```
City      Book      TotalRevenue
-------   -------   ------------
Boston    Book A    1000
Boston    Book B    1500
Boston    ALL       2500        -- subtotal per city
Miami     Book A    800
Miami     Book B    1200
Miami     ALL       2000        -- subtotal per city
ALL       Book A    1800        -- subtotal per book
ALL       Book B    2700        -- subtotal per book
ALL       ALL       4500        -- grand total
```

**Note**: ALL is typically implemented as NULL (with special function to distinguish from real NULLs)

---

## Additional Resources

**Books:**
- Ralph Kimball: "The Data Warehouse Toolkit"
- Bill Inmon: "Building the Data Warehouse"

**Standards:**
- SQL:1999 (CUBE operator)
- SQL:2003 (Window functions)
- MDX specification

**Systems:**
- Microsoft SQL Server Analysis Services (SSAS)
- Oracle OLAP
- IBM Cognos
- SAP BusinessObjects
- Open-source: Mondrian, Apache Kylin

---

## Quick Reference Card

### Must-Know for Exam

| Category | Key Points |
|----------|------------|
| **Multidimensional Model** | Facts (events) + Measures (numbers) + Dimensions (context) |
| **OLAP vs OLTP** | Analysis vs Transactions; Historical vs Current; Read-heavy vs Write-heavy; Denormalized vs Normalized |
| **FASMI** | Fast, Analysis, Shared, Multidimensional, Information |
| **Data Warehouse (Inmon)** | Subject-oriented, Integrated, Time-variant, Non-volatile |
| **Fact Types** | Event (unpredictable events), Snapshot (predefined state), Measureless (no measure) |
| **Measure Types** | Additive (all dims), Semi-additive (not Time), Non-additive (none) |
| **Star Schema** | 1 dim table/dim, 1 fact table; redundant but fast; dimensions 1-5% storage |
| **Snowflake Schema** | Multiple tables/dim (1 per level); normalized; slower queries, more joins |
| **Granularity** | Level of detail; determines what fact means; finer = more flexible but more storage |
| **Roll-Up** | Go UP hierarchy (less detail); aggregate measures |
| **Drill-Down** | Go DOWN hierarchy (more detail); expand measures |
| **Drill-Across** | Combine cubes via shared (conformed) dimensions |
| **Slice/Dice** | Select subset of cube; slice = 1 value, dice = complex condition |
| **ETL** | Extract-Transform-Load; feeds data warehouse from operational systems |
| **Kimball Process** | 1) Business process, 2) Granularity, 3) Dimensions, 4) Measures |
| **Conformed** | Dimensions/measures identical or subset across cubes |
| **SQL Extensions** | CUBE operator (subtotals), Window functions (aggregates per row) |
| **MDX** | Produces pivot tables; axes (COLUMNS, ROWS); WHERE = slicer |

---

**Good luck with your exam!**
