---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.12
    jupytext_version: 1.8.0
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---

# Latitude-dependent grey radiation

+++

Here is a quick example of using the `climlab.GreyRadiationModel` with a latitude dimension and seasonally varying insolation.

```{code-cell} ipython3
from __future__ import division, print_function
%matplotlib inline
import numpy as np
import matplotlib.pyplot as plt
import climlab
```

```{code-cell} ipython3
model = climlab.GreyRadiationModel(num_lev=30, num_lat=90)
print(model)
```

```{code-cell} ipython3
print(model.lat)
```

```{code-cell} ipython3
insolation = climlab.radiation.DailyInsolation(domains=model.Ts.domain)
```

```{code-cell} ipython3
model.add_subprocess('insolation', insolation)
model.subprocess.SW.flux_from_space = insolation.insolation
```

```{code-cell} ipython3
print(model)
```

```{code-cell} ipython3
model.compute_diagnostics()
```

```{code-cell} ipython3
plt.plot(model.lat, model.SW_down_TOA)
```

```{code-cell} ipython3
model.Tatm.shape
```

```{code-cell} ipython3
model.integrate_years(1)
```

```{code-cell} ipython3
plt.plot(model.lat, model.Ts)
```

```{code-cell} ipython3
model.integrate_years(1)
```

```{code-cell} ipython3
plt.plot(model.lat, model.timeave['Ts'])
```

```{code-cell} ipython3
def plot_temp_section(model, timeave=True):
    fig = plt.figure()
    ax = fig.add_subplot(111)
    if timeave:
        field = model.timeave['Tatm'].transpose()
    else:
        field = model.Tatm.transpose()
    cax = ax.contourf(model.lat, model.lev, field)
    ax.invert_yaxis()
    ax.set_xlim(-90,90)
    ax.set_xticks([-90, -60, -30, 0, 30, 60, 90])
    fig.colorbar(cax)
```

```{code-cell} ipython3
plot_temp_section(model)
```

```{code-cell} ipython3
model2 = climlab.RadiativeConvectiveModel(num_lev=30, num_lat=90)
insolation = climlab.radiation.DailyInsolation(domains=model2.Ts.domain)
model2.add_subprocess('insolation', insolation)
model2.subprocess.SW.flux_from_space = insolation.insolation
```

```{code-cell} ipython3
model2.integrate_years(1)
```

```{code-cell} ipython3
model2.integrate_years(1)
```

```{code-cell} ipython3
plot_temp_section(model2)
```

## Testing out multi-dimensional Band Models

```{code-cell} ipython3
#  Put in some ozone
import netCDF4 as nc

datapath = "http://ramadda.atmos.albany.edu:8080/repository/opendap/latest/Top/Users/BrianRose/CESM_runs/"
endstr = "/entry.das"

ozone = nc.Dataset( datapath + 'som_input/ozone_1.9x2.5_L26_2000clim_c091112.nc' + endstr )

#  Dimensions of the ozone file
lat = ozone.variables['lat'][:]
lon = ozone.variables['lon'][:]
lev = ozone.variables['lev'][:]

# Taking annual, zonal average of the ozone data
O3_zon = np.mean( ozone.variables['O3'],axis=(0,3) )
```

```{code-cell} ipython3
#  make a model on the same grid as the ozone
model3 = climlab.BandRCModel(lev=lev, lat=lat)
insolation = climlab.radiation.DailyInsolation(domains=model3.Ts.domain)
model3.add_subprocess('insolation', insolation)
model3.subprocess.SW.flux_from_space = insolation.insolation
print(model3)
```

```{code-cell} ipython3
#  Set the ozone mixing ratio
O3_trans = np.transpose(O3_zon)
#  model and ozone data are on the same grid, after the transpose.
print(O3_trans.shape)
print(lev)
print(model3.lev)
```

```{code-cell} ipython3
# Put in the ozone
model3.absorber_vmr['O3'] = O3_trans
```

```{code-cell} ipython3
print(model3.absorber_vmr['O3'].shape)
print(model3.Tatm.shape)
```

```{code-cell} ipython3
model3.step_forward()
```

```{code-cell} ipython3
model3.integrate_years(1.)
```

```{code-cell} ipython3
model3.integrate_years(1.)
```

```{code-cell} ipython3
#plt.contour(model3.lat, model3.lev, model3.Tatm.transpose())
plot_temp_section(model3)
```

This is now working. Will need to do some model tuning.

And start to add dynamics!

+++

## Adding meridional diffusion!

```{code-cell} ipython3
print(model2)
```

```{code-cell} ipython3
diffmodel = climlab.process_like(model2)
```

```{code-cell} ipython3
# thermal diffusivity in W/m**2/degC
D = 0.05
# meridional diffusivity in 1/s
K = D / diffmodel.Tatm.domain.heat_capacity[0]
print(K)
```

```{code-cell} ipython3
d = climlab.dynamics.MeridionalDiffusion(K=K, state={'Tatm': diffmodel.Tatm}, **diffmodel.param)
```

```{code-cell} ipython3
diffmodel.add_subprocess('diffusion', d)
```

```{code-cell} ipython3
print(diffmodel)
```

```{code-cell} ipython3
diffmodel.step_forward()
```

```{code-cell} ipython3
diffmodel.integrate_years(1)
```

```{code-cell} ipython3
diffmodel.integrate_years(1)
```

```{code-cell} ipython3
plot_temp_section(model2)
plot_temp_section(diffmodel)
```

This works as long as K is a constant.

The diffusion operation is broadcast over all vertical levels without any special code.

```{code-cell} ipython3
def inferred_heat_transport( energy_in, lat_deg ):
    '''Returns the inferred heat transport (in PW) by integrating the net energy imbalance from pole to pole.'''
    from scipy import integrate
    from climlab import constants as const
    lat_rad = np.deg2rad( lat_deg )
    return ( 1E-15 * 2 * np.math.pi * const.a**2 * integrate.cumtrapz( np.cos(lat_rad)*energy_in,
            x=lat_rad, initial=0. ) )
```

```{code-cell} ipython3
#  Plot the northward heat transport in this model
Rtoa = np.squeeze(diffmodel.timeave['ASR'] - diffmodel.timeave['OLR'])
plt.plot(diffmodel.lat, inferred_heat_transport(Rtoa, diffmodel.lat))
```

### Band model with diffusion

```{code-cell} ipython3
diffband = climlab.process_like(model3)
```

```{code-cell} ipython3
# thermal diffusivity in W/m**2/degC
D = 0.05
# meridional diffusivity in 1/s
K = D / diffband.Tatm.domain.heat_capacity[0]
print(K)
```

```{code-cell} ipython3
d = climlab.dynamics.MeridionalDiffusion(K=K, state={'Tatm': diffband.Tatm}, **diffband.param)
diffband.add_subprocess('diffusion', d)
print(diffband)
```

```{code-cell} ipython3
diffband.integrate_years(1)
```

```{code-cell} ipython3
diffband.integrate_years(1)
```

```{code-cell} ipython3
plot_temp_section(model3)
plot_temp_section(diffband)
```

```{code-cell} ipython3
plt.plot(diffband.lat, diffband.timeave['ASR'] - diffband.timeave['OLR'])
```

```{code-cell} ipython3
#  Plot the northward heat transport in this model
Rtoa = np.squeeze(diffband.timeave['ASR'] - diffband.timeave['OLR'])
plt.plot(diffband.lat, inferred_heat_transport(Rtoa, diffband.lat))
```

```{code-cell} ipython3

```