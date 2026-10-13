# Real-Test | 8 types of multicarrier modulation identification dataset generation? Principle + complete MATLAB process breakdown [with MATLAB code]

[Multicarrier Modulation Identification Practical Application]
Real-Test | How to Generate 8 Multicarrier Modulation Identification Data Sets? Principle + Complete MATLAB Process Dissection for Automated Modulation Identification (AMC)
Friends who are into automatic modulation identification (AMC) often fall into this pit: the accuracy of their model on public datasets is 95%, but when they switch to their own collected data, it drops directly to 60%.
The problem often lies not in the model structure, but in your ignorance of the source of your training data.
How are public datasets generated? What channels are used? What non-ideal factors are added? What parameters are for each waveform? If you can't answer these questions, even if you adjust many network structures, you are just "fitting other people's datasets," and it immediately fails in a new environment.
This article fully disassembles the entire process of generating datasets for 8 multicarrier waveforms (OFDM, FBMC, UFMC, FOFDM, WOLA, GFDM, OTFS, AFDM), from the principle of waveforms, channel modeling to MATLAB core code fragments, and explains every step thoroughly. After reading, you will at least avoid 80% of dataset generation pitfalls.
I. Why Do We Need a Multicarrier Modulation Identification Dataset?
In the era of 5G/6G, there are a large number of multicarrier waveforms coexisting in the spectrum: OFDM is the absolute mainstay of 4G/5G, FBMC, UFMC, FOFDM, WOLA are candidates for the new air interface of 5G, OTFS focuses on high Doppler scenarios, AFDM is a chirp domain waveform that became popular after 2022, and GFDM has its place in non-orthogonal transmission.
Spectrum regulation, interference recognition, cognitive radio, specific radiation source identification (SEI), all rely on a prerequisite ability: recognizing what kind of waveform the counterpart uses. And the first step in training an identification model is to have a "waveform variety comprehensive, channel enough real, parameter reproducible" dataset. This dataset must meet three conditions:
1. The waveform variety covers mainstream multicarrier schemes, so the model can learn "inter-class differences";
2.
1. Channels contain factors such as fading, Doppler, frequency offset, and time offset that are not ideal; the model should not only focus on AWGN.
2. Diversity in subcarrier modulation (QPSK/16QAM/64QAM) is adopted to avoid treating "density of constellation points" as a clue for distinguishing waveforms.
I have written a script in MATLAB to generate an output file named MWR_all_AMC_diffmod.mat, which will be explained step by step.

II. Understanding the principles of 8 waveforms in 30 seconds:
2.1 Frequency domain orthogonal class: OFDM and its "relatives"
OFDM divides a broadband channel into narrowband subcarriers, with independent modulation symbols on each subcarrier:
To combat inter-symbol interference (ISI), a cyclic prefix (CP) is inserted before each symbol, at the cost of approximately 7% to 10% spectrum efficiency loss.
FBMC (Filter Bank Multicarrier): Uses prototype filters (such as PHYDYAS, overlap factor K=4) to shape each subcarrier, allowing adjacent subcarriers to overlap in spectrum without the need for CP alignment, thus achieving higher spectrum efficiency; however, it comes with high complexity and implementation difficulties.
UFMC (Universal Filter Multicarrier): No longer performs filtering on each subcarrier but instead groups subcarriers into subbands, performing global filtering on these subbands, offering lower complexity than FBMC but also eliminating the need for CP alignment;
FOFDM (Filtered OFDM): Adds filtering to the entire signal block on top of OFDM, suppressing out-of-band leakage;
WOLA (Windowed Overlapping Addition): Adds windows to OFDM symbols and performs overlapping addition, using window functions to suppress sidelobes.
In summary, the key points to remember are: FBMC filters on individual subcarriers, UFMC filters on subbands, FOFDM filters globally, and WOLA adds windows.
2.2 Time-delay-Doppler domain class: OTFSFOTFS directly modulates symbols in the time-delay-Doppler (TDD) domain, mapping them through an inverse Singular Fourier Transform (ISFFT) to the time-frequency domain, then through a Heisenberg transformation to be transmitted.
In high-mobility scenarios (such as high-speed trains and drones), the channel is quasi-statically sparse in the time-delay-Doppler domain, making OTFS naturally resistant to Doppler spread, which is a core selling point distinguishing it from OFDM.
2.3 Chirp domain class: AFDMAFDM is based on Discrete Affine Fourier Transform (DAFT), essentially a chirp-modulated Fourier transform:
It consists of a DFT matrix, a chirp phase diagonal matrix that satisfies the orthogonality condition. It is similar to OTFS in its suitability for dual selective channels, but with a closer "transform + CP" structure to OFDM.
2.4 Non-orthogonal class: GFDMGFDM uses cyclic filters to pulse-shape the time-frequency grid, with subcarriers × subsymbols per block, where subcarriers can be non-orthogonal, offering high flexibility:
However, it requires higher complexity at the receiver end for equalization.
III. Data Set Design: Simulation parameters determine the dataset's upper limit. The quality of a dataset largely depends on eight key parameters. The most critical groups of parameters in the script include:
Parameter category  
Value  
Description  
Sampling rate  
15e3 × 256  
Needs to be greater than the number of subcarriers × subcarrier interval  
SNR range  
-18 : 2 : 20 dB  
Total of 20 SNR points  
Number of samples per SNR point  
1800  
Monte Carlo repeat  
Subcarrier modulation  
QPSK / 16QAM / 64QAM  
Each 600 frames, evenly distributed  
Channel model  
VehicularA  
Vehicle-based multipath channel  
Moving speed  
60 km/h  
Jakes Doppler spectrum  
Carrier frequency  
1 GHz  
Corresponding maximum Doppler around 55 Hz  
Normalized frequency offset  
0.4  
Relative sampling rate's frequency offset  
Phase noise line width  
200 kHz  
IMD non-idealness  
Normalized time offset  
0.05  
Relative subcarrier interval
Here lies a key design: subcarrier modulation is evenly mixed between QPSK/16QAM/64QAM. Without this step, the model may infer waveforms indirectly through "number of constellation points," leading to an overly high recognition accuracy—that is, not identifying the capability for waveform recognition but rather the ability to classify modulation orders.
IV. Core Code Snippets: Every step is a pitfall 4.1 AFDM modulation: chirp phase + DFTfunction [S] = AFDM_modulation(X, c1, c2, cp_len)% AFDM modulation based on DAFT (discrete affine Fourier transform)N = size(X, 1);% chirp phase diagonal matricesL1 = diag(exp(-1j*2*pi*c1*((0:N-1).2)));L2 = diag(exp(-1j*2*pi*c2*((0:N-1).2)));F = dftmtx(N)/sqrt(N);IA = L1'*F'*L2';     % DAFT matrixS = IA*X;        % transform-domain modulationif cp_len ~= 0  n = -cp_len:1:-1;  Ro = exp(-1j.*2.*pi.*c1.*(N^2+2*N*n));% chirp-periodic extension  Ro = repmat(Ro, size(X, 2), 1)'; S = [S(end-cp_len+1:end, :).*Ro; S]; % chirp CPendS = S(:); End Rather, it is chirp period expansion. Use S(end-cp_len+1:end). :) Extract and paste it together. When receiving end performs DAFT, it will destroy the orthogonality. Performance plummets under high Doppler conditions.A phase compensation must be made by multiplying with Ro = e{-j2\pi c_1(N2+2Nn)}.
Point 2: The selection of the angles must meet the orthogonality condition. In the script, it calculates using maximum Doppler:
maxDoppler = maxVel/(physconst("lightspeed")/Channel_CarrierFrequency) ...      * 2/SamplingRate*AFDM_NumberOfSubcarriers;c1_AFDM = (2*(maxDoppler + AFDM_guardWidth) + 1)/(2*AFDM_NumberOfSubcarriers);c2_AFDM = 1/(AFDM_NumberOfSubcarriers^2*pi);% Small irrational numbers
Notes specifically left two lines of "bookkeeping values" that are commented out—those sets of values work fine in an ideal channel, but reveal themselves when Doppler shift is added.
4.2 OTFS modulation: ISFFT + Heisenbergfunction y = OTFS_modulation(N, M, x, padlen, varargin)% OTFS: ISFFT + Heisenberg transform% x : M x N delay-Doppler symbols (M subcarriers, N symbols)X = fft(ifft(x).').'/sqrt(M/N); % ISFFTy = ifft(X.')*sqrt(M);% Heisenberg transformy = y(:); % CP / ZP, choose one. Four variants available switch varargin{1} % 'CP' / 'ZP' / 'RZP' / 'RCP' / 'NONE'  case 'CP'    y = [y(end-padlen+1:end, :); y]; y = y(:); case 'ZP'    y = [y; zeros(padlen, N)]; y = y(:); % ...endend\nPoint 3: The CP/ZP of OTFS is applied to each column (each symbol) rather than at the end of the frame. In the code, `y` after Heisenberg transformation is a matrix, and `[y(end-padlen+1:end,:); y]` is for adding CP to each column before serialization. If the dimensions are mismatched, the delay spread will directly penetrate into adjacent symbols.
4.3 Channel and Damage Injection: Making the dataset "dirty" for a real receiver, it faces not an ideal signal but a mixture of "multipath + Doppler + Frequency offset + Time offset + Noise". The generation script is layered in sequence:
% 1) Frequency offset: Normalized frequency offset is 0.4, relative sampling rate t = (0:N_sig-1)/fs; freq_offset_signal = s_moded * exp(1j * 2 * pi * freq_offset * SubcarrierSpacing * t);
% 2) Time offset: Random time delay between 0 and 0.05/ΔfM_NormalizedTimeOffset_Samples = round((0.05/SubcarrierSpacing)/dt);M_NormalizedTimeOffset_Samples = randi(M_NormalizedTimeOffset_Samples);time_offset_signal = [zeros(M_NormalizedTimeOffset_Samples,1); ...           freq_offset_signal(1:end-M_NormalizedTimeOffset_Samples)];% 3) VehicularA multipath channel (Jakes Doppler, 60km/h)r_noNoise = ChannelModel.Convolution(time_offset_signal); % 4) Noise Addition: Pn_time = 1/OFDM.GetSymbolNoisePower(1)*10^(-SNR_OFDM_dB/10); noise = sqrt(Pn_time/2)*(randn(N, 1)+1j*randn(N, 1));rxSig = r_noNoise + noise(1:N_sig);
Issue 4: The expression for frequency offset injection is freq_offset * SubcarrierSpacing. Here, freq_offset is the normalized frequency offset (0.4), which needs to be multiplied by the subcarrier spacing to obtain the absolute frequency offset. Many people only multiply by the sampling rate, leading to a significant increase in frequency offset, making it completely unrecognizable for different waveforms.
Point 5: Each sample is regenerated to implement a channel. The `ChannelModel.NewRealization` must be called within the `i_rep` loop. Move it outside the loop, and all 1800 samples will use the same channel implementation—the model directly learns "backchannel".
4.4 Data Standardization: The final step, the most prone to errors temp_gen_data = (temp_gen_data - mean(temp_gen_data,'all')) ... ./ std(temp_gen_data,0,'all');
Challenge 6: Standardization must be done per SNR, not all at once. Variance is small for high SNR samples and large for low SNR samples. If global statistics are used, it leaks SNR information to the model—the "noise resistance" evaluated is entirely false.
4.5 Data format: 1024 sample points per frame, with two channels for real and imaginary parts, data format:
% Monotonic Modulation Type: snr_num × 1800 × 2 × 1024% dim1: SNR points (-18~20 dB, Total 20% % dim2: Monte Carlo repeat (1800 frames, QPSK/16QAM/64QAM each 600% dim3: channel (1=real part) 2=Imaginary part)% dim4: Sample points (1024)\n8 types of modulation are stored separately as xAFDM. xGFDM,  xFOFDM,  xUFMC,  xFBMC,  xOTFS,  xOFDM,  xWOLA,  The final write is written to MWR_all_AMC_diffmod.mat.
5. How to interpret the simulation results? After completion, the script will draw a 3x6 comparison chart of the real parts, imaginary parts, and envelopes for 6 waveform types. From the time-domain waveforms, it is evident:
OFDM/WOLA/FOFDM waveform amplitudes are relatively stable, with OFDM being a typical Gaussian-like envelope;
FBMC due to the overlapping filter groups and adjacent symbols in the time domain, the envelope is "rougher";
OTFS/AFDM after unfolding in the time-delay-Doppler domain, the time domain presents an approximate uniform distribution characteristic;
GFDM due to the roll-off filter (roll-off=0.1), there is a transition band at the edge of the block. These temporal differences are the "fingerprints" for deep learning models in waveform recognition. The simulation diagrams are recommended to be directly saved as PNG files and inserted into the article for a more intuitive display.
The CP of AFDM is chirp period extension, not a regular cyclic prefix, remember to multiply the phase compensation term;
2  
When calculating maximum Doppler, do not use the "universal value" from textbooks;
3  
OTFS's CP/ZP should be added to each column, first reshaped and then filled, do not fill the entire sequence;
4  
Frequency offset should be multiplied by the subcarrier interval to get the absolute frequency offset;
5  
Each sample should be regenerated with the channel implementation again, otherwise the model will be backscattered through the channel;
6  
Normalization should be done step by step according to SNR, to prevent SNR information leakage into features.
Next issue preview: [Multi-Carrier Modulation Identification Real-Time] Time-frequency diagram + Deep Learning identification model how to build? Looking forward to your follow-up updates.

## Images

![Measured | 8 types of multicarrier modulation identification datasets how to generate? Principle + complete MATLAB process breakdown [attached MATLAB code]](01.jpg)
![Measured | 8 types of multicarrier modulation identification datasets how to generate? Principle + complete MATLAB process breakdown [attached MATLAB code]](02.jpg)
![Measured | 8 types of multicarrier modulation identification datasets how to generate? Principle + complete MATLAB process breakdown [attached MATLAB code]](03.jpg)
![Measured | 8 types of multicarrier modulation identification datasets how to generate? Principle + complete MATLAB process breakdown [attached MATLAB code]](04.jpg)

<!-- PAYMENT_BLOCK -->

## Get this dataset

- Payment link (Stripe): [Buy now](https://buy.stripe.com/3cs8yP7sY87d0vu9AB)
- Payment QR code: ![Payment QR code](pay.png)

<!-- /PAYMENT_BLOCK -->
