---
title: "Aggregating Data"
teaching: 20
exercises: 15
questions:
- "How to I aggregate data over various dimensions?"
objectives:
- "Lean how to apply aggregating functions across multidimensional DataArrays which, in other languages, would require looping."
keypoints:
- "Aggregation functions are used often in climate data analysis, and are made easy with xarray"
---

The Oceanic Niño Index [ONI](https://origin.cpc.ncep.noaa.gov/products/analysis_monitoring/ensostuff/ONI_v5.php) is used by the Climate Prediction Center to monitor and predict El Niño and La Niña. It is defined as the 3-month running mean of SST anomalies in the Niño3.4 region. We will use aggregation methods from `xarray` to calculate this index. 

## First Steps

Create a new notebook and save it as Agg_Mask_Interp.ipynb

Import the standard set of packages we use:

~~~
import xarray as xr
import numpy as np
import cartopy.crs as ccrs
import matplotlib.pyplot as plt
~~~
{: .language-python}

Change the next cell from **code** to **markdown** and add a title for this episode:

~~~
## Aggregation
~~~
{: .language-python}

Read in the dataset we wrote in the last notebook.

~~~
file='/scratch/[your_userid]/nino34_1982-2019.oisstv2.nc'
ds=xr.open_dataset(file)
ds
~~~
{: .language-python}

## Take the mean over a lat-lon region

~~~
ds_nino34_index=ds.mean(dim=['lat','lon'])
ds_nino34_index
~~~
{: .language-python}

Our data now has only a time dimension. Make a plot.

~~~
plt.plot(ds_nino34_index['time'],ds_nino34_index['sst'])
~~~
{: .language-python}

## Take the weighted mean over a lat-lon region

When performing a spatial average of data on a lat-lon grid we need to take into account the fact that the area of each grid cell is not the same because lines of longitude converge as they approach the poles. This convergence leads to a decrease in delta x (east-west distance in m) following the cosin of latitude, and therefore grid cell area (delta x * delta y). One must therefore perform an area weighted average = sum(var * weights) / sum(weights) where the weights are proportional to the area of each grid cell i.e. the cosin of latitude

Xarray has some build in funtionality to assist in calculating this weighted area average. First you will need to create a “weights” array proportional to cosine of latitude, this is the area-weighting factor for data on a regular lat-lon grid.

~~~
weights = np.cos(np.deg2rad(ds.lat))
weights.dims
~~~
{: .language-python}

We see that the weights variable has the dimensions of latitude.

~~~
plt.plot(ds.lat,weights)
~~~
{: .language-python}

Let's evaluate the impact of multiplying our SST feild by these weights. 

~~~
weighted_sst = ds.sst * weights
plt.plot(ds.lat,weighted_sst[0,:,0])
plt.plot(ds.lat,ds.sst[0,:,0])
~~~
{: .language-python}

This weighting leads to small difference because of our small tropical domain, but would be more important if calucalting an domain average over a larger region or closer to the poles.

Next let's use Xarray's build in fuctionality for weighted reductions. It create a special intermediate DataArrayWeighted object, to which different reduction operations can applied - https://xarray-contrib.github.io/xarray-tutorial/scipy-tutorial/03_computation_with_xarray.html

~~~
sst_weighted = ds.sst.weighted(weights)
sst_weighted
~~~
{: .language-python}

Now let's use this to calculate the area weighted mean

~~~
ds_nino34_index_weighted=sst_weighted.mean(dim=("lon", "lat"))
~~~
{: .language-python}

Because we are in a small tropical domain this makes a very small difference, but is very important when taking the gloabl mean of a feild for example.

~~~
plt.plot(ds_nino34_index['time'],ds_nino34_index['sst'])
plt.plot(ds_nino34_index_weighted['time'],ds_nino34_index_weighted)
~~~
{: .language-python}

## Calculate Anomalies

An anomaly is a departure from normal (i.e., the climatology).
We often calculate anomalies for working with climate data.
We will spend time in a future class learning about calculating climatology and anomalies using another feature of `xarray` called `groupby`.
For now we will proceed with little explanation - but you may intuit that in the code below, we are grouping by month.
In other words, we are calculating the mean SST for each month and ending up with a climatology whose time dimension is only 12 months:

~~~
ds_climo=ds_nino34_index.groupby('time.month').mean()
ds_climo
~~~
{: .language-python}

~~~
plt.plot(ds_climo.sst)
~~~
{: .language-python}

Below we calculate anomalies by subtracting the climatology from the original time series:

~~~
ds_anoms=ds_nino34_index.groupby('time.month')-ds_climo
ds_anoms
~~~
{: .language-python}

Plot our data.

~~~
plt.plot(ds_anoms['time'],ds_anoms['sst'])
~~~
{: .language-python}

> ## Why are we constantly printing and ploting the data?
>
> Printing the data is a way to see its dimensions after each
> step. It provides a check on whether your code did what you 
> intended it to do. You can often catch and fix problems early this way.
>
> Plotting also provides a quick check to make sure 
> everything _looks reasonable_.
>
> We encourage you to do the same _checking along the way_ when developing and 
> testing new code!
>
>
{: .callout}

## Rolling (Running Means)

The ONI is calculated using a 3-month running mean.  This can be done using the `rolling` method.

> ## Reading and Learning from Documentation
>
> Read the [documentation](http://xarray.pydata.org/en/stable/generated/xarray.DataArray.rolling.html) for the `xarray.rolling` method.
> Following their example, make a 3-month running mean of the `ds_anoms` data.
>
>
>> ## Solution
>>> ~~~
>>> ds_3m=ds_anoms.rolling(time=3,center=True).mean().dropna(dim='time') 
>>> ds_3m
>>> ~~~
>> {: .language-python}
> {: .solution}
>  
> Note the time dimension and coordinates of `ds_3m` and compare to the time dimension of `ds_anoms`. What has happened?
> 
>> ## Solution
>> The centered 3-month running mean cannot calculate a value for the first and last time steps.
>> Those had been set to `NaN` (not a number) and we clipped those off with the `.dropna()` 
>> Thus, `ds_3m` is two months shorter than `ds_anoms`, starting in February 1982 and ending with November 2019.
>> 
> {: .solution}
{: .challenge}

Let's plot our original and 3-month running mean data together

~~~
plt.plot(ds_anoms['time'],ds_anoms['sst'],color='r')
plt.plot(ds_3m['time'],ds_3m['sst'],color='b')
plt.legend(['original','ONI'])
~~~
{: .language-python}

Note that the plotting was not confused by the fact that the two time series were of different lengths. 
Because we specified the time coordinate of each as its X-axis values, `plt.plot` could navigate the data and plot it correctly.

> ## Some other aggregation functions
>
> There are a number of other aggregate functions such as: `std`,`min`,`max`,`sum`, among others.
>
> Using the original dataset in this notebook `ds`, find and plot the maximum SSTs for each
> gridpoint over the time dimension.
>
>> ## Solution
>>> ~~~
>>> ds_max=ds.max(dim='time')
>>> plt.contourf(ds.lon,ds.lat,ds_max['sst'],cmap='Reds_r')
>>> plt.colorbar()
>>> ~~~
>> {: .language-python}
> {: .solution}
>
> Using the original dataset in this notebook `ds`, calculate and plot the standard deviation at each
> gridpoint.
>
>> ## Solution
>>> ~~~
>>> ds_std=ds.std(dim='time')
>>> plt.contourf(ds.lon,ds.lat,ds_std['sst'],cmap='YlGnBu_r')
>>> plt.colorbar()
>>> ~~~
>> {: .language-python}
> {: .solution}
>
{: .challenge}

