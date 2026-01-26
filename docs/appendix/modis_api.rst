MODIS API Reference
===================

This document provides the API reference for MODIS-related functions and classes
in MELODIES MONET.

Core Classes
------------

analysis
^^^^^^^^

The main ``driver.analysis`` class coordinates MODIS processing.

.. py:class:: melodies_monet.driver.analysis

   Top-level class for managing MODIS observations, models, and pairing.

   .. py:attribute:: obs
      :type: dict

      Dictionary of observation objects, keyed by observation label.

   .. py:attribute:: obs_grid
      :type: dict

      Uniform grid coordinates for gridding swath data.
      Contains ``'longitude'``, ``'latitude'``, and ``'time'`` arrays.

   .. py:attribute:: obs_edges
      :type: dict

      Grid cell edges for binning observations.
      Contains ``'lon_edges'``, ``'lat_edges'``, and ``'time_edges'`` arrays.

   .. py:attribute:: obs_gridded_data
      :type: dict

      Accumulated observation values per grid cell. Keyed by ``'{obs}_{variable}'``.

   .. py:attribute:: obs_gridded_count
      :type: dict

      Observation counts per grid cell. Keyed by ``'{obs}_{variable}'``.

   .. py:attribute:: obs_gridded_dataset
      :type: xarray.Dataset

      Final gridded dataset after normalization.

   .. py:method:: setup_obs_grid()

      Create a uniform observation grid based on ``obs_grid`` configuration.

      Initializes:
        - ``self.obs_grid`` - Grid center coordinates
        - ``self.obs_edges`` - Grid cell edges
        - ``self.obs_gridded_data`` - Zero arrays for data accumulation
        - ``self.obs_gridded_count`` - Zero arrays for count accumulation

      **Configuration (from YAML):**

      .. code-block:: yaml

         obs_grid:
           start_time: '2020-09-09'
           end_time: '2020-09-12'
           ntime: 72        # Number of time bins
           nlat: 180        # Number of latitude bins
           nlon: 360        # Number of longitude bins

   .. py:method:: open_obs(time_interval=None)

      Open observation datasets for the specified time interval.

      For MODIS (``sat_type: 'modis_l2'``), this:
        1. Subsets files to the time interval using ``subset_MODIS_l2()``
        2. Reads HDF files using the MONETIO reader
        3. Returns an ``OrderedDict`` of Datasets keyed by datetime strings

      :param time_interval: ``[start, end]`` pandas Timestamps. If None, reads all files.
      :type time_interval: list of pandas.Timestamp, optional

   .. py:method:: update_obs_gridded_data()

      Accumulate swath observations onto the uniform grid.

      For each observation dataset and time:
        - Extracts lat/lon coordinates and data values
        - Bins observations into grid cells
        - Accumulates sums and counts

      Uses Numba-accelerated ``update_data_grid()`` for performance.

   .. py:method:: normalize_obs_gridded_data()

      Normalize accumulated grid data by dividing by counts.

      Creates ``self.obs_gridded_dataset`` containing:
        - ``{obs}_{variable}_data`` - Mean values per grid cell
        - ``{obs}_{variable}_count`` - Observation counts per grid cell

      Grid cells with zero observations are set to NaN.

observation
^^^^^^^^^^^

.. py:class:: melodies_monet.driver.observation

   Represents a single observation dataset (e.g., Terra_MODIS).

   .. py:attribute:: label
      :type: str

      Observation identifier from YAML configuration.

   .. py:attribute:: obs_type
      :type: str

      Observation type. For MODIS swath data: ``'sat_swath_clm'`` or ``'sat_swath_sfc'``.

   .. py:attribute:: sat_type
      :type: str

      Satellite type identifier. For MODIS: ``'modis_l2'``.

   .. py:attribute:: file
      :type: str

      File path pattern with glob wildcards.

   .. py:attribute:: variable_dict
      :type: dict

      Variable configuration including min/max bounds, scale factors, and units.

   .. py:attribute:: obj
      :type: OrderedDict

      Loaded observation data. For MODIS, an ``OrderedDict`` of xarray Datasets
      keyed by datetime strings (format: ``'YYYYDDDHHMM'``).

   .. py:method:: open_sat_obs(time_interval=None, control_dict=None)

      Open satellite observation data.

      For MODIS L2 (``sat_type == 'modis_l2'``):
        1. Calls ``subset_MODIS_l2()`` to filter files by time
        2. Calls ``monetio.sat._modis_l2_mm.read_mfdataset()`` to load data
        3. Returns ``OrderedDict`` with variables: Latitude, Longitude,
           Scan_Start_Time, and requested parameters

      :param time_interval: ``[start, end]`` time bounds
      :type time_interval: list of pandas.Timestamp, optional
      :param control_dict: Full control dictionary from YAML
      :type control_dict: dict, optional

   .. py:method:: filter_obs()

      Filter observations based on ``filter_dict`` configuration.

      Supported operators:
        - ``'=='``, ``'!='``, ``'>'``, ``'<'``, ``'>='``, ``'<='``
        - ``'isin'``, ``'isnotin'`` - For list membership

   .. py:method:: mask_and_scale()

      Apply masking and unit conversions to observation data.

      Processes each variable according to ``variable_dict``:
        - ``obs_min`` - Set values below threshold to NaN
        - ``obs_max`` - Set values above threshold to NaN
        - ``nan_value`` - Set specific value to NaN
        - ``unit_scale`` / ``unit_scale_method`` - Apply unit conversion

Time Interval Subsetting
------------------------

subset_MODIS_l2
^^^^^^^^^^^^^^^

.. py:function:: melodies_monet.util.time_interval_subset.subset_MODIS_l2(file_path, timeinterval)

   Subset MODIS L2 files to a specified time interval.

   Filters files based on the timestamp encoded in MODIS filenames.

   **Filename Convention:**

   ::

       MOD04_L2.AYYYYDDD.HHMM.collection.timestamp.hdf
       MYD04_L2.AYYYYDDD.HHMM.collection.timestamp.hdf

   Where:
     - ``YYYY`` = 4-digit year
     - ``DDD`` = 3-digit day of year
     - ``HH`` = 2-digit hour (UTC)
     - ``MM`` = 2-digit minute

   :param file_path: Glob pattern for MODIS HDF files (e.g., ``'/data/MODIS/MOD04_L2.*.hdf'``)
   :type file_path: str
   :param timeinterval: ``[start, end]`` time bounds
   :type timeinterval: list of pandas.Timestamp
   :return: List of file paths within the time interval
   :rtype: list of str

   **Example:**

   .. code-block:: python

       from melodies_monet.util import time_interval_subset as tsub
       import pandas as pd

       time_interval = [
           pd.Timestamp('2020-09-09'),
           pd.Timestamp('2020-09-10')
       ]
       files = tsub.subset_MODIS_l2(
           '/data/MODIS/Terra/C61/2020/*/MOD04_L2.*.hdf',
           time_interval
       )

   **Note:** The function uses hourly granularity for subsetting. Files are matched
   using the pattern ``'*M?D04_L2.A{YYYYDDD.HH}*.hdf'``.

Grid Utilities
--------------

generate_uniform_grid
^^^^^^^^^^^^^^^^^^^^^

.. py:function:: melodies_monet.util.grid_util.generate_uniform_grid(start, end, ntime, nlat, nlon)

   Generate a uniform latitude-longitude-time grid.

   :param start: Start time (ISO format string or datetime)
   :type start: str or datetime
   :param end: End time (ISO format string or datetime)
   :type end: str or datetime
   :param ntime: Number of time bins
   :type ntime: int
   :param nlat: Number of latitude bins
   :type nlat: int
   :param nlon: Number of longitude bins
   :type nlon: int
   :return: Tuple of (grid, edges)
   :rtype: tuple(dict, dict)

   **Returns:**

   - ``grid`` (dict): Grid center coordinates

     - ``'longitude'`` - 1D array of longitude centers
     - ``'latitude'`` - 1D array of latitude centers
     - ``'time'`` - 1D array of time centers (Unix timestamps)

   - ``edges`` (dict): Grid cell boundaries

     - ``'lon_edges'`` - Longitude bin edges (nlon + 1)
     - ``'lat_edges'`` - Latitude bin edges (nlat + 1)
     - ``'time_edges'`` - Time bin edges (ntime + 1)

   **Grid Specifications:**

   - Latitude range: -90 to 90 degrees
   - Longitude range: -180 to 180 degrees
   - Time range: Uniformly spaced between start and end

   **Example:**

   .. code-block:: python

       from melodies_monet.util import grid_util

       grid, edges = grid_util.generate_uniform_grid(
           '2020-09-09', '2020-09-12',
           ntime=72, nlat=180, nlon=360
       )
       # 1-degree resolution, 1-hour time steps

update_data_grid
^^^^^^^^^^^^^^^^

.. py:function:: melodies_monet.util.grid_util.update_data_grid(time_edges, x_edges, y_edges, time_obs, x_obs, y_obs, data_obs, count_grid, data_grid)

   Accumulate observation data onto a uniform grid.

   This function is Numba JIT-compiled for high performance.

   :param time_edges: Grid time bin edges (Unix timestamps)
   :type time_edges: numpy.ndarray
   :param x_edges: Grid longitude bin edges
   :type x_edges: numpy.ndarray
   :param y_edges: Grid latitude bin edges
   :type y_edges: numpy.ndarray
   :param time_obs: Observation timestamps
   :type time_obs: numpy.ndarray
   :param x_obs: Observation longitudes
   :type x_obs: numpy.ndarray
   :param y_obs: Observation latitudes
   :type y_obs: numpy.ndarray
   :param data_obs: Observation values
   :type data_obs: numpy.ndarray
   :param count_grid: Array to accumulate observation counts (modified in-place)
   :type count_grid: numpy.ndarray
   :param data_grid: Array to accumulate data sums (modified in-place)
   :type data_grid: numpy.ndarray

   **Notes:**

   - NaN values in ``data_obs`` are skipped
   - Observations outside grid bounds are clipped to edge cells
   - Arrays ``count_grid`` and ``data_grid`` are modified in-place

normalize_data_grid
^^^^^^^^^^^^^^^^^^^

.. py:function:: melodies_monet.util.grid_util.normalize_data_grid(count_grid, data_grid)

   Normalize accumulated grid data by dividing by counts.

   :param count_grid: Observation counts per grid cell
   :type count_grid: numpy.ndarray
   :param data_grid: Accumulated data sums per grid cell (modified in-place)
   :type data_grid: numpy.ndarray

   **Notes:**

   - Grid cells with zero counts are set to NaN
   - ``data_grid`` is modified in-place

MONETIO Reader
--------------

The actual HDF file reading is handled by the MONETIO library.

read_mfdataset
^^^^^^^^^^^^^^

.. py:function:: monetio.sat._modis_l2_mm.read_mfdataset(files, variable_dict, debug=False)

   Read multiple MODIS L2 HDF files into an OrderedDict of Datasets.

   :param files: List of HDF file paths
   :type files: list of str
   :param variable_dict: Variable configuration from YAML
   :type variable_dict: dict
   :param debug: Enable debug output
   :type debug: bool
   :return: Datasets keyed by datetime string (format: ``'YYYYDDDHHMM'``)
   :rtype: collections.OrderedDict

   **Output Dataset Variables:**

   - ``Latitude`` - 2D geolocation array
   - ``Longitude`` - 2D geolocation array
   - ``Scan_Start_Time`` - Observation time
   - Requested AOD variables (flattened with ``lat``/``lon`` coordinates)

   **Note:** This function is part of the MONETIO library and uses the
   ``_mm`` suffix indicating MELODIES MONET variant readers.

Usage Examples
--------------

Basic MODIS Processing
^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    from melodies_monet import driver

    # Initialize and configure
    an = driver.analysis()
    an.control = 'control_modis.yaml'
    an.read_control()

    # Setup grid
    an.setup_obs_grid()

    # Process time intervals
    for time_interval in an.time_intervals:
        an.open_obs(time_interval=time_interval)
        an.update_obs_gridded_data()

    # Finalize
    an.normalize_obs_gridded_data()
    an.obs_gridded_dataset.to_netcdf('modis_aod.nc')

Direct File Subsetting
^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    from melodies_monet.util import time_interval_subset as tsub
    import pandas as pd

    # Define time range
    start = pd.Timestamp('2020-09-09')
    end = pd.Timestamp('2020-09-10')

    # Get files in time range
    files = tsub.subset_MODIS_l2(
        '/data/MODIS/Terra/C61/2020/*/MOD04_L2.*.hdf',
        [start, end]
    )

    print(f'Found {len(files)} files')
    for f in files[:5]:
        print(f)

Custom Grid Generation
^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    from melodies_monet.util import grid_util

    # High-resolution regional grid
    grid, edges = grid_util.generate_uniform_grid(
        start='2020-09-09',
        end='2020-09-10',
        ntime=24,      # Hourly
        nlat=400,      # 0.45 degree
        nlon=800       # 0.45 degree
    )

    print(f"Longitude range: {grid['longitude'].min():.1f} to {grid['longitude'].max():.1f}")
    print(f"Latitude range: {grid['latitude'].min():.1f} to {grid['latitude'].max():.1f}")
    print(f"Time steps: {len(grid['time'])}")

See Also
--------

* :doc:`/users_guide/satellites/modis` - User guide for MODIS processing
* :doc:`/appendix/modis_yaml` - YAML configuration reference
* :doc:`/api` - Full API documentation
