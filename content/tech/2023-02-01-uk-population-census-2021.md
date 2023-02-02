---
title: "Processing UK 2021 census open data with python tools"
date: 2023-01-14T18:54:32Z
draft: false
tags: ["python", "geospatial", "census"]
---

### Introduction
The census provides a rich set of demographic information that could be useful for various data science tasks, such as geomarketing, house price analysis etc. With the results of the 2021 census being published recently, this notebook demonstrates how to manipulate this data with FOSS python tools to build a dataset that can be used for downstream tasks.

For this demo, I will be focusing on output areas (OA) of London:
- download the boundaries for these OA
- download demographics datasets (population, household size, gender, education level)
- convert the irregular geometries of the OA into 100m by 100m cells
- calculate the avgerages for the demographics per bin
- convert the cell data into raster 

These are the packages we will need:

```python
import pandas as pd
import geopandas as gpd
from shapely.geometry import Polygon
import numpy as np
from tqdm import tqdm
import fiona
import rasterio as rio
from geocube.api.core import make_geocube
```

### Datasets

#### Population data 

for Census 2021 according to Output Area (https://www.ons.gov.uk/filters/b9532b29-299e-4a23-9fc8-b99d68e172b9/dimensions)

> Title: Population density
Description: This dataset provides Census 2021 estimates that classify usual residents in England and Wales by population density (number of usual residents per square kilometre). The estimates are as at Census Day, 21 March 2021.


> Output Areas (OAs) are the lowest level of geographical area for census statistics and were first created following the 2001 Census. Each OA is made up of between 40 and 250 households and a usually resident population of between 100 and 625 persons and may change after each census.



```python
population_oa = pd.read_csv('data/uk-census-2021-oa.csv')
population_oa.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Output Areas Code</th>
      <th>Output Areas</th>
      <th>Observation</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>E00000001</td>
      <td>E00000001</td>
      <td>26268.7</td>
    </tr>
    <tr>
      <th>1</th>
      <td>E00000003</td>
      <td>E00000003</td>
      <td>60952.4</td>
    </tr>
    <tr>
      <th>2</th>
      <td>E00000005</td>
      <td>E00000005</td>
      <td>12873.6</td>
    </tr>
    <tr>
      <th>3</th>
      <td>E00000007</td>
      <td>E00000007</td>
      <td>1959.2</td>
    </tr>
    <tr>
      <th>4</th>
      <td>E00000010</td>
      <td>E00000010</td>
      <td>71200.0</td>
    </tr>
  </tbody>
</table>
</div>



### Geographic manipulation-- OA boundaries to cells

To match the above OA codes with actual geography, we have to obtain the boundaries of each output area-- these can be found on [UK's goverment geoportal](https://geoportal.statistics.gov.uk/datasets/ons::output-areas-dec-2021-boundaries-generalised-clipped-ew-bgc/explore?location=52.593248%2C-2.489483%2C7.00)-- note that I am using the 'generalised' version instead of the full version.


```python
boundaries = gpd.read_file('data/Output_Areas_(Dec_2021)_Boundaries_Generalised_Clipped_EW_(BGC)/Output_Areas_(Dec_2021)_Boundaries_Generalised_Clipped_EW_(BGC).shp')
```


```python
boundaries.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>OBJECTID</th>
      <th>OA21CD</th>
      <th>GlobalID</th>
      <th>Shape__Are</th>
      <th>Shape__Len</th>
      <th>geometry</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>E00000001</td>
      <td>bc5eb21b-d42b-4715-a771-2c27575a08f0</td>
      <td>6949.151482</td>
      <td>421.166161</td>
      <td>POLYGON ((532303.492 181814.110, 532213.378 18...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>E00000003</td>
      <td>a1a2b34f-320e-4bb8-acb4-7ca7ca16ef9c</td>
      <td>4492.411072</td>
      <td>307.714653</td>
      <td>POLYGON ((532213.378 181846.192, 532190.539 18...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>E00000005</td>
      <td>9337da1a-fe0f-4210-9c95-ed2d20fd6287</td>
      <td>8565.514214</td>
      <td>385.204781</td>
      <td>POLYGON ((532180.131 181763.020, 532219.161 18...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>E00000007</td>
      <td>b336e11f-af26-48a6-ac67-44f5b8b8840a</td>
      <td>75994.829704</td>
      <td>1408.607657</td>
      <td>POLYGON ((532201.292 181668.180, 532267.728 18...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>E00000010</td>
      <td>ca8f9874-cdf5-4c1a-9d39-f74a410dae44</td>
      <td>2102.876602</td>
      <td>215.271975</td>
      <td>POLYGON ((532127.958 182133.192, 532089.264 18...</td>
    </tr>
  </tbody>
</table>
</div>



In the boundaries geodataframe, the columns we'll need is that 'OA21CD', 'Shape_Are' and 'geometry columns': 'OA21CD' can join to the population dataset from above, Shape__Are gives the area of the OA in m^2 (note that this is different from the poulation measurement unit, which is in km^2). We can in fact cross check it against the geometry column


```python
# check that the projection is indeed british national grid eastings and northings
print(f"geodata has projection of {boundaries.crs}")
# check that the geometry gives ~the same area as what shape_are gives to a 10m^2 tolerance
assert ((boundaries.area.astype(int)- boundaries.Shape__Are.astype(int)).abs() < 10).sum() == len(boundaries)
```

    geodata has projection of epsg:27700



```python

```

Since we are only interested in Greater London, we can filter out rows from the boundaries dataframe that are outside of the london bounding box.

We can get the bounding box of london using the OSM query https://nominatim.openstreetmap.org/search.php?city=london&country=uk&format=jsonv2 (the first entry is the correct one), giving a boundingbox of ymin, ymax, xmin, xmax of "51.2867602","51.6918741","-0.5103751","0.3340155"


```python
ldn_bounding_box = Polygon.from_bounds( -0.5103751, 51.2867602,0.3340155, 51.691874 )
ldn_bounding_box = gpd.GeoSeries([ldn_bounding_box], crs='epsg:4326').to_crs(boundaries.crs)
ldn_bounding_box.geometry
```




    0    POLYGON ((503976.311 155234.131, 503059.759 20...
    dtype: geometry



Convert the geodataframe into a grid for downstream tasks-- Ordnance survey actually has a file of british grids at various resolutions prebuilt (https://github.com/OrdnanceSurvey/OS-British-National-Grids), -- the smallest is 1km by 1km, so we will construct our own 100m x 100m grid for london (We can do it for the whole uk, but we will have to iterate through the dataset as the number of cells will become very large)


---


**diversion**

This demos how to use the grid gpkg file from OS-- the gpkg is a vector file with multiple 'layers' the same way a raster file can have bands


```python
# get the layer names
list(fiona.listlayers('data/os_bng_grids.gpkg'))
```




    ['100km_grid', '50km_grid', '20km_grid', '10km_grid', '5km_grid', '1km_grid']




```python
layername = '1km_grid'
bsng_grid_1km = gpd.read_file('data/os_bng_grids.gpkg', layer=layername)
bsng_grid_1km.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>tile_name</th>
      <th>geometry</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>HL0000</td>
      <td>POLYGON ((0.000 1200000.000, 1000.000 1200000....</td>
    </tr>
    <tr>
      <th>1</th>
      <td>HL0001</td>
      <td>POLYGON ((0.000 1201000.000, 1000.000 1201000....</td>
    </tr>
    <tr>
      <th>2</th>
      <td>HL0002</td>
      <td>POLYGON ((0.000 1202000.000, 1000.000 1202000....</td>
    </tr>
    <tr>
      <th>3</th>
      <td>HL0003</td>
      <td>POLYGON ((0.000 1203000.000, 1000.000 1203000....</td>
    </tr>
    <tr>
      <th>4</th>
      <td>HL0004</td>
      <td>POLYGON ((0.000 1204000.000, 1000.000 1204000....</td>
    </tr>
  </tbody>
</table>
</div>



---

Back to our task, we filter out the OA that intersects the london bounding box:


```python
gpd.GeoDataFrame(ldn_bounding_box.reset_index()).geometry
```




    0    POLYGON ((503976.311 155234.131, 503059.759 20...
    Name: 0, dtype: geometry






```python
ldn_boundaries = boundaries[['OA21CD', 'geometry']].sjoin(gpd.GeoDataFrame(ldn_bounding_box.reset_index()), how='inner')[['OA21CD', 'geometry']]
ldn_boundaries.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>OA21CD</th>
      <th>geometry</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>E00000001</td>
      <td>POLYGON ((532303.492 181814.110, 532213.378 18...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>E00000003</td>
      <td>POLYGON ((532213.378 181846.192, 532190.539 18...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>E00000005</td>
      <td>POLYGON ((532180.131 181763.020, 532219.161 18...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>E00000007</td>
      <td>POLYGON ((532201.292 181668.180, 532267.728 18...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>E00000010</td>
      <td>POLYGON ((532127.958 182133.192, 532089.264 18...</td>
    </tr>
  </tbody>
</table>
</div>




```python
ldn_bounding_box.geometry[0].bounds
```




    (503059.75944664044, 155234.13110756868, 562854.9611320551, 201813.5769950911)




```python
# create a 100m by 100m grid for london
cellsize=100
# British National Grid extent (0,0 700000,1300000)

xmin, ymin,  xmax, ymax = [100 * (x//100 )for x in ldn_bounding_box.geometry[0].bounds] 
matrix = np.mgrid[int(xmin):int(xmax + cellsize):cellsize, int(ymin):int(ymax+ cellsize):cellsize]

xcoor = matrix[0].flatten()
ycoor = matrix[1].flatten()

xcoor_max = xcoor +cellsize
ycoor_max = ycoor + cellsize

polygons = []
for x, y, xmax, ymax in tqdm(zip (xcoor, ycoor, xcoor_max, ycoor_max), total=len(ycoor)):
    polygons.append(Polygon.from_bounds(x, y, xmax, ymax))
grid = gpd.GeoDataFrame(data={'cellid': np.arange(len(polygons))},geometry=polygons, crs=boundaries.crs)
```

    100%|████████████████████████████████| 279733/279733 [00:04<00:00, 65379.51it/s]


Join the grid to the OA polygons to get the demographics info per grid cell


```python
print(grid.shape)
print(ldn_boundaries.shape)

ldn_demographics_grid = grid.sjoin(boundaries)
```

    (279733, 2)
    (30160, 2)



```python
ldn_demographics_grid
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>cellid</th>
      <th>geometry</th>
      <th>index_right</th>
      <th>OBJECTID</th>
      <th>OA21CD</th>
      <th>GlobalID</th>
      <th>Shape__Are</th>
      <th>Shape__Len</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>POLYGON ((503000.000 155200.000, 503000.000 15...</td>
      <td>147286</td>
      <td>147287</td>
      <td>E00155442</td>
      <td>5c96e7ec-ecb5-458b-b6e3-3f170061650d</td>
      <td>312112.652298</td>
      <td>2691.258154</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>POLYGON ((503000.000 155300.000, 503000.000 15...</td>
      <td>147286</td>
      <td>147287</td>
      <td>E00155442</td>
      <td>5c96e7ec-ecb5-458b-b6e3-3f170061650d</td>
      <td>312112.652298</td>
      <td>2691.258154</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>POLYGON ((503000.000 155400.000, 503000.000 15...</td>
      <td>147286</td>
      <td>147287</td>
      <td>E00155442</td>
      <td>5c96e7ec-ecb5-458b-b6e3-3f170061650d</td>
      <td>312112.652298</td>
      <td>2691.258154</td>
    </tr>
    <tr>
      <th>467</th>
      <td>467</td>
      <td>POLYGON ((503100.000 155200.000, 503100.000 15...</td>
      <td>147286</td>
      <td>147287</td>
      <td>E00155442</td>
      <td>5c96e7ec-ecb5-458b-b6e3-3f170061650d</td>
      <td>312112.652298</td>
      <td>2691.258154</td>
    </tr>
    <tr>
      <th>468</th>
      <td>468</td>
      <td>POLYGON ((503100.000 155300.000, 503100.000 15...</td>
      <td>147286</td>
      <td>147287</td>
      <td>E00155442</td>
      <td>5c96e7ec-ecb5-458b-b6e3-3f170061650d</td>
      <td>312112.652298</td>
      <td>2691.258154</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>279671</th>
      <td>279671</td>
      <td>POLYGON ((562800.000 195700.000, 562800.000 19...</td>
      <td>103204</td>
      <td>103205</td>
      <td>E00108990</td>
      <td>53fb368c-3684-4be0-b859-93b09ea9fdd4</td>
      <td>84283.686378</td>
      <td>1503.476417</td>
    </tr>
    <tr>
      <th>279672</th>
      <td>279672</td>
      <td>POLYGON ((562800.000 195800.000, 562800.000 19...</td>
      <td>103204</td>
      <td>103205</td>
      <td>E00108990</td>
      <td>53fb368c-3684-4be0-b859-93b09ea9fdd4</td>
      <td>84283.686378</td>
      <td>1503.476417</td>
    </tr>
    <tr>
      <th>279673</th>
      <td>279673</td>
      <td>POLYGON ((562800.000 195900.000, 562800.000 19...</td>
      <td>103204</td>
      <td>103205</td>
      <td>E00108990</td>
      <td>53fb368c-3684-4be0-b859-93b09ea9fdd4</td>
      <td>84283.686378</td>
      <td>1503.476417</td>
    </tr>
    <tr>
      <th>279674</th>
      <td>279674</td>
      <td>POLYGON ((562800.000 196000.000, 562800.000 19...</td>
      <td>103204</td>
      <td>103205</td>
      <td>E00108990</td>
      <td>53fb368c-3684-4be0-b859-93b09ea9fdd4</td>
      <td>84283.686378</td>
      <td>1503.476417</td>
    </tr>
    <tr>
      <th>279675</th>
      <td>279675</td>
      <td>POLYGON ((562800.000 196100.000, 562800.000 19...</td>
      <td>103204</td>
      <td>103205</td>
      <td>E00108990</td>
      <td>53fb368c-3684-4be0-b859-93b09ea9fdd4</td>
      <td>84283.686378</td>
      <td>1503.476417</td>
    </tr>
  </tbody>
</table>
<p>536587 rows × 8 columns</p>
</div>




```python
# as we can see, the OA size span is quite large --luckily most OA are larger than a grid cell. 
ax =ldn_demographics_grid.drop_duplicates(subset='OA21CD').Shape__Are.apply(np.log10).hist(bins = 100)
ax.set_xlabel('OA area m^2')
ax.set_ylabel('number of OA')
ax.vlines(4,0,1800, colors='b', label='10,000 m^2 (Size of cell)')
ax.legend()
```




    <matplotlib.legend.Legend at 0x7f8b0b33e680>




    
![png](/2023-02-01-uk-population-census-2021/output_26_1.png)
    


There are clearly duplicates-- e.g. when a grid cell span more than 1 OA, and where 1OA span more than 1cell-- this is more likely for the OA to span mutliple cells than the other way round. Base on this assumption, we assign each cell to one OA by dropping duplicadtes


```python
ldn_demographics_grid = ldn_demographics_grid.drop_duplicates(subset='cellid')
```


```python
# drop all the unneed cols
ldn_demographics_grid = ldn_demographics_grid[['cellid', 'OA21CD', 'geometry']]
```

Finally, let's populate these grid cells with demographic data
- [household size by OA](https://www.ons.gov.uk/datasets/TS017/editions/2021/versions/2/filter-outputs/1799bd24-15d0-43ca-81d4-bccb7434f0d4#get-data)
- [education](https://www.ons.gov.uk/datasets/TS067/editions/2021/versions/1/filter-outputs/55e2442d-5a41-4ca2-872f-030d52af39d7#get-data)
- [population density]((https://www.ons.gov.uk/filters/b9532b29-299e-4a23-9fc8-b99d68e172b9/dimensions)
- [sex](https://www.ons.gov.uk/datasets/TS008/editions/2021/versions/3/filter-outputs/4f979d87-7e23-4e29-83ed-db14647e92ba#get-data)

We have discussed the population data above, let's now tidy up the other datasets

#### household size


```python
houshold = pd.read_csv('data/household_size_output_area_census2021.csv')
houshold.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Output Areas Code</th>
      <th>Output Areas</th>
      <th>Household size (9 categories) Code</th>
      <th>Household size (9 categories)</th>
      <th>Observation</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>E00000001</td>
      <td>E00000001</td>
      <td>0</td>
      <td>0 people in household</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>E00000001</td>
      <td>E00000001</td>
      <td>1</td>
      <td>1 person in household</td>
      <td>34</td>
    </tr>
    <tr>
      <th>2</th>
      <td>E00000001</td>
      <td>E00000001</td>
      <td>2</td>
      <td>2 people in household</td>
      <td>44</td>
    </tr>
    <tr>
      <th>3</th>
      <td>E00000001</td>
      <td>E00000001</td>
      <td>3</td>
      <td>3 people in household</td>
      <td>10</td>
    </tr>
    <tr>
      <th>4</th>
      <td>E00000001</td>
      <td>E00000001</td>
      <td>4</td>
      <td>4 people in household</td>
      <td>6</td>
    </tr>
  </tbody>
</table>
</div>



We can use the 'code' as a proxy for household size


```python
houshold[['Household size (9 categories)', 'Household size (9 categories) Code']].drop_duplicates()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Household size (9 categories)</th>
      <th>Household size (9 categories) Code</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0 people in household</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1 person in household</td>
      <td>1</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2 people in household</td>
      <td>2</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3 people in household</td>
      <td>3</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4 people in household</td>
      <td>4</td>
    </tr>
    <tr>
      <th>5</th>
      <td>5 people in household</td>
      <td>5</td>
    </tr>
    <tr>
      <th>6</th>
      <td>6 people in household</td>
      <td>6</td>
    </tr>
    <tr>
      <th>7</th>
      <td>7 people in household</td>
      <td>7</td>
    </tr>
    <tr>
      <th>8</th>
      <td>8 or more people in household</td>
      <td>8</td>
    </tr>
  </tbody>
</table>
</div>




```python
mean_household_size = houshold[['Output Areas Code', 'Household size (9 categories) Code','Observation']]
mean_household_size['counts'] = mean_household_size['Household size (9 categories) Code'] * mean_household_size['Observation']
mean_household_size = mean_household_size.groupby('Output Areas Code')['counts'].sum()
mean_household_size_totals = houshold[['Output Areas Code', 'Household size (9 categories) Code','Observation']].groupby('Output Areas Code')['Observation'].sum()
```

    /tmp/ipykernel_28838/2961719914.py:2: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    
    See the caveats in the documentation: https://pandas.pydata.org/pandas-docs/stable/user_guide/indexing.html#returning-a-view-versus-a-copy
      mean_household_size['counts'] = mean_household_size['Household size (9 categories) Code'] * mean_household_size['Observation']



```python
mean_household_size.reset_index()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Output Areas Code</th>
      <th>counts</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>E00000001</td>
      <td>176</td>
    </tr>
    <tr>
      <th>1</th>
      <td>E00000003</td>
      <td>256</td>
    </tr>
    <tr>
      <th>2</th>
      <td>E00000005</td>
      <td>112</td>
    </tr>
    <tr>
      <th>3</th>
      <td>E00000007</td>
      <td>145</td>
    </tr>
    <tr>
      <th>4</th>
      <td>E00000010</td>
      <td>173</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>188875</th>
      <td>W00010693</td>
      <td>78</td>
    </tr>
    <tr>
      <th>188876</th>
      <td>W00010694</td>
      <td>404</td>
    </tr>
    <tr>
      <th>188877</th>
      <td>W00010695</td>
      <td>189</td>
    </tr>
    <tr>
      <th>188878</th>
      <td>W00010696</td>
      <td>237</td>
    </tr>
    <tr>
      <th>188879</th>
      <td>W00010697</td>
      <td>272</td>
    </tr>
  </tbody>
</table>
<p>188880 rows × 2 columns</p>
</div>




```python
mean_household_size = mean_household_size.reset_index().merge(mean_household_size_totals.reset_index(), on='Output Areas Code')
mean_household_size['avg_household_size'] = mean_household_size['counts'] /  mean_household_size['Observation']
mean_household_size.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Output Areas Code</th>
      <th>counts</th>
      <th>Observation</th>
      <th>avg_household_size</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>E00000001</td>
      <td>176</td>
      <td>94</td>
      <td>1.872340</td>
    </tr>
    <tr>
      <th>1</th>
      <td>E00000003</td>
      <td>256</td>
      <td>109</td>
      <td>2.348624</td>
    </tr>
    <tr>
      <th>2</th>
      <td>E00000005</td>
      <td>112</td>
      <td>63</td>
      <td>1.777778</td>
    </tr>
    <tr>
      <th>3</th>
      <td>E00000007</td>
      <td>145</td>
      <td>87</td>
      <td>1.666667</td>
    </tr>
    <tr>
      <th>4</th>
      <td>E00000010</td>
      <td>173</td>
      <td>125</td>
      <td>1.384000</td>
    </tr>
  </tbody>
</table>
</div>




```python

```

#### gender


```python
gender = pd.read_csv('data/uk-census-2021-sex-oa.csv')
gender.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Output Areas Code</th>
      <th>Output Areas</th>
      <th>Sex (2 categories) Code</th>
      <th>Sex (2 categories)</th>
      <th>Observation</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>E00000001</td>
      <td>E00000001</td>
      <td>1</td>
      <td>Female</td>
      <td>75</td>
    </tr>
    <tr>
      <th>1</th>
      <td>E00000001</td>
      <td>E00000001</td>
      <td>2</td>
      <td>Male</td>
      <td>101</td>
    </tr>
    <tr>
      <th>2</th>
      <td>E00000003</td>
      <td>E00000003</td>
      <td>1</td>
      <td>Female</td>
      <td>123</td>
    </tr>
    <tr>
      <th>3</th>
      <td>E00000003</td>
      <td>E00000003</td>
      <td>2</td>
      <td>Male</td>
      <td>135</td>
    </tr>
    <tr>
      <th>4</th>
      <td>E00000005</td>
      <td>E00000005</td>
      <td>1</td>
      <td>Female</td>
      <td>52</td>
    </tr>
  </tbody>
</table>
</div>




```python
females = gender[gender['Sex (2 categories)'] == 'Female']
gender_total = gender.groupby(['Output Areas Code'])['Observation'].sum().reset_index()
percent_female = females[['Output Areas', 'Observation']].merge(gender_total, left_on='Output Areas', right_on='Output Areas Code')
percent_female['pct'] = percent_female['Observation_x'] / percent_female['Observation_y']
percent_female.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Output Areas</th>
      <th>Observation_x</th>
      <th>Output Areas Code</th>
      <th>Observation_y</th>
      <th>pct</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>E00000001</td>
      <td>75</td>
      <td>E00000001</td>
      <td>176</td>
      <td>0.426136</td>
    </tr>
    <tr>
      <th>1</th>
      <td>E00000003</td>
      <td>123</td>
      <td>E00000003</td>
      <td>258</td>
      <td>0.476744</td>
    </tr>
    <tr>
      <th>2</th>
      <td>E00000005</td>
      <td>52</td>
      <td>E00000005</td>
      <td>112</td>
      <td>0.464286</td>
    </tr>
    <tr>
      <th>3</th>
      <td>E00000007</td>
      <td>67</td>
      <td>E00000007</td>
      <td>144</td>
      <td>0.465278</td>
    </tr>
    <tr>
      <th>4</th>
      <td>E00000010</td>
      <td>71</td>
      <td>E00000010</td>
      <td>178</td>
      <td>0.398876</td>
    </tr>
  </tbody>
</table>
</div>



#### education


```python
education = pd.read_csv('data/uk-census-2021-education-oa.csv')
education.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Output Areas Code</th>
      <th>Output Areas</th>
      <th>Highest level of qualification (8 categories) Code</th>
      <th>Highest level of qualification (8 categories)</th>
      <th>Observation</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>E00000001</td>
      <td>E00000001</td>
      <td>-8</td>
      <td>Does not apply</td>
      <td>15</td>
    </tr>
    <tr>
      <th>1</th>
      <td>E00000001</td>
      <td>E00000001</td>
      <td>0</td>
      <td>No qualifications</td>
      <td>4</td>
    </tr>
    <tr>
      <th>2</th>
      <td>E00000001</td>
      <td>E00000001</td>
      <td>1</td>
      <td>Level 1 and entry level qualifications: 1 to 4...</td>
      <td>7</td>
    </tr>
    <tr>
      <th>3</th>
      <td>E00000001</td>
      <td>E00000001</td>
      <td>2</td>
      <td>Level 2 qualifications: 5 or more GCSEs (A* to...</td>
      <td>10</td>
    </tr>
    <tr>
      <th>4</th>
      <td>E00000001</td>
      <td>E00000001</td>
      <td>3</td>
      <td>Apprenticeship</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
</div>




```python
education[['Highest level of qualification (8 categories) Code','Highest level of qualification (8 categories)'	]].drop_duplicates()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Highest level of qualification (8 categories) Code</th>
      <th>Highest level of qualification (8 categories)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>-8</td>
      <td>Does not apply</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0</td>
      <td>No qualifications</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1</td>
      <td>Level 1 and entry level qualifications: 1 to 4...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2</td>
      <td>Level 2 qualifications: 5 or more GCSEs (A* to...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>3</td>
      <td>Apprenticeship</td>
    </tr>
    <tr>
      <th>5</th>
      <td>4</td>
      <td>Level 3 qualifications: 2 or more A levels or ...</td>
    </tr>
    <tr>
      <th>6</th>
      <td>5</td>
      <td>Level 4 qualifications and above: degree (BA, ...</td>
    </tr>
    <tr>
      <th>7</th>
      <td>6</td>
      <td>Other: vocational or work-related qualificatio...</td>
    </tr>
  </tbody>
</table>
</div>



here make the simplifying assumption that we ignore the 'does not apply category', and the categories code go up in more advanced education


```python
mean_education_level = education[['Output Areas Code', 'Highest level of qualification (8 categories) Code','Observation']]
mean_education_level['counts'] = mean_education_level['Highest level of qualification (8 categories) Code'].clip(lower=0) * mean_education_level['Observation']
mean_education_level = mean_education_level.groupby('Output Areas Code')['counts'].sum()
mean_education_level_totals = education[['Output Areas Code', 'Highest level of qualification (8 categories) Code','Observation']].groupby('Output Areas Code')['Observation'].sum()
mean_education_level = mean_education_level.reset_index().merge(mean_education_level_totals.reset_index(), on='Output Areas Code')
mean_education_level['avg_education_level'] = mean_education_level['counts'] /  mean_education_level['Observation']
mean_education_level.head()
```

    /tmp/ipykernel_28838/566605148.py:2: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    
    See the caveats in the documentation: https://pandas.pydata.org/pandas-docs/stable/user_guide/indexing.html#returning-a-view-versus-a-copy
      mean_education_level['counts'] = mean_education_level['Highest level of qualification (8 categories) Code'].clip(lower=0) * mean_education_level['Observation']





<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Output Areas Code</th>
      <th>counts</th>
      <th>Observation</th>
      <th>avg_education_level</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>E00000001</td>
      <td>714</td>
      <td>175</td>
      <td>4.080000</td>
    </tr>
    <tr>
      <th>1</th>
      <td>E00000003</td>
      <td>1022</td>
      <td>257</td>
      <td>3.976654</td>
    </tr>
    <tr>
      <th>2</th>
      <td>E00000005</td>
      <td>485</td>
      <td>113</td>
      <td>4.292035</td>
    </tr>
    <tr>
      <th>3</th>
      <td>E00000007</td>
      <td>640</td>
      <td>144</td>
      <td>4.444444</td>
    </tr>
    <tr>
      <th>4</th>
      <td>E00000010</td>
      <td>656</td>
      <td>179</td>
      <td>3.664804</td>
    </tr>
  </tbody>
</table>
</div>




```python
# add education level info
ldn_demographics_grid_processed = ldn_demographics_grid.merge(mean_education_level[[
    'Output Areas Code', 'avg_education_level']], left_on='OA21CD', right_on='Output Areas Code').drop(columns='Output Areas Code')
# add % female
ldn_demographics_grid_processed = ldn_demographics_grid_processed.merge(percent_female[[
    'Output Areas Code', 'pct']], left_on='OA21CD', right_on='Output Areas Code').drop(columns='Output Areas Code').rename(columns={'pct': 'pct_females'})
# add mean household size
ldn_demographics_grid_processed = ldn_demographics_grid_processed.merge(mean_household_size[[
    'Output Areas Code', 'avg_household_size']], left_on='OA21CD', right_on='Output Areas Code').drop(columns='Output Areas Code')


ldn_demographics_grid_processed = ldn_demographics_grid_processed.merge(population_oa[[
    'Output Areas Code', 'Observation']], left_on='OA21CD', right_on='Output Areas Code').drop(columns='Output Areas Code').rename(
    columns={'Observation': 'pop_density'})
ldn_demographics_grid_processed['popoulation'] = ldn_demographics_grid_processed['pop_density'] * (100 * 100)/(1000 * 1000)
ldn_demographics_grid_processed.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>cellid</th>
      <th>OA21CD</th>
      <th>geometry</th>
      <th>avg_education_level</th>
      <th>pct_females</th>
      <th>avg_household_size</th>
      <th>pop_density</th>
      <th>popoulation</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>E00155442</td>
      <td>POLYGON ((503000.000 155200.000, 503000.000 15...</td>
      <td>2.839161</td>
      <td>0.522807</td>
      <td>2.567568</td>
      <td>915.5</td>
      <td>9.155</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>E00155442</td>
      <td>POLYGON ((503000.000 155300.000, 503000.000 15...</td>
      <td>2.839161</td>
      <td>0.522807</td>
      <td>2.567568</td>
      <td>915.5</td>
      <td>9.155</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>E00155442</td>
      <td>POLYGON ((503000.000 155400.000, 503000.000 15...</td>
      <td>2.839161</td>
      <td>0.522807</td>
      <td>2.567568</td>
      <td>915.5</td>
      <td>9.155</td>
    </tr>
    <tr>
      <th>3</th>
      <td>467</td>
      <td>E00155442</td>
      <td>POLYGON ((503100.000 155200.000, 503100.000 15...</td>
      <td>2.839161</td>
      <td>0.522807</td>
      <td>2.567568</td>
      <td>915.5</td>
      <td>9.155</td>
    </tr>
    <tr>
      <th>4</th>
      <td>468</td>
      <td>E00155442</td>
      <td>POLYGON ((503100.000 155300.000, 503100.000 15...</td>
      <td>2.839161</td>
      <td>0.522807</td>
      <td>2.567568</td>
      <td>915.5</td>
      <td>9.155</td>
    </tr>
  </tbody>
</table>
</div>




```python
ldn_demographics_grid_processed.to_file('data/ldn_demographics_grid_processed.shp')
```

    /tmp/ipykernel_28838/324827064.py:1: UserWarning: Column names longer than 10 characters will be truncated when saved to ESRI Shapefile.
      ldn_demographics_grid_processed.to_file('data/ldn_demographics_grid_processed.shp')


### Rasterizing geodataframe

Certain tools work better with raster data, and since we have converted our data from a irregular geometry into cells, we can go all the way and convert the vector data into raster, using [geocube](https://corteva.github.io/geocube). 

**Capping/binning population values**

The population values have a fairly long tail-- let's cap this so it is easier to plot and handle


```python
# printing out some stats -- we can see 95% cells have < 150 people

ldn_demographics_grid_processed['popoulation'].quantile([0.2, 0.4, 0.6, 0.8]),ldn_demographics_grid_processed['popoulation'].quantile([0.05]),  ldn_demographics_grid_processed['popoulation'].quantile([0.95]), ldn_demographics_grid_processed['popoulation'].min(), ldn_demographics_grid_processed['popoulation'].max()
```




    (0.2     1.024
     0.4     4.157
     0.6    21.138
     0.8    62.500
     Name: popoulation, dtype: float64,
     0.05    0.321
     Name: popoulation, dtype: float64,
     0.95    130.5817
     Name: popoulation, dtype: float64,
     0.156,
     2393.333)




```python
ldn_demographics_grid_processed['popoulation'] = ldn_demographics_grid_processed['popoulation'].clip(lower=0, upper=150)
```


```python
ldn_demographics_grid_processed['popoulation'].max()
```




    150.0



Now we convert the vector to raster (the resolution is what we are using for each of our cells-- 100m )


```python
ldn_demographics_raster = make_geocube(
    vector_data=ldn_demographics_grid_processed,
    measurements=["popoulation", "avg_education_level", "pct_females"],
    resolution=(-100, 100),
)

```


```python
ldn_demographics_raster['popoulation'].plot()
```




    <matplotlib.collections.QuadMesh at 0x7f8adde759c0>




    
![png](/2023-02-01-uk-population-census-2021/output_56_1.png)
    



```python
ldn_demographics_raster['pct_females'].plot()
```




    <matplotlib.collections.QuadMesh at 0x7f8b0c795db0>




    
![png](/2023-02-01-uk-population-census-2021/output_57_1.png)
    



```python
ldn_demographics_raster['avg_education_level'].plot()
```




    <matplotlib.collections.QuadMesh at 0x7f8b0d76e2f0>




    
![png](/2023-02-01-uk-population-census-2021/output_58_1.png)
    



```python
# save our raster data
ldn_demographics_raster.rio.to_raster("data/ldn_demographics_raster.tiff")
```


```python

```
