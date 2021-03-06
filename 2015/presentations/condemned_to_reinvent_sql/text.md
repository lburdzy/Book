# Condemned to re-invent SQL, poorly - Michał Lowas-Rzechonek

## Introduction

The days when developers were expected to write SQL by hand are long gone. ORMs
are getting more and more popular, especially in web development.

Django, as one of the most popular (if not the most popular?) Python frameworks
for web development comes with the ORM built-in. Because of its prevalence, I'm
going to use it in my examples. If you don't know how it works, their tutorial
is really nice and short [1].

While it's hard to beat the convenience of using such an ORM, it lures us into
thinking about a database as a simple "object storage". One of the problems is
the fact that in most ORMs, querying the database happens pretty much
exclusively via a model class, which constrains the set of results to a known
attributes of our objects.

Focus on collections of objects might be one of the reasons why NoSQL
approaches are more alluring: just throw your objects into a bin and save them
on the disk. Indeed, there is no need to worry about tables, constraints and
indexes if all you do is load data from the storage and process it inside the
application!

I think the "object storage" way of thinking severely limits our ability to
process the data. In a relational model, columns are computed ad-hoc, when
returning query result, not when defining the schema. This is one of the
aspects of so-called "object-relational impedance mismatch" that results in
headaches for people smarter than us.

In this article, I'd like to show a few features of a relational model which,
while mapping poorly to object-oriented world, give the programmer very
powerful data manipulation tools.

As an example, I'm going to use PostgreSQL, which I find the most powerful
open-source database management system. Note that most of the "fancy"
constructs supported by PostgreSQL are actually standard SQL clauses. Still,
the big warning is in order: if you use MySQL, you might as well stop
reading right now.

I'm also going to suggest ways to map these features into application code.
While I believe the complete, robust translation is not achievable for reasons
mentioned above, there are techniques that allow us to tie the data and
application layers together, especially in a flexible, multi-paradigm language
like Python.

## Views

Consider an university schedule. There are buildings, rooms and lectures
happening in these rooms. The schema is a fairly straightforward one:

```python
class Room(models.Model):
    building = models.CharField(max_length=64, null=False, blank=False)
    number = models.IntegerField(null=False, blank=False)

class Lecture(models.Model):
    title = models.CharField(max_length=120, null=False, blank=False)
    room = models.ForeignKey(Room, null=False)
    date = models.DateTimeField(null=False, blank=False)
```

The most obvious case where the ORM falls short, is any non-trivial
aggregation. Let's say we would like to know which months are the most (or
least) busy, per building, so we can plan construction work on the campus.

With SQL, that's easy. First, fetch number of lectures happening in each
building in each month:

```sql
select
    "lectures_room"."building" as "building",
    date_trunc('month', "lectures_lecture"."date") as "month",
    count(*) as "count"
from "lectures_lecture"
left join "lectures_room" on "lectures_room"."id" =
"lectures_lecture"."room_id" group by "building", "month"
```

result is going to be something like this:

```
 building |         month          | count
----------+------------------------+-------
 A2       | 2015-06-01 00:00:00+02 |   275
 B3       | 2015-08-01 00:00:00+02 |   291
 C3       | 2015-12-01 00:00:00+01 |   288
 B4       | 2015-03-01 00:00:00+01 |   299
 B2       | 2015-08-01 00:00:00+02 |   315
 B1       | 2015-01-01 00:00:00+01 |   294
 ...
```

Then, partition the result by building name, within each partition order rows
by number of lectures, then fetch first and last value of "month" column.

The way to do that is called a window function. This feature allows the query
to aggregate results into subsets (windows) and constructs rows by selecting
values from a subset. More on that in [2].

Notice that in this case we're going to run a query on top of another query.
This is called "composition" and is a cornerstone of most programming languages.
Think about that as an equivalent of nested function calls - call this, pass
the result as an argument to call that, and so on. Just that in SQL, you don't
define functions, but rather assign names to such subqueries. The mechanism is
called "Common Table Expressions" [3] and uses `with foo as (select ...)` syntax.

```sql
with "busy_months" as (
    select
        "lectures_room"."building" as "building",
        ... -- the rest of the first query
)
select distinct on ("building")
    "building",
    first_value("month") over "building_window" as "least_busy_month",
    last_value("month")  over "building_window" as "most_busy_month"
from
    "busy_months"
window "building_window" as (
    partition by "building"
    order by "count"
    rows between unbounded preceding and unbounded following
);
```

Which gives us the final report:

```
condemned=> select * from lectures_busy_months;
 building |    least_busy_month    |    most_busy_month
----------+------------------------+------------------------
 A1       | 2015-02-01 00:00:00+01 | 2015-03-01 00:00:00+01
 A2       | 2015-04-01 00:00:00+02 | 2015-01-01 00:00:00+01
 A3       | 2015-02-01 00:00:00+01 | 2015-03-01 00:00:00+01
 B1       | 2015-02-01 00:00:00+01 | 2015-03-01 00:00:00+01
 B2       | 2015-02-01 00:00:00+01 | 2015-12-01 00:00:00+01
 B3       | 2015-04-01 00:00:00+02 | 2015-07-01 00:00:00+02
 B4       | 2015-02-01 00:00:00+01 | 2015-10-01 00:00:00+02
 C1       | 2015-02-01 00:00:00+01 | 2015-12-01 00:00:00+01
 C2       | 2015-01-01 00:00:00+01 | 2015-07-01 00:00:00+02
 C3       | 2015-02-01 00:00:00+01 | 2015-07-01 00:00:00+02
(10 rows)
```

Mapping this to the ORM is a bit of a problem, though:
 * There is no `Building` model, so we can't say
`Building.objects.annotate(count=Count(...))`
 * There is no straightforward way to call `date_trunc` function and aggregate by that
 * The ORM doesn't have any idea about `partition by` and `over` clauses.

This doesn't mean that all hope is lost, though. Sane databases allow you to
create a sort-of dynamic table out of that query. This is called a "view" -
just a name for a query result. Don't confuse that with Django views.

The view might seem like a trivial feature, but the point here is that it allows
us to define database-side abstractions, so do not underestimate them.

```sql
create view "lectures_busy_months" as
    with ...
    select ...;
```

This, for all intents and purposes, behaves just like a (read-only) table. This
means we can explicitly map this view to a model:

```python
class BusyMonths(models.Model):
    building = models.CharField(max_length=64, primary_key=True)
    least_busy_month = models.DateField()
    most_busy_month = models.DateField()

    class Meta:
        managed = False
        db_table = 'lectures_busy_months'
        ordering = ('building',)
```

There's one shortcoming though: Django requires all models to have a
single-column primary key. In our case, building names are unique in the
result, so we can tell Django to just use that. This is not always the case
though, and you might end up with adding superficial auto-incrementing column
just to make the ORM happy:

```sql
create view "needs_id" as
    select
        row_number() over () as "id",
        ...
    from ...
```

The only remaining thing is creating a schema migration that would install
our SQL view in the database. This can be done by executing an SQL script
via `migrations.RunSQL` operation. You can find the details in the sample
project [4].

## Functions

Pure ORM simply does not allow any kind of imperative logic to be defined in
the database. If you need to do non-trivial processing, you need to fetch
everything and do the computation in Python. In some cases, this is a huge I/O
overhead.

What about using this feature to crunch the numbers right where they are, and
passing only the result to the application? The best part is, we can do it in
Python!

Continuing the "university" theme, consider a table of grades:

```python
class Grade(models.Model):
    student = models.ForeignKey(settings.AUTH_USER_MODEL)
    date = models.DateTimeField(auto_now=True, null=False)
    grade = models.IntegerField(null=False)
```

Now, we would like to know who of the students is improving over time and who's
getting worse. One way of doing that is finding a linear interpolation of
grades over time with a "least squares method" [5]

    given a set `S` of points `(x, y)` on a plane
    and a line L: `y = A * x + B`
    find the values of A and B
    such that sum of squared distances between line L and each point in S
    is minimal

As said before, to calculate that in your application, you would need to fetch
*all* grades from the database, group them by student *yourself* and run
the fitting algorithm (i.e. `numpy`) on each set. But what if you could write
an ORM query that looks like this?

```python
User.objects.annotate(grade_trend=LinearFit(
    'grade__grade', 'grade__date'))
```

Fortunately, in PostgreSQL, we can implement database-side functions
in Python. Using these, we can create our own aggregates, besides standard ones
like `count`, `min` and `max`!

The way it works, you need a data type and two functions: the type stores the
"state" of your aggregate. In our case, it's just a list of grades. Then, first
function is supposed to "add" new value to the "state": in our case, just
append the grade to the list. The final function takes the "state", and
calculates the final result.

Sounds scary, but really isn't. The code below is all you need:

```sql
create language plpythonu;

create or replace function linear_fit_finalfunc(p_state float[])
returns float[] as
$$
    import numpy
    return numpy.polyfit(xrange(len(p_state)), p_state, 1)
$$
language plpythonu immutable;

create aggregate linear_fit(float)
(
    stype = float[],      -- our "state" is a list of grades
    initcond = '{}',      -- starting from an empty list
    sfunc = array_append, -- adding grade to the "state" is just appending to
                          -- the list
    -- final function does the line fitting on accumulated points
    finalfunc = linear_fit_finalfunc
);
```

Mind the `create aggregate` clause which is very non-standard and specific to
PostgreSQL. The parameters are:

 - `stype`: the data type holding the "state"
 - `initcond`: initial value of the "state"
 - `sfunc`": function that inserts (aggregates) a "value" into the "state"
 - `finalfunc`: function that takes the "state" and returns final value of
    the aggregation

This allows us to write the following query:

```sql
select
    "auth_user"."username",
    linear_fit("grades_grade"."grade" order by "grades_grade"."date")
from "auth_user"
left join "grades_grade" on "grades_grade"."student_id" = "auth_user"."id"
```

Looks good, but as with the view, we still need to map this to ORM somehow.
This is slightly more complex, we need to define an ORM wrapper for our
custom aggregation function. This is similar to what Django already provides:
`Count` wraps the `count()`, `Avg` wraps `avg` and so on. We need to make our
own `LinearFit` that wraps `linear_fit`.

This is little complicated, as we need to dig deeper into the ORM...

Long story short: when we write a query, Django constructs a tree of
"expressions" to reflect our wishes [6].  Then, this tree is "compiled" to SQL
code and executed as a raw SQL query. The result is translated back from a
table to a set of `Model` instances.

To add a custom aggregate, we need to define a custom "expression". Luckily, we
can base it on built-in base expression classes. There is one for database-side
functions, appropriately named `Func`. Unfortunately, it doesn't support

```sql
select aggregate_function(column order by another_column)
```

syntax, which is vital for our calculations - grades need to be fitted in order
in which they were given (otherwise the whole "trend" doesn't make any sense).
This means we need to extend it a little by modifying `Func.template`
attribute, like this:

```python
class LinearFit(Func):
    contains_aggregate = True
    function = 'linear_fit'
    template = '%(function)s(%(expressions)s order by %(ordering)s)'

    def __init__(self, expression, ordering, **extra):
        super(LinearFit, self).__init__(
            expression,
            output_field=ArrayField(models.FloatField()),
            **extra)
        self.ordering = self._parse_expressions(ordering)[0]

    def resolve_expression(self, *args, **kwargs):
        c = super(Func, self).resolve_expression(*args, **kwargs)
        c.ordering = c.ordering.resolve_expression(*args, **kwargs)
        return c

    def as_sql(self, compiler, connection, function=None, template=None):
        ordering_sql, ordering_params = compiler.compile(self.ordering)
        self.extra['ordering'] = ordering_sql

        return super(LinearFit, self).as_sql(compiler, connection,
            function, template)

    def get_group_by_cols(self):
        return []
```

This seems to do the job (note that grades were given randomly, so trends are
mosly flat)!

```python
>>> for i in User.objects.annotate(grade_trend=LinearFit(
            'grade__grade', 'grade__date')):
...     print i.username, i.grade_trend
...
john [-0.0017908297019, 3.77233748271]
mary [-0.000803833399885, 3.63772475795]
barbara [-0.0018932620358, 3.65124481328]
peter [-0.00281602111148, 3.87818118949]
thomas [0.000325526484835, 3.48609958506]
kevin [0.00103517422177, 3.3887966805]
betty [-0.000701401065991, 3.54215076072]
april [5.72926613309e-05, 3.53482019364]
bob [-0.000128474452681, 3.49868603043]
sean [0.000461379537839, 3.54069847856]
denise [0.000466587961597, 3.36507607192]
```

## Summary

SQL is often considered an "ugly duckling" in the stack, and most of developers
avoid it like a plague. True, the syntax isn't the prettiest and sometimes it
works in a counter-intuitive way. On the other hand, you can achieve amazing
results with very little amount of rather readable code.

The best part is that you don't need to know upfront what kind of reporting
you're going to do. Most of the time, you can just design your models as usual
and write your views/aggregates/whatnot right when you need them.

I would like to encourage everyone dealing with databases to dig a bit deeper
into the relational world and stop relying exclusively on the ORM. The two
examples are just a tip of an iceberg, there is *much* more in SQL than you
think.

## Further reading

1. Django: Model intro https://docs.djangoproject.com/en/1.8/intro/tutorial01/#creating-models
2. Dimitri Fontaine: Understanding Window Functions http://tapoueh.org/blog/2013/08/20-Window-Functions
3. PostgreSQL: Common Table Expressions http://www.postgresql.org/docs/8.4/static/queries-with.html
4. Github: SQL, Condemned https://github.com/mrzechonek/sql_condemned
5. Wikipedia: Simple linear reqression https://en.wikipedia.org/wiki/Simple_linear_regression
6. Django: Query expressions https://docs.djangoproject.com/en/1.8/ref/models/expressions/
