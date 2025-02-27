---
title: '`PACMAN`: A pipeline to reduce and analyze Hubble Wide Field Camera 3 IR Grism data'
tags:
  - HST
  - python
  - astronomy
  - exoplanets
  - spectroscopy
  - photometry
authors:

  - name: Sebastian Zieba #^[zieba@mpia] # note this makes a footnote saying 'co-first author'
    orcid: 0000-0003-0562-6750
    affiliation: "1, 2" # (Multiple affiliations must be quoted)
  - name: Laura Kreidberg
    orcid: 0000-0003-0514-1147
    affiliation: 1
affiliations:
 - name: Max-Planck-Institut für Astronomie, Königstuhl 17, D-69117 Heidelberg, Germany
   index: 1
 - name: Leiden Observatory, Leiden University, Niels Bohrweg 2, 2333CA Leiden, The Netherlands
   index: 2
date: xxx
bibliography: paper.bib

---

# Summary

The Hubble Space Telescope (HST) has become the preeminent workhorse facility for the characterization of extrasolar planets.
Launched in 1990 and never designed for the observations of exoplanets, the STIS spectrograph on HST was used in 2002 to detect the first atmosphere ever discovered on a planet outside of our solar system [@Charbonneau2002].

HST currently has two of the most powerful space-based tools for characterizing exoplanets over a broad spectral range:
The Space Telescope Imaging Spectrograph (STIS; installed in 1997) in the UV and the Wide Field Camera 3 (WFC3; installed in 2009) in the Near Infrared (NIR).
With the introduction of a spatial scan mode on WFC3 [@McCullough2012; @Deming2012] where the star moves perpendicular to the dispersion direction during an exposure, WFC3 observations have become very efficient due to the reduction of overhead time and the possibility of longer exposures without saturation.

For exoplanet characterization, WFC3 is used for transit and secondary eclipse spectroscopy, and phase curve observations.
The instrument has two different grisms: G102 with a spectral range from 800 nm to up to 1150 nm and G141 encompassing 1075 nm to about 1700 nm.
The spectral range of WFC3/G141 is primarily sensitive to molecular absorption from water at approximately 1.4 microns.
This led to the successful detection of water in the atmosphere of over a dozen of exoplanets [e.g., @Deming2013; @Huitson2013; @Fraine2014; @Kreidberg2014b; @Evans2016].
The bluer part of WFC3, the G102 grism, is also sensitive to water and most notably led to the first detection of a helium exosphere [@Spake2018].

Here we present `PACMAN`, an end-to-end pipeline developed to reduce and analyze HST/WFC3 data.
The pipeline includes both spectral extraction and light curve fitting.
The foundation of `PACMAN` has been already used in numerous publications [e.g., @Kreidberg2014a; @Kreidberg2018] and these papers have already accumulated hundreds of citations.


# Statement of need

Exoplanet spectroscopy with Hubble requires very precise measurements that are beyond the scope of standard analysis tools provided by the Space Telescope Science Institute. 
The data analysis is challenging, and different pipelines have produced discrepant results in the literature [e.g., @Kreidberg2019; @Teachey2018].
To facilitate reproducibility and transparency, the data reduction and analysis software should be open-source. 
This will enable easy comparison between different pipelines, and also lower the barrier to entry for newcomers in the exoplanet atmosphere field.

What sets `PACMAN` apart from other tools provided by the community, is that it was specifically designed to reduce and fit HST data.
There are several open-source tools that can fit time series observations of stars to model events like transiting exoplanets, such as `EXOFASTv2` [@Eastman2019], `juliet` [@Espinoza2019], `allesfitter` [@Gunther2019; @Gunther2021], `exoplanet` [@Foreman-Mackey2021a; @Foreman-Mackey2021b], and `starry` [@Luger2019]. 
`PACMAN`'s source code, however, includes fitting models that can model systematics which are characteristic to HST data, 
such as the orbit-long exponential ramps due to charge trapping or the upstream-downstream effect.
This removes the need for the user to write these functions themselves.
`PACMAN` will also retrieve information from the header of the FITS files, automatically detect HST orbits and visits and use this information in the fitting models.

The only other end-to-end open source pipeline specifically developed for the reduction and analysis of HST/WFC3 data is [`Iraclis`](https://github.com/ucl-exoplanets/Iraclis) [@Tsiaras2016].
Another open-source pipeline that has been for example used as an independent check of recent results presented in @Mugnai2021 and @Carone2021 is [`CASCADe`](https://jbouwman.gitlab.io/CASCADe/) (Calibration of trAnsit Spectroscopy using CAusal Data).
For a more detailed discussion of `CASCADe` see Appendix 1 in @Carone2021.


# Outline of the pipeline steps

The pipeline starts with the _ima_ data products provided by the Space Telescope Science Institute that can be easily accessed from [MAST](https://mast.stsci.edu/search/hst).
These files created by the WFC3 calibration pipeline, `calwf3`, have already several calibrations applied (dark subtraction, linearity correction, flat-fielding) to each readout of the IR exposure.

In the following we highlight several steps in the reduction and fitting stages of the code which are typical for HST/WFC3 observations:

- **Wavelength calibration**: We create a reference spectrum based on the throughput of the respective grism (G102 or G141) and a stellar model.
The user can decide if he or she wants to download a stellar spectrum from MAST or use a black body spectrum.
This template is used for the wavelength calibration of the WFC3 spectra. 
We also determine the position of the star in the direct images which are commonly taken at the start of HST orbits to create an initial guess for the wavelength solution using the known dispersion of the grism.
Using the reference spectrum as a template, we determine a shift and scaling in wavelength-space that minimizes the difference between the template and the first spectrum in the visit.
This first exposure in the visit is then used as the template for the following exposures in the visit.

- **Optimal extraction and outlier removal**: `PACMAN` uses an optimal extraction algorithm as presented in @Horne1986 which iteratively masks bad pixels in the image. 
We also mask bad pixels that have been flagged by `calwf3` with data quality DQ = 4 or 512\footnote{for a list of DQ flags see \url{https://wfc3tools.readthedocs.io/en/latest/wfc3tools/calwf3.html}}.

- **Scanning of the detector**: The majority of exoplanetary HST/WFC3 observations use the spatial scanning technique [@McCullough2012] which spreads the light perpendicular to the dispersion direction during the exposure enabling longer integration times before saturation.
The _ima_ files taken in this observation mode consist of a number of nondestructive reads, also known as up-the-ramp samples, each of which we treat as an independent subexposure.
\autoref{fig:figure1} (left panel) shows an example of the last subexposure when using spatial scanning together with the expected position of the trace based on the direct image.

- **Fitting models**: `PACMAN` contains several functions to fit models which are commonly used with HST data. 
The user can fit models like in \autoref{eq:equation1} to the white light curve or to spectroscopic light curves. 
An example of a raw spectroscopic light curve and fitting \autoref{eq:equation1} to it, can be found in \autoref{fig:figure2}.
Here are some examples of the currently implemented models for the instrument systematics and the astrophysical signal:
  - systematics models:
    - visit-long polynomials
    - orbit-long exponential ramps due to charge trapping: NIR detectors like HST/WFC3 can trap photoelectrons [@Smith2008], which will cause the number of recorded photoelectrons to increase exponentially, creating typical hook-like features in each orbit
  - astrophysical models:
    - transit and secondary eclipse curves as implemented in `batman`
    - sinusoids for phase curve fits
    - a constant offset that accounts for the upstream-downstream effect [@McCullough2012] caused by forward and reverse scanning
    
  A typical model to fit an exoplanet transit in HST data is the following [used, for example, by @Kreidberg2014a]: 


    \begin{equation}
    \label{eq:equation1}
    F(t) = T(t) \, (c\,S(t) + k\,t_{\rm{v}}) \, (1 - \exp(-r_1\,t_{\rm{orb}} - r_2 )),
    \end{equation}


  with _T(t)_ being the transit model, _c_ (_k_) a constant (slope), _S(t)_ a scale factor equal to 1 for exposures with spatial scanning in the forward direction, and _s_ for reverse
  scans, $r_{\rm{1}}$ and $r_{\rm{2}}$ are parameters to account for the exponential ramps. $t_{\rm{v}}$ and $t_{\rm{orb}}$ are the times from the first exposure in the visit and in the orbit, respectively.

- **Parameter estimation**: The user has different options to estimate best fitting parameters and their uncertainties:
  - least squared: `scipy.optimize`
  - MCMC: `emcee` [@Foreman-Mackey2013]
  - nested sampling: `dynesty` [@Speagle2020]

- **Multi-visit observations**
  - `PACMAN` has also an option to share parameters across visits.

- **Binning of the light spectrum**: The user can freely specify the bin numbers or locations. 
\autoref{fig:figure1} (right panel) shows the resulting 1D spectrum and a user-defined binning.

\autoref{fig:figure1} and \autoref{fig:figure2} show some figures created by `PACMAN` during a run using three HST visits of GJ 1214 b collected in [GO 13201](https://archive.stsci.edu/proposal_search.php?id=13021&mission=hst) [@Bean2012].
An analysis of all 15 visits was published in @Kreidberg2014a. The analysis of three visits here using `PACMAN`, is consistent with the published results.

![_Left panel_: a typical single exposure showing the raw 2D spectrum. _Right panel_: 1D spectrum after the use of optimal extraction including vertical dashed lines showing the user-set binning to generate spectroscopic light curves.\label{fig:figure1}](figures/fig1.pdf "title-2"){ width=99.9% }

![_panel A_: raw white light curves for each of the three visits. One can clearly see the constant offset between two adjacent exposures due to the spatial scanning mode. _panel B_: white light curve with the best astrophysical model fit using \autoref{eq:equation1}. _panel C_: the transmission spectrum after fitting 11 spectroscopic light curves revealing the flat spectrum of GJ 1214 b as published in @Kreidberg2014a.\label{fig:figure2}](figures/fig2.pdf "title-2"){ width=99.9% }


# Dependencies

`PACMAN` uses typical dependencies of astrophysical Python codes: `numpy` [@numpy2020], `matplotlib` [@matplotlib2007], `scipy` [@scipy2020] and `astropy` [@astropy2013; @astropy2018; @astropy2022].

Other dependencies required for the fitting stage depending on the model and sampler being run are: `batman` [@Kreidberg2015], `emcee` [@Foreman-Mackey2013], `dynesty` [@Speagle2020], and `corner` [@corner2016].

For the barycentric correction, `PACMAN` accesses the [API to JPL's Horizons system](https://ssd-api.jpl.nasa.gov/obsolete/horizons_batch_cgi.html).

If the user decides to use a stellar spectrum for the wavelength calibration, `PACMAN` will download the needed fits file from the [``REFERENCE-ATLASES'' high level science product](https://archive.stsci.edu/hlsps/reference-atlases/cdbs/grid/) hosted on the MAST archive [@STScI2013].


# Documentation

The documentation for `PACMAN` can be found at [pacmandocs.readthedocs.io](https://pacmandocs.readthedocs.io/en/latest/) hosted on [ReadTheDocs](https://readthedocs.org/).
It includes most notably, a full explanation of every parameter in the _pacman control file_ (pcf), the API, and an example of how to download, reduce and analyze observations of GJ 1214 b taken with HST/WFC3/G141.


# Future work

The following features are planned for future development:

- The addition of fitting models like phase curves using the open-source Python package `SPIDERMAN` [@Louden2018].
- Orbit-long ramp fitting using the [RECTE systematic model](https://recte.readthedocs.io/en/latest/).
- Limb darkening calculations for users wanting to fix limb darkening parameters to theoretical models in the fitting stage.
- Extension to WFC3/UVIS data reduction.


# Acknowledgements

We acknowledge B. Zawadzki for the creation of the `PACMAN` logo.
We also acknowledge the comments and contributions by I. Momcheva to `PACMAN`.


# References


