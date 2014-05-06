========
Cookbook 
========

The basics
==========

Loading a table from a CSV
--------------------------

You can use Python's builtin :mod:`csv` to read CSV files.

If your file does not have headers:

.. code-block:: python

    from journalism import Table, TextType, NumberType

    text_type = TextType()
    number_type = NumberType()

    column_names = ('city', 'area', 'population')
    column_types = (text_type, number_type, number_type)

    with open('population.csv') as f:
        rows = list(csv.reader(f) 

    table = Table(rows, column_types, column_names)

If your file does have headers:

.. code-block:: python

    column_types = (text_type, number_type, number_type)

    with open('population.csv') as f:
        rows = list(csv.reader(f))

    column_names = rows.pop(0)

    table = Table(rows, column_types, column_names)

Loading a table from a CSV w/ csvkit
-------------------------------------

Of course, cool kids use `csvkit <http://csvkit.rtfd.org/>`_. (Hint: it supports unicode!)

.. code-block:: python

    import csvkit

    column_types = (text_type, number_type, number_type)

    with open('population.csv') as f:
        rows = list(csvkit.reader(f))

    column_names = rows.pop(0)

    table = Table(rows, column_types, column_names)

Writing a table to a CSV
------------------------

.. code-block:: python

    with open('output.csv') as f:
        writer = csv.writer(f)

        writer.writerow(table.get_column_names())
        writer.writerows(table.rows)

Writing a table to a CSV w/ csvkit
----------------------------------

.. code-block:: python

    with open('output.csv') as f:
        writer = csvkit.writer(f)

        writer.writerow(table.get_column_names())
        writer.writerows(table.rows)

Filtering
=========

Filter by regex
---------------

You can use Python's builtin :mod:`re` module to introduce a regular expression into a :meth:`.Table.where` query.

For example, here we find all states that start with "C".

.. code-block:: python

    import re

    new_table = table.where(lambda row: re.match('^C', row['state']))

This can also be useful for finding values that **don't** match your expectations. For example, finding all values in the "phone number" column that don't look like phone numbers:

.. code-block:: python

    new_table = table.where(lambda row: not re.match('\d{3}-\d{3}-\d{4}', row['phone']))

Filter by glob
--------------

Hate regexes? You can use glob (a.k.a. :mod:`fnmatch`) syntax too!

.. code-block:: python

    from fnmatch import fnmatch

    new_table = table.where(lambda row: fnmatch('C*', row['state'])

Filter to values within a range
-------------------------------

This snippet filters the dataset to incomes between 100,000 and 200,000.

.. code-block:: python

    new_table = table.where(lambda row: 100000 < row['income'] < 200000) 

Random sample
--------------

By combining a random sort with limiting, we can effectively get a random sample from a table.

.. code-block:: python

    import random

    randomized = table.order_by(lambda row: random.random())
    sampled = table.limit(10)

Ordered sample
--------------

With can also get an ordered sample by simply using the :code:`step` parameter of the :meth:`.Table.limit` method to get every Nth row.

.. code-block:: python

    sampled = table.limit(step=10)

Sorting
=======

Basic sort
----------

Order a table by the :code:`last_name` column:

.. code-block:: python

    new_table = table.order_by('last_name')


Multicolumn sort
----------------

Because Python's internal sorting works natively with arrays, we can implement multi-column sort by returning an array from the key function.

.. code-block:: python

    new_table = table.order_by(lambda row: [row['last_name'], row['first_name'])

This table will now be ordered by :code:`last_name`, then :code:`first_name`.

Randomizing order
-----------------

.. code-block:: python

    import random

    new_table = table.order_by(lambda row: random.random())

Statistics
==========

Descriptive statistics
----------------------

journalism includes a full set of standard descriptive statistics that can be applied to any :class:`.NumberColumn`.

.. code-block:: python

    column = table.columns['salary']

    column.sum()
    column.min()
    column.max()
    column.mean()
    column.median()
    column.mode()
    column.variance()
    column.stdev()
    column.mad()

Aggregate statistics
--------------------

You can also generate aggregate statistics for subsets of data (sometimes colloquially referred to as "rolling up".

.. code-block:: python

    summary = table.aggregate('profession', { 'salary': 'mean', 'salary': 'median' }) 

A "count" column is always return in the results. The :code:`summary` table in this example would have these columns: :code:`('profession', 'profession_count', 'salary_mean', 'salary_median')`.

Identifying outliers
--------------------

journalism includes two builtin methods for identifying outliers. The first, and most widely known, is by identifying values which are more than some number of standard deviations from the mean (typically 3).

.. code-block:: python

    outliers = table.stdev_outliers('salary', deviations=3, reject=False)

By specifying :code:`reject=True` you can instead return a table including only those values **not** identified as outliers.

.. code-block:: python

    not_outliers = table.stdev_outliers('salary', deviations=3, reject=True)

The second, more robust, method for identifying outliers is by identifying values which are more than some number of "median absolute deviations" from the median (typically 3).

.. code-block:: python

    outliers = table.mad_outliers('salary', deviations=3, reject=False)

As with the first example, you can specify :code:`reject=True` to exclude outliers in the resulting table.

Modifying data
==============

Computing percent change
------------------------

You could use :meth:`.Table.compute` to calculate percent change, however, for your convenience journalism has a builtin shortcut 

.. code-block:: python

    new_table = table.percent_change('july', 'august', 'pct_change')

This will compute the percent change between the :code:`july` and :code:`august` columns and put the result in a new :code:`pct_change` column in the resulting table.

Rounding to two decimal places
------------------------------

journalism stores numerical values using Python's :class:`decimal.Decimal` type. This data type ensures numerical precision beyond what is supported by the native :func:`float` type, however, because of this we can not use Python's builtin :func:`round` function. Instead we must use :meth:`decimal.Decimal.quantize`.

We can use :meth:`.Table.compute` to apply the quantize to generate a rounded column from an existing one:

.. code-block:: python

    from decimal import Decimal

    def round_price(row):
        return row['price'].quantize(Decimal('0.01'))

    new_table = table.compute('price_rounded', DecimalColumn, round_price)

To round to one decimal place you would simply change :code:`0.01` to :code:`0.1`.

Emulating SQL
=============

journalism's command structure is very similar to SQL. The primary difference between journalism and SQL is that commands like :code:`SELECT` and :code:`WHERE` explicitly create new tables. You can chain them together as you would with SQL, but be aware each command is actually creating a new table.

.. note::

    All examples in this section use the `PostgreSQL <http://www.postgresql.org/>`_ dialect for comparison.

SELECT
------

SQL:

.. code-block:: postgres

    SELECT state, total FROM table;

journalism:

.. code-block:: python

    new_table = table.select(('state', 'total'))

WHERE
-----

SQL:

.. code-block:: postgres

    SELECT * FROM table WHERE LOWER(state) = 'california';

journalism:

.. code-block:: python

    new_table = table.where(lambda row: row['state'].lower() == 'california')

ORDER BY
--------

SQL:

.. code-block:: postgres 

    SELECT * FROM table ORDER BY total DESC;

journalism:

.. code-block:: python

    new_table = table.order_by(lambda row: row['total'], reverse=True)

DISTINCT
--------

SQL:

.. code-block:: postgres

    SELECT DISTINCT ON (state) * FROM table;

journalism:

.. code-block:: python

    new_table = table.distinct('state')

.. note::

    Unlike most SQL implementations, journalism always returns the full row. Use :meth:`.Table.select` if you want to filter the columns first.

INNER JOIN
----------

SQL (two ways):

.. code-block:: postgres

    SELECT * FROM patient, doctor WHERE patient.doctor = doctor.id;

    SELECT * FROM patient INNER JOIN doctor ON (patient.doctor = doctor.id);

journalism:

.. code-block:: python

    joined = patients.inner_join('doctor', doctors, 'id')

LEFT OUTER JOIN
---------------

SQL:

.. code-block:: postgres

    SELECT * FROM patient LEFT OUTER JOIN doctor ON (patient.doctor = doctor.id);

journalism:

.. code-block:: python

    joined = patients.left_outer_join('doctor', doctors, 'id')

GROUP BY
--------

journalism's :meth:`.Table.group_by` works slightly different than SQLs. It does not require an aggregate function. Instead it returns a dictionary of :code:`group`, :meth:`.Table` pairs. To see how to perform the equivalent of a SQL aggregate, see the next example.

.. code-block:: python

    groups = patients.group_by('doctor')

Chaining commands together
--------------------------

SQL:

.. code-block:: postgres

    SELECT state, total FROM table WHERE LOWER(state) = 'california' ORDER BY total DESC;

journalism:

.. code-block:: python

    new_table = table \
        .select(('state', 'total')) \
        .where(lambda row: row['state'].lower() == 'california') \
        .order_by('total', reverse=True)

.. note::

    I don't advise chaining commands like this. Being explicit about each step is usually better.

Aggregate functions
-------------------

SQL:

.. code-block:: postgres

    SELECT mean(age) FROM patient GROUP BY doctor;

journalism:

.. code-block:: python

    new_table = patient.aggregate('doctor', { 'age': 'mean' })

Emulating Excel
===============

One of journalism's most powerful assets is that instead of a wimpy "formula" language, you have the entire Python language at your disposal. Here are examples of how to translate a few common Excel operations.

SUM
---

.. code-block:: python

    def five_year_total(row):
        columns = ('2009', '2010', '2011', '2012', '2013')

        return sum(tuple(row[c] for c in columns)]

    new_table = table.compute('five_year_total', DecimalColumn, five_year_total)  

TRIM
----

.. code-block:: python

    new_table = table.compute('name_stripped', TextType(), lambda row: row['name'].strip())

CONCATENATE
-----------

.. code-block:: python

    new_table = table.compute('full_name', TextType(), lambda row '%(first_name)s %(middle_name)s %(last_name)s' % row) 

IF
--

.. code-block:: python

    new_table = table.compute('mvp_candidate', TextType(), lambda row: 'Yes' if row['batting_average'] > 0.3 else 'No'

VLOOKUP
-------

.. code-block:: python

    states = {
        'AL': 'Alabama',
        'AK': 'Alaska',
        'AZ': 'Arizona',
        ...
    }

    new_table = table.compute('state_name', TextType(), lambda row: states[row['state_abbr']]) 

Pivot tables
------------

You can emulate most of the functionality of Excel's pivot tables using the :meth:`.Table.aggregate` method.

.. code-block:: python

    summary = table.aggregate('profession', { 'salary': 'mean', 'salary': 'median' }) 

A "count" column is always return in the results. The :code:`summary` table in this example would have these columns: :code:`('profession', 'profession_count', 'salary_mean', 'salary_median')`.

Emulating R
===========

aggregate
---------

R:

.. code-block:: r

    aggregate(salary ~ job, data = employees, FUN = mean)

journalism:

.. code-block:: python

    aggregates = employees..aggregate('job', { 'salary': 'mean' })

Emulating Underscore.js
=======================

filter
------

journalism's :meth:`.Table.where` functions exactly like Underscore's :code:`filter`.

.. code-block:: python

    new_table = table.where(lambda row: row['state'] == 'Texas')

reject
------

To simulate Underscore's :code:`reject`, simply negate the return value of the function you pass into journalism's :meth:`.Table.where`.

.. code-block:: python

    new_table = table.where(lambda row: not (row['state'] == 'Texas'))

find
----

journalism's :meth:`.Table.find` works exactly like Undrescore's :code:`find`.

.. code-block:: python

    row = table.find(lambda row: row['state'].startswith('T'))

any
---

journalism's columns have an :meth:`.Column.any` method that functions like Underscore's :code:`any`.

.. code-block:: python

    true_or_false = table.columns['salaries'].any(lambda d: d > 100000)

You can also use :meth:`.Table.where` to filter to columns that pass the truth test.

all
---

journalism's columns have an :meth:`.Column.all` method that functions like Underscore's :code:`all`.

.. code-block:: python

    true_or_false = table.columns['salaries'].all(lambda d: d > 100000)

You can also use :meth:`.Table.where` to filter to columns that pass the truth test.

Plotting with matplotlib
========================

journalism integrates well with Python plotting library `matplotlib <http://matplotlib.org/>`_.

Line chart
----------

.. code-block:: python

    import pylab

    pylab.plot(table.columns['homeruns'], table.columns['wins'])
    pylab.xlabel('Homeruns')
    pylab.ylabel('Wins')
    pylab.title('How homeruns correlate to wins')

    pylab.show()

Histogram
---------

.. code-block:: python

    pylab.hist(table.columns['state'])

    pylab.xlabel('State')
    pylab.ylabel('Count')
    pylab.title('Count by state')

    pylab.show()

Plotting with pygal
===================

`pygal <http://pygal.org/>`_ is a neat library for generating SVG charts. journalism works well with it too.

Line chart
----------

.. code-block:: python

    import pygal

    line_chart = pygal.Line()
    line_chart.title = 'State totals'
    line_chart.x_labels = states.columns['state_abbr']
    line_chart.add('Total', states.columns['total'])
    line_chart.render_to_file('total_by_state.svg') 


