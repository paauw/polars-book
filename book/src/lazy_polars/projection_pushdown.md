# Projection pushdown

Let's expand our query from the previous section by joining the result of the *FILTER* operation with the runescape data
to see which popular Reddit username that have a username starting with an a also played Runescape. That must be something
we are all interested in!

The query that does so may look like this.

```python
{{#include ../examples/lazy_chapter/projection_pushdown_0.py:4:17}}
joined.fetch(int(1e7))
```
And yields the following DataFrame.
```text
{{#include ../outputs/projection_pushdown_0.txt}}
```

## Break it down
Again, let's take a look the query plan. 

``` python
joined.show_graph(optimized=False)
```

![query_plan](../img/projection_pushdown_0.png)

Now were focussed on the projection's indicated with π. The first node shows π 3/6, indicating that
we select 3 out of 6 columns in the DataFrame. If we look the csv scans we see a wildcard π */6 and π */1 meaning that 
we select all of 6 columns of the reddit dataset and the one and only column from the runescape dataset respectively.

This query is not very optimal. We select all columns from both datasets and only show 3/6 after join. That means that
there were some columns computed during the join operation that could have been ignored. There were also columns parsed
during csv scanning only to be dropped at the end. When we are dealing with DataFrame's with a large number of columns the
redundant work that is done can be huge.

### Optimized query
Let's see how Polars optimizes this query.

``` python
joined.show_graph(optimized=True)
```

![query_plan_opt](../img/projection_pushdown_0_optimized.png)

The projections are pushed down the join operation all the way to the csv scans. This  means that both the scanning and 
join operation have become cheaper due to the query optimization.

## Performance

Let's time the result before and after optimization.

**without optimization**

`$ time time python -m book.src.examples.lazy_chapter.projection_pushdown_0_timing False`

```text
real	0m3,273s
user	0m9,284s
sys	0m1,081s
```


**with optimization**

`$ time time python -m book.src.examples.lazy_chapter.projection_pushdown_0_timing True`

```text
real	0m1,732s
user	0m7,581s
sys	0m0,783s
```

We can see that we almost reduced query time by half on this simple query. With real business data often comprising of 
many column, filtering missing data, doing complex groupby and joins we expect this difference between unoptimized queries
and optimized queries to only grow.

