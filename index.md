---
layout: page
title: ""
---

# GSoC 2026: Parallelizing r.proj and Raster Processing Modules in GRASS

## Project Overview

Hi I'm Kaushik, a Computer Science + Crop Sciences student at UIUC working with NumFOCUS and GRASS for Google Summer of Code 2026.

GRASS is one of the oldest open source geographic analysis systems. Researchers and agencies use it to process satellite imagery and map data. One of its most used tools is r.proj, which converts a raster map from one coordinate system to another, for example from plain latitude and longitude into a projection suited for a specific region. On large maps this is slow because the module runs on a single core.

My project parallelizes r.proj and other raster modules with OpenMP so they use all available cores. The memory use stays bounded through a memory option, so big maps still run on ordinary machines instead of needing the whole input in RAM. This gives the user flexibility in how much memory they are willing to allocate and still benefit from a speedup in running the module. The hardest part is not the threading itself. It is proving the parallel output stays exactly identical to the serial version on every projection pair, at every thread count, which is where most of my time goes.

Full project details are on the GRASS wiki: [GRASS GSoC 2026 — Parallelizing r.proj and Raster Processing Modules](https://grasswiki.osgeo.org/wiki/GRASS_GSoC_2026_Parallelizing_r.proj_and_Raster_Processing_Modules_in_GRASS)

## Posts

<details>
<summary><b>Weeks 7 and 8</b></summary>

<!-- COMING SOON -->

</details>

<details>
<summary><b>Weeks 5 and 6</b></summary>

[PASTE_WEEKS_5_6]

</details>

<details>
<summary><b>Weeks 3 and 4</b></summary>

[PASTE_WEEKS_3_4]

</details>

<details>
<summary><b>Weeks 1 and 2</b></summary>

[PASTE_WEEKS_1_2]

</details>

<details>
<summary><b>Week 0 Community Bonding</b></summary>

The bonding period went into reading source code and getting a real benchmarking setup working.

I started by reading through the modules I would be touching or learning from. r.neighbors is already parallelized, so I studied how it does it, mainly the trick of giving every thread its own file handle so they can all read different parts of the map at the same time. Then I read r.param.scale, the first module I would be parallelizing, and found its main obstacle. It processes the map top to bottom through a small window of rows that slides down as it goes, so each step depends on the one before it, and work like that can not simply be split across threads. I did a similar deep read of r.geomorphon and corrected an assumption I had made earlier. I thought it jumped around the whole map unpredictably, but its reads actually stay within a fixed distance of the cell being worked on, which makes it much friendlier to parallelize than I expected. I also skimmed modules like r.slope.aspect, r.patch, and r.series to learn the patterns GRASS already uses, and made a list of which unparallelized modules would benefit the most.

On the setup side I found that OpenMP was not being detected properly on my Mac because of the clang toolchain, so I set up an Ubuntu VM, compiled GRASS with gcc, and got everything working there. That gave me my first real benchmark numbers. r.neighbors scaled to about 3.97x on 8 cores, and the existing r.param.scale reached about 2.26x, which became the baseline I would be trying to beat.

I sent progress reports both weeks and had the first mentor meeting, where I got the go ahead to build two competing versions of a parallel r.param.scale and test them against each other.

</details>
