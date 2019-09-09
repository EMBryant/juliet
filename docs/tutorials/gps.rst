.. _gps:

Incorporating Gaussian Processes
===================

So far in the tutorials we have dealt with gaussian white-noise as a good approximation to the underlying 
signals present behind our transits and radial-velocities. However, this kind of process is very unrealistic 
for real data. Within ``juliet``, we allow to model non-white noise models using Gaussian Proccesses (GPs), 
which are not only good for underlying stochastic processes that might be present in the data, but are also very 
good for modelling underlying deterministic processes for which we do not have a good model at hand. GPs attempt to model 
the likelihood, :math:`\mathcal{L}`, as coming from a multi-variate gaussian distribution, i.e., 

:math:`\mathcal{L} =  -\frac{1}{2}\left[N\ln 2\pi + \ln\left|\mathbf{\Sigma}\right|  + \vec{r}^T \mathbf{\Sigma}^{-1}\vec{r} \right],`

where :math:`N` is the number of datapoints, :math:`\mathbf{\Sigma}` is a covariance matrix and :math:`\vec{r}` is the vector 
of the residuals (where each elements is simply our model --- be it a lightcurve model or radial-velocity model --- minus 
the data). A GP provides a form for the covariance matrix using so-called kernels which define its structure, 
and allow to efficiently fit for this underlying non-white noise structure. Within ``juliet`` we provide a wide variety of kernels 
which are implemented through `george <https://george.readthedocs.io/en/latest/>`_ and 
`celerite <https://celerite.readthedocs.io/en/stable/>`_. In this tutorial we test their capabilities using real exoplanetary data!

Detrending lightcurves with GPs
-------------------------------
A very popular use of GPs is to use them for "detrending" lightcurves. This means using the data outside of the feature 
of interest (e.g., a transit) in order to predict the behaviour of the lightcurve inside the feature and remove it, in 
order to facilitate or simplify the lightcurve fitting. To highlight the capabilities of ``juliet``, here we will play around 
with *TESS* data obtained in Sector 1 for the HATS-46b system (`Brahm et al., 2017 <https://arxiv.org/abs/1707.07093>`_). We already 
analyzed transits in Sector 2 for this system in the :ref:`transitfit` tutorial, but here we will tackle Sector 1 data as the systematics 
in this sector are much stronger than the ones of Sector 2.

Let's start by downloading and plotting the *TESS* data for HATS-46b in Sector 1 using ``juliet``:

.. code-block:: python

   import juliet
   import matplotlib.pyplot as plt

   # First, get arrays of times, normalized-fluxes and errors for HATS-46 
   #from Sector 1 from MAST:
   t, f, ferr  = juliet.get_TESS_data('https://archive.stsci.edu/hlsps/'+\
                                      'tess-data-alerts/hlsp_tess-data-'+\
                                      'alerts_tess_phot_00281541555-s01_'+\
                                      'tess_v1_lc.fits')
   # Plot the data:
   plt.errorbar(t,f,yerr=ferr,fmt='.')
   plt.xlim([np.min(t),np.max(t)])
   plt.xlabel('Time (BJD - 2457000)')
   plt.ylabel('Relative flux') 

.. figure:: hats-46_plot.png
   :alt: Sector 1 data for HATS-46b.

As can be seen, the data has a fairly strong long-term trend going around. In fact, the trend is so strong that you cannot 
see the transit by eye! Let us try to get rid of this trend by fitting a GP to the out-of-transit data, and then *predicting* 
the in-transit flux to remove it. Let us first isolate the out-of-transit data from the in-transit data using the ephemerides 
published in `Brahm et al., 2017 <https://arxiv.org/abs/1707.07093>`_ --- we know where the transits should be, so we will 
simply phase-fold the data and remove all datapoints out-of-transit (which judging from the plots in that paper, should be all 
points at absolute phases above 0.02). Let us save this out-of-transit data in dictionaries so we can feed them to ``juliet``:

.. code-block:: python

   # Period and t0 from Anderson et al. (201X):
   P,t0 =  4.7423729 ,  2457376.68539 - 2457000
   # Get phases --- identify out-of-transit (oot) times by phasing the data 
   # and selecting all points at absolute phases larger than 0.02:
   phases = juliet.utils.get_phases(t, P, t0)
   idx_oot = np.where(np.abs(phases)>0.02)[0]   
   # Save the out-of-transit data into dictionaries so we can feed them to juliet:
   times, fluxes, fluxes_error = {},{},{}
   times['TESS'], fluxes['TESS'], fluxes_error['TESS'] = t[idx_oot],f[idx_oot],ferr[idx_oot]

Now, let us fit a GP to this data. To do this, we will use a simple (approximate) Matern kernel, which was implemented via 
`celerite <https://celerite.readthedocs.io/en/stable/>`_ and which can accomodate itself to both rough and smooth signals. On top of this, 
the selection was also made because this is implemented in ``celerite``, which makes the computation of the 
log-likelihood blazing fast --- this in turn speeds up the posterior sampling within ``juliet``. The kernel is given by

:math:`k(\tau_{i,j}) = \sigma^2_{GP}\tilde{M}(\tau_{i,j},\rho) + (\sigma^2_{i} + \sigma^2_{w})\delta_{i,j}`,

where :math:`k(\tau_{i,j})` gives the element :math:`i,j` of the covariance matrix :math:`\mathbf{\Sigma}`, :math:`\tau_{i,j} = |t_i - t_j|` 
with the :math:`t_i` and :math:`t_j` being the :math:`i` and :math:`j` GP regressors (typically --- as in this case --- the times), 
:math:`\sigma_i` the errorbar of the :math:`i`-th datapoint, :math:`\sigma_{GP}` sets the amplitude (in ppm) of the GP, :math:`\sigma_w` (in ppm) is an added 
(unknown) *jitter* term, :math:`\delta_{i,j}` a Kronecker's delta (i.e., zero when :math:`i \neq j`, one otherwise) and where

:math:`\tilde{M}(\tau_{i,j},\rho) = [(1+1/\epsilon)\exp(-[1-\epsilon]\sqrt{3}\tau/\rho) + (1- 1/\epsilon)\exp(-[1+\epsilon]\sqrt{3}\tau/\rho)]`

is the (approximate) Matern part of the kernel, which has a characteristic length-scale :math:`\rho`.

To use this kernel within ``juliet`` you just have to give the priors for these parameters in the prior dictionary or file (see below for 
a full list of all the available kernels). ``juliet`` will automatically recognize which kernel you want based on the priors selected for 
each instrument. In this case, if you define a parameter ``GP_sigma`` (for :math:`\sigma_{GP}`) and ``rho`` (for the 
Matern time-scale, :math:`\rho`), ``juliet`` will automatically recognize you want to use this (approximate) Matern kernel. Let's thus give 
these priors --- for now, let us set the dilution factor ``mdilution`` to 1, give a normal prior for the mean out-of-transit flux ``mflux`` and 
wide log-uniform priors for all the other parameters:

.. code-block:: python
    :emphasize-lines: 16

    # Set the priors:
    params =  ['mdilution_TESS', 'mflux_TESS', 'sigma_w_TESS', 'GP_sigma_TESS', \
               'GP_rho_TESS']
    dists =   ['fixed',          'normal',     'loguniform',   'loguniform',\
               'loguniform']
    hyperps = [1., [0.,0.1], [1e-6, 1e6], [1e-6, 1e6],\
               [1e-3,1e3]]

    priors = {}
    for param, dist, hyperp in zip(params, dists, hyperps):
        priors[param] = {}
        priors[param]['distribution'], priors[param]['hyperparameters'] = dist, hyperp

    # Perform the juliet fit. Load dataset first (note the GP regressor will be the times):
    dataset = juliet.load(priors=priors, t_lc = times, y_lc = fluxes, \
                          yerr_lc = fluxes_error, GP_regressors_lc = times, \
                          out_folder = 'hats46_detrending')
    # Fit:
    results = dataset.fit()

Note that the only new part in terms of loading the dataset is that one has to now add a new piece of data, the ``GP_regressors_lc``, 
in order for the GP to run (emphasized in the code above). This is also a dictionary, which specifies the GP regressors for each instrument. 
For ``celerite`` kernels, in theory the regressors have to be one-dimensional and ordered in ascending or descending order --- however, 
internally ``juliet`` performs this ordering so the user doesn't have to worry about this last part. Let us now plot the GP fit and some 
residuals below to see how we did:

.. code-block:: python

    # Import gridspec:
    import matplotlib.gridspec as gridspec
    # Get juliet model prediction for the full lightcurve:
    model_fit = results.lc.evaluate('TESS')

    # Plot:
    fig = plt.figure(figsize=(10,4))
    gs = gridspec.GridSpec(2, 1, height_ratios=[2,1])

    # First the data and the model on top:
    ax1 = plt.subplot(gs[0])
    ax1.errorbar(times['TESS'], fluxes['TESS'], fluxes_error['TESS'],fmt='.',alpha=0.1)
    ax1.plot(times['TESS'], model_fit, color='black', zorder=100)
    ax1.set_ylabel('Relative flux')
    ax1.set_xlim(np.min(times['TESS']),np.max(times['TESS']))
    ax1.xaxis.set_major_formatter(plt.NullFormatter())

    # Now the residuals:
    ax2 = plt.subplot(gs[1])
    ax2.errorbar(times['TESS'], (fluxes['TESS']-model_fit)*1e6, \
                 fluxes_error['TESS']*1e6,fmt='.',alpha=0.1)
    ax2.set_ylabel('Residuals (ppm)')
    ax2.set_xlabel('Time (BJD - 2457000)')
    ax2.set_xlim(np.min(times['TESS']),np.max(times['TESS']))    

.. figure:: hats-46_GPfitmatern.png
   :alt: Sector 1 data for HATS-46b with an approximate Matern kernel on top

Seems we did pretty good! By default, the ``results.lc.evaluate`` function evaluates the model on the input dataset (i.e., on the 
input GP regressors and input times). In our case, this was the out-of-transit data. To detrend the lightcurve, however, we have to *predict* 
the model on the full time-series. This is easily done using the same function but giving the times and GP regressors we want to predict the 
data on. So let us detrend the original lightcurve (stored in the arrays ``t``, ``f`` and ``ferr`` that we extracted at the beggining of 
this section), and fit a transit to it to see how we do:

.. code-block:: python

    # Get model prediction from juliet:
    model_prediction = results.lc.evaluate('TESS', t = t, GPregressors = t)

    # Repopulate dictionaries with new detrended flux:
    times['TESS'], fluxes['TESS'], fluxes_error['TESS'] = t, f/model_prediction, \
                                                          ferr/model_prediction

    # Set transit fit priors:
    priors = {}

    params = ['P_p1','t0_p1','r1_p1','r2_p1','q1_TESS','q2_TESS','ecc_p1','omega_p1',\
                  'rho', 'mdilution_TESS', 'mflux_TESS', 'sigma_w_TESS']

    dists = ['normal','normal','uniform','uniform','uniform','uniform','fixed','fixed',\
                     'loguniform', 'fixed', 'normal', 'loguniform']

    hyperps = [[4.7,0.1], [1329.9,0.1], [0.,1], [0.,1.], [0., 1.], [0., 1.], 0.0, 90.,\
                       [100., 10000.], 1.0, [0.,0.1], [0.1, 1000.]]

    # Populate the priors dictionary:
    for param, dist, hyperp in zip(params, dists, hyperps):
        priors[param] = {}
        priors[param]['distribution'], priors[param]['hyperparameters'] = dist, hyperp

    # Perform juliet fit:
    dataset = juliet.load(priors=priors, t_lc = times, y_lc = fluxes, \
                      yerr_lc = fluxes_error, out_folder = 'hats46_detrended_transitfit')

    results = dataset.fit()

    # Extract transit model prediction given the data:
    transit_model = results.lc.evaluate('TESS')

    # Plot results:
    fig = plt.figure(figsize=(10,4))
    gs = gridspec.GridSpec(1, 2, width_ratios=[2,1])
    ax1 = plt.subplot(gs[0])

    # Plot time v/s flux plot:
    ax1.errorbar(dataset.times_lc['TESS'], dataset.data_lc['TESS'], \
             yerr = dataset.errors_lc['TESS'], fmt = '.', alpha = 0.1)

    ax1.plot(dataset.times_lc['TESS'], transit_model,color='black',zorder=10)
 
    ax1.set_xlim([1328,1350])
    ax1.set_ylim([0.96,1.04])
    ax1.set_xlabel('Time (BJD - 2457000)')
    ax1.set_ylabel('Relative flux')
   
    # Now phased transit lightcurve:
    ax2 = plt.subplot(gs[1])
    ax2.errorbar(phases, dataset.data_lc['TESS'], \
                 yerr = dataset.errors_lc['TESS'], fmt = '.', alpha = 0.1)
    idx = np.argsort(phases)
    ax2.plot(phases[idx],transit_model[idx], color='black',zorder=10)
    ax2.yaxis.set_major_formatter(plt.NullFormatter())
    ax2.set_xlim([-0.03,0.03])
    ax2.set_ylim([0.96,1.04])
    ax2.set_xlabel('Phases')

.. figure:: juliet_h46_transit_fit.png
   :alt: juliet fit to Sector 1 detrended data for HATS-46b. 

Pretty good! In the next section, we explore *joint* fitting for the transit model and the GP process.

Joint GP and lightcurve fits
-----------------------------

One might wonder what the impact of doing the two-stage process mentioned above is when compared with fitting *jointly* 
the GP process and the transit model. This latter method, in general, seems more appealing because it can take into 
account in-transit non-white noise features, which in turn might give rise to more realistic errorbars on the retrieved 
planetary parameters. Within ``juliet`` performing this kind of model fit is fairly easy to do: one just has to add the 
priors for the GP process to the transit paramenters, and feed the GP regressors. Let us use the same GP kernel as in the 
previous section then to model the underlying process for HATS-46b *jointly* with the transit parameters:

.. code-block:: python
    :emphasize-lines: 7,11,15

    # First define the priors:
    priors = {}

    # Same priors as for the transit-only fit, but we now add the GP priors:
    params = ['P_p1','t0_p1','r1_p1','r2_p1','q1_TESS','q2_TESS','ecc_p1','omega_p1',\
              'rho', 'mdilution_TESS', 'mflux_TESS', 'sigma_w_TESS', \
              'GP_sigma_TESS', 'GP_rho_TESS']

    dists = ['normal','normal','uniform','uniform','uniform','uniform','fixed','fixed',\
             'loguniform', 'fixed', 'normal', 'loguniform', \
             'loguniform', 'loguniform']

    hyperps = [[4.7,0.1], [1329.9,0.1], [0.,1], [0.,1.], [0., 1.], [0., 1.], 0.0, 90.,\
               [100., 10000.], 1.0, [0.,0.1], [0.1, 1000.], \
               [1e-6, 1e6], [1e-3, 1e3]]

    # Populate the priors dictionary:
    for param, dist, hyperp in zip(params, dists, hyperps):
        priors[param] = {}
        priors[param]['distribution'], priors[param]['hyperparameters'] = dist, hyperp

    times['TESS'], fluxes['TESS'], fluxes_error['TESS'] = t,f,ferr
    dataset = juliet.load(priors=priors, t_lc = times, y_lc = fluxes, \
                          yerr_lc = fluxes_error, GP_regressors_lc = times, out_folder = 'hats46_transitGP', verbose = True)

    results = dataset.fit()

Note that in comparison with the transit-only fit, we have just added the priors for the GP parameters 
(highlighted lines above). The model being fit in this case by ``juliet`` is the one given in Section 2 
of the `juliet paper <https://arxiv.org/abs/1812.08549>`_, i.e., a model of the form

:math:`\mathcal{M}_{\textrm{TESS}}(t) + \epsilon(t)`,

where 

:math:`\mathcal{M}_{\textrm{TESS}}(t) = [T(t)D_{\textrm{TESS}} + (1-D_{\textrm{TESS}})]\left(\frac{1}{1+D_{\textrm{TESS}}}M_{\textrm{TESS}}\right)`

is the photometric model composed of the dilution factor :math:`D_{\textrm{TESS}}` (``mdilution_TESS``) and the mean out-of-transit 
flux :math:`M_{\textrm{TESS}}` (``mflux_TESS``). This is the *deterministic* part of the model, as 
:math:`\mathcal{M}_{\textrm{TESS}}(t)` is a process that, given a time and a set of parameters, will always be the same: you can easily 
evaluate the model from the above definition. :math:`\epsilon(t)`, on the other hand, is the *stochastic* part of our model: a noise model which 
in our case is being modelled as a GP. Given a set of parameters and times for the GP model, the process *cannot* directly be evaluated because 
it defines a probability distribution, not a deterministic function like :math:`\mathcal{M}_{\textrm{TESS}}(t)`. This means that every time 
you sample from this GP, you would get a different curve --- ours was just *one realization* of many possible ones. However, we do have a 
(noisy) realization (our data) and so our process can be constrained by it. This is what we plotted in the previous section of this tutorial 
(which in strict rigor is a filter). Also note that in this model the GP is an additive process.

Once the fit is done, ``juliet`` allows to retrieve (1) the full median posterior model (i.e., the deterministic part of the model **plus** the 
median GP process) via the ``results.lc.evaluate()`` function already used in the previous section and (2) all parts of the model 
separately via the ``results.lc.model`` dictionary, which holds the ``deterministic`` key which hosts the deterministic part of the model 
(:math:`\mathcal{M}_{\textrm{TESS}}(t)`) and the ``GP`` key which holds the stochastic part of the model (:math:`\epsilon(t)`, constrained 
on the data). To show how this works, let us extract these components below in order to plot the full model, and remove the median GP process 
from the data in order to plot the ("systematics-corrected") phase-folded lightcurve:

.. code-block:: python

    # Extract full model:
    transit_plus_GP_model = results.lc.evaluate('TESS')

    # Deterministic part of the model (in our case transit divided by mflux):
    transit_model = results.lc.model['TESS']['deterministic']

    # GP part of the model:
    gp_model = results.lc.model['TESS']['GP']

    # Now plot. First preambles:
    fig = plt.figure(figsize=(12,4))
    gs = gridspec.GridSpec(1, 2, width_ratios=[2,1])
    ax1 = plt.subplot(gs[0])

    # Plot data
    ax1.errorbar(dataset.times_lc['TESS'], dataset.data_lc['TESS'], \
                 yerr = dataset.errors_lc['TESS'], fmt = '.', alpha = 0.1)

    # Plot the (full, transit + GP) model:
    ax1.plot(dataset.times_lc['TESS'], transit_plus_GP_model, color='black',zorder=10)

    ax1.set_xlim([1328,1350])
    ax1.set_ylim([0.96,1.04])
    ax1.set_xlabel('Time (BJD - 2457000)')
    ax1.set_ylabel('Relative flux')

    ax2 = plt.subplot(gs[1])

    # Now plot phase-folded lightcurve but with the GP part removed:
    ax2.errorbar(phases, dataset.data_lc['TESS'] - gp_model, \
                 yerr = dataset.errors_lc['TESS'], fmt = '.', alpha = 0.3)

    # Plot transit-only (divided by mflux) model:
    idx = np.argsort(phases)
    ax2.plot(phases[idx],transit_model[idx], color='black',zorder=10)
    ax2.yaxis.set_major_formatter(plt.NullFormatter())
    ax2.set_xlabel('Phases')
    ax2.set_xlim([-0.03,0.03])
    ax2.set_ylim([0.96,1.04])

.. figure:: gp_joint_fit.png
   :alt: Simultaneous GP and transit juliet fit to Sector 1 data for HATS-46b.

Looks pretty good! As can be seen, the ``results.lc.model['TESS']['deterministic']`` dictionary holds the deterministic 
part of the model. This includes the transit model which is distorted by the dilution factor (set to 1 in our case) and the 
mean out-of-transit flux, which we fit together with the other parameters in our joint fit --- this deterministic model is the one 
that is plotted in the right panel in the above presented figure. The ``results.lc.model['TESS']['GP']`` dictionary, on the other 
hand, holds the GP part of the model --- because this is an additive process in this case, we can just substract it from the data 
in order to get the "systematic-corrected" data that we plot in the right panel in the figure above.
