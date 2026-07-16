---
layout: page
title: ""
---

<style>
  body {
    background-color: #faf6ee;
    color: #2b2a26;
    font-family: Georgia, 'Times New Roman', serif;
    max-width: 720px;
    margin: 0 auto;
    padding: 1.5rem 1.25rem;
  }
  .project-overview {
    font-size: 0.95em;
  }
</style>

# <u>GSoC 2026: Parallelizing r.proj and Raster Processing Modules in GRASS</u>

<div class="project-overview" markdown="1">

## Project Overview

Hi I'm Kaushik, a Computer Science + Crop Sciences student at UIUC working with NumFOCUS and GRASS for Google Summer of Code 2026.

GRASS is one of the oldest open source geographic analysis systems. Researchers and agencies use it to process satellite imagery and map data. One of its most used tools is r.proj, which converts a raster map from one coordinate system to another, for example from plain latitude and longitude into a projection suited for a specific region. On large maps this is slow because the module runs on a single core.

My project parallelizes r.proj and other raster modules with OpenMP so they use all available cores. The memory use stays bounded through a memory option, so big maps still run on ordinary machines instead of needing the whole input in RAM. This gives the user flexibility in how much memory they are willing to allocate and still benefit from a speedup in running the module. The hardest part is not the threading itself. It is proving the parallel output stays exactly identical to the serial version on every projection pair, at every thread count, which is where most of my time goes.

Full project details are on the GRASS wiki: [GRASS GSoC 2026 — Parallelizing r.proj and Raster Processing Modules](https://grasswiki.osgeo.org/wiki/GRASS_GSoC_2026_Parallelizing_r.proj_and_Raster_Processing_Modules_in_GRASS)

</div>

## Posts

<details markdown="1">
<summary><b>Weeks 7 and 8</b></summary>

<!-- COMING SOON -->

</details>

<details markdown="1">
<summary><b>Weeks 5 and 6</b></summary>

[PASTE_WEEKS_5_6]

</details>

<details markdown="1">
<summary><b>Weeks 3 and 4</b></summary>

[PASTE_WEEKS_3_4]

</details>

<details markdown="1">
<summary><b>Weeks 1 and 2</b></summary>

**What I worked on in week 1**

I built and benchmarked two different ways to parallelize r.param.scale. The first version reads strips straight from the disk, conceptually very similar to how r.neighbors is parallelized ([draft PR #7440](https://github.com/OSGeo/grass/pull/7440)), and the second loads them into RAM per thread ([draft PR #7442](https://github.com/OSGeo/grass/pull/7442)). Both gave exactly the same output as the original serial code across window sizes 5 to 51. There was some speedup compared to the serial version, but not as fast as it should be. In Friday’s meeting the mentors and I agreed #7440 is conceptually the better route to take. I also benchmarked r.neighbors on my computer to make sure it wasn’t a reason I was seeing poor speedup ratios. It scaled well there at larger windows, so my code was the reason for the weaker scaling, not the machine. I ended with doing a much deeper analysis of what the differences were between my disk approach and r.neighbors. I found the main gaps to be not having the two-level band structure, each thread holding too much of the map in memory, no memory limit option, and no mask handling.

**What I worked on in week 2:**

I refactored the code for draft PR #7440 to fix the weak speedup ratios I had last week. The parallel version now uses the same two level band setup that r.neighbors uses. It works through the map band by band, splits each band’s rows across the threads, and each thread only holds a small rolling window of rows instead of a large part of the map. I also limited memory usage with the standard memory= option and added mask handling. On a 100 million cell raster the peak went from about 620 MB down to about 330 MB. The output is still identical to the serial module’s output at one thread and with a mask.

While reworking the module I found and fixed a bug that’s in the original serial module too. When the window is larger than the region, it used to write past the end of the map. The parallel version handles that case cleanly now.

I tested the refactored code with a bunch of different benchmarking scripts and different window sizes, every test case was with a 100 million cell raster. For one of the test cases, I used window size 31 and I saw about 1.91x at 2 threads, 3.37x at 4, and 4.55x at 8. I added the benchmark script and a script to generate the test raster to the PR so mentors can reproduce it on their own machines.

<img width="541" style="max-width:100%; height:auto;" alt="r.param.scale cold-isolated parallel scaling benchmark table" src="/param_scale_image.png" />

The main thing I took away from these two weeks is that matching the serial output is the easy part, and getting a real speedup is where the actual engineering is. Most of my gains did not come from the threading itself but from being careful about how much of the map each thread holds at once, which meant closely following the way r.neighbors already manages memory.

</details>

<details markdown="1">
<summary><b>Week 0 Community Bonding</b></summary>

The bonding period went into reading source code and getting a real benchmarking setup working.

I started by reading through the modules I would be touching or learning from. r.neighbors is already parallelized, so I studied how it does it, mainly the trick of giving every thread its own file handle so they can all read different parts of the map at the same time. Then I read r.param.scale, the first module I would be parallelizing, and found its main obstacle. It processes the map top to bottom through a small window of rows that slides down as it goes, so each step depends on the one before it, and work like that can not simply be split across threads. I did a similar deep read of r.geomorphon and corrected an assumption I had made earlier. I thought it jumped around the whole map unpredictably, but its reads actually stay within a fixed distance of the cell being worked on, which makes it much friendlier to parallelize than I expected. I also skimmed modules like r.slope.aspect, r.patch, and r.series to learn the patterns GRASS already uses, and made a list of which unparallelized modules would benefit the most.

On the setup side I found that OpenMP was not being detected properly on my Mac because of the clang toolchain, so I set up an Ubuntu VM, compiled GRASS with gcc, and got everything working there. That gave me my first real benchmark numbers. r.neighbors scaled to about 3.97x on 8 cores, and the existing r.param.scale reached about 2.26x, which became the baseline I would be trying to beat.

I sent progress reports both weeks and had the first mentor meeting, where I got the go ahead to build two competing versions of a parallel r.param.scale and test them against each other.

</details>
