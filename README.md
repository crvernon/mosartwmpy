![build](https://github.com/IMMM-SFA/mosartwmpy/workflows/build/badge.svg) [![codecov](https://codecov.io/gh/IMMM-SFA/mosartwmpy/branch/main/graph/badge.svg?token=IPOY8984MB)](https://codecov.io/gh/IMMM-SFA/mosartwmpy)

## mosartwmpy

`mosartwmpy` is a python translation of MOSART-WM, a model for water routing and reservoir management written in Fortran. The original code can be found at [IWMM](https://github.com/IMMM-SFA/iwmm) and [E3SM](https://github.com/E3SM-Project/E3SM), in which MOSART is the river routing component of a larger suite of earth-science models. The motivation for rewriting is largely for developer convenience -- running, debugging, and adding new capabilities were becoming increasingly difficult due to the complexity of the codebase and lack of familiarity with Fortran. This version aims to be intuitive, lightweight, and well documented, while still being highly interoperable.

## getting started

Install `mosartwmpy` with:
```shell
pip install mosartwmpy
```

Download a sample input dataset spanning 1980-1985 by running the following and selecting option `1`. This will download and unpack the inputs to your current directory. Note that this data is about 1.5GB in size.

```shell
python -m mosartwmpy.download
```

Settings are defined by the merger of the `mosartwmpy/config_defaults.yaml` and a user specified file which can override any of the default settings. Create a `config.yaml` file that defines your simulation:

> `config.yaml`
> ```yaml
> simulation:
>   name: tutorial
>   start_date: 1981-05-24
>   end_date: 1981-05-26
> 
> grid:
>   path: ./input/domains/MOSART_NLDAS_8th_20160426.nc
>   land:
>     path: ./input/domains/domain.lnd.nldas2_0224x0464_c110415.nc
> 
> runoff:
>   read_from_file: true
>   path: ./input/runoff/Livneh_NLDAS_1980_1985.nc
> 
> water_management:
>   enabled: true
>   demand:
>     read_from_file: true
>     path: ./input/demand/RCP8.5_GCAM_water_demand_1980_1985.nc
>   reservoirs:
>     path: ./input/reservoirs/US_reservoir_8th_NLDAS3_updated_20200421.nc
> ```

`mosartwmpy` implements the [Basic Model Interface](https://csdms.colorado.edu/wiki/BMI) defined by the CSDMS, so driving it should be familiar to those accustomed to the BMI. To launch the simulation, open a python shell and run the following:

```python
from mosartwmpy import Model

# path to the configuration yaml file
config_file = "config.yaml"

# initialize the model
mosart_wm = Model()
mosart_wm.initialize(config_file)

# advance the model one timestep
mosart_wm.update()

# advance until the `simulation.end_date` specified in config.yaml
mosart_wm.update_until(mosart_wm.get_end_time())
```

Alternatively, one can update the settings via code in the driving script using dot notation:

```python
from mosartwmpy import Model
from datetime import datetime

mosart_wm = Model()
mosart_wm.initialize()
 
mosart_wm.config['simulation.name'] = 'Tutorial'
mosart_wm.config['simulation.start_date'] = datetime(1981, 5, 24)
mosart_wm.config['simulation.end_date'] = datetime(1985, 5, 26)
# etc...
```

One can use the usual python plotting libraries to visualize data. Model state and output are stored as one-dimensional numpy ndarrays, so they must be reshaped to visualize two-dimensionally:

```python
import xarray as xr
import matplotlib.pyplot  as plt
from mosartwmpy import Model

mosart_wm = Model()
mosart_wm.initialize('./config.yaml')

mosart_wm.update_until(mosart_wm.get_end_time())

surface_water = mosart_wm.get_value_ptr('surface_water_amount')

# create an xarray from the data, which has some convenience wrappers for matplotlib methods
data_array = xr.DataArray(
    surface_water.reshape(mosart_wm.get_grid_shape()),
    dims=['latitude', 'longitude'],
    coords={'latitude': mosart_wm.get_grid_x(), 'longitude': mosart_wm.get_grid_y()},
    name='Surface Water Amount',
    attrs={'units': mosart_wm.get_var_units('surface_water_amount')}
)

# plot as a pcolormesh
data_array.plot(robust=True, levels=32, cmap='winter_r')

plt.show()

```

## model input

Several input files in NetCDF format are required to successfully run a simulation, which are not shipped with this repository due to their large size. The grid files, reservoir files, and a small range of runoff and demand input files can be obtained using the download utility by running `python -m mosartwmpy.download` and choosing option 1 for "sample_input". Currently, all input files are assumed to be at the same resolution (for the sample files this is 1/8 degree over the CONUS). Below is a summary of the various input files:

<table>
<thead>
<tr>
<th>
    Name
</th>
<th>
    Description
</th>
<th>
    Configuration Path
</th>
<th>
    Notes
</th>
</tr>
</thead>
<tbody>
<tr>
<td>
    Grid
</td>
<td>
    Spatial constants dimensioned by latitude and longitude relating to the physical properties of the river channels 
</td>
<td>
    <code>grid.path</code>
</td>
<td/>
</tr>
<tr>
<td>
    Land Fraction
</td>
<td>
    Fraction of grid cell that is land (as opposed to i.e. ocean water) dimensioned by latitude and longitude 
</td>
<td>
    <code>grid.land.path</code>
</td>
<td>
    As a TODO item, this variable should be merged into the grid file (historically it was separate for the coupled land model)
</td>
</tr>
<tr>
<td>
    Reservoirs
</td>
<td>
    Locations of reservoirs (possibly aggregated) and their physical and political properties
</td>
<td>
    <code>water_management.reservoirs.path</code>
</td>
<td/>
</tr>
<tr>
<td>
    Runoff
</td>
<td>
    Surface runoff, subsurface runoff, and wetland runoff per grid cell averaged per unit of time; used to drive the river routing
</td>
<td>
    <code>runoff.path</code>
</td>
<td/>
</tr>
<tr>
<td>
    Demand
</td>
<td>
    Water demand of grid cells averaged per unit of time; currently assumed to be monthly
</td>
<td>
    <code>water_management.reservoirs.demand</code>
</td>
<td>
    There are plans to support other time scales, such as epiweeks
</td>
</tr>
</tbody>
</table>

Alternatively, certain model inputs can be set using the BMI interface. This can be useful for coupling `mosartwmpy` with other models. If setting an input that would typically be read from a file, be sure to disable the `read_from_file` configuration value for that input. For example:
```python
import numpy as np
from mosartwmpy import Model

mosart_wm = Model()
mosart_wm.initialize()

# get a list of model input variables
mosart_wm.get_input_var_names()

# disable the runoff read_from_file
mosart_wm.config['runoff.read_from_file'] = False

# set the runoff values manually (i.e. from another model's output)
surface_runoff = np.empty(mosart_wm.get_grid_size())
surface_runoff[:] = # <values from coupled model>
mosart_wm.set_value('surface_runoff_flux', surface_runoff)

# advance one timestep
mosart_wm.update()

# continue coupling...
```

## model output

By default, key model variables are output on a monthly basis at a daily averaged resolution to `./output/<simulation name>/<simulation name>_<year>_<month>.nc`. See the configuration file for examples of how to modify the outputs, and the `./mosartwmpy/state/state.py` file for state variable names.

Alternatively, certain model outputs deemed most important can be accessed using the BMI interface methods. For example:
```python
import numpy as np
from mosartwmpy import Model

mosart_wm = Model()
mosart_wm.initialize()

# get a list of model output variables
mosart_wm.get_output_var_names()

# get the flattened numpy.ndarray of values for an output variable
supply = mosart_wm.get_value_ptr('supply_water_amount')
```

## testing and validation

Before running the tests or validation, make sure to download the "sample_input" and "validation" datasets using the download utility `python -m mosartwmpy.download`.

To execute the tests, run `./test.sh` or `python -m unittest discover mosartwmpy/tests` from the repository root.

To execute the validation, run a model simulation that includes the years 1981 - 1982, note your output directory, and then run `python -m mosartwmpy.validate` from the repository root. This will ask you for the simulation output directory, think for a moment, and then open a figure with several plots representing the NMAE (Normalized Mean Absolute Error) as a percentage and the spatial sums of several key variables compared between your simulation and the validation scenario. Use these plots to assist you in determining if the changes you have made to the code have caused unintended deviation from the validation scenario. The NMAE should be 0% across time if you have caused no deviations. A non-zero NMAE indicates numerical difference between your simulation and the validation scenario. This might be caused by changes you have made to the code, or alternatively by running a simulation with different configuration or parameters (i.e. larger timestep, fewer iterations, etc). The plots of the spatial sums can assist you in determining what changed and the overall magnitude of the changes.

If you wish to merge code changes that intentionally cause significant deviation from the validation scenario, please work with the maintainers to create a new validation dataset.
