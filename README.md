# DataFlow

DataFlow is a novel and intuitive way of building data processing flows.

- It's built for medium-data processing - data that fits on your hard drive, but is too big to load in Excel or as-is into Python, and not big enough to require spinning up a Hadoop cluster...
- It's built upon the foundation of the Frictionless Data project - which means that all data prduced by these flows is easily reusable by others.

## QuickStart / Tutorial

Let's start with the traditional 'hello, world' example:

```python
from datastream import Flow

data = [
  {'data': 'Hello'},
  {'data': 'World'}
]

def lowerData(row):
	row['data'] = row['data'].lower()

f = Flow(
      data,
      lowerData
)
data, *_ = f.results()

print(data)

# -->
# [
#   [
#     {'data': 'hello'}, 
#     {'data': 'world'}
#   ]
# ]
```

This very simple flow takes a list of `dict`s and applies a row processing function on each one of them.

We can load data from a file instead:

```python
from datastream import Flow, load

# beatles.csv:
# name,instrument 
# john,guitar
# paul,bass
# george,guitar
# ringo,drums

def titleName(row):
    row['name'] = row['name'].title()

f = Flow(
      load('beatles.csv'),
      titleName
)
data, *_ = f.results()

print(data)

# -->
# [
#   [
#     {'name': 'John', 'instrument': 'guitar'}, 
#     {'name': 'Paul', 'instrument': 'bass'}, 
#     {'name': 'George', 'instrument': 'guitar'}, 
#     {'name': 'Ringo', 'instrument': 'drums'}
#   ]
# ]
```

The source file can be a CSV file, an Excel file or a Json file. You can use a local file name or a URL for a file hosted somewhere on the web.

Data sources can be generators and not just lists or files. Let's take as an example a very simple scraper:

```python
from datastream import Flow

from xml.etree import ElementTree
from urllib.request import urlopen

# Get from Wikipedia the population count for each country
def country_population():
    # Read the Wikipedia page and parse it using etree
    page = urlopen('https://en.wikipedia.org/wiki/List_of_countries_and_dependencies_by_population').read()
    tree = ElementTree.fromstring(page)
    # Iterate on all tables, rows and cells
    for table in tree.findall('.//table'):
        if 'wikitable' in table.attrib.get('class', ''):
            for row in table.findall('tr'):
                cells = row.findall('td')
                if len(cells) > 3:
                    # If a matching row is found...
                    name = cells[1].find('.//a').attrib.get('title')
                    population = cells[2].text
                    # ... yield a row with the information
                    yield dict(
                        name=name,
                        population=population
                    )

f = Flow(
      country_population(),
)
data, *_ = f.results()

print(data)
# ---> 
# [
#   [
#     {'name': 'China', 'population': '1,391,090,000'}, 
#     {'name': 'India', 'population': '1,332,140,000'}, 
#     {'name': 'United States', 'population': '327,187,000'},
#     {'name': 'Indonesia', 'population': '261,890,900'},
#     ...
#   ]
# ]
```

This is nice, but we do prefer the numbers to be actual numbers and not strings.

In order to do that, let's simply define their type to be numeric:

```python
from datastream import Flow, set_type

def country_population():
    # same as before
	...

f = Flow(
	country_population(),
    set_type('population', type='number', groupChar=',')
)
data, *_ = f.results()

print(data)
# -->
# [
#   [
#     {'name': 'China', 'population': Decimal('1391090000')}, 
#     {'name': 'India', 'population': Decimal('1332140000')}, 
#     {'name': 'United States', 'population': Decimal('327187000')}, 
#     {'name': 'Indonesia', 'population': Decimal('261890900')},
#     ...
#   ]
# ]

```

Data is automatically converted to the correct native Python type.

Apart from data-types, it's also possible to set other constraints to the data. If the data fails validation (or does not fit the assigned data-type) an exception will be thrown - making this method highly effective for validating data and ensuring data quality. 

What about large data files? In the above examples, the results are loaded into memory, which is not always preferrable or acceptable. In many cases, we'd like to store the results directly onto a hard drive - without having the machine's RAM limit in any way the amount of data we can process.

We do it by using _dump_ processors:

```python
from datastream import Flow, set_type, dump_to_path

def country_population():
    # same as before
	...

f = Flow(
	country_population(),
    set_type('population', type='number', groupChar=','),
    dump_to_path('country_population')
)
*_ = f.process()

```

Running this code will create a local directory called `county_population`, containing two files:

```
├── country_population
│   ├── datapackage.json
│   └── res_1.csv
```

The CSV file - `res_1.csv` - is where the data is stored. The `datapackage.json` file is a metadata file, holding information about the data, including its schema.

We can now open the CSV file with any spreadsheet program or code library supporting the CSV format - or using one of the **data package** libraries out there, like so:

```python
from datapackage import Package
pkg = Package('country_population/res_1.csv')
it = pkg.resources[0].iter(keyed=True)
print(next(it))
# prints:
# {'name': 'China', 'population': Decimal('1391110000')}
```

Note how using the data package meta-data, data-types are restored and there's no need to 're-parse' the data. This also works with other types too, such as dates, booleans and even `list`s and `dict`s.

So far we've seen how to load data, process it row by row, and then inspect the results or store them in a data package.

Let's see how we can do more complex processing by manipulating the entire data stream:

```python
from datastream import Flow, set_type, dump_to_path

# Generate all triplets (a,b,c) so that 1 <= a <= b < c <= 20
def all_triplets():
    for a in range(1, 20):
        for b in range(a, 20):
            for c in range(b+1, 21):
                yield dict(a=a, b=b, c=c)

# Yield row only if a^2 + b^2 == c^1
def filter_pythagorean_triplets(rows):
    for row in rows:
        if row['a']**2 + row['b']**2 == row['c']**2:
            yield row

f = Flow(
    all_triplets(),
    set_type('a', type='integer'),
    set_type('b', type='integer'),
    set_type('c', type='integer'),
    filter_pythagorean_triplets,
    dump_to_path('pythagorean_triplets')
)
_ = f.process()

# -->
# pythagorean_triplets/res_1.csv contains:
# a,b,c
# 3,4,5
# 5,12,13
# 6,8,10
# 8,15,17
# 9,12,15
# 12,16,20
```

The `filter_pythagorean_triplets` function takes an iterator of rows, and yields only the ones that pass its condition. 

The flow framework knows whether a function is meant to hande a single row or a row iterator based on its parameters: 

- if it accepts a single `row` parameter, then it's a row processor.
- if it accepts a single `rows` parameter, then it's a rows processor.
- if it accepts a single `package` parameter, then it's a package processor.

Let's see an example of what we can do with a package processor:

```python
from datastream import Flow, load, dump_to_path, printer

def find_double_winners(datapackage):
	
    # Fetch the emmy nominees stream
    emmy = next(datapackage)
    # filter all winners and create a set from their names
    emmy_winners = set(
        map(lambda x: x['nominee'], 
            filter(lambda x: x['winner'],
                   emmy))
    )
    # yield an empty iterator for the emmy data
    yield iter([])

    # Fetch the academy nominees stream
    academy = next(datapackage)
    # Filter all winners whose name appears in the emmy winners
    # And yield the result
    yield filter(lambda row: row['Winner'] and row['Name'] in emmy_winners,
                 academy)

f = Flow(
    # Emmy award nominees and winners
    load('emmy.csv'),
    # Academy award nominees and winners
    load('academy.csv', encoding='utf8'),
    # Find academy award winners who also won an Emmy 
    find_double_winners,
    dump_to_path('double_winners')
)
_ = f.process()

```
