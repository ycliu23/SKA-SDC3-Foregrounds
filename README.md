# Camrbidge SKA Science Data Challenge 3 (SKA SDC3) - Foregrounds
## OSKAR Simulation
The first step is to remove the point sources by performing a mock observation of SKA1-Low using `OSKAR` ([Dulwich et al. 2009](https://pos.sissa.it/132/031/pdf)) using the same settings as the SKA SDC3. The `OSKAR` simulator is available from [this website](https://github.com/OxfordSKA/OSKAR) and the SKA End-to-End simulation pipeline can be found [here](https://github.com/ycliu23/SKA_Power_Spectrum_and_EoR_Window), 
which is based on [this version](https://github.com/oharao/SKA_Power_Spectrum_and_EoR_Window) with a modified sky model. The full sky is composed of an outer sky that covers the $\mathrm{2\pi}$ steradians above the horizon and an inner sky model defined within the first null of the station beam pattern at 106 MHz. The point source model we used is the composite GLEAM and LoBES source catalogue provided by the SKAO and can be found [here](https://drive.google.com/file/d/14nfYmwlyqL7NzMqWtMxYfaFBccrjxKll/view?usp=drive_link). The mock observation covers 4 hours to track the point sources in the catalogue and consists of 1440 time steps, each integrating over 10 seconds. The simulation also spans the same frequency range as in the SDC3 from 106 MHz to 196 MHz, with intervals of 0.1 MHz.

The setting passed to `OSKAR`:
```
FILENAME = 'gleam_lobes'
MIN_FREQ = 106
MAX_FREQ = 196
FREQ_INTERVAL = 0.1

STATION = 'ska_one_station'
OBS_START_TIME = '2021-09-21 14:12:40.13'
OBS_LENGTH = '04:00:00'
OBS_NUM_TIMES = 1440
INT_TIME = 10

EOR = False
DIFFUSE = False
PS = True
NOISE = False
```

## Imaging
The imaging process utilizes `WSCLEAN` ([Offringa et al., 2014](https://arxiv.org/pdf/1407.1943.pdf)) to grid and inverse Fourier transform the visibilities generated by `OSKAR` to images by accounting for the sky curvature (ie. w-terms) and synthesized beam patterns (ie. point spread functions).
```
wsclean -reorder -use-wgridder -parallel-gridding 10 -weight natural -oversampling 4095 -kernel-size 15 -nwlayers 1000 -grid-mode kb -taper-edge 100 -padding 2 -name OUTFILE -size 256 256 -scale 128asec -niter 0 -pol xx -make-psf INFILE
```
The desourced images are obtained by subtracting the image cube of GLEAM and LoBES sources from the SDC3 image cube. As PSF deconvolution needs to be performed in the Fourier space, the images are transformed to gridded visibilities using the Python package `ps_eor` that can be obtained [here](https://gitlab.com/flomertens/ps_eor).

## Foreground Removal
This step requires Gaussian Process Regression in a Bayesian framework using nested sampling. The nested sampling is enable by [PolyChord](https://github.com/PolyChord/PolyChordLite/tree/master) ([Handley et al. 2015a](https://arxiv.org/abs/1502.01856), [2015b](https://arxiv.org/abs/1506.00171)).

```
python posterior_gpr_clean.py
```
After sampling the hyperparameters for the GPR model,
```
python posterior_gpr_clean.py
```
We also use `anesthetic` (available [here](https://github.com/handley-lab/anesthetic)) to post-process the MCMC sampling chain results and to obtain the posterior density distribution of each hyperparameter in the GPR model:
```
python posterior_plot.py
```

## Power Spectrum Analysis
Once completing the GPR foreground cleaning, the residual 2D power spectra can be calculated at each $k_\parallel$ and $k_\perp$ bin:

```
python cal_ps.py
```

