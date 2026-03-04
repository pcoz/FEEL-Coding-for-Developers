# Appendix C: FEEL-to-Language Rosetta Stone

---

## Values and Literals

| Concept | FEEL | JavaScript | Python | Java | SQL |
|---------|------|-----------|--------|------|-----|
| Integer | `42` | `42` | `42` | `42` | `42` |
| Decimal | `3.14` | `3.14` | `3.14` | `3.14` | `3.14` |
| String | `"hello"` | `"hello"` | `"hello"` | `"hello"` | `'hello'` |
| Boolean true | `true` | `true` | `True` | `true` | `TRUE` |
| Boolean false | `false` | `false` | `False` | `false` | `FALSE` |
| Null | `null` | `null` | `None` | `null` | `NULL` |
| Date | `@"2024-03-15"` | `new Date("2024-03-15")` | `date(2024,3,15)` | `LocalDate.of(2024,3,15)` | `DATE '2024-03-15'` |
| Duration | `@"P2D"` | (no native) | `timedelta(days=2)` | `Duration.ofDays(2)` | `INTERVAL '2' DAY` |

## Collections

| Operation | FEEL | JavaScript | Python | SQL |
|-----------|------|-----------|--------|-----|
| Create list | `[1, 2, 3]` | `[1, 2, 3]` | `[1, 2, 3]` | `ARRAY[1,2,3]` |
| First element | `L[1]` | `L[0]` | `L[0]` | (1-based) |
| Last element | `L[-1]` | `L.at(-1)` | `L[-1]` | — |
| Length | `count(L)` | `L.length` | `len(L)` | `CARDINALITY(L)` |
| Append | `append(L, x)` | `[...L, x]` | `L + [x]` | — |
| Contains | `list contains(L, x)` | `L.includes(x)` | `x in L` | `x = ANY(L)` |
| Filter | `L[item > 5]` | `L.filter(x=>x>5)` | `[x for x in L if x>5]` | `WHERE x > 5` |
| Map | `for x in L return f(x)` | `L.map(f)` | `[f(x) for x in L]` | — |
| Sum | `sum(L)` | `L.reduce((a,b)=>a+b)` | `sum(L)` | `SUM(col)` |
| Sort | `sort(L, function(a,b) a<b)` | `L.sort((a,b)=>a-b)` | `sorted(L)` | `ORDER BY` |
| Distinct | `distinct values(L)` | `[...new Set(L)]` | `list(set(L))` | `DISTINCT` |
| Flatten | `flatten(L)` | `L.flat(Infinity)` | (itertools) | `UNNEST` |

## Key-Value Structures

| Operation | FEEL | JavaScript | Python | Java |
|-----------|------|-----------|--------|------|
| Create | `{a: 1, b: 2}` | `{a: 1, b: 2}` | `{"a": 1, "b": 2}` | `Map.of("a",1,"b",2)` |
| Access | `ctx.a` | `obj.a` | `d["a"]` | `map.get("a")` |
| Dynamic access | `get value(ctx, k)` | `obj[k]` | `d[k]` | `map.get(k)` |
| Keys | `get entries(ctx).key` | `Object.keys(obj)` | `d.keys()` | `map.keySet()` |

## Logic

| Operation | FEEL | JavaScript | Python | SQL |
|-----------|------|-----------|--------|-----|
| And | `a and b` | `a && b` | `a and b` | `a AND b` |
| Or | `a or b` | `a \|\| b` | `a or b` | `a OR b` |
| Not | `not(a)` | `!a` | `not a` | `NOT a` |
| If/else | `if c then a else b` | `c ? a : b` | `a if c else b` | `CASE WHEN c THEN a ELSE b END` |
| Null check | `x = null` | `x === null` | `x is None` | `x IS NULL` |

## Iteration

| Operation | FEEL | JavaScript | Python | SQL |
|-----------|------|-----------|--------|-----|
| Transform | `for x in L return f(x)` | `L.map(f)` | `[f(x) for x in L]` | — |
| Any match | `some x in L satisfies p(x)` | `L.some(p)` | `any(p(x) for x in L)` | `EXISTS(...)` |
| All match | `every x in L satisfies p(x)` | `L.every(p)` | `all(p(x) for x in L)` | `NOT EXISTS(... NOT ...)` |
| Range | `for i in 1..n return ...` | `Array.from({length:n},(_,i)=>...)` | `for i in range(1,n+1)` | `generate_series(1,n)` |

## String Operations

| Operation | FEEL | JavaScript | Python |
|-----------|------|-----------|--------|
| Length | `string length(s)` | `s.length` | `len(s)` |
| Substring | `substring(s, 3, 2)` (1-based) | `s.slice(2, 4)` | `s[2:4]` |
| Upper | `upper case(s)` | `s.toUpperCase()` | `s.upper()` |
| Lower | `lower case(s)` | `s.toLowerCase()` | `s.lower()` |
| Contains | `contains(s, "x")` | `s.includes("x")` | `"x" in s` |
| Replace | `replace(s, "a", "b")` | `s.replaceAll("a","b")` | `s.replace("a","b")` |
| Split | `split(s, ",")` | `s.split(",")` | `s.split(",")` |
| Join | `string join(L, ",")` | `L.join(",")` | `",".join(L)` |
| Starts with | `starts with(s, "x")` | `s.startsWith("x")` | `s.startswith("x")` |
| Ends with | `ends with(s, "x")` | `s.endsWith("x")` | `s.endswith("x")` |

## Temporal

| Operation | FEEL | JavaScript | Python |
|-----------|------|-----------|--------|
| Current date | `today()` | `new Date()` | `date.today()` |
| Current timestamp | `now()` | `new Date()` | `datetime.now()` |
| Add duration | `d + @"P30D"` | (manual) | `d + timedelta(30)` |
| Difference | `d1 - d2` | `d1 - d2` (ms) | `d1 - d2` (timedelta) |
| Year extract | `d.year` | `d.getFullYear()` | `d.year` |
| Day of week | `day of week(d)` | `d.getDay()` (0=Sun) | `d.strftime("%A")` |

## Type Checking

| Operation | FEEL | JavaScript | Python | Java |
|-----------|------|-----------|--------|------|
| Type test | `x instance of number` | `typeof x === 'number'` | `isinstance(x, (int, float))` | `x instanceof Number` |
| Null check | `x = null` | `x === null` | `x is None` | `x == null` |

## Functions

| Operation | FEEL | JavaScript | Python | Java |
|-----------|------|-----------|--------|------|
| Lambda | `function(x, y) x + y` | `(x, y) => x + y` | `lambda x, y: x + y` | `(x, y) -> x + y` |
| Named function | Context entry: `{add: function(x,y) x+y}` | `const add = (x,y) => x+y` | `def add(x,y): return x+y` | Method definition |
| Named parameters | `f(a: 1, b: 2)` | (not supported) | `f(a=1, b=2)` | (not supported) |

## Ranges and Membership

| Operation | FEEL | JavaScript | Python | SQL |
|-----------|------|-----------|--------|-----|
| Range check | `x in [1..10]` | `x >= 1 && x <= 10` | `1 <= x <= 10` | `x BETWEEN 1 AND 10` |
| Between | `x between 5 and 10` | `x >= 5 && x <= 10` | `5 <= x <= 10` | `x BETWEEN 5 AND 10` |
| Set membership | `x in ("A","B","C")` | `["A","B","C"].includes(x)` | `x in ("A","B","C")` | `x IN ('A','B','C')` |

---

[Previous: Appendix B: Exercise Solutions](appendix-b-exercise-solutions.md) | [Next: Appendix D: Engine Compatibility Matrix](appendix-d-engine-compatibility.md)
