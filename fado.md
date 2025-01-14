
*fado* is a trigkey minipc running a few services from my apartment, this
document contains notes about the setup & maintenance of the plex server
running there.

Here and now, in early January 2025, I am resurrecting an old service that has
been shut down for years due to an easily overheated external drive (and
laziness), serving movies, tv shows, and a few collections of music from a new
replacement drive.

# setup
The Plex server is running via the `plexinc/pms-docker` image, configured via
`docker-compose.fado.yml` in this directory, which is pointed to via a symlink:
```
kenneth@fado ~/git/plex-pms-docker (fado) $ ls -lh docker-compose.yml
lrwxrwxrwx 1 kenneth kenneth 23 Feb  4  2024 docker-compose.yml -> docker-compose.fado.yml
```
Allowing one to manage the server process via the usual commands:
```
kenneth@fado ~/git/plex-pms-docker (fado) $ docker-compose up
kenneth@fado ~/git/plex-pms-docker (fado) $ docker ps
CONTAINER ID   IMAGE                COMMAND          CREATED        STATUS                  PORTS                                                                       
   NAMES
1e0f25d31684   plexinc/pms-docker   "/init"          16 hours ago   Up 16 hours (healthy)                               plex
```

# useful commands
A reference of mostly CLI tools useful for maintinaing a digital library

## ripping dvds

### rip dvd with subtitle track
First, concat all of the `.VOB` files into one big one:
```
$ cat VTS_03_{1,2,3,4,5}.VOB > out/Heimat\ Chapter\ 1\ -\ Fernweh.vob
```
Next, examine the file to identify what's what:
```
$ ffmpeg -analyzeduration 100M -probesize 100M -i out/Heimat\ Chapter\ 1\ -\ Fernweh.vob
```
This example asks `ffmpeg` to scan the first 100mb of the file, `.VOB` files
don't include a header describing the streams contained within, and if we only
look at the first frame the subtitle info will probably not yet be present.

This reports a bunch of stuff about your current version version of `ffmpeg`,
options it was compiled with and so on, at the end there is a list of streams
found in the `.VOB` file:
```
Input #0, mpeg, from 'out/Heimat Chapter 1 - Fernweh.vob':
  Duration: 01:59:22.18, start: 0.280000, bitrate: 5168 kb/s
  Stream #0:0[0x1bf]: Data: dvd_nav_packet
  Stream #0:1[0x1e0]: Video: mpeg2video (Main), yuv420p(tv, top first), 720x576 [SAR 16:15 DAR 4:3], 25 fps, 25 tbr, 90k tbn
    Side data:
      cpb: bitrate max/min/avg: 9000000/0/0 buffer size: 1835008 vbv_delay: N/A
  Stream #0:2[0x80]: Audio: ac3, 48000 Hz, stereo, fltp, 224 kb/s
  Stream #0:3[0x20]: Subtitle: dvd_subtitle
```
Stream `0:0` is for dvd navigation, we don't need that, `0:1` is the movie
itself, `0:2` is audio, and `0:3` the english subtitles, we want to keep all
three of those.

For a Plex-compatible output capable of combining these three streams into a
single container, we will use a `.mkv` file format, encoding the video with
`H.264` codec and audio as `mp3`.
```
$ ffmpeg -analyzeduration 100M -probesize 100M  \
    -i Heimat\ Chapter\ 1\ -\ Fernweh.vob \
    -map 0:1 -map 0:2 -map 0:3 \
    -metadata:s:s:0 language=eng -metadata:s:s:0 title="English" \
    -codec:v libx264 -crf 21 -codec:a libmp3lame -qscale:a 2 -codec:s copy \
    -threads 16 \
    Heimat\ Chapter\ 1\ -\ Fernweh.mkv
```
Once again, we need the `-analyzeduration 100M -probesize 100M` args (and they
must come first) to instruct `ffmpeg` to scan for all the streams we want to
extract.

Alter the `threads` count to match the number of cores you have.

The other options are copied from, and summarized at, the first link in the
resources section below.

### naming TV Show files for the Plex scanner
I don't know exactly what the requirements are to make Plex recognize the
added files, but thanks to **ATotalBastard** on reddit, I finally got my `.mkv` to
show up in my collection & have verified that the subtitles work by putting
the `.mkv` in a directory of exactly the same name, and including the `s01e01`
style episode numbering scheme (previously I just had "Chapter 1"):
```
kenneth@fado /mnt/seagate/tv/Heimat (1984) $ ls -lhR
.:
drwxr-sr-x 2 kenneth media 4.0K Jan 14 17:35 'Heimat (1984) - s01e01 - Fernweh'

'./Heimat (1984) - s01e01 - Fernweh':
-rw-r--r-- 1 kenneth media 984K Jan 14 13:51  cover.jpg
-rw-r--r-- 1 kenneth media 1.8G Jan 14 15:34 'Heimat (1984) - s01e01 - Fernweh.mkv'
```

## ripping cds

## tagging mp3/ogg/flac files
There are a lot of options for this since the old days when I created my own,
I am not particularly endorsing these, just leaving notes here 
### lltag
```
$ lltag -a "Maths Balance Volumes" -A "Cattle Skulls and Railroad Tracks" --rename %a--%A--%n--%t  --rename-min -n4 -t "Midwestern Mountain Rumble/The Rolling Muddist Rumble"  track_4.mp3
$ lltag -S maths\ balance\ volumes--cattle\ skulls\ and\ railroad\ tracks--04--midwestern\ mountain\ rumble-the\ rolling\ muddist\ rumble.mp3
maths balance volumes--cattle skulls and railroad tracks--04--midwestern mountain rumble-the rolling muddist rumble.mp3:
  ARTIST=Maths Balance Volumes
  TITLE=Midwestern Mountain Rumble/The
  ALBUM=Cattle Skulls and Railroad Tra
  NUMBER=4
  COMMENT=Chocolate Monk
```
Slightly cumbersome, should probably define an alias with preferred options:
```
alias lltag-rename="lltag --rename %a--%A--%n--%t --rename-min --rename-sep _ --rename-regexp \"s/'//\""
```

# files
The files are served from an external usb3 drive formatted with an ext4 fs.

For maintenance, each of the directories being served is owned by me, group `media`:
```
drwxrwsr-x 175 kenneth media 4.0K Jan 12 19:35 /mnt/seagate/lossless
drwxrwsr-x  46 kenneth media 8.0K Feb  6  2024 /mnt/seagate/movies
drwxr-sr-x 609 kenneth media  16K Jan 13 01:14 /mnt/seagate/music
drwxr-xr-x  56 kenneth media 4.0K Jan 12 23:51 /mnt/seagate/plex-audioblogs
drwxr-xr-x 255 kenneth media  12K Jan 13 19:10 /mnt/seagate/plex-live-shows
drwxr-sr-x 207 kenneth media  12K Feb  6  2024 /mnt/seagate/plex-movies
drwxr-xr-x   2 kenneth media 4.0K Jan 13 19:51 /mnt/seagate/plex-mst3k
drwxrwsr-x  32 kenneth media 4.0K Jan 24  2022 /mnt/seagate/torrents
drwxrwsr-x  45 kenneth media 4.0K Jan 13 03:12 /mnt/seagate/tv
```

# system

## uname -a
```
Linux fado 6.1.0-28-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.119-1 (2024-11-22) x86_64 GNU/Linux
```

## lsusb
```
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 003: ID 8087:0029 Intel Corp. AX200 Bluetooth
Bus 003 Device 002: ID 25a7:fa23 Areson Technology Corp 2.4G Receiver
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 002 Device 002: ID 0bc2:2038 Seagate RSS LLC Expansion HDD
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

## lscpu
```
Architecture:                         x86_64
CPU op-mode(s):                       32-bit, 64-bit
Address sizes:                        48 bits physical, 48 bits virtual
Byte Order:                           Little Endian
CPU(s):                               16
On-line CPU(s) list:                  0-15
Vendor ID:                            AuthenticAMD
Model name:                           AMD Ryzen 7 5800H with Radeon Graphics
CPU family:                           25
Model:                                80
Thread(s) per core:                   2
Core(s) per socket:                   8
Socket(s):                            1
Stepping:                             0
Frequency boost:                      enabled
CPU(s) scaling MHz:                   35%
CPU max MHz:                          4462.5000
CPU min MHz:                          1200.0000
BogoMIPS:                             6388.09
Flags:                                fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx mmxext fxsr_opt pdpe1gb rdtscp lm constant_tsc rep_good nopl nonstop_tsc cpuid extd_apicid aperfmperf rapl pni pclmulqdq monitor ssse3 fma cx16 sse4_1 sse4_2 movbe popcnt aes xsave avx f16c rdrand lahf_lm cmp_legacy svm extapic cr8_legacy abm sse4a misalignsse 3dnowprefetch osvw ibs skinit wdt tce topoext perfctr_core perfctr_nb bpext perfctr_llc mwaitx cpb cat_l3 cdp_l3 hw_pstate ssbd mba ibrs ibpb stibp vmmcall fsgsbase bmi1 avx2 smep bmi2 erms invpcid cqm rdt_a rdseed adx smap clflushopt clwb sha_ni xsaveopt xsavec xgetbv1 xsaves cqm_llc cqm_occup_llc cqm_mbm_total cqm_mbm_local clzero irperf xsaveerptr rdpru wbnoinvd cppc arat npt lbrv svm_lock nrip_save tsc_scale vmcb_clean flushbyasid decodeassists pausefilter pfthreshold avic v_vmsave_vmload vgif v_spec_ctrl umip pku ospke vaes vpclmulqdq rdpid overflow_recov succor smca fsrm
Virtualization:                       AMD-V
L1d cache:                            256 KiB (8 instances)
L1i cache:                            256 KiB (8 instances)
L2 cache:                             4 MiB (8 instances)
L3 cache:                             16 MiB (1 instance)
NUMA node(s):                         1
NUMA node0 CPU(s):                    0-15
Vulnerability Gather data sampling:   Not affected
Vulnerability Itlb multihit:          Not affected
Vulnerability L1tf:                   Not affected
Vulnerability Mds:                    Not affected
Vulnerability Meltdown:               Not affected
Vulnerability Mmio stale data:        Not affected
Vulnerability Reg file data sampling: Not affected
Vulnerability Retbleed:               Not affected
Vulnerability Spec rstack overflow:   Mitigation; safe RET, no microcode
Vulnerability Spec store bypass:      Mitigation; Speculative Store Bypass disabled via prctl
Vulnerability Spectre v1:             Mitigation; usercopy/swapgs barriers and __user pointer sanitization
Vulnerability Spectre v2:             Mitigation; Retpolines; IBPB conditional; IBRS_FW; STIBP always-on; RSB filling; PBRSB-eIBRS Not affected; BHI Not affected
Vulnerability Srbds:                  Not affected
Vulnerability Tsx async abort:        Not affected
```

## /proc/meminfo
```
MemTotal:       13196896 kB
MemFree:          312928 kB
MemAvailable:   11481100 kB
Buffers:          771628 kB
Cached:         10298508 kB
SwapCached:         2096 kB
Active:          4666400 kB
Inactive:        6914020 kB
Active(anon):      67412 kB
Inactive(anon):   440916 kB
Active(file):    4598988 kB
Inactive(file):  6473104 kB
Unevictable:           0 kB
Mlocked:               0 kB
SwapTotal:        999420 kB
SwapFree:         957436 kB
Zswap:                 0 kB
Zswapped:              0 kB
Dirty:                84 kB
Writeback:             0 kB
AnonPages:        489476 kB
Mapped:           176592 kB
Shmem:              1148 kB
KReclaimable:     424372 kB
Slab:            1022560 kB
SReclaimable:     424372 kB
SUnreclaim:       598188 kB
KernelStack:        8000 kB
PageTables:         5572 kB
SecPageTables:         0 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     7597868 kB
Committed_AS:    1786812 kB
VmallocTotal:   34359738367 kB
VmallocUsed:      121552 kB
VmallocChunk:          0 kB
Percpu:            18176 kB
HardwareCorrupted:     0 kB
AnonHugePages:    202752 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
FileHugePages:         0 kB
FilePmdMapped:         0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
DirectMap4k:      740432 kB
DirectMap2M:    12812288 kB
DirectMap1G:           0 kB
```

## resources

  1. instructions to rip dvd with subtitles based on https://www.internalpointers.com/post/convert-vob-files-mkv-ffmpeg


