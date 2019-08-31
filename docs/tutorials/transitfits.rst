.. _quicktest:

Fitting transits
===================

Two ways of using juliet
-------------------------

In the spirit of accomodating the code for everyone to use, `juliet` can be used in two different ways: as 
an *imported library* and also in *command line mode*. Both give rise to the same results because the command 
line mode simply calls the `juliet` libraries in a python script.

To use `juliet` as an *imported library*, inside any python script you can simply do:

.. code-block:: python

    import juliet
    dataset = juliet.load(priors,t_lc=times,y_lc=flux,yerr_lc=flux_error)

In this example, `juliet` will simply load your dataset into a ``juliet.load`` object with which you can play around 
to fit, plot or predict  a lightcurve defined by a dictionary of times ``times``, of relative fluxes ``flux`` and errors 
on those fluxes ``flux_error`` given some prior information ``prior`` which, as we will see below, is defined through either 
a dictionary or a file. 


In *command line mode*, `juliet` can be used through a simple call in any terminal. To do this, after 
installing juliet, you can from anywhere in your system simply do:

.. code-block:: bash

    juliet -flag1 -flag2 --flag3

In this example, `juliet` is performing a fit using different inputs defined by `-flag1`, `-flag2` and `--flag3`. 
There are several flags that can be used to accomodate your `juliet` runs. If this mode suits your needs, 
check out the `project's wiki page to find out more about this mode <https://github.com/nespinoza/juliet/wiki>`_.

A primer on transit fitting
-----------------------------------------------

Let us first perform an extremely simple fit to data using `juliet` as an *imported library*. In this first 
tutorial, we will fit the TESS data of TOI-141 b, which was shown to host a 1-day transiting exoplanet 
by `Espinoza et al. (2019) <https://arxiv.org/abs/1903.07694>`_. Let us first load the data corresponding to this 
object, which is hosted in MAST:

.. code-block:: python

    import numpy as np
    from astropy.io import fits

    def get_TESS_data(filename):
        """
        Given a filename, this function returns an array of times, 
        fluxes and errors on those fluxes.
        """
        # Manipulate the fits file:
        data = fits.getdata(filename)

        # Identify zero-flux values to take them out of the data arrays:
        idx = np.where(data['PDCSAP_FLUX']!=0.)[0]

        # Return median-normalized flux:
        return data['TIME'][idx],data['PDCSAP_FLUX'][idx]/np.median(data['PDCSAP_FLUX'][idx]), \
               data['PDCSAP_FLUX_ERR'][idx]/np.median(data['PDCSAP_FLUX'][idx])
     
    # First, get times, normalized-fluxes and errors for TOI-141 from MAST:
    t,f,ferr  = get_TESS_data('https://archive.stsci.edu/hlsps/tess-data-alerts/hlsp_tess'+\
                               '-data-alerts_tess_phot_00403224672-s01_tess_v1_lc.fits')
    
Now, in order to load this dataset into `juliet`, we need to put the times, fluxes and errors into dictionaries. 
This makes it extremely easy to add more instruments, as simply other instruments will be other keys of this 
dictionary. For now, let us just use this TESS data; we put them in dictionaries that `juliet` likes as follows:

.. code-block:: python

    # Create dictionaries:
    times, fluxes, fluxes_error = {},{},{}
    # Save data into those dictionaries:
    times['TESS'], fluxes['TESS'], fluxes_error['TESS'] = t,f,ferr
    # If you had data from other instruments you would simplt do, e.g.,
    # times['K2'], fluxes['K2'], fluxes_error['K2'] = t_k2,f_k2,ferr_k2

The final step to fit the data with `juliet` is to define the priors for the different parameters that we 
are going to fit. This can be done in two ways. The longest (but more jupyter-notebook-friendly?) is to 
create a dictionary that, on each key, has the parameter to be fitted. Each key, in turn, is a dictionary 
containing the ``distribution`` and its ``hyperparameters`` (for details on what distributions 
`juliet` can handle, what are the hyperparameters and what each parameter name mean, see Sections 2.1, 2.2 and 
2.3 of the `juliet` `wiki page <https://github.com/nespinoza/juliet/wiki/Installing-and-basic-usage>`_ ). 

Let us give normal priors for the period ``P_p1``, time-of-transit center ``t0_p1``, mean out-of-transit 
flux ``mflux_TESS``, uniform distributions for the ``r1_p1`` and ``r2_p1`` of the `Espinoza (2018) <https://ui.adsabs.harvard.edu/abs/2018RNAAS...2d.209E/abstract>_` parametrization 
for the impact parameter and planet-to-star radius ratio, same for the ``q1_p1`` and ``q2_p1`` `Kipping (2013) <https://ui.adsabs.harvard.edu/abs/2013MNRAS.435.2152K/abstract>_` 
limb-darkening parametrization, log-uniform distributions for the stellar density ``rho`` (in kg/m3) and 
jitter term ``sigma_w_TESS``, and leave the rest of the parameters (eccentricity ``ecc_p1``, argument of 
periastron ``omega_p1`` and dilution factor ``mdilution_TESS``) fixed: 

.. code-block:: python

    priors = {}

    params = ['P_p1','t0_p1','r1_p1','r2_p1','q1_TESS','q2_TESS','ecc_p1','omega_p1',\
                  'rho', 'mdilution_TESS', 'mflux_TESS', 'sigma_w_TESS']

    dists = ['normal','normal','uniform','uniform','uniform','uniform','fixed','fixed',\
                     'loguniform', 'fixed', 'normal', 'loguniform']

    hyperps = [[1.,0.1], [1325.55,0.1], [0.,1], [0.,1.], [0., 1.], [0., 1.], 0.0, 90.,\
                       [100., 10000.], 1.0, [0.,0.1], [0.1, 1000.]]

    for param, dist, hyperp in zip(params, dists, hyperps):
        priors[param] = {}
        priors[param]['distribution'], priors[param]['hyperparameters'] = dist, hyperp

With these definitions, to fit this dataset with `juliet` one would simply do:

.. code-block:: python

    # Load dataset into juliet, save results to a temporary folder called toi141_fit:
    dataset = juliet.load(priors=priors, t_lc = times, y_lc = fluxes, \
                          yerr_lc = fluxes_error, out_folder = 'toi141_fit')

    # Fit and absorb results into a juliet.fit object:
    results = dataset.fit(n_live_points = 300)

This code will run `juliet` and save the results to the ``toi141_fit`` folder. 

The second way to define the priors for `juliet` (and perhaps the most simple) is to create a text file where 
in the first column one defines the parameter name, in the second column the name of the ``distribution`` and 
in the third column the ``hyperparameters``. The priors defined above would look like this in a text file:

.. code-block:: bash

    P_p1                 normal               1.0,0.1   
    t0_p1                normal               1325.55,0.1 
    r1_p1                uniform              0.0,1.0 
    r2_p1                uniform              0.0,1.0    
    q1_TESS              uniform              0.0,1.0 
    q2_TESS              uniform              0.0,1.0 
    ecc_p1               fixed                0.0 
    omega_p1             fixed                90.0
    rho                  loguniform           100.0,10000.0
    mdilution_TESS       fixed                1.0
    mflux_TESS           normal               0.0,0.1
    sigma_w_TESS         loguniform           0.1,1000.0

To run the same fit as above, suppose this prior file is saved under ``toi141_fit/priors.dat``. Then, to load this 
dataset into `juliet` and fit it, one would do:

.. code-block:: python

    # Load dataset into juliet, save results to a temporary folder called toi141_fit:
    dataset = juliet.load(priors='toi141_fit/priors.dat', t_lc = times, y_lc = fluxes, \
                          yerr_lc = fluxes_error, out_folder = 'toi141_fit')

    # Fit and absorb results into a juliet.fit object:
    results = dataset.fit(n_live_points = 300)

And that's it! Cool `juliet` fact is that, once you have defined an ``out_folder``, **all your data will be saved there --- 
not only the prior file and the results of the fit, but also the photometry or radial-velocity you fed into juliet will 
be saved**. This makes it easy to come back later to this dataset without having to download the data all over again. So, 
for example, if we ran the above defined code and we wanted to come back at this dataset again with another `python` 
session and say, plot the data, one can simply do:

.. code-block:: python

   # Load already saved dataset with juliet:
   import juliet
   dataset = juliet.load(input_folder = 'toi141_fit', out_folder = 'toi141_fit')
   
   import matplotlib.pyplot as plt
   plt.errorbar(dataset.times_lc['TESS'], dataset.data_lc['TESS'], \
                yerr = dataset.errors_lc['TESS'], fmt = '.')
   plt.show()

