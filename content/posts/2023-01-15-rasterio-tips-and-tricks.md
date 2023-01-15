There are two main ways geospatial data are stored
- rasters, where each 'pixel' stores data values. This corresponds to files such as geotiffs.
- vectors, where information is stored more like a table form, and each row will have a 'geometry' field which stores information that allows you to recreate a geographical feature, such as a point , a polygon etc. It might also store the coordinate reference system so you can correctly place the shapes on a map

This set of notes corresponds to handling raster data. For this, we will make use of the [rasterio](https://rasterio.readthedocs.io/en/latest/index.html) library for manipulating the data, as well as [pyproj](https://pyproj4.github.io/pyproj/stable/) for coordinate transforms.

(You can access a running version of this notebook on colab [here](https://colab.research.google.com/drive/1ZW7633qzhV7UWxcBNYV8jo4Ok9a5oI2i))

### Setup

For this demo, I will be using a population dataset from NASA ["Gridded Population of the World, Version 4 (GPWv4): Population Count Adjusted to Match 2015 Revision of UN WPP Country Totals"](http://sedac.ciesin.columbia.edu/data/collection/gpw-v4/sets/browse)


```python
!pip install rasterio pyproj --quiet
```


```python
import rasterio
import pyproj
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

%matplotlib inline
```

### peak at data:
asc is much like a text file and we can use conventional text handling tools and a bit of pandas to parse it, but it is much less mistake prone if we use rasterio to handle it


```python
!cat gpw_v4_population_count_adjusted_to_2015_unwpp_country_totals_rev11_2020_1_deg.asc | head -6
```

    ncols         360
    nrows         180
    xllcorner     -180
    yllcorner     -90
    cellsize      1.0000000000001
    NODATA_value  -9999
    -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 -9999 
    ...



```python
!cat gpw_v4_population_count_adjusted_to_2015_unwpp_country_totals_rev11_2020_1_deg.prj | head -10
```

    Projection    GEOGRAPHIC
    Datum         WGS84
    Spheroid      WGS84
    Units         DD
    Zunits        NO
    Parameters    


### parsing the data

For this example, we would like to convert the raster to a tabular form we can store in a 'conventional' database/csv file. For this, we will do these tasks:
1. read the file into memory
2. filter out any pixels/cells with no data
3. convert the data into a 'long' table with the format `longitude | latitude | population | `
4. obtain the lat/lng values in [web mercator coordinates](https://epsg.io/3857)

1. Read the file: note that rasterio can also open files on certain cloud providers, such as GCS/S3 etc, just pass it the relevant filepath in `gs://`, `s3://` format


```python
dataset =  rasterio.open("gpw_v4_population_count_adjusted_to_2015_unwpp_country_totals_rev11_2020_1_deg.asc") 
```

the dataset object holds metadata:


```python
# extent of the data and crs -- note we are getting the CRS info from the .prj file also supplied
dataset.bounds, dataset.crs
```




    (BoundingBox(left=-180.0, bottom=-90.0, right=180.0, top=90.0),
     CRS.from_epsg(4326))



The actual values of the pixels are in the 1st 'band'. There can be more than 1 band-- for example satellite imagery can comes in > 10 bands. We can either read all the bands in at once, or one by one. (Note that the indexing starts with 1 for the bands if reading in one by one)


```python

data= dataset.read()
data.shape
```




    (1, 180, 360)




```python
# from the above, we know that there is only 1 band, so we can convert the data to a 2d array
data = data.squeeze()
```

2. remove all pixels with no data


```python

valid_rows, valid_cols = np.nonzero(data != dataset.get_nodatavals())
valid_data_vals = data[(valid_rows, valid_cols)]
```


```python
plt_data = np.zeros_like(data)
plt_data[(valid_rows, valid_cols)] = valid_data_vals

f, ax = plt.subplots(figsize=(12, 8))
ax1 = ax.imshow(np.log10(plt_data+0.001), cmap='pink')
f.colorbar(ax1, label=f"log10(population density)");
```


    
![png](/2023-01-15-rasterio-tips-and-tricks_files/rasterio_tipsandtricks_17_0.png)
    


3. Convert pixel positions to coordinates:


```python
# covert rows, cols to coordinates:
valid_lngs, valid_lats = rasterio.transform.xy(dataset.transform, valid_rows, valid_cols)
output_dataframe = pd.DataFrame(
    data = dict(
        lat=valid_lats,
        lng = valid_lngs,
        population = valid_data_vals
    )
)
output_dataframe.sample(frac=1.).head()
```





  <div id="df-e5bdb33b-903f-4a08-a4cf-9f0c6dceae91">
    <div class="colab-df-container">
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
      <th>lat</th>
      <th>lng</th>
      <th>population</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>6506</th>
      <td>53.5</td>
      <td>124.5</td>
      <td>9.351563e+03</td>
    </tr>
    <tr>
      <th>1035</th>
      <td>75.5</td>
      <td>141.5</td>
      <td>1.780356e+02</td>
    </tr>
    <tr>
      <th>3034</th>
      <td>66.5</td>
      <td>72.5</td>
      <td>1.972377e+03</td>
    </tr>
    <tr>
      <th>7488</th>
      <td>48.5</td>
      <td>-113.5</td>
      <td>9.566622e+03</td>
    </tr>
    <tr>
      <th>10740</th>
      <td>32.5</td>
      <td>116.5</td>
      <td>4.994749e+06</td>
    </tr>
  </tbody>
</table>
</div>
      <button class="colab-df-convert" onclick="convertToInteractive('df-e5bdb33b-903f-4a08-a4cf-9f0c6dceae91')"
              title="Convert this dataframe to an interactive table."
              style="display:none;">

  <svg xmlns="http://www.w3.org/2000/svg" height="24px"viewBox="0 0 24 24"
       width="24px">
    <path d="M0 0h24v24H0V0z" fill="none"/>
    <path d="M18.56 5.44l.94 2.06.94-2.06 2.06-.94-2.06-.94-.94-2.06-.94 2.06-2.06.94zm-11 1L8.5 8.5l.94-2.06 2.06-.94-2.06-.94L8.5 2.5l-.94 2.06-2.06.94zm10 10l.94 2.06.94-2.06 2.06-.94-2.06-.94-.94-2.06-.94 2.06-2.06.94z"/><path d="M17.41 7.96l-1.37-1.37c-.4-.4-.92-.59-1.43-.59-.52 0-1.04.2-1.43.59L10.3 9.45l-7.72 7.72c-.78.78-.78 2.05 0 2.83L4 21.41c.39.39.9.59 1.41.59.51 0 1.02-.2 1.41-.59l7.78-7.78 2.81-2.81c.8-.78.8-2.07 0-2.86zM5.41 20L4 18.59l7.72-7.72 1.47 1.35L5.41 20z"/>
  </svg>
      </button>

  <style>
    .colab-df-container {
      display:flex;
      flex-wrap:wrap;
      gap: 12px;
    }

    .colab-df-convert {
      background-color: #E8F0FE;
      border: none;
      border-radius: 50%;
      cursor: pointer;
      display: none;
      fill: #1967D2;
      height: 32px;
      padding: 0 0 0 0;
      width: 32px;
    }

    .colab-df-convert:hover {
      background-color: #E2EBFA;
      box-shadow: 0px 1px 2px rgba(60, 64, 67, 0.3), 0px 1px 3px 1px rgba(60, 64, 67, 0.15);
      fill: #174EA6;
    }

    [theme=dark] .colab-df-convert {
      background-color: #3B4455;
      fill: #D2E3FC;
    }

    [theme=dark] .colab-df-convert:hover {
      background-color: #434B5C;
      box-shadow: 0px 1px 3px 1px rgba(0, 0, 0, 0.15);
      filter: drop-shadow(0px 1px 2px rgba(0, 0, 0, 0.3));
      fill: #FFFFFF;
    }
  </style>

      <script>
        const buttonEl =
          document.querySelector('#df-e5bdb33b-903f-4a08-a4cf-9f0c6dceae91 button.colab-df-convert');
        buttonEl.style.display =
          google.colab.kernel.accessAllowed ? 'block' : 'none';

        async function convertToInteractive(key) {
          const element = document.querySelector('#df-e5bdb33b-903f-4a08-a4cf-9f0c6dceae91');
          const dataTable =
            await google.colab.kernel.invokeFunction('convertToInteractive',
                                                     [key], {});
          if (!dataTable) return;

          const docLinkHtml = 'Like what you see? Visit the ' +
            '<a target="_blank" href=https://colab.research.google.com/notebooks/data_table.ipynb>data table notebook</a>'
            + ' to learn more about interactive tables.';
          element.innerHTML = '';
          dataTable['output_type'] = 'display_data';
          await google.colab.output.renderOutput(dataTable, element);
          const docLink = document.createElement('div');
          docLink.innerHTML = docLinkHtml;
          element.appendChild(docLink);
        }
      </script>
    </div>
  </div>




Add lat lng values in web mercator:

(note that it is a good idea to  `always_xy` param to be true, this means the order of the output is always in the coordinates' equivalent to longitude, latitude. There are CRS that can end up swapping the x, y axis from the source CRS)


```python
dataset.crs
```




    CRS.from_epsg(4326)




```python
transformer = pyproj.Transformer.from_crs("EPSG:4326", "EPSG:3857", always_xy=True)
```


```python
lng_3857, lat_3857 = transformer.transform(output_dataframe.lng.values.squeeze(), output_dataframe.lat.values.squeeze())

output_dataframe['lng_3857'] = lng_3857
output_dataframe['lat_3857'] = lat_3857
output_dataframe.sample(frac=1.).head()
```





  <div id="df-f0660fcc-3190-43dc-91e0-eafb6d469c12">
    <div class="colab-df-container">
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
      <th>lat</th>
      <th>lng</th>
      <th>population</th>
      <th>lng_3857</th>
      <th>lat_3857</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>11941</th>
      <td>25.5</td>
      <td>124.5</td>
      <td>2.870859e+01</td>
      <td>1.385928e+07</td>
      <td>2.937284e+06</td>
    </tr>
    <tr>
      <th>17692</th>
      <td>-22.5</td>
      <td>117.5</td>
      <td>4.557170e+03</td>
      <td>1.308004e+07</td>
      <td>-2.571663e+06</td>
    </tr>
    <tr>
      <th>44</th>
      <td>82.5</td>
      <td>-77.5</td>
      <td>3.311810e-02</td>
      <td>-8.627261e+06</td>
      <td>1.738069e+07</td>
    </tr>
    <tr>
      <th>9618</th>
      <td>38.5</td>
      <td>46.5</td>
      <td>1.455383e+06</td>
      <td>5.176356e+06</td>
      <td>4.650302e+06</td>
    </tr>
    <tr>
      <th>16594</th>
      <td>-12.5</td>
      <td>122.5</td>
      <td>1.345766e-03</td>
      <td>1.363664e+07</td>
      <td>-1.402665e+06</td>
    </tr>
  </tbody>
</table>
</div>
      <button class="colab-df-convert" onclick="convertToInteractive('df-f0660fcc-3190-43dc-91e0-eafb6d469c12')"
              title="Convert this dataframe to an interactive table."
              style="display:none;">

  <svg xmlns="http://www.w3.org/2000/svg" height="24px"viewBox="0 0 24 24"
       width="24px">
    <path d="M0 0h24v24H0V0z" fill="none"/>
    <path d="M18.56 5.44l.94 2.06.94-2.06 2.06-.94-2.06-.94-.94-2.06-.94 2.06-2.06.94zm-11 1L8.5 8.5l.94-2.06 2.06-.94-2.06-.94L8.5 2.5l-.94 2.06-2.06.94zm10 10l.94 2.06.94-2.06 2.06-.94-2.06-.94-.94-2.06-.94 2.06-2.06.94z"/><path d="M17.41 7.96l-1.37-1.37c-.4-.4-.92-.59-1.43-.59-.52 0-1.04.2-1.43.59L10.3 9.45l-7.72 7.72c-.78.78-.78 2.05 0 2.83L4 21.41c.39.39.9.59 1.41.59.51 0 1.02-.2 1.41-.59l7.78-7.78 2.81-2.81c.8-.78.8-2.07 0-2.86zM5.41 20L4 18.59l7.72-7.72 1.47 1.35L5.41 20z"/>
  </svg>
      </button>

  <style>
    .colab-df-container {
      display:flex;
      flex-wrap:wrap;
      gap: 12px;
    }

    .colab-df-convert {
      background-color: #E8F0FE;
      border: none;
      border-radius: 50%;
      cursor: pointer;
      display: none;
      fill: #1967D2;
      height: 32px;
      padding: 0 0 0 0;
      width: 32px;
    }

    .colab-df-convert:hover {
      background-color: #E2EBFA;
      box-shadow: 0px 1px 2px rgba(60, 64, 67, 0.3), 0px 1px 3px 1px rgba(60, 64, 67, 0.15);
      fill: #174EA6;
    }

    [theme=dark] .colab-df-convert {
      background-color: #3B4455;
      fill: #D2E3FC;
    }

    [theme=dark] .colab-df-convert:hover {
      background-color: #434B5C;
      box-shadow: 0px 1px 3px 1px rgba(0, 0, 0, 0.15);
      filter: drop-shadow(0px 1px 2px rgba(0, 0, 0, 0.3));
      fill: #FFFFFF;
    }
  </style>

  </div>





```python
# verify and sanity check
output_dataframe.isnull().sum()
```




    lat           0
    lng           0
    population    0
    lng_3857      0
    lat_3857      0
    dtype: int64

