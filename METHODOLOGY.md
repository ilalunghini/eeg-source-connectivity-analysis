# EEG Source Connectivity Analysis

## Overview

This academic project follows a source-level EEG workflow: scalp recordings are projected to cortical sources, summarised within atlas-defined regions, and analysed with a directed connectivity measure.

This page documents the complete methodological pipeline. It does not include EEG recordings, participant information, MRI or atlas resources, cortical surfaces, derived source estimates, connectivity matrices, figures, or the complete analysis code.

The central idea is to move through three levels of description: the EEG sensors measure electrical activity at the scalp; a source model estimates the cortical regions that may have generated those measurements; and a network analysis studies interactions between those regions. Each level depends on the assumptions made in the previous one.
## 1. Data preparation

The analysis begins with a preprocessed EEGLAB dataset. It is loaded in EEGLAB and converted to a FieldTrip structure, which is used for source modelling and connectivity analysis.

```matlab
EEG = pop_loadset('recording.set');
eegData = eeglab2fieldtrip(EEG, 'preprocessing');
```

The input dataset itself is not distributed. Channel labels, electrode positions, sampling rate, and the EEG time series are retained in the FieldTrip structure for the following steps.

EEG does not measure the activity of one isolated brain area at a time. Electrical fields spread through the brain, cerebrospinal fluid, skull, and scalp before reaching the electrodes. As a result, every sensor usually contains a mixture of contributions from several sources. This is the reason source reconstruction is needed before interpreting the signal anatomically.

## 2. Template head model and source space

A template MRI is segmented into scalp, skull, and brain compartments. These surfaces are meshed to build a three-layer boundary-element head model. A cortical surface provides the candidate source locations (solution points).

```matlab
mri = ft_read_mri('template_mri.nii');
mri.coordsys = 'mni';

cfg = [];
cfg.output = {'scalp', 'skull', 'brain'};
segmentedMri = ft_volumesegment(cfg, mri);

cfg = [];
cfg.method = 'projectmesh';
cfg.tissue = {'scalp', 'skull', 'brain'};
mesh = ft_prepare_mesh(cfg, segmentedMri);

cfg = [];
cfg.method = 'bemcp';
headmodel = ft_prepare_headmodel(cfg, mesh);
```

The head model represents how electrical activity inside the brain is conducted to the scalp.

This step defines the geometry and conductive boundaries of the forward problem: if a source were active at a known cortical location, the model could predict how its signal would appear at the electrodes. The source space defines the locations that are allowed to carry activity. It is therefore a spatial model of the possible generators, not a measurement of neural activity by itself.

## 3. Sensor layout and electrode alignment

A two-dimensional channel layout is prepared for sensor-level quality-control plots. Electrode positions are then projected onto the scalp surface of the head model and converted to a common unit system.

This alignment is important because the forward model requires electrode locations, the head model, and the cortical source space to be expressed in compatible coordinates.

An error at this stage would shift the relationship between the electrodes and the anatomy. Even when the later algorithms are configured correctly, a geometrical mismatch can lead to inaccurate source estimates.

## 4. Timelocked signal and sensor-level inspection

The EEG is converted to a timelocked representation with a sensor covariance estimate. Sensor time courses and global mean field power are inspected before source reconstruction.

```matlab
cfg = [];
cfg.covariance = 'yes';
timelockedEeg = ft_timelockanalysis(cfg, eegData);

cfg = [];
cfg.method = 'amplitude';
gmfp = ft_globalmeanfield(cfg, timelockedEeg);
```

These checks make the transition from sensor data to source analysis explicit; no sensor-level recordings or plots are shared.

The timelocked representation organises the signal in a form that can be used by the subsequent FieldTrip routines. The global mean field power (GMFP) gives a time-resolved measure of the overall field strength across sensors. It describes how strongly the scalp field changes at each time point, rather than identifying a particular anatomical region.

## 5. Forward model: lead-field calculation

The lead field is calculated for each cortical solution point. It describes how a hypothetical source at each location would be observed across the EEG electrodes.

```matlab
cfg = [];
cfg.elec = alignedElectrodes;
cfg.channel = timelockedEeg.label;
cfg.headmodel = headmodel;
cfg.grid = sourceSpace;
leadfield = ft_prepare_leadfield(cfg);
```

This forward model is the link between the anatomical model and the inverse source reconstruction.

In matrix form, the relationship can be written conceptually as $Y = LS$, where $S$ is the activity of the candidate sources, $L$ is the lead field, and $Y$ is the measured sensor signal. The forward problem is well defined once the head model, source locations, and electrode positions are specified.

## 6. eLORETA source reconstruction

The inverse solution estimates cortical activity from the sensor-level EEG and the lead field. eLORETA is used with regularisation, and source moments are retained for later regional and connectivity analyses.

```matlab
cfg = [];
cfg.method = 'eloreta';
cfg.sourcemodel = leadfield;
cfg.headmodel = headmodel;
cfg.eloreta.lambda = 0.05;
cfg.eloreta.keepmom = 'yes';
cfg.eloreta.keepfilter = 'yes';

sourceEstimate = ft_sourceanalysis(cfg, timelockedEeg);

cfg = [];
cfg.projectmom = 'yes';
cfg.keeppow = 'yes';
sourceEstimate = ft_sourcedescriptives(cfg, sourceEstimate);
```

Source time courses can then be inspected on the cortical surface.

The inverse problem is underdetermined: there are many more possible source locations than independent EEG measurements, so more than one source configuration could explain the same scalp signal. eLORETA selects a stable, spatially regularised solution that is compatible with the data and the forward model. The regularisation parameter controls the balance between following the measured signal and suppressing unstable or excessively irregular solutions.

The resulting source time courses are model-based estimates, not direct recordings from individual neurons. Their interpretation depends on the head geometry, electrode alignment, source space, noise, and inverse method.

## 7. Interpolation and atlas-based parcellation

The source estimate is interpolated to the template MRI and summarised within atlas-defined regions of interest (ROIs). Averaging within each parcel produces a representative source-level time course for each region.

```matlab
cfg = [];
cfg.parameter = {'mom', 'pow'};
sourceInTemplate = ft_sourceinterpolate(cfg, sourceEstimate, mri);

cfg = [];
cfg.method = 'mean';
cfg.parcellation = 'tissue';
cfg.parameter = {'mom', 'pow'};
sourceParcels = ft_sourceparcellate(cfg, sourceInTemplate, atlas);
```

The atlas, regional time courses, and source-derived maps are not distributed.

Parcellation reduces a large set of cortical points to one representative signal per anatomical region. This makes the next step more interpretable and reduces the dimensionality of the connectivity problem. The regional signal is an aggregate of the source estimates assigned to that parcel, rather than an independent measurement from a single location.

## 8. Spectral representation and directed connectivity

The regional time courses are arranged in a FieldTrip-compatible data structure and transformed into the frequency domain using multitaper Fourier analysis. Partial directed coherence (PDC) is then estimated between ROIs and averaged within the delta band (1–4 Hz).

```matlab
cfg = [];
cfg.method = 'mtmfft';
cfg.output = 'fourier';
cfg.taper = 'dpss';
cfg.tapsmofrq = 2;
cfg.foilim = [1 10];
frequencyData = ft_freqanalysis(cfg, parcelData);

cfg = [];
cfg.method = 'pdc';
connectivity = ft_connectivityanalysis(cfg, frequencyData);

cfg = [];
cfg.parameter = 'pdcspctrm';
cfg.frequency = [1 4];
cfg.avgoverfreq = 'yes';
deltaPdc = ft_selectdata(cfg, connectivity);
```

PDC is directional: the resulting matrix represents directed relationships between ROI signals rather than undirected similarity alone.

The analysis first represents each regional signal in the frequency domain. Multitaper Fourier analysis uses several optimised tapers to obtain a more stable spectral estimate than a single-window transform. The project then focuses on the delta band, from 1 to 4 Hz.

Partial directed coherence is derived from a multivariate autoregressive model. It asks whether the activity of one region contributes information that helps explain the frequency-specific activity of another region while considering the other modelled regions. Therefore, the connection from region A to region B is conceptually distinct from the connection from B to A. PDC is a statistical directionality measure; by itself, it does not prove anatomical or physiological causality.

## 9. Network-level summary and visualisation

The PDC matrix is prepared for network inspection by handling missing values and normalising the weights. Weighted node strength is then used as a summary of the total connectivity associated with each ROI. Connectivity matrices may also be exported to node-and-edge formats for brain-network visualisation.

In the resulting network, each ROI is a node and each directed PDC value is a weighted edge. Weighted node strength summarises how strongly a region is connected according to the selected direction and normalisation. A high value indicates a prominent position in this modelled network, not necessarily that the region controls the other regions biologically.
