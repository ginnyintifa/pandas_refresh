# pandas + NumPy Notes

## `object` dtype

In pandas, `object` dtype means the column holds Python objects — in practice, almost always **strings**.

Pandas doesn't have a dedicated string dtype by default, so any column with text gets stored as `object`.

```python
df.dtypes
# name      object   ← strings
# age        int64
# score    float64
```

You may also see `object` if a column has **mixed types** (e.g. some ints, some strings) — pandas falls back to `object` when it can't pick a single numeric type.

---

## `df[mask, 'col']` doesn't work — use `.loc`

Plain `[]` only accepts one thing at a time — a mask **or** a column name, not both:

```python
df[mask]          # rows — ok
df['col']         # column — ok
df[mask, 'col']   # TypeError — can't do both at once
```

`.loc` has two slots separated by a comma — rows first, columns second:

```python
df.loc[mask, 'col']   # correct
```

**Single vs double brackets in `.loc`:**
```python
df.loc[mask, 'col']     # single brackets → Series
df.loc[mask, ['col']]   # double brackets → DataFrame (one column)
```

This matters when passing to `np.mean()` — it expects a Series, not a DataFrame:
```python
np.mean(res.loc[res['is_api'], 'status_code'] >= 400)    # scalar ✓
np.mean(res.loc[res['is_api'], ['status_code']] >= 400)  # Series with one value ✗
```

---

## `.loc` vs `.iloc` — label vs position

`.loc` is **label-based** — the row selector can be an index label, a boolean mask, or a slice of labels. The second argument selects columns by name:

```python
df.loc[5, 'price']                          # row with label 5, column 'price'
df.loc[df['cut'] == 'Ideal', 'price']       # boolean mask + column name
df.loc['Ideal']                             # all columns for rows where index label == 'Ideal'
```

`.iloc` is **position-based** — always uses integer positions (0, 1, 2...):

```python
df.iloc[0]        # first row by position
df.iloc[0, 2]     # first row, third column
```

**Common mistake:** after `set_index('cut')`, the index is now cut names — `df.loc[1]` raises a `KeyError` because `1` is not a label. Use `df.iloc[0]` for the first row by position.

**`.loc` slicing requires the labels to exist and be in order** — `df.loc[0:5]` finds label `0` and label `5` in the index and returns everything between them. After sorting or filtering, the integer labels get shuffled, so `loc[0:5]` can return an empty DataFrame or unexpected rows:

```python
df = df.sort_values('price', ascending=False)
df.loc[0:5]    # label 0 may now be in the middle — unpredictable result
df.iloc[0:5]   # always the first 5 rows by position — use this after sorting
```

To use `.loc` safely after sorting, use the actual index labels:
```python
df.loc[df.index[0:5]]   # gets the labels of the first 5 rows, then selects by label
```

**Can't mix slice and column name in `[]`:**
```python
planets[0:5, 'method']    # TypeError — doesn't work

planets.iloc[0:5]['method']     # correct
planets.loc[0:5, 'method']      # correct (when index is default integers)
```

**`set_index` doesn't modify in place** — always assign the result:
```python
d = diamonds.set_index('cut')   # correct
diamonds.set_index('cut')       # does nothing — result is discarded
```

**Prefer `.loc` over chained indexing** — `df[mask]['col']` works for reading but can cause `SettingWithCopyWarning` on assignment. `df.loc[mask, 'col']` selects rows and column in one step and is always safe:

```python
# Chained — avoid
mpg1.loc[mpg1['cylinders'] > 4]['origin']

# Correct — filter rows and select column in one step
mpg1.loc[mpg1['cylinders'] > 4, 'origin']
```

---

## Negating a boolean — `~` vs `not` vs `!`

| Operator | Use case |
|---|---|
| `~` | Negate a **boolean Series or array** (element-wise) |
| `not` | Negate a **single scalar** boolean |
| `!` | Doesn't exist in Python |

```python
# Series — use ~
~df['is_alone']
~(df['fare'] > 50)

# Scalar — use not
not True          # False

# Inside .apply() lambda — scalars, so use not
df['col'].apply(lambda x: 'yes' if not x else 'no')
```

`!` is from JavaScript/C — it does not work in Python.

---

## Boolean filtering — operator precedence

`&` and `|` bind tighter than comparison operators like `>`, `<`, `==`. Always wrap each condition in parentheses:

```python
# Wrong — evaluates as: products['rating'] > (4.0 & products['in_stock'])
products[products['rating'] > 4.0 & products['in_stock']]

# Correct
products[(products['rating'] > 4.0) & (products['in_stock'])]
```

---

## Counting nulls and finding NaN locations

```python
df.isnull().sum()        # nulls per column (default)
df.isnull().sum(axis=1)  # nulls per row
```

**Dropping rows with nulls — `dropna()`:**

```python
df.dropna()                                        # drop rows with NaN in ANY column
df.dropna(subset=['orbital_period', 'mass'])       # drop if EITHER column is NaN (default: how='any')
df.dropna(subset=['orbital_period', 'mass'], how='all')  # drop only if BOTH are NaN
```

**Finding the index labels of NaN values:**

```python
# Series — index labels where value is NaN:
s.index[s.isna()]

# DataFrame — which rows have any NaN:
df.index[df.isna().any(axis=1)]

# DataFrame — which columns have any NaN:
df.columns[df.isna().any(axis=0)]

# DataFrame — exact (row, col) pairs with NaN (useful for pivot tables):
[(r, c) for r in df.index for c in df.columns if pd.isna(df.loc[r, c])]
```

---

## NumPy functions vs array methods

Some operations are available both as NumPy functions and as array methods — they do the same thing:

```python
arr.mean()       # method
np.mean(arr)     # function — same result
```

Basic stats available as both: `mean`, `sum`, `std`, `min`, `max`

More advanced operations are function-only (no array method):
```python
np.median(arr)
np.percentile(arr, 75)
np.unique(arr)
np.where(condition, true_val, false_val)
```

---

## NaN-safe NumPy functions

When working with plain NumPy arrays that contain NaN, use the `nan`-prefixed versions:

```python
np.nanmean(arr)
np.nanmedian(arr)
np.nanstd(arr)
```

These ignore NaN values instead of returning nan.

**Series vs plain array — important difference:**

```python
np.mean(df['col'])             # skips NaN — Series handles it
np.mean(df['col'].to_numpy())  # returns NaN — plain array doesn't
```

Converting to a plain array loses pandas' NaN-handling behavior.

---

## `np.where` — compact if/else for arrays

Works like a vectorized if/else — operates on the entire array at once:

```python
np.where(condition, value_if_true, value_if_false)

# Example
np.where(sensors['reading'] > 80, True, False)
```

Similar idea to lambda + apply, but faster because it's vectorized:
```python
# lambda equivalent (slower, element by element)
sensors['reading'].apply(lambda x: True if x > 80 else False)
```

---

## `pd.cut()` — right-inclusive bins

`pd.cut()` bins are **right-inclusive** (closed on the right) by default:

```python
pd.cut(age, bins=[0, 17, 35, 60, 100], labels=['Minor', 'Young Adult', 'Adult', 'Senior'])
# bins are (0, 17], (17, 35], (35, 60], (60, 100]
# age=17 → 'Minor', age=35 → 'Young Adult'
```

The left edge is excluded, the right edge is included. Keep this in mind when setting bin boundaries.

Use `np.inf` as the upper bound when the last bin has no upper limit:

```python
pd.cut(df['age'], bins=[0, 18, 35, 60, np.inf], labels=['Minor', 'Young', 'Adult', 'Senior'])
# last bin is (60, ∞] — captures everything above 60
```

---

## `pd.pivot_table()` default aggfunc

`pd.pivot_table()` uses `aggfunc='mean'` by default — you don't need to specify it unless you want something else:

```python
pd.pivot_table(df, values='tip_pct', index='sex', columns='day')               # mean (default)
pd.pivot_table(df, values='tip_pct', index='sex', columns='day', aggfunc='sum')
pd.pivot_table(df, values='tip_pct', index='sex', columns='day', aggfunc='count')
```

**Counting rows — `'count'` vs `'size'`:**

```python
aggfunc='count'   # counts non-NaN values in the `values` column — skips NaN
aggfunc='size'    # counts all rows in each group, including NaN
```

Use `'size'` when you want the true number of rows per group. Use `'count'` when you want to know how many rows have a non-null value. They give the same result if your `values` column has no nulls.

`groupby()` can produce the same values but returns a Series instead of a 2D table:

```python
df.groupby(['sex', 'day'])['tip_pct'].mean()                       # Series — same numbers, flat format
pd.pivot_table(df, values='tip_pct', index='sex', columns='day')   # 2D table — easier to read
```

---

## `pivot_table()` — NaN handling

**NaN values in the data** are ignored by default when computing the mean (equivalent to `skipna=True`). There's no `skipna` argument — to change this, pass a custom `aggfunc`:

```python
tips.pivot_table(index='day', columns='time', values='tip_pct',
                 aggfunc=lambda x: x.mean(skipna=False))
```

**`dropna` parameter** — controls whether all-NaN *columns* are kept in the output, not whether NaNs in the data are skipped:

```python
pivot_table(..., dropna=True)   # default — drops columns where every cell is NaN
pivot_table(..., dropna=False)  # keeps those all-NaN columns
```

**`fill_value` parameter** — replaces NaN *result* cells after aggregation (doesn't affect the mean calculation):

```python
pivot_table(..., fill_value=0)
```

---

## `groupby().transform()` vs `groupby().apply()`

Both operate group-by-group, but they return different shapes:

| | `apply()` | `transform()` |
|---|---|---|
| Output size | One row per group (reduced) | Same length as original DataFrame |
| Use case | Summarize each group | Add a per-group value back to each row |

**`apply()` — reduces to one result per group:**
```python
df.groupby('category')['price'].apply(np.mean)
# category
# A    15.0
# B    22.0
# dtype: float64
```

**`transform()` — broadcasts the group result back to every row:**
```python
df['category_avg'] = df.groupby('category')['price'].transform('mean')
# Every row gets its group's mean — same length as df
```

`transform()` is the right tool when you want to add a group-level stat as a new column (e.g. group mean, group rank, normalized values within a group). Use `apply()` when you want a summary.

---

## `observed=True` — when and where to use it

When a `category` column is used to group or reshape data, pandas by default includes **all possible categories** — even ones with no data. This causes phantom empty rows/columns and a `FutureWarning`.

**Rule: add `observed=True` any time a `category` column is used as a grouping key.**

This applies to three places:

**`groupby()`:**
```python
df.groupby('size', observed=True)['price'].mean()   # only real groups
```

**`pivot_table()`** — when `index` or `columns` are categorical:
```python
df.pivot_table(index='cut', columns='color', values='price', observed=True)
```

**`transform()` inside `groupby()`:**
```python
df.groupby('sex', observed=True)['tip'].transform('mean')
```

Without `observed=True` you get NaN rows/columns for categories that don't appear in the data, plus a `FutureWarning` that will become an error in a future pandas version.

---

## Mapping a grouped result back to a DataFrame

After a `groupby().apply()`, the result is a Series indexed by the group key. Use `.map()` to add it back as a column:

```python
gpa = merged.groupby('student_id').apply(
    lambda g: np.dot(g['score'], g['credits']) / g['credits'].sum()
)

students['gpa'] = students['student_id'].map(gpa)
```

`.map(series)` looks up each value in `student_id` against the Series index and fills in the corresponding value.

---

## When to use `.to_numpy()`

Usually you don't need it — NumPy functions and pandas operations accept Series directly.

Only convert when:
- A library requires a plain array
- Using `np.dot()` or matrix operations where the index causes issues
- Something breaks and you suspect the index is the cause
- **Positional indexing with an array of indices** (e.g. after `np.argsort()`)

**Label vs positional indexing:**
```python
aa = np.argsort(sales['revenue'])

sales['rep'][aa]            # label-based — works only if index is 0,1,2,3...
sales['rep'].to_numpy()[aa] # positional — always reliable
```

If the DataFrame has been filtered and the index is no longer `0, 1, 2...`, the first version gives wrong results or a KeyError. Always use `.to_numpy()` when indexing with an array of positions.

**Concrete example — why it breaks:**
```python
df = pd.DataFrame({
    'name':   ['Alice', 'Bob', 'Carol', 'Dave'],
    'salary': [3000, 1000, 4000, 2000],
}, index=[10, 11, 12, 13])  # non-default index

aa = np.argsort(df['salary'])  # → [1, 3, 0, 2]  (positions)

df['name'][aa]            # looks for labels 1, 3, 0, 2 → KeyError!
df['name'].to_numpy()[aa] # looks for positions 1, 3, 0, 2 → correct
```

---

## MultiIndex — set, sort, and slice

**Creating a MultiIndex:**
```python
df = df.set_index(['region', 'quarter']).sort_index()
# region and quarter become index levels, no longer columns
```

**Retrieving rows with `.loc`:**
```python
df.loc['West']                    # all rows where first level == 'West'
df.loc[('East', 'Q3')]            # exact match on both levels
df.loc['C':'S']                   # range on first level (label-inclusive)
df.loc[('East', 'Q3'), 'revenue'] # both levels + column
```

**Slicing by second level** — use `pd.IndexSlice` or `slice(None)` for "all" on a level:
```python
idx = pd.IndexSlice
df.loc[idx[:, 'Q3'], :]            # all Q3 rows across every region (readable)
df.loc[(slice(None), 'Q3'), :]     # same thing, without pd.IndexSlice
```

**`.loc` vs `.xs()` on a MultiIndex:**
- `s.loc[('C', 1)]` — selects that exact row, keeps all columns
- `s.xs(('C', 1))` — same result, but `.xs()` also lets you specify `level=` to slice a non-first level more easily

Use `.loc` when you know the exact label(s). Use `.xs()` when you want to slice by a specific level that isn't the first one.

**Mean per group using NumPy:**
```python
for region in df.index.get_level_values('region').unique():
    print(region, np.mean(df.loc[region, 'revenue']))
```

---

## Re-grouping a MultiIndex result with `groupby(level=)`

After a multi-column `groupby`, the result has a **MultiIndex**. You can group it again using `level=` to operate within each top-level group.

**Use case — find the best category per region:**
```python
# Step 1: sum revenue by region+category → MultiIndex Series
totals = ecommerce.groupby(['region', 'category'])['revenue'].sum()

# Step 2: re-group by level=0 (region), then idxmax within each region
totals.groupby(level=0).idxmax()
# North    ('North', 'Electronics')
# South    ('South', 'Home')
# ...
```

- `level=0` → first index level (region)
- `level=1` → second index level (category)

Compare to just `.idxmax()` with no re-groupby — that returns a single global max tuple, not one per region.

---

## `groupby().agg()` with all numeric columns

`groupby().agg()` fails on non-numeric columns (e.g. strings). Use `select_dtypes` to limit to numeric columns first:

```python
numeric_cols = df.select_dtypes(include='number').columns
df.groupby('species')[numeric_cols].agg(['mean', 'std'])
```

**What doesn't work:**
```python
df.groupby('species').agg(['mean', 'std'])                       # TypeError on object columns
df.groupby('species').agg(['mean', 'std'], numeric_only=True)    # numeric_only not a valid agg kwarg
```

**Different stat per column — pass a dict of tuples:**

```python
t = planets.groupby('method', observed=True).agg(
    total_planets=('number', 'sum'),
    mean_distance=('distance', 'mean'),
    median_orbital_period=('orbital_period', 'median')
)
```

The keys are the **new column names**, the values are tuples of `(source_column, aggregation)`. Result has flat columns — no multi-level — so filtering is straightforward:

```python
t.loc[t['total_planets'] > 100]
```

Compare to passing a list like `['sum', 'mean']` — that applies every stat to every column and gives a multi-level column structure: `t[('number', 'sum')]`.

---

## Avoiding SettingWithCopyWarning — `.copy()`

When you slice a DataFrame (e.g. with `.xs()`, `.loc[]`, boolean filter), the result may be a **view**, not a new DataFrame. Assigning to it raises a `SettingWithCopyWarning`:

```python
q3 = sales_q.xs('Q3', level='quarter')
q3['total'] = q3['revenue'] * q3['units']  # SettingWithCopyWarning
```

Fix: call `.copy()` to make it an independent DataFrame:

```python
q3 = sales_q.xs('Q3', level='quarter').copy()
q3['total'] = q3['revenue'] * q3['units']  # no warning
```

---

## `.stack()` returns a Series, not a DataFrame

`.stack()` folds columns into rows — the result is a **Series** with a MultiIndex, not a DataFrame:

```python
energy            # DataFrame — buildings as index, months as columns
energy_long = energy.stack()   # Series — index is (building, month), values are numbers
```

Because it's a Series (one column of values), you can't add new columns to it directly. Use `.to_frame()` first:

```python
energy_long = energy.stack().to_frame(name='kwh')   # now a DataFrame
energy_long['log_kwh'] = np.log(energy_long['kwh']) # can add columns now
```

---

## `groupby('name')` vs `groupby(level='name')` — columns vs index levels

When you call `groupby('city')`, pandas looks for a **column** named `'city'` first. If it doesn't find one, it falls back to checking index level names.

This means `groupby('city')` can accidentally work after `set_index(['city', 'month'])` — because there's no longer a column called `'city'`, so pandas falls back to the index level.

Use `groupby(level='city')` to be explicit and unambiguous:

```python
# After set_index(['city', 'month']):
weather.groupby('city')['temp'].max()         # works by coincidence — city is no longer a column
weather.groupby(level='city')['temp'].max()   # explicit — always means the index level
```

The difference matters if a column and an index level share the same name — pandas would group by the column, not the level.

---

## `sort_values()` vs `sort_index()`

- **`sort_values()`** — sorts by the **data** in a column
- **`sort_index()`** — sorts by the **index labels**

```python
s = pd.Series([30, 10, 20], index=['b', 'c', 'a'])

s.sort_values()
# c    10
# a    20
# b    30

s.sort_index()
# a    20
# b    30
# c    10
```

**Common use case:** after a `groupby`, the index is the group keys — `sort_index()` alphabetizes the groups, `sort_values()` ranks them by their aggregated value.

---

## DateTime operations with `.dt`

Convert a string column to datetime, then use the `.dt` accessor to extract parts:

```python
df['hire_date'] = pd.to_datetime(df['hire_date'])

df['year']  = df['hire_date'].dt.year
df['month'] = df['hire_date'].dt.month
```

**Calculating days since a date:**

```python
df['tenure_days'] = (pd.Timestamp('today') - df['hire_date']).dt.days
```

Subtracting two datetime columns gives a `Timedelta` — `.dt.days` converts it to an integer.

**Summarise with NumPy:**

```python
np.mean(df['tenure_days'])
np.median(df['tenure_days'])
```

**Extracting hours from a Timedelta Series — use `.dt.total_seconds()`:**

Subtracting two datetime columns gives a Timedelta Series. To get the difference in hours, you need `.dt` (required for any time component on a Series) and `.total_seconds()` — `.hour` doesn't exist on Timedelta:

```python
# Wrong — .hour doesn't exist on Timedelta, and .dt is missing
(flights['arrive_utc'] - flights['depart_utc']).hour       # AttributeError

# Correct
(flights['arrive_utc'] - flights['depart_utc']).dt.total_seconds() / 3600
```

---

This also happens after `.dropna()`, `.drop_duplicates()`, or any filter — the index keeps the original row numbers, so it's no longer `0, 1, 2...`:
```python
df = df[df['salary'] > 1500]
# index is now [10, 12, 13] — df['name'][aa] will break
```

**Assigning a Series with non-matching index to a DataFrame column:**
```python
# w has index ['Alice', 'Bob', 'Carol'] (from groupby key)
# employees has index [0, 1, 2]
employees['score'] = w            # index mismatch → NaN
employees['score'] = w.to_numpy() # assigns by position → correct
```

This happens whenever the Series index doesn't match the DataFrame index — common after groupby.

---

## FutureWarning — what it means

pandas releases new versions over time and sometimes changes how things behave. A `FutureWarning` means:

> "This currently works, but we're going to change it in a future version. Update your code now."

If you upgrade pandas without fixing the warning, your code will break.

**Common example — positional vs label indexing:**

```python
g['month'][np.argmax(g['revenue'])]  # FutureWarning — ambiguous: position or label?
```

`np.argmax` returns an integer position. Passing it to `[]` is ambiguous — pandas currently treats it as a position, but will treat it as a label in future.

Fix by being explicit:

```python
g['month'].iloc[np.argmax(g['revenue'])]   # clearly positional
g['month'].to_numpy()[np.argmax(g['revenue'])]  # outside pandas, no ambiguity
```

**Using `np.argmax` on a grouped result to get back a label:**

```python
means = df.groupby('hotel')['occupancy'].mean()
means.index[np.argmax(means)]   # returns the label (e.g. 'Grand'), not the position
```

This is the `np.argmax` equivalent of `.idxmax()` — use it when you want the label of the max group.

---

## `.apply()` — if/else inside a lambda

Inside `.apply()`, each value is a **scalar**, so use `if/else` — not `np.where`:

```python
# Series-level (element-wise):
df['col'].apply(lambda x: 'Premium' if x > 50 else 'Standard')

# Row-level (axis=1), multi-condition:
df.apply(lambda row: 'High'   if row['age'] > 60 and row['severity'] > 7
                else 'Medium' if row['age'] > 60 or  row['severity'] > 7
                else 'Low', axis=1)
```

`&`/`|` also work on boolean scalars and give correct results — but `and`/`or` is the more natural Python style inside a lambda.

`np.where` is for whole Series/arrays — don't nest it inside `.apply()`.

**When to use `.apply()` with `DateOffset` vs vectorized:**

`DateOffset` works element-wise on a Series — use vectorized when the offset is fixed:
```python
df['review_date'] = df['deadline'] - pd.DateOffset(weeks=2)   # fixed offset — no .apply() needed
```

Use `.apply(axis=1)` only when the offset varies per row:
```python
df['deadline'] = df.apply(lambda row: row['start'] + pd.DateOffset(weeks=row['duration']), axis=1)
```

---

## `.str.contains()` and `.str.extract()` — regex string matching

**`.str.contains(pattern)`** — returns a boolean Series: True where the string matches.
```python
df['name'].str.contains('ford')                  # substring match
df['name'].str.contains('ford', case=False)      # case-insensitive
df['name'].str.contains(r'^toyota')              # regex: starts with 'toyota'
df['name'].str.contains(r'pickup|truck')         # either word
df['col'].str.contains(r'A|B|C', na=False)       # na=False treats NaN as non-match
```

**`.str.extract(pattern)`** — pulls out the part matching a capture group `()`.
```python
df['col'].str.extract(r'(\d+)')[0]       # extract digits: 'Model-42' → '42'
df['col'].str.extract(r'^(\w+)')         # first word
df['name'].str.extract(r'([A-Z][a-z]+\.)') # title like 'Mr.', 'Mrs.'
```

**Capture group `()` — when it matters:**
```python
# .str.contains() — () is optional, makes no difference to results
df['deck'].str.contains(r'A|B|C', na=False)
df['deck'].str.contains(r'(A|B|C)', na=False)   # identical output

# .str.extract() — () is required, tells pandas what to return
df['deck'].str.extract(r'(A|B|C)')   # works
df['deck'].str.extract(r'A|B|C')     # error — no capture group
```

**Extracting everything after a character:**
```python
df['email'].str.extract(r'@(.+)')[0]   # 'alice@gmail.com' → 'gmail.com'
```
`@` matches the literal `@`, `(.+)` captures everything after it.

**Non-capturing group `(?:...)`** — group for logic without returning in the result:
```python
r'(GET|POST) (\S+)'    # two capture groups → extract returns two columns
r'(?:GET|POST) (\S+)'  # one capture group  → extract returns one column (the path)
```
Use `(?:...)` when you need `|` or grouping for matching, but don't want that part in the output. `\S` means any non-whitespace character.

**Raw strings `r''` — always use for regex:**
```python
r'\d+'   # correct — \d means digit
'\\d+'   # same but harder to read
'\d+'    # works by accident for \d, but breaks for \n, \t etc
```

**`r''` vs `''` only differs when backslashes are involved** — `r'A|B|C'` and `'A|B|C'` are identical.

**`.map()` for code → label mapping** (not `.str.extract()`):
```python
port_map = {'S': 'Southampton', 'C': 'Cherbourg', 'Q': 'Queenstown'}
df['port'] = df['embarked'].map(port_map)
```
`.str.extract()` extracts substrings — it can't rename them. Use `.map()` when you have a fixed lookup table.

---

## `.str` — slicing and extracting substrings

`.str` applies string/sequence operations element-wise across a Series. It works on both strings and lists — so after `.str.split()`, each element is a list, and `.str[i]` picks index `i` from each one:

```python
files['path'].str.split('/').str[2]   # grab index 2 from each resulting list
```

This is the vectorized shorthand for `.apply(lambda x: x[2])`.

`.str` also supports slicing on strings directly:

```python
s.str[1:]    # drop the first character: 'E03' → '03'
s.str[:3]    # first 3 characters
```

To extract digits (or any pattern) with regex:

```python
s.str.extract(r'(\d+)')[0]   # pull out the numeric part: 'E03' → '03'
```

Use slicing when the structure is fixed (e.g. always one prefix character). Use `.str.extract()` when the position varies and you need a pattern match.

**Vectorized string concatenation with `+`:**

pandas overloads `+` for string Series — it concatenates element-wise, no loop needed:

```python
df['island_species'] = df['island'] + '_' + df['species']
# e.g. 'Torgersen' + '_' + 'Adelie' → 'Torgersen_Adelie'
```

This only works when the columns are `object` dtype. For `category` columns, convert first:
```python
df['island'].astype(str) + '_' + df['species'].astype(str)
```

**`(?:...)` — when you actually need it:**

`(?:)` is a non-capturing group. You don't need it just to skip over a literal prefix — write the prefix inline:

```python
r'status=(\d+)'       # fine — literal prefix, one capture group
r'(?:status=)(\d+)'   # unnecessary here
```

Use `(?:)` when you need grouping for logic, not just to match text:
```python
r'(?:foo|bar)(\d+)'   # alternation — group foo|bar without capturing
r'(?:ab)+'            # repeat a multi-char sequence
```

**`\S+` — match non-whitespace:**

Useful when extracting a value that runs until the next space:
```python
r'endpoint=(\S+)'   # captures everything after 'endpoint=' up to first space
```
Cleaner than `(.+)(?: +)`, which is greedy and can leave trailing spaces in the captured value.

---

## `len()` vs `.str.len()`

In base Python, `len()` is a built-in function — always call it as `len(x)`, never as a method:

```python
len("hello")      # 5
len([1, 2, 3])    # 3
```

`.len()` only exists as part of the pandas string accessor:

```python
df['col'].str.len()                  # length of each string in a Series
df['col'].str.split().str.len()      # length of each resulting list
```

Never write `.len()` on a plain Python object — it will raise `AttributeError`.

## `.str` and `.contains()` — pandas only, not plain Python strings

`.str` and `.contains()` only exist on pandas Series. Plain Python strings use different syntax:

| Task | Plain Python string | pandas Series |
|------|-------------------|---------------|
| Check substring | `'score' in cn` | `s.str.contains('score')` |
| Get length | `len(cn)` | `s.str.len()` |
| Slice | `cn[:3]` | `s.str[:3]` |

Common mistake — iterating over columns gives plain strings, not a Series:
```python
# WRONG — cn is a plain string, not a Series
[cn for cn in df.columns if cn.str.contains('score')]   # AttributeError

# CORRECT
[cn for cn in df.columns if 'score' in cn]
```

---

## Regex — pandas `.str` vs plain Python `re`

The regex syntax (patterns) is identical in both. The difference is the API.

**Plain Python** — `re` module, one string at a time:
```python
import re

s = 'TXN-2024-001-USD'
re.search(r'(\d{4})', s).group(1)   # '2024'
re.findall(r'\d+', s)               # ['2024', '001']
re.sub(r'\d+', 'X', s)             # 'TXN-X-X-X'
```

**pandas** — `.str` accessor, entire Series at once (vectorized):
```python
df['col'].str.extract(r'(\d{4})')    # DataFrame, one col per capture group
df['col'].str.contains(r'\d+')       # boolean Series
df['col'].str.replace(r'\d+', 'X')  # string Series
```

Use `re` when working with a single string outside a DataFrame. Use `.str` methods inside a DataFrame — no loop needed.

---

## `pivot` vs `pivot_table`

**`pivot`** — simple reshaping, no aggregation:
- Each (index, column) combination must be **unique** — no duplicates allowed
- Just restructures the data, no math involved

```python
df.pivot(index='date', columns='city', values='temp')
```

**`pivot_table`** — when you need to **aggregate** duplicates:
- Multiple rows share the same (index, column) pair → needs `aggfunc` to combine them
- Defaults to `aggfunc='mean'`

```python
df.pivot_table(index='region', columns='category', values='sales', aggfunc='sum')
```

**Quick rule:** Ask "could there be duplicate (index, column) pairs in my data?"
- No → `pivot`
- Yes → `pivot_table`

If you use `pivot` on duplicate data, it raises a `ValueError`.

---

## `.xs()` — cross-section of a MultiIndex

`.xs()` pulls out a slice of a MultiIndex DataFrame or Series by specifying a value at a given level:

```python
df.xs('Mar', level=1)        # all rows where the second index level == 'Mar'
df.xs('Mar', level='month')  # same, using the level name
```

After `web.stack()`, the MultiIndex is `(country, month)`:
```python
traffic_long = web.stack().to_frame('visits')
traffic_long.xs('Mar', level=1)        # all March rows across every country
traffic_long.xs('Brazil', level=0)     # all rows for Brazil across every month
```

`.xs()` is cleaner than `pd.IndexSlice` when you're slicing a single value from one level and want all values from the other level.

**Selecting both levels at once with a tuple:**
```python
# Two chained .xs() calls:
df.xs(2, level='pclass').xs('female', level='sex')

# Same thing in one call — pass a tuple:
df.xs((2, 'female'))
```

The tuple version is shorter when you want a specific combination across multiple levels.

**`.xs()` requires a MultiIndex — it doesn't work on plain column indexes:**
```python
# sales_wide has a plain column index (month names) — .xs() won't work
sales_wide.xs('Mar', axis=1)   # TypeError: Index must be a MultiIndex

# Just use direct column selection instead
sales_wide['Mar']              # correct
```

`.xs(key, axis=1)` only makes sense when the *columns* form a MultiIndex (e.g. after `stack()` + `unstack()` produces multi-level columns).

---

## Survival rate with `groupby` — use `.mean()`

For a 0/1 column like `survived`, `.mean()` gives the fraction directly — no need for `.apply()`:

```python
titanic.groupby(['sex', 'pclass'])['survived'].mean()
```

Avoid this pattern:
```python
titanic.groupby(['sex', 'pclass']).apply(lambda x: x['survived'].sum() / x.agg('count'))
```

`x.agg('count')` returns a count *per column*, so dividing a scalar by a Series duplicates the survival rate across every column — the output looks like a full DataFrame but is just noise.

---

## `.unstack(level)` — choosing which level becomes columns

With no argument, `.unstack()` moves the **last (innermost)** index level into the columns by default.

Pass a level name or position to control which level moves:

```python
# After groupby + stack, MultiIndex is (species, measurement)
s = iris.groupby('species')[cols].mean().stack()

s.unstack('species')      # → rows = measurements, columns = species
s.unstack('measurement')  # → rows = species, columns = measurements (same as default)
s.unstack()               # → moves last level (measurement) into columns — same as above
```

Use the level name to be explicit about what you want in the rows vs columns.

---

## `category` dtype and the `.cat` accessor

Convert a column to save memory when it has many repeated values:
```python
df['col'] = df['col'].astype('category')
```

The `.cat` accessor gives you category-specific tools:
```python
df['col'].cat.categories   # the unique values (Index)
df['col'].cat.codes        # integer code for each row (Series)
```

**Getting a category → code mapping:**
```python
# Most idiomatic:
{cat: code for code, cat in enumerate(df['col'].cat.categories)}
# e.g. {'Biscoe': 0, 'Dream': 1, 'Torgersen': 2}
```

`cat.categories` only gives names — `enumerate` generates the matching code numbers (0, 1, 2...) since pandas assigns codes in the same order as `cat.categories`.

**Ordered categories** — enable `<`, `>` comparisons:
```python
dtype = pd.CategoricalDtype(categories=['Fair', 'Good', 'Very Good', 'Premium', 'Ideal'], ordered=True)
df['cut'] = df['cut'].astype(dtype)
df[df['cut'] > 'Good']   # returns Very Good, Premium, Ideal
```

The order is left = lowest, right = highest — exactly as you list them.

**Removing unused categories after filtering:**
```python
filtered = df[df['island'] == 'Biscoe']
filtered['species'].cat.categories          # still shows all 3 species
filtered['species'].cat.remove_unused_categories()  # only keeps species that appear
```

**Memory check:**
```python
df.memory_usage(deep=True)   # always use deep=True for accurate string/category sizes
```

---

## List comprehensions

A compact way to build a list using a loop and optional condition — all in one line:

```python
[expression for item in iterable if condition]
```

**Basic examples:**
```python
[x * 2 for x in [1, 2, 3]]              # [2, 4, 6]
[x for x in [1, 2, 3, 4] if x > 2]     # [3, 4]
```

**With a DataFrame — collect column names by dtype:**
```python
[col for col in df.columns if df[col].dtype.name == 'category']
[col for col in df.columns if df[col].dtype == 'object']
```

**With a dict — collect keys by value:**
```python
[k for k, v in my_dict.items() if v < 10]
```

**With null counts — columns with >10% nulls:**
```python
[col for col in df.columns if df[col].isnull().mean() > 0.1]
```

**Filtering rows from two columns in parallel — use `zip`:**
```python
[name for name, band in zip(df['name'], df['band']) if band == 'exec']
```
This pairs each name with its corresponding band value. Without `zip`, `if band == 'exec'` would compare the entire Series to a scalar, raising `ValueError: The truth value of a Series is ambiguous`.

The pattern is always: `[what_to_keep for each_item in something if condition]`.

The `if` part is optional — leave it out to include every item:
```python
[col.upper() for col in df.columns]   # all column names, uppercased
```

Think of it as a compact for loop that collects results into a list:
```python
# Loop version:
result = []
for col in df.columns:
    if df[col].dtype == 'object':
        result.append(col)

# List comprehension version — same thing:
result = [col for col in df.columns if df[col].dtype == 'object']
```

---

## Set comprehensions

Same syntax as a list comprehension but with `{}` — produces a set (unordered, unique values):

```python
{species for species, bc in zip(df['species'], df['bill_class']) if bc == 'wide'}
```

Use when you want unique values and don't care about order.

```python
[...]   # list comprehension  → ordered, allows duplicates
{...}   # set comprehension   → unordered, unique values only
{k: v}  # dict comprehension  → key-value pairs
```

**With a groupby result — filter groups by their aggregated value:**
```python
rates = titanic.groupby('embark_town')['survived'].mean()
{et for et, r in zip(rates.index, rates) if r > 0.5}
```
After groupby, the index holds the group keys — zip `rates.index` with `rates` to pair each town with its survival rate.

---

## Dict comprehensions

Same idea as list comprehensions but builds a dict — use `{key: value for ...}`:

```python
{key_expr: value_expr for item in iterable if condition}
```

**Basic example:**
```python
{x: x**2 for x in [1, 2, 3]}    # {1: 1, 2: 4, 3: 9}
```

**Category → code mapping (most common use):**
```python
{cat: code for code, cat in enumerate(df['col'].cat.categories)}
# e.g. {'Thur': 0, 'Fri': 1, 'Sat': 2, 'Sun': 3}
```

**From an existing dict — filter by value:**
```python
{k: v for k, v in my_dict.items() if v > 10}
```

The key mistake to avoid — you can't use `and` to combine two `for` clauses:
```python
{k: v for k in keys and v in values}   # SyntaxError — wrong
{k: v for k, v in zip(keys, values)}   # correct
```

**Mapping keys → lists of values (grouping):**

Dict comprehensions only work when each key maps to a single value. When you need a key → list mapping, use a plain loop with `.setdefault()`:

```python
result = {}
for plan, sid in zip(subs['plan'], subs['subscriber_id']):
    result.setdefault(plan, []).append(sid)
# {'Basic': [101, 104], 'Pro': [102], 'Enterprise': [103]}
```

`setdefault(key, default)` returns the existing value if the key exists, or inserts `default` and returns it — so `.append()` always works without an `if` check.

---

## When `.copy()` is redundant after a slice

`.copy()` is needed when you get a view from a slice and then assign new columns to it:

```python
# ps might be a view — assigning to it can trigger SettingWithCopyWarning
ps = penguins.loc[penguins['species'] == 'Adelie', ['bill_length_mm', 'bill_depth_mm']]
ps['new_col'] = 0   # warning

# .copy() makes it safe
ps = penguins.loc[...].copy()
ps['new_col'] = 0   # fine
```

But if your chain ends with something that already produces a new DataFrame — `.dropna()`, `.reset_index()`, `.rename()`, etc. — the result is already independent. Adding `.copy()` before it is redundant:

```python
# .dropna() returns a new DataFrame, so ps is not a view — no .copy() needed
ps = penguins.loc[penguins['species'] == 'Adelie', cols].dropna()
ps['new_col'] = 0   # safe
```

**Rule:** only add `.copy()` when the slice itself is the last step and you plan to mutate it afterward.

---

## `.pipe()` modifies the original DataFrame — use `.copy()`

Inside pipe functions, `df['new_col'] = ...` modifies the DataFrame in place. pandas passes the original object, not a copy, so your source DataFrame also changes.

Fix — add `.copy()` before the chain:

```python
res = df.copy().pipe(fn1).pipe(fn2).pipe(fn3)
```

Or copy inside each function:
```python
def add_value(df):
    df = df.copy()
    df['total_value'] = df['units_in_stock'] * df['unit_cost']
    return df
```

The `.copy()` before the chain is cleaner.

---

## `shift(n)` vs `shift(n, freq=...)` — values vs index

These two do fundamentally different things:

**`shift(n)`** — moves the **values** down by `n` rows. The index stays fixed. Leaves `NaN` at the top:
```python
sales.shift(4)
# 2024-01-07    NaN       ← first 4 rows become NaN
# 2024-01-14    NaN
# 2024-01-21    NaN
# 2024-01-28    NaN
# 2024-02-04    120.0     ← value from 4 rows earlier
# ...
```
Use this to align values with a lagged version of themselves — e.g. comparing this week's sales to 4 weeks ago.

**`shift(n, freq='W')`** — moves the **index** forward by `n` weeks. The values don't change. No NaN introduced:
```python
sales.shift(4, freq='W')
# 2024-02-04    120       ← same value, index shifted forward 4 weeks
# 2024-02-11    135
# ...
```
Use this when you want to re-timestamp a copy of the data — e.g. creating a `lagged` DataFrame that can be joined back to the original on its index.

**Key difference — join behavior:**
```python
sales.join(lagged, lsuffix='_now', rsuffix='_4w_ago')
```
With `freq=` shift, `lagged`'s index lines up with the original's index 4 weeks later, so each row shows current vs. what was happening 4 weeks prior — without any NaN from the shift itself.

---

## `join()` vs `merge()` — index vs column joins

Both do the same thing under the hood — `join` is a convenience wrapper around `merge`.

**`merge()`** — full control, join on columns or index:
```python
df1.merge(df2, on='key')                          # join on a column
df1.merge(df2, left_on='id', right_on='emp_id')   # different column names
df1.merge(df2, left_index=True, right_index=True) # join on index
```
Default is inner join.

**`join()`** — shortcut for index-on-index (or index-on-column) joins:
```python
df1.join(df2)              # aligns on index by default
df1.join(df2, on='key')    # df1 column matched to df2's index
```
Default is left join.

**Rule of thumb:** use `join()` when aligning on the index, `merge()` when joining on columns.

**`join()` arguments:**
```python
df.join(other, on=None, how='left', lsuffix='', rsuffix='', sort=False)
```
- `other` — the DataFrame/Series to join
- `on` — column in `df` to match against `other`'s index (if omitted, uses `df`'s index)
- `how` — `'left'` (default), `'right'`, `'inner'`, `'outer'`
- `lsuffix` / `rsuffix` — suffix to add to overlapping column names
- `sort` — sort the result index; default `False`
