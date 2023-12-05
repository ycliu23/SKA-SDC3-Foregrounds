# Camrbidge SKA Science Data Challenge 3 (SKA SDC3 Cantabrigians) - Foregrounds
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.10223261.svg)](https://doi.org/10.5281/zenodo.10223261)

<img src="https://uc9e62fbdf8f7783342b43806158.previews.dropboxusercontent.com/p/thumb/ACFT81THAHQgC3tmtABXCEcjCq-zQXPIetE0LWVjHhJxHJhGFLwr4DKHhF5TTG7pNXAbQTnKv42bWuZsmcnwzC147D6Ezzt0ixyZ55b39_8ifDcfgwtucK6Amws1q94uwDf-uTSA_p3WQftzDm6H43M85246IyH6ezHiI7rmRq6wmKI1_UPw6mtRnRRkIR2Zi6YPokxEmuoTkMx06mpPbZ8CIKoVILsHTtIz8br67iTA58HRR8DD5FfLj_ZQC36NI_wwWL-FKcZZxN88YErKmmqvKdGfkuVUS512018brpnhAF4wjm40gR0u8vUJbwYU1dt-hgnKozhzsMFOtg3O47e73BO2Bk_PCiACN2GsR5HUKzvC_aw1anOuhL9fk3XUFwU/p.png">

## Pipeline flowchart
<p align="center">
<img src="illustration/ska_sdc3_pipeline.png" width=80% height=80%>
</p>

The data analysis pipeline, designed for the Square Kilometre Array Science Data Challenge 3a (SKA SDC3a), is intended to separate the 21-cm signal from astrophysical foregrounds. 

The foreground mitigation is based on a hybrid strategy based on both foreground removal with Gaussian Process Regression (GPR) models and foreground avoidance. Nested sampling is incorporated in the pipeline for Bayesian inference of the model parameters. 

The flowchart illustrates the SKA data processing and can be found in the `illustration` directory, and scripts are provided in `pipeline` for realization of the steps implemented in the workflow.

## Licence
This SKA SDC3 pipeline is free to use for academic purposes. However, users are required to acknowledge and agree to comply with the terms and conditions under [the MIT  License](LICENSE) by using, modifying and distributing this repository.

## Citation
```
@misc{SKAO_Science_Data_Challenge_3a,
author = {Liu, Y., O’Hara, O. S. D., de Lera Acedo, E., Coles, J., Dulwich, F., Leeney, S. A. K., Gessey-Jones, T., Anstey, D.},
title = {SKAO SDC3a Cantabrigians},
month = {nov},
year = {2023},
publisher = {Github},
version = {1.0.0},
doi = {10.5281/zenodo.10223261},
howpublished = {\url{https://github.com/ycliu23/Cambridge-SKA-SDC3-Foregrounds}}
}
```

## Installation

### OSKAR installation
The `OSKAR` simulator is available from [this release](https://github.com/OxfordSKA/OSKAR) and instructions for the installation are described in the [documentation](https://ska-telescope.gitlab.io/sim/oskar/install/install.html). 

For usage on HPC clusters, we recommend using the containerized image available from [here](https://github.com/OxfordSKA/OSKAR/releases), or alternatively built from `docker` or `singularity`.

### Pipeline environment
The latest version of the pipeline can be obtained by running:
```
git clone https://github.com/ycliu23/Cambridge-SKA-SDC3-Foregrounds.git
```
The pipeline utilizes the dependencies specified in `environment.yaml`. The `conda` environment configuration file can be used to set up a virtual environment with which to run the pipeline scripts.

To set up the environment for running the pipeline:
```
conda env create --file environment.yaml
```
This will create a `conda` environment named `sdc3a` and to activate the created environment:
```
conda activate sdc3a
```

## OSKAR simulation
We start the foreground mitigation with the removal of bright point sources in the SDC3 images. 

The total sky intensity is desourced by first performing mock SKA1-Low observations with `OSKAR` ([Dulwich et al. 2009](https://pos.sissa.it/132/031/pdf)) of the discrete sources for the entire sky.

The simulation pipeline can be found [here](https://github.com/ycliu23/SKA_Power_Spectrum_and_EoR_Window), 
which is based on the [SKA End-to-End pipeline](https://github.com/oharao/SKA_Power_Spectrum_and_EoR_Window) with a modified sky model. 

The settings provided to `OSKAR`:
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
The modified sky model is composed of an outer sky that covers the $\mathrm{2\pi}$ steradians above the horizon and an inner sky model defined within the first null of the station beam pattern (~11.4&deg;) at 106 MHz. 

The point source model, the composite GLEAM and LoBES source catalogue, is kindly provided by the SKA Observatory in the SDC3 and can be found [here](https://drive.google.com/file/d/14nfYmwlyqL7NzMqWtMxYfaFBccrjxKll/view?usp=drive_link). 

The mock observations cover 4 hours to track the point sources in the catalogue and consist of 1440 time steps, each integrating over 10 seconds. The simulation also spans the same frequency range as in the SDC3 from 106 MHz to 196 MHz, with intervals of 0.1 MHz.

## Imaging
The imaging process utilizes `WSCLEAN` ([Offringa et al., 2014](https://arxiv.org/pdf/1407.1943.pdf)) to grid and inverse Fourier transform the visibilities generated by `OSKAR` to images by accounting for the sky curvature (ie. w-terms) and synthesized beam response (ie. point spread functions).
```
wsclean -reorder -use-wgridder -parallel-gridding 10 -weight natural -oversampling 4095 -kernel-size 15 -nwlayers 1000 -grid-mode kb -taper-edge 100 -padding 2 -name OUTFILE -size 256 256 -scale 128asec -niter 0 -pol xx -make-psf INFILE
```
The desourced images are obtained by subtracting the image cube of GLEAM and LoBES sources from the SDC3 image cube. As PSF deconvolution needs to be performed in the Fourier space, the images are then transformed into gridded visibilities using the Python package [ps-eor](https://gitlab.com/flomertens/ps_eor):
```
python cal_vis.py
```

## Foreground removal
This step involves Gaussian Process Regression (GPR) within a Bayesian framework. The GPR model is capable of distinguishing different components of the observed signal by pre-defining analytic forms for their covariance.

The model evidence and parameter optimization are computed with the nested sampler [PolyChord](https://github.com/PolyChord/PolyChordLite/tree/master) ([Handley et al. 2015a](https://arxiv.org/abs/1502.01856), [2015b](https://arxiv.org/abs/1506.00171)).

```
python ns_gpr.py
```
After sampling the hyperparameters for the GPR model,
```
python posterior_gpr_clean.py
```
The post-processing of the MCMC sampling chain results is based on [anesthetic](https://github.com/handley-lab/anesthetic) that can be used to summarize and plot the posterior density distribution of each hyperparameter in the GPR model:
```
python posterior_plot.py
```

## Power spectrum analysis
Once completing the GPR foreground cleaning, the residual gridded visibilities are Fourier transformed along the frequency axis and cylindrically averaged into the 2D power spectra in each $k_\parallel$ and $k_\perp$ bin, where $k_\perp^2 = \sqrt{k_x^2 + k_y^2}$ and $k_\parallel = k_z$:
```
python cal_ps.py
```
In this final step of power spectrum estimation, Fourier modes, dominated by the intrinsic and instrumentally corrupted foregrounds, are also masked in the resultant cylindrical power spectra based on a foreground avoidance strategy, separated from a region that represents the EoR window.
