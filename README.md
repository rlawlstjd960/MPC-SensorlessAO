# MPC-SensorlessAO
This is a `Matlab` implementation of the Model Predictive Control of Time-varying Aberrations for Sensorless Adaptive Optics from Jinsung~Kim et al. [[1]], designed for Sensorless adaptive optics. It maintains estimates of the moments of the gradient independently for each parameter.

<img src="images/SensorlessAO_overview.eps" />
<img src="images/block_diagram.png" />

[![Hits](https://hits.seeyoufarm.com/api/count/incr/badge.svg?url=https%3A%2F%2Fgithub.com%2Frlawlstjd960%2Fhit-counter&count_bg=%2379C83D&title_bg=%23555555&icon=&icon_color=%23E7E7E7&title=hits&edge_flat=false)](https://hits.seeyoufarm.com)

##  Generating Time-varying Phase aberration
The time-varying phase aberration caused by atmospheric turbulence is generated by **Object-Oriented MATLAB & Adaptive Optics (OOMAO) Toolbox** [[2]].
The OOMAO sources are hosted by GitHub (https://github.com/rconan/OOMAO).
To illustrate a complex environment such as the real-world atmosphere, we designed a *mainly frozen-flow model* with three layers at diferent heights, wind speeds, and directions [[3]].

```matlab
clear;clc;close all;
addpath(genpath('D:\Users\Her\Documents\MATLAB\AO\4F-Optical-Correlator-Simulation-kim\OOMAO-master'))

samplingFreq = 200; % turbulence sampling frequency
nSimSteps = 2000;
T_final = nSimSteps;
nPolynomials = 28; % number of zernike modes (radial number = 6)

D = 1; % telescope diameter [m]
r0 = 0.2; % Fried parameter [m] % The strength of atmospheric turbulence (D/r0) is set as 5.
L0 = 42; % outer scale [m]

% mainly frozen-flow model
h = [1000, 5000, 12000]; % height [m]
v0 = [5, 7.5, 10]; % wind speed [m/s]
vD = [0, pi/3, 5*pi/3];

f_r0 = [0.7, 0.1, 0.2]./25; % Fractional r0

atm = atmosphere(photometry.V,r0,L0,'fractionnalR0',f_r0,'altitude',h,'windSpeed',v0,'windDirection',vD); % atmosphere

% telescope setting
nL = 512; % number of lenslet
nPx = 1; % number or pixel per lenslet
nRes = nL*nPx; % resolution
xaxis = 1:nRes;
d = D/nL; % lenslet pitch

tel = telescope(D,'resolution',nRes,'fieldOfViewInArcsec',2.5,'samplingTime',1/samplingFreq); % telescope
tel = tel + atm;

% Photometric system
% Band ( wavelength , bandwidth , zero point )
% V  ( 0.550e-6 , 0.090e-6 , 3.3e12 )
ngs = source('wavelength',photometry.V); % natural guide star
ngs = ngs.*tel;

sprintf('Calculating wavefront for %d steps ...',nSimSteps);

phase = []; time_phase = [];
for iSimStep=1:nSimSteps
    tic;
    +ngs;
    +tel;
    phase(:,:,iSimStep) = ngs.meanRmPhase; % time-varying phase aberrations (mainly frozen-flow model)
    time_phase(iSimStep) = ngs.timeStamp;
    zern2 = zern.\(ngs.meanRmPhase); % zernike coefficient generation
    toc;
end
```
##  Model Identification based on Time-series method
To determine the VAR model, any system identification method such as time-series, machine learning, and extrapolation can be used.
In this study, we designed an AR model based on a time-series method.
An open-loop wavefront dataset fxt[k]jk = 1;    ; ttraing was used to identify the model parameters 
![equation](http://latex.codecogs.com/gif.latex?O_t%3D%5Ctext%20%7B%20Onset%20event%20at%20time%20bin%20%7D%20t):
```math
SE = \frac{\sigma}{\sqrt{n}}
```


```matlab

```



##  Model Predictive Control Simulation
```matlab
% The strength of atmospheric turbulence (D/r0) is magnified by 10, 15, 20.
mag_conv_5 = 1;
mag_conv_10 = 1.781797436291855;
mag_conv_15 = 2.498049532979032;
mag_conv_20 = 3.174802103932926;

phase = phase .* mag_conv_10;
ad_acc = ad_acc .* mag_conv_10;
```



## References
[[1]] Jinsung Kim, Kye-Sung Lee, Sang-Chul Lee, Ji Yong Bae, Dong Uk Kim, I Jong Kim, Ki Soo Chang and Hwan Hur. "Model Model Predictive Control of Time-varying Aberrations for Sensorless Adaptive Optics", arXiv preprint. [https://arxiv.org/abs/--](https://arxiv.org/abs/--)

[[2]] R. Conan and C. Correia, “Object-oriented Matlab adaptive optics toolbox,” in Adaptive Optics Systems IV (E. Marchetti, L. M. Close, and J.-P. Vran, eds.), vol. 9148 of Society of Photo-Optical Instrumentation Engineers (SPIE) Conference Series, p. 91486C, Aug. 2014.

[[3]] L. Prengere, C. Kulcs´ar, and H.-F. Raynaud, “Zonal-based highperformance control in adaptive optics systems with application to astronomy and satellite tracking,” J. Opt. Soc. Am. A, vol. 37, pp. 1083–1099, Jul 2020.

[1]: https://arxiv.org/abs/--
[2]: https://www.spiedigitallibrary.org/conference-proceedings-of-spie/9148/91486C/Object-oriented-Matlab-adaptive-optics-toolbox/10.1117/12.2054470.short
[3]: https://www.osapublishing.org/josaa/abstract.cfm?uri=josaa-37-7-1083
