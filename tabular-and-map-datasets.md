---
jupyter:
  jupytext:
    formats: ipynb,md
    target_format: ipynb,md
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.1'
      jupytext_version: 1.2.0
  kernelspec:
    display_name: dados.gov.br API
    language: python
    name: dadosgovbr
---

# Counting tabular and map datasets in CKAN


It has come to my attention that some international open data ranking systems,
specifically, the
[Open-Useful-Reusable Government Data (OURdata) Index](http://www.oecd.org/gov/digital-government/open-government-data.htm),
measured by the Organization for Economic Co-operation and Development (OECD),
do measure not only how many datasets a given national open government data
portal has, but also how many of those are tabular and how many are maps.


I don't think measuring the number of datasets in an open government data
portal is a very useful metric, considering that governments may just as
well split large datasets into smaller ones in order to achieve a larger
"amount of datasets", without adding any benefit or value to the data user.

On the contrary, that practice might make relevant data more difficult for
the user to find, as it is more spread out in the midst of an ocean of other
datasets that may not be relevant in the context the user is looking for.


Similar opinions on the number of datasets as a metric are shared by civil
society organizations and governments alike, as can be seen in blog posts
by the
[Sunlight Foundation](https://sunlightfoundation.com/2019/08/13/measuring-the-impact-of-community-engagement-around-open-data/)
and [DataSF](https://datasf.org/blog/how-to-measure-open-data/),
the open data portal of the City of San Francisco.


> * **It incents the wrong behavior.** If we publish every year as a different dataset, our N goes up but data usability goes down. If we publish low value data, our N goes up but no one wants to use it. While it’s important to track - # of datasets shouldn’t be a key performance indicator (KPI).
> * **It diverts focus from quality.** Publishing to increase your N distracts from ensuring high quality publishing - data that is documented, updated regularly and of value.

As a matter of fact, other established international open data rankings, the
[Open Data Index](https://index.okfn.org/),
[Open Data Barometer](https://opendatabarometer.org/) and
[Open Data Inventory](http://odin.opendatawatch.com/) do not measure the
quantity of datasets.

## What is a dataset?


According to W3C's Data on the Web Best Practices Recommendation, a [dataset](https://www.w3.org/TR/dwbp/#dataset) is:

> defined as a collection of data, published or curated by a single agent, and available for access or download in one or more formats. A dataset does not have to be available as a downloadable file.


[CKAN](https://ckan.org/), the open data catalog software most often used
around the globe, also operates with the notion of a
[dataset](https://docs.ckan.org/en/2.8/user-guide.html#datasets-and-resources).
It further divides datasets into resources, which can follow several possible
criteria, such as temporal slices, interrelated tables, regional or
departmental divisions, etc. Resources may also be made available in many
different formats.

From CKAN's documentation:

> A dataset is a parcel of data - for example, it could be the crime
> statistics for a region, the spending figures for a government department,
> or temperature readings from various weather stations. When users search
> for data, the search results they see will be individual datasets.
> 
> A dataset contains two things:
> 
> * Information or “metadata” about the data. For example, the title and
>   publisher, date, what formats it is available in, what license it is
>   released under, etc.
> * A number of “resources”, which hold the data itself. CKAN does not mind
>   what format the data is in. A resource can be a CSV or Excel spreadsheet,
>   XML file, PDF document, image file, linked data in RDF format, etc. CKAN
>   can store the resource internally, or store it simply as a link, the
>   resource itself being elsewhere on the web. A dataset can contain any
>   number of resources. For example, different resources might contain the
>   data for different years, or they might contain the same data in different
>   formats.



From that we can infer that datasets may contain tabular data, map data,
both or none of those.


Counting how many tabular and how many map datasets are there in a given
CKAN instance presents some challenges:

* they are not disjuct sets, i.e., there may be datasets that have some
  tabular resources and some maps in the same dataset
* there are multiple tabular and map formats that must be all accounted for
* the *format* metadata in resources is a free text field, often allowing
  for a multitude of ways to write the same format (e.g. "geoJSON",
  "geo-JSON", "Geo JSON", etc.)
* the dataset search interface allows the user to filter datasets by either
  format at a time or a combination of format conditions that must all be
  satisfied simultaneously, i.e., it's an "and", not "or" operation. You
  can't catch all format variants by simply selecting each of the acceptable
  format values that would be map data. For instance, selecting "KML" and
  "GeoJSON" will only bring datasets that have resources in KML **and**
  resources in GeoJSON. If only one of those is available, the dataset
  isn't included.


## Approach


To overcome those challenges, one possible solution would be to use the CKAN
API to sum the datasets according to specified criteria, which we will define
as conditions for formats to be accepted as tabular and/or maps.


For that, we make a list of all formats recognized as tabular and another
list of all formats considered as maps. Considering that these sets are
not disjunct sets, we will also count how many datasets are in either one
but not the other sets, as well as datasets that are in both sets.


For this example, we shall use the Brazilian Open Data Portal, available at
dados.gov.br.

```python
CKAN_URL = "http://dados.gov.br"
UA = "ckan-tabular-and-map-counter/1.0 (+https://github.com/augusto-herrmann/ckan-tabular-and-map-counter)"
```

```python
from ckanapi import RemoteCKAN
```

```python
catalog = RemoteCKAN(CKAN_URL, UA)
```

```python
dataset_ids = catalog.action.package_list()
```

The following code will make one request per dataset to the CKAN instance.
That may take a while, depending on the number of datasets present.
Make sure to ajust the delay between requests in order not to overload
the server.

```python
# Time our delays between requests
import time
import random
```

```python
# Minimum and maximum delay, in miliseconds
MIN_DELAY = 100
MAX_DELAY = 5000
```

```python
# Let's track our progress
from tqdm import tqdm_notebook as tqdm
```

```python
formats = {}
```

This should take a while to run.

```python
for dataset_id in tqdm(dataset_ids):
    time.sleep(random.uniform(MIN_DELAY, MAX_DELAY)/1000)
    dataset = catalog.action.package_show(id=dataset_id)
    if dataset['state'] == 'active' and not dataset['private']:
        formats[dataset_id] = [
            resource['format'].lower() 
            for resource in dataset['resources']
        ]
```

To count the datasets, we need first do determine which formats are to be
considered tabular, and which ones are maps. To avoid redundancies, we shall
treat all format in lowercase.

We could use the [MIME Type](https://en.wikipedia.org/wiki/Media_type)
metadata, but we found that in practice this field os often let empty by
the people who input them.

```python
TABULAR_FORMATS = [
    'csv',
    'xlsx',
    'ods',
    'xls',
    'zip+csv',
    'txt',
    'zip+xls',
    'zip+sas',
    'csv+zip',
    'zip+rar+csv',
    'zip+txt',
]
```

```python
MAP_FORMATS = [
    'kml',
    'geojson',
    'zip shp',
    'esri rest',
    'ogc wms',
    'shp-zip',
    'wms',
    'kmz',
]
```

## Counting datasets


Now let's count datasets in each of those groups.


The following are datasets that include at least one resource in a format that is considered tabular.

```python
tabular_datasets = [
    dataset_id
    for dataset_id in dataset_ids
    if set(formats[dataset_id]).intersection(set(TABULAR_FORMATS))
]
```

```python
len(tabular_datasets)
```

The following are datasets that include at least one resource in a format that is considered map data.

```python
map_datasets = [
    dataset_id
    for dataset_id in dataset_ids
    if set(formats[dataset_id]).intersection(set(MAP_FORMATS))
]
```

```python
len(map_datasets)
```

We now cont datasets that do have tabular resources, but don't have any map resources.

```python
tabular_but_not_map_datasets = [
    dataset_id
    for dataset_id in dataset_ids
    if set(formats[dataset_id]).intersection(set(TABULAR_FORMATS))
    and not set(formats[dataset_id]).intersection(set(MAP_FORMATS))
]
```

```python
len(tabular_but_not_map_datasets)
```

And vice versa, datasets that do have map resources, but none that are tabular.

```python
map_but_not_tabular_datasets = [
    dataset_id
    for dataset_id in dataset_ids
    if set(formats[dataset_id]).intersection(set(MAP_FORMATS))
    and not set(formats[dataset_id]).intersection(set(TABULAR_FORMATS))
]
```

```python
len(map_but_not_tabular_datasets)
```

At last, these are the datasets that contain both tabular and map resources.

```python
tabular_and_map_datasets = [
    dataset_id
    for dataset_id in dataset_ids
    if set(formats[dataset_id])\
        .intersection(set(TABULAR_FORMATS))\
        .intersection(set(MAP_FORMATS))
]
```

```python
len(tabular_and_map_datasets)
```

## Conclusion


However useful the metric would be, it is indeed possible to count tabular
and map datasets in CKAN. Even though it isn't a readily available metric,
it can be counted with relative ease (albeit a little slowly if you have
many datasets), by using the CKAN API.
