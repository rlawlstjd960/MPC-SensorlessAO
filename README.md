# MPC for Sensorless AO
[![Hits](https://hits.seeyoufarm.com/api/count/incr/badge.svg?url=https%3A%2F%2Fgithub.com%2Frlawlstjd960%2Fhit-counter&count_bg=%2379C83D&title_bg=%23555555&icon=&icon_color=%23E7E7E7&title=hits&edge_flat=false)](https://hits.seeyoufarm.com)

This repository is a `Matlab` implementation of the model predictive control of time-varying aberrations for sensorless adaptive optics from Jinsung~Kim et al. [[1]].
A schematic diagram of the sensorless AO system is shown below.

<img src="images/SensorlessAO_overview.png" />

Incoming light with time-varying phase aberration ![equation](https://latex.codecogs.com/png.image?\dpi{110}%20\phi_{\textrm{distort}}) caused by atmospheric turbulence is directed to the DM with a corrected wavefront ![equation](https://latex.codecogs.com/png.image?\dpi{110}%20\phi_{\textrm{cor}}).
The reflected beam with residual wavefront aberration ![equation](https://latex.codecogs.com/png.image?\dpi{110}%20\phi_{\textrm{res}}%20=%20\phi_{\textrm{distort}}%20+%20\phi_{\textrm{cor}}) is entered onto a scientific camera, such as a CCD or CMOS, to measure the PSF.
The estimator determines the Zernike coefficient constituting ![equation](https://latex.codecogs.com/png.image?\dpi{110}%20\phi_{\textrm{res}}) without using a wavefront sensor, based on the approximate model.
The controller computes the applied voltage to each actuator of the DM to correct the aberration of the newly incoming light.

The block diagram for `Matlab` simulation is represented as follow.

<img src="images/block_diagram.png" />

Before implementing this MATLAB code, you must install the appropriate MPC optimizer such as ***CVX, fmincon, fastMPC***.
***CVX*** which is a MATLAB package for specifying and solving convex optimization problem can be executed through the *cvx_setup* command after installing the package provided in [[2]].
***fmincon*** is a built-in function provided by MATLAB and does not need to be installed separately.
Lastly, ***fastMPC***, an algorithm that fixes the barrier parameters and Newton steps of the interior-point method, can be used to solve the resultant constrained optimal control problem in real time [[3]].

##  Generating Time-varying Phase aberration
The time-varying phase aberration caused by atmospheric turbulence is generated by **Object-Oriented MATLAB & Adaptive Optics (OOMAO) Toolbox** [[4]].
The OOMAO sources are hosted by GitHub (https://github.com/rconan/OOMAO).
To illustrate a complex environment such as the real-world atmosphere, we designed a *mainly frozen-flow model* with three layers at diferent heights, wind speeds, and directions [[5]].

```matlab
clear;clc;close all;
addpath(genpath('.\OOMAO-master'))

samplingFreq = 200; % turbulence sampling frequency
nSimSteps = 2000; % total number of phase screen
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
nL = 128; % number of lenslet
nPx = 1; % number or pixel per lenslet
nRes = nL*nPx; % resolution
xaxis = 1:nRes;
d = D/nL; % lenslet pitch

tel = telescope(D,'resolution',nRes,'fieldOfViewInArcsec',2.5,'samplingTime',1/samplingFreq); % telescope
tel = tel + atm;

% V ( 0.550e-6 , 0.090e-6 , 3.3e12 ) : Photometric system where Band ( wavelength , bandwidth , zero point )
ngs = source('wavelength',photometry.V); % natural guide star
ngs = ngs.*tel;

sprintf('Calculating wavefront for %d steps ...',nSimSteps);

phase = []; time_phase = [];
for iSimStep=1:nSimSteps
    +ngs;
    +tel;
    phase(:,:,iSimStep) = ngs.meanRmPhase; % time-varying phase aberrations for each sampling time
    time_phase(iSimStep) = ngs.timeStamp;
end

% Fitting the Zernike coefficients constituting the time-varying phase aberrations
x = (-(nL-1):2:(nL-1))/(nL-1);
[X,Y] = meshgrid(x);
[theta,r] = cart2pol(X,Y);    % convert to polar coordinate
is_in = r <= max(abs(x));

r = r(is_in);
theta = theta(is_in);

N = 6; % order of zernike mode

ad_acc = []; 
for j=1:size(phase,3)
    z = squeeze(phase(:,:,j)); % Unit : [rad]
    [ad_new,nm_new] = zernmodfit(r,theta,z(is_in),N); % Modified Zernike Decomposition
    ad_acc = [ad_acc; ad_new(:,1)']; % Zernike coefficients constituting the time-varying phase
end
```

##  Model Identification based on Time-series method
The temporal dynamics of the Zernike coefficient ![equation](https://latex.codecogs.com/png.image?\dpi{110}%20x_t) constituting the time-varying phase aberration ![equation](https://latex.codecogs.com/png.image?\dpi{110}%20\phi_{\textrm{distort}}) can be represented by a vector-valued autoregressive (VAR) model of order N_v

![equation](https://latex.codecogs.com/png.image?\dpi{110}%20\begin{align*}x_t[k]%20=%20A_1%20x_t[k-1]%20+%20\cdots%20+%20A_{N_v}%20x_t[k-N_v]%20+%20w[k]\end{align*})

where ![equation](https://latex.codecogs.com/png.image?\dpi{110}%20k) represents the time index, ![equation](https://latex.codecogs.com/png.image?\dpi{110}%20A_{i}%20\in%20\mathbb{R}^{n\times%20n}) are coefficient matrices, and ![equation](https://latex.codecogs.com/png.image?\dpi{110}%20w[k]\sim\mathcal{N}(0,Q_w)) is the white Gaussian noise.

To determine the VAR model, any system identification method such as time-series, machine learning, and extrapolation can be used.
In this study, we designed an AR model based on a time-series method.
An open-loop wavefront dataset ![equation](https://latex.codecogs.com/png.image?\dpi{110}%20\{x_t[k]|k=1,\cdots,t_{\textrm{train}}%20\}) was used to identify the model parameters ![equation](https://latex.codecogs.com/gif.latex?%5Cinline%20A_i)

```matlab
num_data = size(ad_acc,1);

ad_acc(:,1) = []; % piston element is removed

num_train = 1000; % number of training set
num_valid = 500; % number of validation set
num_test = 500; % number of test set

PN = 2; % order of VAR model
AA = []; % model of training set
BB = [];

for i=PN+1:num_train
    for j=1:PN
        AA(i-PN,size(ad_acc,2)*(j-1)+1:size(ad_acc,2)*j) = ad_acc(i-j,:);
    end
    BB(i-PN,:) = ad_acc(i,:);
end

PARA = (AA'*AA)\AA'*BB;

A1 = (PARA(1:size(ad_acc,2),:))';
A2 = (PARA(1+size(ad_acc,2):2*size(ad_acc,2),:))';

BB_ts_train = AA*PARA;

% validation set
AA_valid = []; % model of validation set
BB_valid = [];
for i=1:num_test
    for j=1:PN
        AA_valid(i,size(ad_acc,2)*(j-1)+1:size(ad_acc,2)*j) = ad_acc(i+num_train-j,:);
    end
    BB_valid(i,:) = ad_acc(i+num_train,:);
end

BB_ts_valid = AA_valid*PARA;


ad_diff_test = BB_ts_valid - BB_valid;

RMSE_valid = []; RRMSE_valid = [];
for j=1:size(ad_acc,2)
    RMSE_valid = [RMSE_valid; sqrt(mean((BB_ts_valid(:,j) - BB_valid(:,j)).^2))];
    RRMSE_valid = [RRMSE_valid; RMSE_valid(j,1)/(max(BB_valid(:,j)) - min(BB_valid(:,j)))];
end

RMSE_valid = [0; RMSE_valid]; RRMSE_valid = [0; RRMSE_valid];

% Compare the zernike coefficients of validation set between original and generation via PARA
BB_valid = [zeros(size(BB_valid,1),1) BB_valid];
BB_ts_valid = [zeros(size(BB_ts_valid,1),1) BB_ts_valid];

BB_valid = [zeros(num_train,size(BB_valid,2)); BB_valid];
BB_ts_valid = [zeros(num_train,size(BB_ts_valid,2)); BB_ts_valid];

for i=1:size(ad_acc,2)
    j = i;
    figure(15)
    subplot(4,7,i)
    hold on
    plot(BB_ts_valid(:,j),'b-','LineWidth',1.5)
    plot(BB_valid(:,j),'r--','LineWidth',1.5)
    hold off
    axis square
    title(['\alpha_{k}(' num2str(j) ')'])
    xlabel('Time index, k')
    xlim([(num_train + 1) (num_train + num_test)])
    ylim([-2 2])
    set(gca,'FontSize',15)
end
legend('Predicted','Actual')
```

<img src="images/result1_validation.png" />

## The influence matrix of Deformable Mirror (DM) generation
The influence matrix for each actuators was designed as a Gaussian influence function: 
![equation](https://latex.codecogs.com/png.image?\dpi{110}%20I_j(\chi)%20=%20\textrm{exp}\left(\textrm{ln}(c)%20\displaystyle\frac{(\chi-\chi_{0,j})^2}{d^2}%20\right))

If the DM is operated as a modal method, the corrected wavefront and influence functions are decomposed into a set of Zernike polynomials:
![equation](https://latex.codecogs.com/png.image?\dpi{110}%20I_j(\chi)%20=%20\sum_{r=1}^{n}%20b_{r,j}%20Z_r(\chi)%20%20\Leftrightarrow%20%20%20\mathcal{I}%20=%20\mathcal{Z}B)

Therefore, the influence matrix can be obtained by the least-squares method ![equation](https://latex.codecogs.com/png.image?\dpi{110}%20B%20=%20{\mathcal{Z}}^{\dagger}%20{\mathcal{I}})

```matlab
dx = 6.5e-6; %0.1e-6;   % pixel spacing (m)

coupling = 0.1; % coupling parameter, defining the width of functions [0.05 ~ 0.15]
D = 4.4e-3; % telescope diameter, [m]
m1 = 12; % number of actuator array, m = (m1)^2
m = m1^2; % total number of actuators

n = 6; % order of zernike mode -> total 28 modes
nx = (n+1)*(n+2)/2;

d = D/(m1-1); % distance between actuators in pupil plane

len_dm = round((2.2e-3)*2/dx);
xaxis_dm = ((-len_dm/2):(len_dm/2-1))*dx;
yaxis_dm = -xaxis_dm;

[XX_dm,YY_dm] = meshgrid(xaxis_dm,yaxis_dm);  %2-D arrays hold x location and fy location of all points

diff_dm_idx = floor(len_dm/(m1-1));
x0_dm_idx = 1;
for i=1:(m1-1)
    x0_dm_idx = [x0_dm_idx; 1+i*diff_dm_idx];
end
x0_dm_idx(end) = len_dm;

x0_dm_axis = xaxis_dm(x0_dm_idx)';
y0_dm_axis = yaxis_dm(x0_dm_idx)';

act_idx = 0;
B_dm = zeros(len_dm,len_dm,m);
for i=1:m1
    for j=1:m1
        x_diff = XX_dm - x0_dm_axis(j);
        y_diff = YY_dm - y0_dm_axis(i);
        act_idx = act_idx + 1;
        
        B_g = 0 + 1*exp( log(coupling)*( ((x_diff).^2 + (y_diff).^2 )/(d^2) ) );
        
        B_dm(:,:,act_idx) = B_g;
    end
end

% pupil plane
len = 512; % number of pixels in the array
cen = len/2 + 1;
df = 1/(len*dx); % spacing in the spatial frequency domain (cycles/m)

xaxis = ((-len/2):(len/2-1))*dx;
yaxis = -xaxis;

[XX,YY] = meshgrid(xaxis,yaxis);  %2-D arrays hold x location and fy location of all points

N = len-1;
x = (-N:2:N)/N;
[X,Y] = meshgrid(x);
[theta,r] = cart2pol(X,Y);    % convert to polar coordinate
is_in = r <= max(abs(x));
% z(~is_in) = nan;
r = r(is_in);
theta = theta(is_in);

[t1,max_idx] = min(abs(xaxis_dm - xaxis(end)));
[t2,min_idx] = min(abs(xaxis_dm - xaxis(1)));

B_pupil = zeros(len,len,m); B_pupil_isin = zeros(size(r,1),m);
for t=1:size(B_dm,3)
    B_pupil(:,:,t) = B_dm(min_idx:max_idx,min_idx:max_idx,t);
    temp = B_pupil(:,:,t);
    B_pupil_isin(:,t) = temp(is_in);
end

% Fitted by Zernike polynomials
load("./Zs.mat") % load Zernike polynomials

Zs_new = reshape(Zs,size(Zs,1),len^2)'; 
B_pupil_new = reshape(B_pupil,len^2,size(B_pupil,3));

B = pinv(Zs_new'*Zs_new)*Zs_new'*B_pupil_new; % Influence matrix
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

A1 = A1'; % input_data에 저장된 A가 x*A 형태로 쓰였기 때문에 변환 필요
A2 = A2';

% piston element is removed
B(1,:) = []; A_s(:,1) = []; A_s_est(:,1) = []; ad_acc(:,1) = []; 
```
### Setup the simulation parameters
``` matlab
load("./model_approx.mat") % load the first-order approximation model of PSF
load("./SNR_10.mat") % load measurement noise (SNR = 10 [dB])
sample = 1;

mag_conv_5 = 1;
mag_conv_10 = 1.781797436291855;
mag_conv_15 = 2.498049532979032;
mag_conv_20 = 3.174802103932926;

phase = phase .* mag_conv_10;
ad_acc = ad_acc .* mag_conv_10;

% Plot the magnitude of time-varying phase aberration used in the simulation.
norm_ad = [];
for i=1:size(ad_acc,1)
    norm_ad(i,1) = norm(ad_acc(i,:));
end

x1 = (1:num_train)*T_s_tur;
x2 = (num_train:num_train+num_valid)*T_s_tur;
x3 = (num_train+num_valid:num_train+num_valid+num_test)*T_s_tur;

figure(51)
hold on
plot(x1,norm_ad(1:num_train,1),'r-','LineWidth',1.5)
plot(x2,norm_ad(num_train:num_train+num_valid,1),'b-','LineWidth',1.5)
plot(x3,norm_ad(num_train+num_valid:num_train+num_valid+num_test,1),'g-','LineWidth',1.5)
hold off
grid on
xlim([1 size(norm_ad,1)]*T_s_tur)
ylim([0 6])
xlabel('Time [sec]')
ylabel('Aberration Magnitde [rad]')
legend('Training data','Validation data','Test data')
set(gca,'FontSize',15)


B(1,:) = []; A_s(:,1) = []; % piston element is removed

nx = size(A1,1); % number of state
nu = size(B,2); % number of input
p = size(A_s,1); % number of measurements   

n_mode = 6; % radial order of ernike modes
N = 2; % prediction horizon for MPC
T_final = 500; % num_test = size(phase,3) - num_train - num_valid;
T_settle = 1.0e-3;
T_s = 0.1e-3; % sampling time of MPC Simulation

% Cost function : J = U'*H*U + r'*U + c
Q = (1.5e+4)*eye(nx,nx); % weighting matrix for error state
P = (1e+0)*Q; % weighting matrix for terminal error 
R = (1e+0)*eye(nu,nu);% weighting matrix for cost function

% SNR = 10; % the magnitude of desired SNR [dB]

coeff_a = 0.047275; coeff_b = 2.709264; coeff_c = 0; % Unit change parameters ( [V] → [nm] )

u_min = -28*ones(nu,1); % input box constraint [rad]
u_max = 28*ones(nu,1);  % 28 [rad] → 200 [V]

du_min = -0.2121*ones(nu,1); % ramp-rate constraint [rad]
du_max = 0.2121*ones(nu,1);

U_min = repmat(u_min,N,1); % input box constraint for vector representation
U_max = repmat(u_max,N,1);

dU_min = repmat(du_min,N,1); % ramp-rate constraint for vector representation
dU_max = repmat(du_max,N,1);
```


### Make pixel array for image plane and pupil plane
```matlab
% y_ref generation by FFT2
mag = 1; % FFT magnification
res = len*mag; % resolution of FFT 
dx = 6.5e-6; %0.1e-6;   % pixel spacing [m]
wavelength = 532.0e-9;   % [m]
unit_change = wavelength/(2*pi)*1e+9; % Unit conversion : [rad] to [nm]

xaxis_res = ((-res/2):(res/2-1))*dx/mag;
yaxis_res = -xaxis_res;

range_min = find(abs(xaxis_res-(-1.0e-4)) < 3e-6, 1, 'first');
range_max = find(abs(xaxis_res-(+1.0e-4)) < 3e-6, 1, 'last');
diff = (range_max - range_min + 1);
AU = 1e+12; % Arbitrary unit for point spread function

fxaxis = ((-len/2):(len/2-1))*df;
fyaxis = -fxaxis;
[FX,FY] = meshgrid(fxaxis,fyaxis);  % 2-D arrays hold fx location and fy location of all points
freq_rad = sqrt(FX.^2 + FY.^2);
maxfreq = (len/2-1)*df;

pupil_radius = 1.0*maxfreq; % NA / wavelength % radius of the pupil, inverse microns
pupil_area = pi*pupil_radius^2; % pupil area
pupil = double(freq_rad <= pupil_radius); % pin-hole

idx2 = 5; % Defocus (diversity function) index of Zernike polynomials

zd_dist = 3; % phase diversiy
zd_list = (-zd_dist:zd_dist:zd_dist); % [-3, 0, 3]
```

### MPC Simulation (Estimator + Controller)
First, we define one based on the Taylor series expansion around zero aberration:

![equation](https://latex.codecogs.com/png.image?\dpi{110}%20y_{i,j}%20=%20D_{0,j}(\beta_i)%20+%20D_{1,j}(\beta_i)\alpha%20+%20\mathcal{O}(\|\alpha\|^2)).

A vector representation of each pixel j for all phase diversities ![equation](https://latex.codecogs.com/png.image?\dpi{110}%20i=1,\cdots,n_p) is given by

![equation](https://latex.codecogs.com/png.image?\dpi{110}%20y%20=%20b_s%20+%20A_s\alpha).


Then, the estimate of ![equation](https://latex.codecogs.com/png.image?\dpi{110}%20\alpha) is obtained by least squares:

![equation](https://latex.codecogs.com/png.image?\dpi{110}%20\hat{\alpha}%20:=%20{A^{\dagger}_s}(y-b_s))


```matlab
% Design matrix generation for Linear MPC
[H, M1, M2, Q_tilda, R_tilda, B_conv, B_conv_pre1, B_conv_pre2, E] = MPC_DesignMatrices(A1,A2,B,nx,nu,N,Q,P,R);
closed_form_matrix = -0.5*(pinv(H'*H))*(H');

% MPC Simulation for T_final step
U_acc = []; U_v_acc = []; dU_acc = []; X_acc = []; X_acc_err = [];
X_err = zeros(T_final,N); X_est_err = []; X_err_low = []; X_err_high = [];
ad_est_acc = []; ad_cor_acc = []; J = [];
T_mldivide = zeros(T_final,1); T_pinv = zeros(T_final,1); T_lsqmin = zeros(T_final,1); T_geninv = zeros(T_final,1); T_cvx = zeros(T_final,1); T_fmincon = zeros(T_final,1); T_sim = zeros(T_final,1); 

phase_cor = zeros(len,len,T_final); % corrected aberration by DM
phase_res = zeros(len,len,T_final); % residual aberration (phase_res = phase + phase_cor)
u_prev = zeros(nu,1); Y_M_acc = [];

ad_acc_valid = ad_acc(1+num_train+num_test:end,:);
phase_valid = phase(:,:,1+num_train+num_test:end);

for iSimStep = 1:T_final % Simulation start (Estimator → Controller)
    tic
    if iSimStep == 1
        phase_res(:,:,iSimStep) = phase_valid(:,:,iSimStep);
    else
        % ramp constraint for first control input, u[0|k]
        dU_min(1:nu,1) = du_min + u_prev;
        dU_max(1:nu,1) = du_max + u_prev;
        
        phase_res(:,:,iSimStep) = phase_valid(:,:,iSimStep) + phase_cor(:,:,iSimStep-1);
    end
    
    % Estimator
    Y_M = []; Y_M_noise = [];
    
    scrn = phase_res(:,:,iSimStep); % phase screen, unit : [rad]

    for k=1:size(zd_list,2)
        zd_diversity = zd_list(k); % phase diversity value (-3, 0, 3)
                        
        kW = zd_diversity.*squeeze(Zs(idx2,:,:)); % diversity function
        P_defocus = pupil.*exp(1i*(scrn+kW));

        % transfer function
        I_defocus = fftshift(fft2(fftshift(P_defocus),res,res))*dx^2;
        im = abs(I_defocus).^2;
        v_im(:,:,k) = im(range_min:range_max,range_min:range_max)*AU;
        Y_M = [Y_M; reshape(v_im(:,:,k),(range_max-range_min+1)^2,1)];
    end
    Y_M_noise = squeeze(SNR_data(sample,iSimStep,:));
    
    Y_M = Y_M + Y_M_noise;
    Y_M_acc = [Y_M_acc; Y_M'];
    
    ad_est = lsqminnorm((A_s'*A_s),((A_s)'*(Y_M-b_s)));
    X_est_err = [X_est_err; norm(ad_est,2)];
    ad_est_acc = [ad_est_acc; ad_est']; % Zernike coefficient for residual aberration
    
    % initial condition for MPC Controller
    x0 = ad_est;
    if iSimStep == 1
        x0_pre = zeros(nx,1);
    else
        x0_pre = ad_est_acc(iSimStep-1,:)';
    end
    
    %% b_ref generation
    if iSimStep == 1
        b_ref = zeros(N*nx,1);
    elseif iSimStep == 2
        b_ref = -M1*B*U_acc(iSimStep-1,:)';
    else
        b_ref = -M1*B*U_acc(iSimStep-1,:)' -M2*B*U_acc(iSimStep-2,:)';
    end

    %% r, c generation
    r = 2*(B_conv)'*Q_tilda*(M1*x0 + M2*x0_pre + b_ref);
    c = (M1*x0 + M2*x0_pre + b_ref)'*Q_tilda*(M1*x0 + M2*x0_pre + b_ref); % constant
    
    %% Simulate MPC Opmizer (case_num : 1 = CVX, 2 = fmincon, 3 = fastMPC)
    switch case_num
        case 1
            %% Solving MPC via CVX
            tic
            cvx_precision best % cvx precision settings - low, medium, default, high, best options are exist.
            cvx_begin
            cvx_solver SDPT3 % solver option : sedumi, mosek
            variable U(nu*N,1)
            subject to
            %         F*U5 <= f; % linear constraints
            U <= U_max; % input box constraints
            U_min <= U;
            E*U <= dU_max; % ramp-rate constraints
            dU_min <= E*U;
            minimize( (U)'*H*(U) + r'*(U) + c )
            cvx_end
            T_cvx(iSimStep,1) = toc;

        case 2
            %% Solving MPC via fmincon
            fun = @(U6) (U6)'*H*(U6) + r'*(U6) + c; % objective function
            Aieq = []; bieq = []; Aeq = []; beq = []; lb = []; ub = []; 
            % lb = U_min; ub = U_max;
            U0 = zeros(N*nu,1);

            nonlcon = @mycon_uncon;

            options = optimoptions(@fmincon,'Algorithm','interior-point','MaxIterations',1e+4,'MaxFunctionEvaluations',1e+5);
            tic
            U = fmincon(fun,U0,Aieq,bieq,Aeq,beq,lb,ub,nonlcon,options);
            T_fmincon(iSimStep,1) = toc;

        case 3
            %% parameter setting
            Xmax = 100;                  % State upper limit (arbitrary value for fastMPC)
            x_min = -Xmax*ones(nx,1);     % State lower bound
            x_max = Xmax*ones(nx,1);      % State upper bound
            u_min = U_min(1:nu,1);     % Control lower bound
            u_max = U_max(1:nu,1);      % Control upper bound
            du_min = dU_min(1:nu,1); % Control ramp lower bound
            du_max = dU_max(1:nu,1); % Control ramp upper bound

            w = b_ref; % offset value for state equation
            xf = [];  % Terminal state
            test = Fast_MPC2(Q,R,[],Qf,[],[],[],x_min,x_max,u_min,u_max,du_min,du_max,N,x0,x0_pre,u_prev,A1,A2,B,w,xf,[]);   % Build class

            %% Solve by mpc_fixed_log_newton (fixed log + fixed newton)
            k_fix = 1e-2; % Fixed log barrier parameter (k=0.01)
            n_fix = 1; % Fixed newton step (n=1)

            tic;
            [x_opt_lgnw] = test.mpc_fixed_log_newton(n_fix,k_fix);
            T_lgnw(iSimStep,1) = toc;

            x_lgnw = zeros(N*nx,1);
            u_lgnw = zeros(N*nu,1);
            for i=1:(nu+nx):length(x_opt_lgnw)
                if i==1
                    u_lgnw(i:i+nu-1) = x_opt_lgnw(i:i+nu-1);
                    x_lgnw(i:i+nx-1) = x_opt_lgnw(i+nu:i+nu+nx-1);
                else
                    u_lgnw((i-1)/(nu+nx)*nu+1:(i-1)/(nu+nx)*nu+nu) = x_opt_lgnw(i:i+nu-1);
                    x_lgnw((i-1)/(nu+nx)*nx+1:(i-1)/(nu+nx)*nx+nx) = x_opt_lgnw(i+nu:i+nu+nx-1);
                end
            end

            U = u_lgnw;

        otherwise
            disp('other value')
    end
    
    %% Unit change for input variable : [rad] → [V]
    for i=1:size(U,1)
        if U(i,1) < 0
            U_v(i,1) = -( -coeff_b + sqrt((coeff_b)^2 - 4*coeff_a*U(i,1)*unit_change) )/(2*coeff_a);
        else
            U_v(i,1) = ( -coeff_b + sqrt((coeff_b)^2 + 4*coeff_a*U(i,1)*unit_change) )/(2*coeff_a);
        end
    end

    U_v_acc = [U_v_acc; U_v(1:nu)'];
    
    %% Phase correction by DM 
    J = [J; U'*H*U + r'*U + c]; % cost function
    u_prev = U(1:nu,1); % u_prev = U(nu*(N-1)+1:nu*N,1);
    ad_cor = B*u_prev;

    X_predicted = M1*x0 + M2*x0_pre + B_conv*U + b_ref;

    x_prev = X_predicted(1:nx,1);

    Zs_cor = zeros(nx,len,len);
    for j1=1:nx
        Zs_cor(j1,:,:) = ad_cor(j1,1).*squeeze(Zs(j1+1,:,:)); % piston element is removed
    end
    Zs_cor = squeeze(sum(Zs_cor,1));
    phase_cor(:,:,iSimStep) = Zs_cor;
    
    for i1=1:N
        X_err(iSimStep,i1) = norm((X_predicted((i1-1)*nx+1:i1*nx,1)),2);
    end

    X_err_low = [X_err_low; norm(x_prev(1:nx,1))];
    X_err_high = [X_err_high; norm(x_prev(nx+1:end,1))];


    if iSimStep == 1
        du_prev = u_prev - zeros(nu,1);
    else
        du_prev = u_prev - U_acc(iSimStep-1,:)';
    end

    ad_cor_acc = [ad_cor_acc; ad_cor'];
    U_acc = [U_acc; u_prev'];
    dU_acc = [dU_acc; du_prev'];

    X_acc = [X_acc; x_prev'];
    X_acc_err = [X_acc_err; norm(x_prev)];

    T_sim(iSimStep,1) = toc;
    disp(iSimStep)
```




## References
[[1]] Jinsung Kim, Kye-Sung Lee, Sang-Chul Lee, Ji Yong Bae, Dong Uk Kim, I Jong Kim, Ki Soo Chang and Hwan Hur. "Model Model Predictive Control of Time-varying Aberrations for Sensorless Adaptive Optics", arXiv preprint. [https://arxiv.org/abs/--](https://arxiv.org/abs/--)

[[2]] Michael Grant and Stephen Boyd. CVX: Matlab software for disciplined convex programming, version 2.0 beta. http://cvxr.com/cvx, September 2013.

[[3]] Wang, Yang, and Stephen Boyd. "Fast model predictive control using online optimization." IEEE Transactions on control systems technology 18.2 (2009): 267-278.

[[4]] R. Conan and C. Correia, “Object-oriented Matlab adaptive optics toolbox,” in Adaptive Optics Systems IV (E. Marchetti, L. M. Close, and J.-P. Vran, eds.), vol. 9148 of Society of Photo-Optical Instrumentation Engineers (SPIE) Conference Series, p. 91486C, Aug. 2014.

[[5]] L. Prengere, C. Kulcs´ar, and H.-F. Raynaud, “Zonal-based highperformance control in adaptive optics systems with application to astronomy and satellite tracking,” J. Opt. Soc. Am. A, vol. 37, pp. 1083–1099, Jul 2020.

[1]: https://arxiv.org/abs/--
[2]: http://cvxr.com
[3]:
[4]: https://www.spiedigitallibrary.org/conference-proceedings-of-spie/9148/91486C/Object-oriented-Matlab-adaptive-optics-toolbox/10.1117/12.2054470.short
[5]: https://www.osapublishing.org/josaa/abstract.cfm?uri=josaa-37-7-1083
