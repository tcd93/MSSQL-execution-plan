# MSSQL Plan Optimizer
An article is an introduction to Microsoft SQL Server's plan optimizer & common operators, and provide an more in-depth view & analysis of slow queries (with example)

_note: I don't provide sample data here as it's private & pretty huge, but even if you don't run these data yourself, you should have a pretty good grasp of 
SQL Server's optimizer after reading this_

Tool: Microsoft Sql Server Management Tool (MSSM)

---

## Execution Plan & Optimizer
### What is an Execution Plan?
![](img/2021-03-01-09-54-56.png)

An execution plan is a set of physical operations (operators) that can be performed to produce the required result

The data flow order is from right to left, the thickness of the arrow indicate the amount of data compared to the entire plan; hovering on the icons show extra details

![](img/2021-03-01-09-55-13.png)

### Retrieving the estimated plan
Retrieving the estimated plan is just telling SQL Server to return the execution plan without actually executing it, helpful in debugging

From MSSM (Microsoft SQL Server Management tool), select an SQL block:
- Press `CTRL + L`
- or: Right click → Display estimated execution plan
- or Query → Display estimated execution plan

### Retrieving the actual plan
In the query menu, tick the “Include Actual Execution Plan” icon

![](img/2021-03-01-09-53-33.png)

Select & run the query, the plan will be open on a new tab next to result tab

### Estimated vs. Actual
They can differ in cases where the query involves parallelism, variable, hints, current CPU usage… Actual execution plan contains extra runtime information, such as the actual usage metrics (memory grant, actual rows, executions…), and any runtime warnings

<div>
    <figure style="display:inline-block;margin-left:0;">
        <img src="img/2021-03-01-10-03-45.png"></img>
        <figcaption style="font-size:80%;font-style:italic;">Estimated</figcaption>
    </figure>
    vs
    <figure style="display:inline-block;">
        <img src="img/2021-03-01-10-04-03.png"></img>
        <figcaption style="font-size:80%;font-style:italic;">Actual</figcaption>
    </figure>
</div>

Actual Plan also include the number of rows processed by each thread, runtime memory allocation...

<div>
    <figure style="display:inline-block;margin-left:0;">
        <img src="img/2021-03-01-10-11-17.png"></img>
    </figure>
    <figure style="display:inline-block;">
        <img src="img/2021-03-01-10-11-09.png"></img>
    </figure>
</div>


## Query Processor
What happens when a query is submitted?

<img style="background:white" src="img/2021-03-01-10-18-10.png"/>

The algebrizer resolves all the names of various objects, tables, and columns referred to within the query string. It identifies at the individual column level, all the data types (varchar, datetime…) for the objects being accessed. It also determines the location of aggregates (SUM, MAX…)

The algebrizer outputs a binary tree which gives the optimizer knowledge of the logical query structure and the underlying tables and indexes, the output also includes a hash representing the query, the optimizer uses it to see if there is already a plan for this stored in plan cache & whether it’s still valid, if there’s one, then the process stops and __the cached plan is reused__, if not, then it'll compile out an execution plan based on __*statistics*__ & __*cost*__

Once the query is optimized, the generated execution plan may be stored in the plan cache and be executed step-by-step by the physical operators in that plan

### Cost of the plan
The _estimated cost_ is based on a complex mathematical model, and it considers various factors, such as cardinality, row size, expected memory usage and number of sequential and random I/O operations, parallelism overhead… 

__*This number is meaningless outside of the query optimizer's context and should be used for comparison only*__

![](img/2021-03-01-11-38-28.png)
- _Operator Cost_: Cost taken by the operator
- _Subtree Cost_: Cumulative cost associated with the whole subtree up to the node

### Ways to select a plan
The query optimizer finds a number of candidate execution plans for a given query, estimates the cost of each of these plans and selects the plan with the lowest cost. 

For some queries, the optimizer cannot consider every possible plan for every query, it actually has to consider both the cost of finding potential plans and the costs of plans themselves

![](img/2021-03-01-12-42-44.png)
![](img/2021-03-01-12-42-54.png)

### Plan cache
Whenever a query is run for the first time in SQL Server, it is compiled and a query plan is generated for the query. Every query requires a query plan before it is actually executed. This query plan is stored in SQL Server query plan cache, when that query is run again, SQL Server doesn’t need to create another query plan

The duration that a query plan stays in the plan cache depends upon how often a query is executed. Query plans that are used more often, stay in the query plan cache for longer durations, and vice-versa

_Cache is not used when specific [hints](https://docs.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-query?view=sql-server-ver15) are specified (RECOMPILE hint)_

## Statistics
Why is it important?

### The data of data
- The statistics contain information about tables and indexes such as number of rows, the histogram, the density of values from a sample of data; these values are stored in system tables 
- Costs are generated based on statistics, if the stats are incorrect or out-of-date (stale), cost will be wrongly calculated, and the optimizer may choose a sub-optimal plan
- Statistics can be updated automatically, periodically, or manually

### Histogram
_Histogram_ measures the frequency of occurrence for each distinct value in a data set

To create the histogram, SQL server split the data into different buckets (called steps) based on the value of first column of the index. Each record in the output is called as bucket or step

The maximum number of bucket is 200, this can cause problems for larger set of data, where there can be points of _skewed data distributions_, leading to un-optimized plans for special ranges

<blockquote style="font-size:80%">

For example, customer A usually makes 5 purchases per week, but suddenly, at a special day (like Black Friday), he made over 10000 transactions, that huge spike might not get captured in the transaction bucket, and the query for that week would likely get much slower than normal as the optimizer'd still think he makes 
very little purchases in that week

</blockquote>

In MSSM, expand _Table > Statistics > Double click a stat name_; some stat names are auto-generated, some are user-defined 

![](img/2021-03-01-13-00-42.png)

This is a sample histogram of column `MasterID` from _Customer_ table:

![](img/2021-03-01-13-03-01.png)

Explanation for the 4th bucket:
* `RANGE_ROWS`: There are 2861 rows with keys from 4183 - 62833
* `EQ_ROWS`: There are 272 rows with key 62834
* `DISTINCT_RANGE_ROWS`: There are 47 distinct rows with keys from 4183 - 62833

Now if we selects 30% of the 4th bucket (`21778` = `(62833 - 4183) * 0.3 + 4183`):
```sql
SELECT * FROM Customer WHERE MasterID BETWEEN 4183 AND 21778
```

This is the generated plan:

![](img/2021-03-01-13-23-21.png)

There are 2861 rows from ID 4183 - 62833, so if we’re selecting 30% of that range, it should also results in 30% of 2861 which is 858 rows, that’s the estimated number of the optimizer

### Density
_Density_ is the ratio of unique values with in the given column or a set of columns

![](img/2021-03-01-13-41-06.png)

Let's go with this query:
```sql
DECLARE @N INT = 4178
SELECT * FROM Customer WHERE MasterID = @N
```

![](img/2021-03-01-13-44-53.png)

_Histogram cannot be used when we're using paramater_, it then falls back to Density, which is estimated as `Total rows * Density` = `1357786 * 2.020488E-05` = `27.43` rows - but in actualality there is 2134 rows! (as showed in Histogram `EQ_ROWS` attribute). Optimizer failed pretty hard there 🤔

### Memory Grant
- Memory Grant value (kb) can only be seen in Actual execution mode
- This memory is used to store temporary rows for sort, hash join & [parallelism exchange operators](###-parallelism-operators)
- SQL Server calculates this based on statistics, lack of available memory grant causes a tempdb spill ([tempDB](https://docs.microsoft.com/en-us/sql/relational-databases/databases/tempdb-database?view=sql-server-ver15) is a global resource that is used to stores all temporary objects)

<blockquote style="font-size:80%">

In SQL server 2012+, a yellow warning icon is displayed in plan explorer when the processor detects a spill (not enough RAM to store data)

For SQL server 2008R2, check the “sort warnings” event in [SQL profiler](https://www.sqlshack.com/an-overview-of-the-sql-server-profiler/) to detect memory spill

</blockquote>

#### TempDB Spill
By adding a “order by” clause to the above example, we can produce a _sort warnings_ event in SQL Profiler

![](img/2021-03-01-14-00-23.png)

![](img/2021-03-01-14-03-49.png)

The engine only granted 1136 Kb of memory buffer to perform sorting, but in reality the operation needed way more because actual rows are much higher than estimated returned rows, so the input data has to be split into smaller chunks in _tempDB_ to accommodate the granted space to be sorted, then extra passes are performed to merge these sorted chunks

To fix this, we can simply add the __RECOMPILE__ hint to the query, this forces the parse to replace the `@N` parameter with actual value, therefore correctly using the Histogram table

---

## A little about B+Tree Index
Index is a set of ordered values stored in 8kb pages, the pages form a B+tree structure, and the value contains pointer to the pages in the next level of the tree

The pages at the leaf nodes can be data pages (clustered index) or index pages (nonclustered index)

Clustered index (CI) is the table itself, 1 table can only have 1 CI; NonCI’s leaf may refer to the CI’s key, so _any changes to the CI’s key will force changes to every NonCI’s structures_

![](img/2021-03-01-14-47-26.png)

With scan, we need to scan 6 pages to reach key 28, whereas going top-down (seek), we just need to read 2 index pages and 1 data page (3 logical/physical reads = 3 * 8kb = 24kb)

Seek & scan can be combined, where a seek happens first to find where to start scanning, this is still displayed as an _index seek_ operator in plan view

---

## Common Operators
### Sort
<table>
    <tr>
        <th>Icon</th>
        <th>Name</th>
        <th>Description</th>
    </tr>
    <tr>
        <td><img src="img/2021-03-01-14-14-50.png" style="width:50px;height:50px;"/></td>
        <td>Sort</td>
        <td>Reads all input rows, sorts them, and then returns them in the specified order</td>
    </tr>
</table>

Sort is a _blocking_ operation, it has to read all data into RAM, and sort it. It is both time & memory consuming

If the data is too big for granted memory, a [spill](####-tempdb-spill) happens, making Sort less efficient

### Data Retrievers
<table>
    <tr>
        <th>Icon</th>
        <th>Name</th>
        <th>Description</th>
    </tr>
    <tr>
        <td><img src="img/2021-03-01-14-22-15.png" style="width:50px;height:50px;"/></td>
        <td>Index seek / Non-clustered index seek</td>
        <td>Finds a specific row in an index, based on key value; and optionally continues to scan from there in logical (index) order</td>
    </tr>
    <tr>
        <td><img src="img/2021-03-01-14-22-16.png" style="width:50px;height:50px;"/></td>
        <td>Index scan / Non-clustered index scan</td>
        <td>Reads all data from an index, either in <a href="https://sqlperformance.com/2015/01/t-sql-queries/allocation-order-scans">allocation order</a> or in logical (index) order</td>
    </tr>
    <tr>
        <td><img src="img/2021-03-01-14-26-54.png" style="width:50px;height:50px;"/></td>
        <td>Key lookup</td>
        <td>Reads a single row from a clustered index, based on a key value that was retrieved from a non-clustered index on the same table. A Key lookup is a very expensive operation because it performs a random I/O into the clustered index. For every row of the non-clustered index, SQL Server has to go to the Clustered Index to read their data. We can take advantage of knowing this to improve the query performance</td>
    </tr>
    <tr>
        <td><img src="img/2021-03-01-14-27-28.png" style="width:50px;height:50px;"/></td>
        <td>Table scan</td>
        <td>Reads all data from a heap table, in allocation order</td>
    </tr>
</table>

### Joins / Aggregator
<table>
    <tr>
        <th>Icon</th>
        <th>Name</th>
        <th>Description</th>
    </tr>
    <tr>
        <td><img src="img/2021-03-01-14-53-57.png" style="width:50px;height:50px;"/></td>
        <td>Hash match/aggregate</td>
        <td>Builds a hash table from its first input, then uses that hash table to either join to its second input, or produce aggregated values</td>
    </tr>
    <tr>
        <td><img src="img/2021-03-01-14-54-32.png" style="width:50px;height:50px;"/></td>
        <td>Merge join</td>
        <td>Joins two inputs that are ordered by the join key(s), exploiting the known sort order for optimal processing efficiency</td>
    </tr>
    <tr>
        <td><img src="img/2021-03-01-14-55-06.png" style="width:50px;height:50px;"/></td>
        <td>Stream aggregate</td>
        <td>Computes aggregation results by reading a sorted input stream and returning a single row for each set of rows with the same key value</td>
    </tr>
    <tr>
        <td><img src="img/2021-03-01-14-55-55.png" style="width:50px;height:50px;"/></td>
        <td>Nested loop</td>
        <td>Joins two inputs by repeatedly executing the second input for each row in the first input</td>
    </tr>
</table>

### Parallelism operators
<table>
    <tr>
        <th>Icon</th>
        <th>Name</th>
        <th>Description</th>
    </tr>
    <tr>
        <td><img src="img/2021-03-01-15-01-35.png" style="width:50px;height:50px;"/></td>
        <td>Distribute streams</td>
        <td rowspan="3">The parallelism operators, also known as exchange operators, manage the distribution of rows between threads in parallel plans</td>
    </tr>
    <tr>
        <td><img src="img/2021-03-01-15-02-20.png" style="width:50px;height:50px;"/></td>
        <td>Repartition streams</td>
    </tr>
    <tr>
        <td><img src="img/2021-03-01-15-02-50.png" style="width:50px;height:50px;"/></td>
        <td>Gather streams</td>
    </tr>
</table>

#### Nested loop
[Source](https://bertwagner.com/posts/visualizing-nested-loops-joins-and-understanding-their-implications/)

<div>
    <span style="display:inline-block;width:500px;">
        <ul>
            <li>O(n.m) / <i>O(nlog(m))</i>*</li>
            <li>Require data sorted: No</li>
            <li>CPU cost: Low</li>
            <li>Memory grant: Maybe</li>
            <li>Spillable: No</li>
            <li>Blocking: No / <i>Semi</i>**</li>
            <li>
                Optimal for:
                <ul>
                    <li>Small outer input → Small/Medium (indexed) inner input</li>
                    <li>Low cardinality data</li> 
                    <li>OLTP</li>
                </ul>
            </li>
        </ul>
    </span>
    <span style="display:inline-block;">
        <figure>
            <img src="https://bertwagner.com/wp-content/uploads/2018/12/Nested-Loop-Join-50fps-1.gif">
            <figcaption style="font-size:80%;font-style:italic;">this gif demonstrates “brute-force” type of NL</figcaption>
        </figure>
    </span>
</div>

<blockquote style="font-size:80%">

(*) SQL Server can use multiple ways to optimize a nested loop (to get Big O of _nlog(m)_ time complexity)
- [Spool](https://sqlserverfast.com/epr/table-spool/#:~:text=The%20Table%20Spool%20operator%20is,operators%20to%20produce%20them%20again.) in inner loop to maximize reusability
- Perform index seek on inner loop
- [Prefetch data in inner loop](#####-nested-loop-prefetching)

(**) Order inner loop implicitly to create Semi-blocking nested loop

</blockquote>

##### Nested loop prefetching (`WithUnorderedPrefetch: True`)
Example plan:

![](img/2021-03-01-15-29-22.png)

Scans `IX_agent` index, for each agent, seek the corresponding customer __asynchronously__ from `IX_custid`, forward the result whenever it's available

<blockquote style="font-size:80%">

When `WithUnorderedPrefetch` is set to False, the seeked result will be forwarded only when the previous ordered key is fetched & forwarded

</blockquote>

##### Optimized nested loop (`Optimized: True`)
Example plan:

![](img/2021-03-01-15-43-34.png)

![](img/2021-03-01-15-48-59.png)

1. Scans `IX_tnx_type` index
2. May implicitly perform an (partial) "_order by_" to create less random seeks; hence the high memory usage
3. If memory does not fit, it’ll fill what it can, so it does not spill

<blockquote style="font-size:80%">

- Although getting just 10 rows, the above plan still requires 189,312 KB of sorting space
- Concurrent runs of above query cause high _RESOURCE_SEMAPHORE_ wait, leading to slower performance (fixed in 2016)
- The sort method & memory grant algorithm is different to a normal sort operator, there’s no guarantee that it is faster than same query without optimization
- This is treated as a “safety net” in case the statistics are out-of-date

</blockquote>

#### Hash match
[Source](https://bertwagner.com/posts/hash-match-join-internals/)

<div>
    <span style="display:inline-block;width:500px;">
        <ul>
            <li>O(n + m)</li>
            <li>Require data sorted: No</li>
            <li>CPU cost: High</li>
            <li>Memory grant: <b>Yes</b></li>
            <li>Spillable: Yes</li>
            <li>Blocking: Yes</li>
            <li>
                Optimal for:
                <ul>
                    <li>Medium build input → Medium/Large probe input</li>
                    <li>Medium/high cardinality data</li>
                </ul>
            </li>
            <li><b>Scales well with parallelism</b></li>
        </ul>
    </span>
    <span style="display:inline-block;">
        <figure>
            <img src="https://bertwagner.com/wp-content/uploads/2018/12/Hash-Match-Join-Looping-1.gif">
        </figure>
    </span>
</div>

#### Merge join
[Source](https://bertwagner.com/posts/visualizing-merge-join-internals-and-understanding-their-implications/)

<div>
    <span style="display:inline-block;width:500px;">
        <ul>
            <li>O(n + m)</li>
            <li>Require data sorted: <b>Yes</b></li>
            <li>CPU cost: Low</li>
            <li>Memory grant: No</li>
            <li>Spillable: No</li>
            <li>Blocking: No</li>
            <li>
                Optimal for:
                <ul>
                    <li>Evenly sized inputs</li>
                    <li><a href="https://sqlserverfast.com/blog/hugo/2017/12/many-many-reads-many-many-merge-join/">One to many</a></li>
                </ul>
            </li>
            <li><b>Scales badly parallelism</b></li>
        </ul>
    </span>
    <span style="display:inline-block;">
        <figure>
            <img src="https://bertwagner.com/wp-content/uploads/2018/12/Merge-Join-1.gif">
        </figure>
    </span>
</div>

#### Making sense of parallel scan

This is the explan plan produced from the following query:
```sql
SELECT [product].id, [tnx_table].amount...
FROM tnx_table
INNER JOIN product
ON [tnx_table].prod_id = [product].id
```

![](img/2021-03-01-16-02-41.png)

First, the engine scans the `IX_prod` index, in parallel, the distribution of rows among threads can be considered as “random”; each time the query runs, each thread will handle different number of rows

![](img/2021-03-01-16-21-38.png)

After scanning, SQL Server repartitions the rows in each thread, arranging them in a __deterministic__ order, rows are now distributed “correctly” among threads; _each time the query runs, each thread will handle same number of rows_

This operator requires some buffer space to do the sorting

Next, it'll allocate some space to create a [bloom filter](https://en.wikipedia.org/wiki/Bloom_filter) (bitmap)

![](img/2021-03-01-16-25-20.png)

When the second index scan starts, it also include a probe action that checks on the bitmap net. If the bit is “0”, that means the key does not exists in the first index, if the bit is “1”, that means the key _may_ exists in the first index and can pass through into repartition streams

![](img/2021-03-01-16-37-55.png)

With bitmap, the actual number of rows after the scan is reduced

With the two sources ready & optimized, the Hash join operation can be done quickly in parallel and finally merged together

Here's a summary (_note that `mod % 2` & `mod % 10` are not actual MS hash function implementation, this is for demontrastion purpose only_):

<img src="img/2021-03-01-16-40-37.png" style="background:white"/>

<blockquote style="font-size:80%">

__Types of scan:__

__Unordered scan__ (Allocation Order Scan) using using internal page allocation information
- Favorable for _Hash match_

__Ordered scan__, the engine will scan the index structure
- Favorable for _Merge join_
- _During order-preserving re-partition exchange, it does not do any sorting_, it just keep the order of output stream the same as the input stream

</blockquote>

#### Comparing Merge & Hash, in parallel plans
This is a side-by-side comparation of a merge join & hash join, both produce same set of records

The query is simple:
```sql
--hash
select f.custid, d.Week, sum(f.ActualStake) 
from Fact f
inner join DimDate d
on f.RptDate = d.PK_Date
-- where d.PK_Date >= '2020-01-01' (uncomment to get merge join plan)
group by f.custid, d.Week
```

Merge join plan is evaludated by adding a where clause filter by date, the optimizer will now go for _index seek_ in the `DimDate` table, but `2020-01-01` is way lower than the actual data range in `Fact` table, so both queries produce same result

<div>
    <figure style="display:inline-block;margin-left:0;">
        <img src="img/2021-03-01-17-05-12.png"></img>
        <figcaption style="font-size:80%;font-style:italic;">Merge</figcaption>
    </figure>
    vs
    <figure style="display:inline-block;">
        <img src="img/2021-03-01-17-05-27.png"></img>
        <figcaption style="font-size:80%;font-style:italic;">Hash</figcaption>
    </figure>
</div>

Since the _index seek_ generate an ordered result set, optimizer tries to make use of an _merge join_ plan, but data from `Fact` table's _clustered index scan_ are not yet ordered, the engine do it implicitly in the _ordered repartition streams_ operator, thus giving very high cost comprared to the _hash join_ one

<blockquote style="font-size:80%">
    We can keep track of these symptoms by monitoring the CXPACKET & SLEEP_TASK wait types (for SQL Server 2008)
</blockquote>

**Where the fun begins**

Now, put the system CPU under load (by running many queries at same time using __SQL Stress Test__), the _merge join_ becomes slower the more threads used, whereas _hash join_'s performance is very consistent

__Why?__ 

In _merge_, the order-preserving exchange operator has to run sequentially to get pages from the scan, so at this point it is actually running in _single thread_ mode, and when the CPU is under pressure, it’ll have to wait upto 4ms (a _quantum_, see [SQLOS](https://blog.sqlauthority.com/2015/11/11/sql-server-what-is-sql-server-operating-system/)) to get the next batch of pages

In _hash_, at no point the execution is done syncronously, parallel execution is used at 100% power, so it is very effective