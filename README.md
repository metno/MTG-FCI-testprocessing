# TEST processing of MTG FCI TEST dataset

![TEST Processing overview](okd-satellite-MTG-FCI-TEST-processing-ppi.svg)

## Running the test.
A simple commandline together with the `at` command and a list of the files is used to simulate the reception of the data. Each slot ( every 10 minute) starts a sequence of scp commans every 11 second. This is approximately the rate the segment( if 41 segments) will be received in.

```
delay_slot=0;while read p; do echo "$delay_slot $p";at now +$delay_slot minutes <<END
/software/pytroll/scripts/start-mtg-test.sh "$p"
END
delay_slot=$((delay_slot+10));done </software/pytroll/scripts/test-run-mtg.sh
```

Where `/software/pytroll/scripts/test-run-mtg.sh` is a list of all the slots:
```
W_XX-EUMETSAT-Darmstadt\,IMG+SAT\,MTI1+FCI-1C-RRAD-FDHSI-FD--CHK-*---NC4E_C_EUMT_2020040???????_GTT_DEV_20200404??????_2020040???????_N__T_0139_00??.nc
W_XX-EUMETSAT-Darmstadt\,IMG+SAT\,MTI1+FCI-1C-RRAD-FDHSI-FD--CHK-*---NC4E_C_EUMT_2020040???????_GTT_DEV_20200404??????_2020040???????_N__T_0140_00??.nc
...
W_XX-EUMETSAT-Darmstadt\,IMG+SAT\,MTI1+FCI-1C-RRAD-FDHSI-FD--CHK-*---NC4E_C_EUMT_2020040???????_GTT_DEV_20200406??????_2020040???????_N__T_0005_00??.nc
W_XX-EUMETSAT-Darmstadt\,IMG+SAT\,MTI1+FCI-1C-RRAD-FDHSI-FD--CHK-*---NC4E_C_EUMT_2020040???????_GTT_DEV_20200406??????_2020040???????_N__T_0006_00??.nc
```
(can be generated and a bit copy/paste with `for d in {4..6};do echo $d;for c in {0001..0144};do echo $c;for f in W_XX-EUMETSAT-Darmstadt\,IMG+SAT\,MTI1+FCI-1C-RRAD-FDHSI-FD--CHK-TRAIL---NC4E_C_EUMT_2020040???????_GTT_DEV_2020040${d}??????_2020040???????_N__T_${c}_00??.nc;do echo $f;done;done;done`)

And `/software/pytroll/scripts/start-mtg-test.sh` is:
```
#!/bin/bash

for f in $1
do
    echo "$f"
    scp -pv $f eumetcast@sater6:/data/test-mtg-fci/
    sleep 11
done
```

This gives you something like this for the `at -l`:
```
318	Tue Dec  7 14:54:00 2021 a polarsat
315	Tue Dec  7 14:24:00 2021 a polarsat
320	Tue Dec  7 15:14:00 2021 a polarsat
316	Tue Dec  7 14:34:00 2021 a polarsat
317	Tue Dec  7 14:44:00 2021 a polarsat
314	Tue Dec  7 14:14:00 2021 = polarsat
319	Tue Dec  7 15:04:00 2021 a polarsat
321	Tue Dec  7 15:24:00 2021 a polarsat
```
ie. one slot starting every 10 minute. Be carefull to start exactly at 0, 10, 20 etc if you want the slot to start at those times, but this is not important for the test.

The processing is using a conda environment based on miniconda and pytroll/satpy=0.32.0 and python=3.9 with saving to geotiffs with tiling.

We have several queues. In the test research-el7.q was used.

## Results

Total 6396 MTG FCI netcdf test files where ingested into the processing. 6396 were received. 100% reception for this test starting 2021-12-06 12:34:00 completing 2021-12-07 14:34:00

|          | Avg       | Min      | Max      |
|----------|----------:|---------:|---------:|
|wait_time |    1.84s  |   1.00s  |   86.00s |
|run_time  |  412.87s  | 168.00s  |  876.00s |
|maxvmem   |   37.60G  |   1.77G  |   44.29G |
|cpu       |  910.77   |  32.85   | 1227.27  |

Wait time is the time between submission of the job to the queue and start of the processing.
Run time is the run time of the processing
Maxvmem is the maximum memory (in GB) used by the processing
cpu is the cpu time used.

There looks like one or more processes failed as it was fast and used little memory. This was the 20200405_212000 lsot, and logs indicate an external download of pyspectral_rsr_data.tgz failed.(These needs to be cached somehow.)

9 slots used more than 10 minutes to process ie. the data is not complete before a new dataset has been received. It must be a goal to have a processing faster than 10 minutes.

The following 14 full disk native resampling products in geotiff:
'airmass', 'ash', 'cloudtop', 'convection', 'day_microphysics', 'dust', 'fog', 'green_snow', 'ir108_3d', 'ir_cloud_day', 'natural_color', 'night_fog', 'night_microphysics', 'true_color_raw'

This is similar to what to expect in production. true_color is not made, so the processing inproduction will be longer. In addition resampled products are needed.

Typical CPUs if the processing servers are: 
Intel(R) Xeon(R) CPU E5-2650 v4 @ 2.20GHz
Intel(R) Xeon(R) Gold 5118 CPU @ 2.30GHz

But the processing allocated 8 slots using 16 dask workers.

Of 2184 expected geotiff products, 2170 products where produced ( due to one failed slot as explained above)

## Conclusion

Even if one slot failed, 9 slots used longer than 10 minutes and we might need more products in operation I'm confident that we will manage to process the MTG FCI data when we first reveive data in Q1/Q2 2023 using our PPI infrastructure.

## Environment

The following package version were used:
```
dask                      2021.11.2          pyhd8ed1ab_0    conda-forge
dask-core                 2021.11.2          pyhd8ed1ab_0    conda-forge
geotiff                   1.7.0                h90a4e78_5    conda-forge
libgdal                   3.4.0               hbe510e8_10    conda-forge
libtiff                   4.3.0                h6f004c6_2    conda-forge
numpy                     1.21.4           py39hdbf815f_0    conda-forge
pyresample                1.22.1           py39hde0f152_1    conda-forge
rasterio                  1.2.10           py39he0fb565_3    conda-forge
satpy                     0.32.0             pyhd8ed1ab_0    conda-forge
xarray                    0.20.1             pyhd8ed1ab_0    conda-forge
```