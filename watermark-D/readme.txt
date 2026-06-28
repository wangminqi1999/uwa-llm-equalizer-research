Watermark Version 1 distribution
30.11.2016, Paul van Walree <paul.vanwalree@ffi.no>

This text file is a mini user guide for Watermark. The official user manual is issued as FFI-rapport 2016/01378, which will become available via www.ffi.no. It is currently in the signature stage. The UCOMMS conference paper [1] can be regarded as an introduction. A post-UCOMMS development is array functionality, made possible by vertical line array (VLA) data provided by Telecom Bretagne [2] and Scripps Scripps Institution of Oceanography.


Avialable channels:

NOF1 - Norway Oslofjord 1. Play time 33 minutes; 10-18 kHz; single hydrophone (SISO). 
NCS1 - Norway Continental Shelf 1. Play time 33 minutes; 10-18 kHz; single hydrophone (SISO).
BCH1 - Brest Commercial Harbour 1. Play time 1 minute; 32.5-37.5 kHz; VLA with 4 hydrophones (SIMO).
KAU1 - Kauai 1. Play time 33 seconds; 4-8 kHz; VLA with 16 hydrophones (SIMO).
KAU2 - Kauai 2. Play time 33 seconds; 4-8 kHz; VLA with 16 hydrophones (SIMO).

The archive of a given channel consists of a collection of so-called channel files. A channel file is a .mat file with a time-varying impulse response (TVIR) estimate. The collection represents consecutive TVIR measurements in the SISO case, and a single TVIR measurement on consecutive hydrophones in the SIMO case.

User instructions for Watermark V1.0, after unzipping the zip file:

1. Prepare a communication signal and save a .mat file in \Watermark\input\signals\ containing:

x         signal time series in passband
fs_x      sampling rate (Hz)
nBits     number of information bits
  
The name of the mat file will be used as a signal identifier throughout the benchmarking. Suitable choices are for instance 'my_signal.mat', 'dsss4.mat', 'ofdm.mat'.

Make the signal as compact as possible by avoiding unnecessary samples such as trailing zeroes. Such samples lower the effective bit rate and may reduce the number of packets in simulation. Excessively high sampling rates unnecessarily increase prosessing time and disk space requirements, because the Watermark output is provided at the same sampling rate (fs_x) as the input signal. 

It is the responsibility of the user to provide a signal in the frequency band of a given test channel (see channel description above). Signal power outside this band will be strongly attenuated.

2. Go to \Watermark\matlab\ and pass your signal through a replay channel, e.g.

watermark('ofdm', 'NOF1', 'all');  

to process the entire NOF1 channel archive, or, for rapid testing,

watermark('ofdm', 'NOF1', 'single'); 

to process a single channel file. Make a note of the total number of simulated packets. 

3. Use the functions sfetch (serial fetch) and pfetch (parallel fetch) to retrieve packets. Serial fetch is used for NOF1 and NCS1, parallel fetch for the array data of BCH1, KAU1, and KAU2. Examples:

[y, fs] = sfetch('ofdm', 'NOF1', 1);
returns a column vector y corresponding to the first OFDM packet received in NOF1. 

[y, fs] = pfetch('ofdm', 'KAU1', 1);
returns a 16-column matrix y representing the first OFDM packet received on the 16 elements of the array. 

The sfetch and pfetch functions can be built into a receiver batch processing loop. Averaging error rates, output SNRs, etc. should be done by the user. 

See the help text in the .m files for more information, and make sure you read FFI-rapport 2016/01378 when it is published.


[1] P. van Walree, R. Otnes, and T. Jenserud, “Watermark: A realistic benchmark for underwater acoustic modems,” in 2016 IEEE Third Underwater Communications and Networking Conference (UComms), Aug 2016, pp. 1–4.
[2] F.-X. Socheleau, A. Pottier, and C. Laot, “Watermark: BCH1 dataset description,” Institut Mines-Telecom; TELECOM Bretagne, UMR CNRS 6285 Lab-STICC, Research report 17331, hal-01404491, 2016.
