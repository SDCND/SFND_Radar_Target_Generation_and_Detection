# SFND Radar Target Generation and Detection

## Implementation steps for the 2D CFAR process

* Select the number of Training Cells in both the dimensions:
```
Tr = 10;
Td = 8;
```
* Select the number of Guard Cells in both dimensions around the Cell under test (CUT) for accurate estimation
```
Gr = 4;
Gd = 4;
```
* Offset the threshold by SNR value in dB:
```
offset = 1.4;
```
* Design a loop such that it slides the CUT across range doppler map by giving margins at the edges for Training and Guard Cells. For every iteration sum the signal level within all the training cells. To sum convert the value from logarithmic to linear using db2pow function. Average the summed values for all of the training cells used. After averaging convert it back to logarithimic using pow2db. Further add the offset to it to determine the threshold. Next, compare the signal under CUT with this threshold. If the CUT level > threshold assign it a value of 1, else equate it to 0. Use RDM[x,y] as the matrix from the output of 2D FFT for implementing CFAR:
```
for i = Tr + Gr + 1 : (Nr / 2) - (Gr + Tr)
    
    for j = Td + Gd + 1 : Nd - (Gd + Td)
        
        noise_level = zeros(1, 1);
        
        for p = i - (Tr + Gr) : i + Tr + Gr
            
            for q = j - (Td + Gd) : j + Td + Gd
                
                if(abs(i - p) > Gr || abs(j - q) > Gd)
                    
                    noise_level = noise_level + db2pow(RDM(p, q));
                    
                end
                
            end
            
        end
        
        threshold = pow2db(noise_level / (2 * (Td + Gd + 1) * 2 * (Tr + Gr + 1) - (Gr * Gd) - 1));
        threshold = threshold + offset;
                    
        CUT = RDM(i, j);
        
        if(CUT < threshold)
            
            RDM(i, j) = 0;
            
        else 
            
            RDM(i, j)=1;
            
        end
        
    end
    
end
```
* The process above will generate a thresholded block, which is smaller than the Range Doppler Map as the CUT cannot be located at the edges of matrix. Hence,few cells will not be thresholded. To keep the map size same set those values to 0:
```
RDM(union(1 : (Tr + Gr), end - (Tr + Gr - 1) : end), :) = 0;
RDM(:, union(1 : (Td + Gd), end - (Td + Gd - 1) : end)) = 0; 
```
* Display the CFAR output using the Surf function like we did for Range Doppler Response output:
```
figure('Name','CA-CFAR (filtered RDM)'), surf(doppler_axis, range_axis, RDM);
title('CFAR RDM');
xlabel('Velocity');
ylabel('Range');
zlabel('Amplitude');
colorbar;
```

## Selection of Training, Guard cells and offset

The number of training cells is decided based on the environment. In a dense traffic scenario, fewer training cells should be used as closely spaced targets can impact the noise estimate.
* Number of Training Cells in range dimension:
```
Tr = 10;
```
* Number of Training Cells in doppler dimension:
```
Td = 8;
```
The purpose of the guard cells is to avoid the target signal from leaking into the training cells, which can adversely affect the noise estimate. The number of guards cells is decided based on the leakage of the target signal out of the cell and the test. If the cell target size is large, they often get into surrounding bins.
* Number of Guard Cells in range dimension:
```
Gr = 4;
```
* Number of Guard Cells in doppler dimension:
```
Gd = 4;
```
The offset value is used to scale the noise threshold. Principle of the offset selection: If the signal strength is defined in logarithmic form then add this offset value to the average noise estimate, else multiply it.
* Offset the threshold by SNR value in dB:
```
offset = 1.4;
```

## Steps taken to suppress the non-thresholded cells at the edges

The process above will generate a thresholded block, which is smaller than the Range Doppler Map as the CUT cannot be located at the edges of matrix. Hence,few cells will not be thresholded. To keep the map size same set those values to 0. By sliding through the RDM matrix, and checking for value in each cell; then if value is different from 0 or 1, then assign it the value 0.
* Rows:
```
RDM(union(1 : (Tr + Gr), end - (Tr + Gr - 1) : end), :) = 0;
```
* Columns:
```
RDM(:, union(1 : (Td + Gd), end - (Td + Gd - 1) : end)) = 0; 
```

## Scripting

```
clear all
clc;

%% Radar Specifications 
%%%%%%%%%%%%%%%%%%%%%%%%%%%
% Frequency of operation = 77GHz
fc   = 77e9;
% Max Range = 200m
Rmax = 200;
% Range Resolution = 1 m
Rres = 1;
% Max Velocity = 100 m/s
Vmax = 100;
% Light Velocity = 299,792,458 m/s
c    = 299792458;
%%%%%%%%%%%%%%%%%%%%%%%%%%%

%speed of light = 3e8
%% User Defined Range and Velocity of target
% *%TODO* :
% define the target's initial position and velocity. Note : Velocity
% remains contant
R = 110;
v = -20;

%% FMCW Waveform Generation

% *%TODO* :
%Design the FMCW waveform by giving the specs of each of its parameters.
% Calculate the Bandwidth (B), Chirp Time (Tchirp) and Slope (slope) of the FMCW
% chirp using the requirements above.
Tchirp    = 5.5 * 2 * Rmax / c; 
Bandwidth = c / (2 * Rres);
Slope     = Bandwidth / Tchirp;

%Operating carrier frequency of Radar 
%fc = 77e9;             %carrier freq
                                                          
%The number of chirps in one sequence. Its ideal to have 2^ value for the ease of running the FFT
%for Doppler Estimation. 
Nd = 128;                   % #of doppler cells OR #of sent periods % number of chirps

%The number of samples on each chirp. 
Nr = 1024;                  %for length of time OR # of range cells

% Timestamp for running the displacement scenario for every sample on each
% chirp
t = linspace(0, Nd * Tchirp ,Nr * Nd); %total time for samples

%Creating the vectors for Tx, Rx and Mix based on the total samples input.
Tx  = zeros(1, length(t)); %transmitted signal
Rx  = zeros(1, length(t)); %received signal
Mix = zeros(1, length(t)); %beat signal

%Similar vectors for range_covered and time delay.
r_t = zeros(1,length(t));
td  = zeros(1,length(t));

%% Signal generation and Moving Target simulation
% Running the radar scenario over the time. 

for i = 1:length(t)         
    
    % *%TODO* :
    %For each time stamp update the Range of the Target for constant velocity. 
    r_t(i) = R + v * t(i);
    td(i)  = r_t(i) * 2 / c;
    
    % *%TODO* :
    %For each time sample we need update the transmitted and
    %received signal. 
    Tx(i) = cos(2 * pi * (fc * t(i) + 0.5 * Slope * t(i)^2));
    Rx(i) = cos(2 * pi * (fc * (t(i) - td(i)) + 0.5 * Slope * (t(i) - td(i))^2));
    
    % *%TODO* :
    %Now by mixing the Transmit and Receive generate the beat signal
    %This is done by element wise matrix multiplication of Transmit and
    %Receiver Signal
    Mix(i) =  Tx(i).* Rx(i);
    
end

%% RANGE MEASUREMENT

 % *%TODO* :
%reshape the vector into Nr*Nd array. Nr and Nd here would also define the size of
%Range and Doppler FFT respectively.
Mix = reshape(Mix, [Nr, Nd]);

 % *%TODO* :
%run the FFT on the beat signal along the range bins dimension (Nr) and
%normalize.
sig_fft1 = fft(Mix, Nr);
sig_fft1 = sig_fft1./ Nr;

 % *%TODO* :
% Take the absolute value of FFT output
sig_fft1 = abs(sig_fft1);

 % *%TODO* :
% Output of FFT is double sided signal, but we are interested in only one side of the spectrum.
% Hence we throw out half of the samples.
sig_fft1 = sig_fft1(1 : Nr / 2);

%plotting the range
figure ('Name','Range from First FFT')
%subplot(2,1,1)

 % *%TODO* :
 % plot FFT output 
plot(sig_fft1);
title('FFT1');
ylabel('Amplitude');
xlabel('Range');
axis ([0 200 0 1]);

%% RANGE DOPPLER RESPONSE
% The 2D FFT implementation is already provided here. This will run a 2DFFT
% on the mixed signal (beat signal) output and generate a range doppler
% map.You will implement CFAR on the generated RDM


% Range Doppler Map Generation.

% The output of the 2D FFT is an image that has reponse in the range and
% doppler FFT bins. So, it is important to convert the axis from bin sizes
% to range and doppler based on their Max values.

Mix=reshape(Mix,[Nr, Nd]);

% 2D FFT using the FFT size for both dimensions.
sig_fft2 = fft2(Mix, Nr, Nd);

% Taking just one side of signal from Range dimension.
sig_fft2 = sig_fft2(1 : Nr / 2, 1 : Nd);
sig_fft2 = fftshift(sig_fft2);

RDM = abs(sig_fft2);
RDM = 10 * log10(RDM);

%use the surf function to plot the output of 2DFFT and to show axis in both
%dimensions
doppler_axis = linspace(-100, 100, Nd);
range_axis = linspace(-200, 200, Nr / 2) * ((Nr / 2) / 400);

figure('Name','Range Estimation from Second FFT'), surf(doppler_axis, range_axis, RDM);
title('FFT2');
ylabel('Velocity');
xlabel('Range');
zlabel('Amplitude');

%% CFAR implementation

%Slide Window through the complete Range Doppler Map

% *%TODO* :
%Select the number of Training Cells in both the dimensions.
Tr = 10;
Td = 8;

% *%TODO* :
%Select the number of Guard Cells in both dimensions around the Cell under 
%test (CUT) for accurate estimation
Gr = 4;
Gd = 4;

% *%TODO* :
% offset the threshold by SNR value in dB
offset = 1.4;

% *%TODO* :
%Create a vector to store noise_level for each iteration on training cells
%noise_level = zeros(1,1);

% *%TODO* :
%design a loop such that it slides the CUT across range doppler map by
%giving margins at the edges for Training and Guard Cells.
%For every iteration sum the signal level within all the training
%cells. To sum convert the value from logarithmic to linear using db2pow
%function. Average the summed values for all of the training
%cells used. After averaging convert it back to logarithimic using pow2db.
%Further add the offset to it to determine the threshold. Next, compare the
%signal under CUT with this threshold. If the CUT level > threshold assign
%it a value of 1, else equate it to 0.


   % Use RDM[x,y] as the matrix from the output of 2D FFT for implementing
   % CFAR

RDM = RDM / max(max(RDM));

for i = Tr + Gr + 1 : (Nr / 2) - (Gr + Tr)
    
    for j = Td + Gd + 1 : Nd - (Gd + Td)
        
        noise_level = zeros(1, 1);
        
        for p = i - (Tr + Gr) : i + Tr + Gr
            
            for q = j - (Td + Gd) : j + Td + Gd
                
                if(abs(i - p) > Gr || abs(j - q) > Gd)
                    
                    noise_level = noise_level + db2pow(RDM(p, q));
                    
                end
                
            end
            
        end
        
        threshold = pow2db(noise_level / (2 * (Td + Gd + 1) * 2 * (Tr + Gr + 1) - (Gr * Gd) - 1));
        threshold = threshold + offset;
                    
        CUT = RDM(i, j);
        
        if(CUT < threshold)
            
            RDM(i, j) = 0;
            
        else 
            
            RDM(i, j)=1;
            
        end
        
    end
    
end

% *%TODO* :
% The process above will generate a thresholded block, which is smaller 
%than the Range Doppler Map as the CUT cannot be located at the edges of
%matrix. Hence,few cells will not be thresholded. To keep the map size same
% set those values to 0. 
 
RDM(union(1 : (Tr + Gr), end - (Tr + Gr - 1) : end), :) = 0;
RDM(:, union(1 : (Td + Gd), end - (Td + Gd - 1) : end)) = 0; 

% *%TODO* :
%display the CFAR output using the Surf function like we did for Range
%Doppler Response output.
figure('Name','CA-CFAR (filtered RDM)'), surf(doppler_axis, range_axis, RDM);
title('CFAR RDM');
xlabel('Velocity');
ylabel('Range');
zlabel('Amplitude');
colorbar;
```

## Output Plots

**FFT1**
<p align = "center">
  <img src="results/FFT1.png" width="562" height="506">
</p>

**FFT2**
<p align = "center">
  <img src="results/FFT2.png" width="562" height="506">
</p>

**CFAR RDM**
<p align = "center">
  <img src="results/CFAR.png" width="562" height="506">
</p>
