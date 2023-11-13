# Camrbidge SKA Science Data Challenge - Foregrounds
## OSKAR Simulation
The SKA End-to-End simulation pipeline used in the SKA SDC3 can be found [here](https://github.com/ycliu23/SKA_Power_Spectrum_and_EoR_Window), 
which is based on [this version](https://github.com/oharao/SKA_Power_Spectrum_and_EoR_Window) with a modified sky model. The full sky is composed of an outer sky that covers the $2\pi$ steradians above the horizon and an inner sky model defined within the first null of the station beam pattern at 106 MHz. A detailed description can be found in [SDC3 data description](https://docs.google.com/document/d/1UZCsztjZDlbGbz3uqvEbPiIRXkct3rJDVtR_EiVM_so/edit). We use the GLEAM and LoBES catalogue provided by the SKAO to simulate a 4-hour track observation of point sources that consists of 1440 time steps, each integrating over 10 seconds. The observation also covers the same frequency range as in the SDC3 from 106 MHz to 196 MHz, with a interval of 0.1 MHz. The simulation is for the point source subtraction from the images of 

## Imaging
The imaging process utilizes WSCLEAN (Offringa et al., 2014) to grid and inverse Fourier transform the visibilities generated by $OSKAR$ to images by accounting for the sky curvature (ie. w-terms). 

"wsclean -reorder -use-wgridder -parallel-gridding 10 -weight natural -oversampling 4095 -kernel-size 15 -nwlayers 1000 -grid-mode kb -taper-edge 100 -padding 2 -name outfile -size 256 256 -scale 128asec -niter 0 -pol xx -make-psf infile" where infile is the input MeasurementSet files that contain visibility data and outfile is the output image file WSCLEAN will generate.

## Foreground Removal
## MCMC Sampling
## Power Spectrum Analysis
