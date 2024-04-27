+++
title = '5600X vs 5800X3D on Linux'
date =  2022-12-02
draft = false
tags = ['linux', 'benchmark']
+++

Recently bought a 5800X3D to replace my 5600X, and ran some benchmarks to see if it was worth it.

**Update:** It wasn't worth the 8% performance loss with most regular tasks. Returned it for a full refund.

# Benchmarks

Did three quick ones, two work related and one gaming:
1. starting PostgreSQL and applying a lot of migrations
2. building a Next.js application
3. running Cyberpunk 2077 on all the presets

Running Pop!_OS 22.04, on 2022-12-02.

## PostgreSQL

### 5600X

```
./loop.sh  19.87s user 3.77s system 95% cpu 24.709 total
./loop.sh  19.75s user 3.53s system 95% cpu 24.336 total
./loop.sh  20.02s user 3.63s system 95% cpu 24.808 total
```

### 5800X3D

```
./loop.sh  22.02s user 3.83s system 95% cpu 27.086 total
./loop.sh  21.90s user 3.70s system 95% cpu 26.883 total
./loop.sh  21.87s user 3.77s system 95% cpu 26.879 total
```

### Comparison

Total time, smaller is better.

```
. 5600X #1 ▏ 24.709 ██████████████████████▊
  5600X #2 ▏ 24.336 ██████████████████████▍
  5600X #3 ▏ 24.808 ██████████████████████▉
5800X3D #1 ▏ 27.086 █████████████████████████
5800X3D #2 ▏ 26.883 ████████████████████████▊
5800X3D #3 ▏ 26.879 ████████████████████████▊
```

On average, the 5800X3D was 91.35% the performance of the 5600X, or 8.65% *slower*.

## Next.js

### 5600X

```
npm run build  210.73s user 7.92s system 307% cpu 1:11.06 total
npm run build  216.67s user 7.51s system 305% cpu 1:13.29 total
npm run build  214.16s user 7.20s system 311% cpu 1:10.98 total
```

### 5800X3D

```
npm run build  247.76s user 8.57s system 323% cpu 1:19.18 total
npm run build  242.66s user 8.39s system 333% cpu 1:15.27 total
npm run build  248.84s user 8.10s system 323% cpu 1:19.38 total
```

### Comparison

Total time, smaller is better.

```
. 5600X #1 ▏ 1:11.06 ██████████████████████▍
  5600X #2 ▏ 1:13.29 ███████████████████████
  5600X #3 ▏ 1:10.98 ██████████████████████▎
5800X3D #1 ▏ 1:19.18 ████████████████████████▉
5800X3D #2 ▏ 1:15.27 ███████████████████████▋
5800X3D #3 ▏ 1:19.38 █████████████████████████
```

On average, the 5800X3D was 92.09% the performance of the 5600X, or 7.91% *slower*.

## Cyberpunk 2077

These were all done at the various graphics presets, with a resolution of 2560x1440 and a 1070 Ti.

I should've just turned off FSR to help control things better. =/

### 5600X

| Preset Name | Average FPS | Min FPS | Max FPS | Time  | Number of Frames | FSR 2.1 |
|-------------|-------------|---------|---------|-------|------------------|---------|
| Low         | 41.19       | 32.92   | 50.94   | 64.22 | 2645             | Auto    |
| Medium      | 40.88       | 31.56   | 51.93   | 64.24 | 2626             | Auto    |
| High        | 40.78       | 31.08   | 50.99   | 64.22 | 2619             | Auto    |
| ~~Ultra~~       | ~~21.29~~       | ~~17.91~~   | ~~27.99~~   | ~~64.17~~ | ~~1366~~             | ~~Quality~~ |
| ~~Steam Deck~~  | ~~31.32~~       | ~~24.55~~   | ~~38.67~~   | ~~64.22~~ | ~~2011~~             | ~~Auto~~    |

### 5800X3D

| Preset Name | Average FPS | Min FPS | Max FPS | Time  | Number of Frames | FSR 2.1 |
|-------------|-------------|---------|---------|-------|------------------|---------|
| Low         | 41.93       | 32.56   | 52.19   | 64.22 | 2693             | Auto    |
| Medium      | 41.75       | 32.41   | 53.80   | 64.23 | 2682             | Auto    |
| High        | 41.58       | 31.43   | 52.38   | 64.21 | 2670             | Auto    |
| ~~Ultra~~       | ~~31.48~~       | ~~25.18~~   | ~~39.31~~   | ~~64.20~~ | ~~2021~~             | ~~Quality~~ |
| ~~Steam Deck~~  | ~~41.66~~       | ~~32.24~~   | ~~52.65~~   | ~~64.20~~ | ~~2675~~             | ~~Auto~~    |

## Comparison

Frames per second, higher is better.

```
.        5600X Low ▏ 41.19 ████████████████████████▌
       5800X3D Low ▏ 41.93 █████████████████████████
      5600X Medium ▏ 40.88 ████████████████████████▎
    5800X3D Medium ▏ 41.75 ████████████████████████▉
        5600X High ▏ 40.78 ████████████████████████▎
      5800X3D High ▏ 41.58 ████████████████████████▊
       5600X Ultra ▏ 21.29 ████████████▋
     5800X3D Ultra ▏ 31.48 ██████████████████▊
  5600X Steam Deck ▏ 31.32 ██████████████████▋
5800X3D Steam Deck ▏ 41.66 ████████████████████████▊
```

For a preset, the 5800X3D was:
* Low: 1.80% faster
* Medium: 2.13% faster
* High: 1.96% faster
* ~~Ultra: 47.9% faster~~
* ~~Steam Deck: 33.0% faster~~

I've crossed out the Ultra and Steam Deck preset benchmarks, since I'm pretty sure I messed up the benchmarks there. Noticed that it wasn't consistently logging the same FSR option I had selected in the output, too. If I ever do this again, I'll try and do some more benchmarks and control things better.

# Was it worth it?

No.
