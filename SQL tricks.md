# SQL patterns based on past problems

# Execution order

FROM/JOIN -> WHERE -> GROUP BY -> HAVING -> SELECT -> DISTINCT -> ORDER BY -> LIMIT/OFFSET

---

- For NULL checks, use IS/IS NOT NULL (never = NULL or <> NULL, which fail silently).

- Derived tables (created in FROM) are isolated and cannot be re-used or opened by subqueries. Use CTEs (WITH clause) if you need global access to a temporary set.

- When using **window** functions, tie the aggregate function to the OVER statement and apply any further functions on the whole statement.

  FUNC (AGG_FUNC OVER (PARTITION BY .... ORDER BY ....))

- For pattern matching there are two options:
  
  - LIKE: Provides only two options: % indicates any number of characters, and _ indicates a single character.
    
    WHERE string LIKE ....
  - REGEXP: Provides many more options to specify characters. Advanced matching
  
    A-Za-z0-9 : Uppercase, lowercase and digits

    \b : Word boundary marker
    
    ^ : Placed at the start of regex = Start of string.
    
    $ : Placed at the end of regex = End of string.
    
    [^...] : Placed inside brackets = NOT these characters.
    
    . : Wildcard, matches any character. However, a single dot will represent a single character, and does not extend to infinite characters.
    
    + : One or more instances | * : Zero or more instances
    
    \ : Used as an escape character, usually put when regexp syntax characters need to be matched. Double backslash is needed in MySQL

    WHERE string REGEXP .....

  - To enforce exact case, we can use the **BINARY** keyword with REGEXP.

- **GROUP_CONCAT**:
Aggregates strings from multiple rows into a single comma-separated text field for a GROUP BY bucket.

- If we want the primary key for some aggregate value (like MAX or MIN) a more robust approach is order by and limit. Avoid partial aggregation.

- Conditional aggregation

  When we want to apply any aggregate function to a single column without filtering the rest of the rows, we can use conditional aggregation.
  
  That implies using COUNT/SUM/AVG with CASE WHEN within it, which checks for equality or the required condition.

- When joining, there is a high possibility of occurence of rows with NULL values, in such cases, we can use the COALESCE function. This function will replace those nulls with the specified value.

---

# Pandas

- We can split a list of strings into multiple rows using the EXPLODE 
function.

- Using .groupby().agg() will lead to shrinking the dataframe. To maintain dataframe shape, we can use .groupby.transform('agg').  It will replicate that same aggregated value for all rows belonging to that group.

---

- **RANK() and DENSE_RANK()**

  Both are window functions used to rank based on certain value. However, the distinction is how they handle ties. For ties, RANK() will assign the same rank to the value, and skip the subsequent ranks. DENSE_RANK() will also assign the same rank, but will not skip any ranks.

- Finding duplicates in SQL

  - Using GROUP BY and HAVING
  
    Group by the non-unique column, and use HAVING to filter out the duplicates.

  - Self Join

- If we want to check the aggregation of another value based on some temporal variable, check if MIN() or MAX() can be used to check the conditions for the temporal var.

- Dual-Ranking for First/Last Comparisons (The Student Problem)
  - The Technique: Use ROW_NUMBER() twice over the same partition—once sorted ascending (ORDER BY date) and once sorted descending (ORDER BY date DESC).

  - Conditional Aggregation: Use MAX(CASE WHEN forward_no = 1 THEN value END) to pivot the first and last entries into a single row per group.

  - The Hidden Trick: Filtering with HAVING first_value < last_value serves a dual purpose: it guarantees an increase and inherently filters out groups with only one entry (since a single entry makes both ranks equal 1, and a value cannot be strictly less than itself).

- Multi-Group Top-1 Analysis (The Seasons Problem)
  - The UNION Syntax Rule: If you use ORDER BY and LIMIT inside individual SELECT queries combined by a UNION, you must wrap each individual query in parentheses (...). Otherwise, the SQL parser throws an error.

  - Stacked CTE Optimization: Instead of running a UNION on 4 separate queries (which forces the database to scan the underlying tables 4 separate times), you can use a stacked CTE approach:

    CTE 1 (Aggregate): Group by the main categories and calculate totals.

    CTE 2 (Rank): Use ROW_NUMBER() OVER (PARTITION BY group_var ORDER BY total DESC) to rank them in memory.

  - Main Query (Filter): Select where rank = 1. This scans the heavy tables exactly once.

- **Gaps and Islands Problem**
  
  - How to identify: Will involve something related to **streaks**

  - How to solve: 

    Maintain two pointers: 
    
      - One which is over a group partitioned by the primary key

      - One which is partitioned within a group over the variable which determines the streak

    Both pointers usually arranged chronologically
    
    Get the difference between the two pointers and group over that difference and you will get some intermediate result which can be used in the final answer.


    The Two Types of Streak Problems

      - Event-Based Streaks: A change in status/state breaks the streak (e.g., a "Loss" breaks a "Win" streak). Gaps in time are ignored. 

      - Time-Based Streaks: The absence of an event breaks the streak (e.g., failing to log in on Tuesday breaks a daily streak).

    Time-Based Islands (Consecutive Dates)
      - How to identify: The prompt asks for consecutive days, weeks, or months (e.g., "Longest daily login streak").

      - How to solve (The Date Math Method):
      
        Generate a single sequential pointer (ROW_NUMBER() or rank) partitioned by the user and ordered chronologically.

        Subtract that pointer (as days) from the actual Date column: Date - Row Number = Streak ID.

        Because both the date and the row number increase by 1 for consecutive days, subtracting them yields a constant, identical "reference date" for the entire sequence. Group by this reference date to measure the streak.

    Finding Gaps (Return Windows / Missing Time)
      - How to identify: The prompt asks for time between events, inactivity periods, or return metrics (e.g., "Users who made a second purchase within 7 days" or "Find users who went inactive for 3+ days").

      - How to solve (The Shift/Lead Method):

        Use a window function (LEAD() in SQL, .shift(-1) in Pandas) strictly partitioned by the primary key (User ID) and ordered chronologically.

        This pulls the Next Event Date onto the exact same row as the Current Event Date.

        Calculate the difference: Next Date - Current Date.

        Filter based on the difference (e.g., > 1 means a calendar day was skipped; <= 7 means they returned within a week).


Using multiple exists clauses:
Instead of that, use IN for WHERE, followed by GROUP BY and making sure the count is correct.
OR Intersection of sets
OR join.


