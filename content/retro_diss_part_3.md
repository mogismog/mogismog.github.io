Title: Retrospective Dissertation - Part 3: Reforecast Data
Date: 2020-07-18
Category: retrodiss
Tags: python, weather
Slug: retro-diss-part-2
Author: fma
Summary: Getting The GEFS Reforecast Dataset

*In [Part 2](https://mogismog.github.io/retro-diss-part-2.html) of Retrospective Dissertation, we downloaded, processed, and explored SPC's tornado reports dataset. In Part 3, we'll introduce the GEFS Reforecast dataset, which we'll use as our features for tornado prediction!*

## 0. A Bit of Backstory

Back in 2010, I had finished my master's research on [rapid cyclogenesis](https://en.wikipedia.org/wiki/Explosive_cyclogenesis) (more commonly known as "bomb" cyclones) within the first version of NOAA's Climate Forecast System (CFSv1). I was excited to be done, and the plan going forward was to continue more in-depth research for my dissertation. I had presented the research at both the American Meteorological Society's and the American Geophysical Union's annual conferences, and it was (mostly) well received, so it made sense to continue the research.

In order to be classified as a rapid cyclogenesis event, an extra-tropical cyclone needs to have a decrease in pressure over 24 hours (corrected for latitude):

$$\frac{(24 *sin(\phi)/sin(60))mb}{24hr}$$

Which means if this value, known as a Bergeron, is >= 1, then it is an explosive cyclogenesis event, and the higher above 1, the more intense the storm. So around August of 2010, I'm looking over my FORTRAN 90 code that processed these events, and that's when I noticed a bug in the code: Some where along the way, probably when I wanted to test something out, I had it hardcoded to exclude any storm that was fewer than 3.5 Bergerons. I removed the filter, reran the code, and not only was I throwing out like 95% of all of the data, I also got much different results than in any of my conference posters/presentations and my thesis. Basically, I invalidated all of my research.

Needless to say, I was distraught and kind of just went through the motions the next few months. I found myself questioning whether I was good enough to even do a PhD, clearly someone smart wouldn't have made this mistake. I let my advisor know how I didn't want to continue on with the research, and that I wasn't sure I wanted to do a PhD. Thankfully, my advisor was (and still is) a really great person, and said that I could switch topics and just hammer out the dissertation research in 3-4 years. He also mentioned that, when he worked at NOAA, he came across a super fascinating concept of generating a ton of retrospective forecasts from the same weather model and building post-processing models from the output. My advisor asked me to write up a 2-3 pager suggesting a research idea that used these retrospective forecasts, and that he would reach out to Tom Hamill at NOAA's Earth Systems Research Laboratory.

I wrote up an, uh, interesting idea involving genetic algorithms and precipitation prediction, which I thought was super cool. Tom, who is an absolutely incredible person, was very kind and polite in his response: That's an ok idea, but how about we work on one of these three other ideas that may have more value? One of those ideas was, of course, probabilistic tornado prediction. I decided on that one, and the best decision I could have made was started by one of the worst mistakes I have made.

## 1. Reforecast Data

But enough backstory, let's talk about the data! So the idea behind reforecasts is that we have high quality reanalysis data (basically, a best guess at the state of the atmosphere based on various observations) and we have high quality numerical weather prediction (NWP) models. If we were to "freeze" the NWP model, so we would use the same data assmiliation and parameterization techniques, we could use the reanalysis data as our initial conditions and generate a large number of historical weather forecasts. This dataset, then, would have the same state-dependent model biases and errors across the entire time series. Having a large sample size allows for the correction of these state-dependent model biases that may have a low signal relative to the noise, while also having a good representation for rare or unusual events, such as heavy precipitation or, I dunno, tornadoes.

Back in 2011, we used the then [state-of-the-art GEFS Reforecast Version 2](https://psl.noaa.gov/forecasts/reforecast2/) dataset, which was great to use but difficult to get. The data was at a 1 degree x 1 degree resolution, was in grib files, and prior to the web app required pulling grib files from an FTP server. I used a bash script and a tool from NOAA called [`wgrib2`](https://www.cpc.ncep.noaa.gov/products/wesley/wgrib2/) to select the slices of data I needed within each file, pull it down, and save it to a set of 3, 1TB external hard drives I had bought. It was a different world back then...

However, it's 2020 (for better and _definitely_ for worse), and that just won't cut it. Tom let me know that there is a 💫awesome brand new reforecast dataset 💫 that is publicly available on AWS! This data is a huge upgrade from the second version, with the spatial resolution around 0.25 degrees (0.5 degrees for upper air data to save space), forecast steps every three hours, and brand new fields that would be useful for severe weather forecasting!!! Additionally, this data begins in 2000, so uh forget everything I had said in the prior post about tornadoes and needing to use only data from 1990, because we are going to use this new data set from here on out.

## 2. Getting The Data

As part of NOAA's "big data" initiative, the Reforecast V3 data is located on S3 at `s3://noaa-gefs-retrospective/GEFSv12/reforecast`. The data is set up in the following way:

- Forecasts were generated every day at 00Z from 2000-01-01 through 2019-12-31
- Forecast data is provided every three hours out to 10 days, and every six hours from 10-16 days.
- The spatial resolution is 0.25-degree lat/lon grid spacing out to 10 days, and 0.5-degree from 10-16 days. An exception is the upper air data (above 700mb), which always has 0.5-degree spatial resolution.
- The current GEFSv12 provides 31 ensemble members (one control and 30 forecasts with small amounts of noise added to the initial conditions), whereas the Reforecast V3 has one control (`c0`) and four perturbed members (`p1` through `p4`).
- Directory structure on S3 follows a `YYYY/YYYYMMDD00/member/days/field` format, where `days` is either `'Days:1-10'` or `'Days:10-16'`.

Following my dissertation's research, we'll opt to pull the following fields:

- Convective Available Potential Energy (`cape_sfc`): Basically the amount of energy available for a statically unstable parcel lifted to a specific level (higher CAPE can lead to more severe thunderstorms).
- Convective Inhibition (`cin_sfc`): A measure of the energy of negative buoyancy that can prevent rising parcels from reaching the level of free convection.
- 0-3 km Storm Relative Helicity (`hlcy_hgt`): [According to SPC](https://www.spc.noaa.gov/exper/mesoanalysis/help/help_srh1.html), this is a measure of the potential for cyclonic updraft rotation in right-moving supercells.
- 950mb and 500mb U/V wind components (`vgrd_pres` and `ugrd_pres`): We'll use these to compute 0-6km wind shear, which research has shown to be important for super cell and tornado development.
- Convective precipitation (`acpcp_sfc`), which we'll use as a feature simply because I used it in my dissertation research as kind of a indicator feature for thunderstorm activity.

As I mentioned before, I got the old reforecast data through a bash script, which sequentially downloaded daily files for seven different fields (CAPE, CIN, precip, and the u/v wind components). These files were then opened up within Python via `pygrib`, and I did whatever processing was required. That, uh, was pretty slow.

As a comparison, I wrote up a script to download the V3 grib files, process them on the fly, save them as individual netCDF files, and then combine all of them into a single netCDF file via `xarray`.

You can check out the full script [here](https://github.com/mogismog/redissertation/blob/master/redissertation/process_reforecast_data.py), but for this post we will just look at two of the functions that do the heavy lifting:

```python
def try_to_open_grib_file(path: str,) -> xr.Dataset:
    """Try a few different ways to open up a grib file.

    Parameters
    ----------
    path : str
        Path pointing to location of grib file

    Returns
    -------
    ds : xr.Dataset
        The xarray Dataset that contains information
        from the grib file.
    """
    try:
        ds = xr.open_dataset(path, engine='cfgrib')
    except Exception as e:
        try:
            import cfgrib
            ds = cfgrib.open_datasets(path)
            ds = xr.combine_by_coords(ds)
        except:
            logger.error(f'Oh no! There was a problem opening up {path}: {e}')
            return
    return ds


def download_and_process_grib(s3_prefix: str, latitude_bounds: Iterable[float],
                              longitude_bounds: Iterable[float],
                              forecast_days_bounds: Iterable[float],
                              save_dir: str,
                              pressure_levels: Iterable[float]=[950.0, 500.0]) -> str:
    """Get a reforecast grib off S3, process, and save locally as netCDF file.

    Parameters
    ----------
    s3_prefix : str
        S3 key/prefix/whatever it's called of a single grib file.
    latitude_bounds : Iterable[float]
        An iterable that contains the latitude bounds, in degrees,
        between -90-90.
    longitude_bounds : Iterable[float]
        An iterable that contains the longitude bounds, in degrees,
        between 0-360.
    forecast_days_bounds : Iterable[float]
        An iterable that contains the first/last forecast days.
    save_dir : str
        Local directory to save resulting netCDF file.
    pressure_levels : Iterable[float]
        Pressure levels to extract from fields that have pressure levels.

    Returns
    -------
    saved_file_path : str
        The location of the saved file.
    """
    base_file_name = s3_prefix.split('/')[-1]
    saved_file_path = os.path.join(save_dir, f"{base_file_name.split('.')[0]}.nc")
    if pathlib.Path(saved_file_path).exists():
        return saved_file_path

    logger.info(f'Processing {s3_prefix}')
    selection_dict = create_selection_dict(latitude_bounds, longitude_bounds, forecast_days_bounds)

    fs = s3fs.S3FileSystem(anon=True)
    try:
        with TemporaryDirectory() as t:
            grib_file = os.path.join(t, base_file_name)
            with fs.open(s3_prefix, 'rb') as f, open(grib_file, 'wb') as f2:
                f2.write(f.read())
            ds = try_to_open_grib_file(grib_file)
            if ds is None:
                return
            if 'isobaricInhPa' in ds.coords:
                pressure_level = [p for p in pressure_levels if p in ds['isobaricInhPa']]
                selection_dict['isobaricInhPa'] = pressure_level
            ds = ds.sel(selection_dict)
            # NOTE: The longitude is originally between 0-360, but
            # for our purpose, we'll convert it to be between -180-180.
            ds['longitude'] = (('longitude',), np.mod(ds['longitude'].values + 180.0, 360.0) - 180.0)
            ds = ds.assign(time=ds.time + ds['step'].max())
            if 'pcp' in base_file_name:
                ds = ds.sum('step')
            else:
                ds = ds.mean('step')
            # now, we need to reshape the data
            ds = ds.expand_dims('time', axis=0).expand_dims('number', axis=1)
            # set data vars to float32
            for v in ds.data_vars.keys():
                ds[v] = ds[v].astype(np.float32)
            ds = ds.drop(COMMON_COLUMNS_TO_DROP, errors='ignore')
            ds.to_netcdf(saved_file_path, compute=True)
    except Exception as e:
        logging.error(f'Oh no! There was an issue processing {grib_file}: {e}')
        return
    logging.info(f'All done with {saved_file_path}')
    return saved_file_path
```

So we first download a grib file via `s3fs` into a temporary directory, use `xarray` and `cfgrib` to try and open up each file a number of ways, process the data via defined selection criteria, and save it locally as a netCDF file (which, at this point, is a really small file!). These functions are part of a script you can use on the command line. I use the excellent `click` for help with our command line arguments and options, and `joblib` to parallelize the entire process: 

```python
@click.command()
@click.argument('start_date')
@click.argument('end_date')
@click.option('-df', '--date-frequency', default=1, help='Date frequency between start_date and end_date',)
@click.option('-m', '--members', default=['c00',], help='Gridded fields to download.',
              multiple=True)
@click.option('-v', '--var-names', default=['cape_sfc', 'cin_sfc', 'hlcy_hgt'], help='Gridded fields to download.',
              multiple=True)
@click.option('-p', '--pressure-levels', default=[950.0, 500.0], multiple=True,
              help="Pressure levels to use, if some pressure field is used.")
@click.option('--latitude-bounds', nargs=2, type=click.Tuple([float, float]), default=(22, 55),
              help='Bounds for latitude range to keep when processing data.',)
@click.option('--longitude-bounds', nargs=2, type=click.Tuple([float, float]), default=(230, 291),
              help='Bounds for longitude range to keep when processing data, assumes values between 0-360.',)
@click.option('--forecast-days-bounds', nargs=2, type=click.Tuple([float, float]), default=(5.5, 6.5),
              help='Bounds for forecast days, where something like 5.5 would be 5 days 12 hours.',)
@click.option('--local-save-dir', default='./reforecast_v3', help='Location to save processed data.',)     
@click.option('--final-save-path', default='./combined_reforecast_data.nc', help='Saved name of the combined netCDF file.',)    
@click.option('--n-jobs', default=28, help='Number of jobs to run in parallel.',)
def get_and_process_reforecast_data(start_date, end_date, date_frequency, members, var_names, pressure_levels,
                                    latitude_bounds, longitude_bounds, forecast_days_bounds,
                                    local_save_dir, final_save_path, n_jobs):
    # let's do some quick checks here...
    if not all([min(latitude_bounds) > -90, max(latitude_bounds) < 90]):
        raise ValueError(f'Latitude bounds need to be within -90 and 90, got: {latitude_bounds}')
    if not all([min(longitude_bounds) >= 0, max(longitude_bounds) < 360]):
        raise ValueError(f'Longitude bounds must be positive and between 0-360 got: {longitude_bounds}')
    if not all([min(forecast_days_bounds) >= 0, max(forecast_days_bounds) <= 16]):
        raise ValueError(f'Forecast hour bounds must be between 0-16 days, got: {forecast_days_bounds}')
    if max(forecast_days_bounds) < 10:
        days = DAYS_PREFIX['1-10']
    else:
        days = DAYS_PREFIX['10-16']
    # now, let's make sure the local save directory exists.
    pathlib.Path(local_save_dir).mkdir(parents=True, exist_ok=True)
    globbed_list = [f'{S3_BUCKET}/{BASE_S3_PREFIX}/{dt.strftime("%Y/%Y%m%d00")}/{member}/{days}/{var_name}_{dt.strftime("%Y%m%d00")}_{member}.grib2'
                    for dt in pd.date_range(start_date, end_date, freq=f'{date_frequency}D')
                    for var_name in var_names for member in members]
    logger.info(f"Number of files to be downloaded and processed: {len(globbed_list)}")
    # TODO: Should this be async and not parallelized?
    _ = Parallel(n_jobs=n_jobs, verbose=25)(delayed(download_and_process_grib)(f, latitude_bounds, longitude_bounds, forecast_days_bounds, local_save_dir, pressure_levels) for f in globbed_list)

    ds = xr.open_mfdataset(os.path.join(local_save_dir, '*.nc'), combine='by_coords')
    ds.to_netcdf(final_save_path, compute=True)
    logger.info('All done!')
```

Writing the bash script and getting it to work as expected, along with running it on my suddenly struggling MacBook (the one with the black shell!) took about 2-3 months (and probably more time off of my life due to the frustration). I would have used more compute power, but my grad department didn't have a cluster or anything, and I didn't know AWS was a thing (was it a thing back then?). In fact, it even got to the point where Tom suggested I send a cleaned/wiped hard drive that he would load all the data up on and just mail it back (assuming, of course, all of this would be ok with the government's security restrictions).

Meanwhile, in 2020, I simply spun up a VM instance on Google Cloud with a bunch of cores and let it run. It took about 3-4 hours to run, which amounted to something like $10-15. Combined with the 30-60 minutes it took to write the script, we're basically looking at 5 hours of time here.

Up next, exploratory data analysis and feature engineering!