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

**What I worked on in week 7:**

I finished the fix for the lu.c bug this week. The fix itself is just removing the parallel pragma from the pivot search, since none of the callers actually benefit from it. I implemented it, tested it, and added it to [#7440](https://github.com/OSGeo/grass/pull/7440), which also let me remove the single threaded workaround in r.param.scale and the pytest skip, since neither is needed once the library itself is correct. To make sure the fix is real and not just hiding the problem, I built a separate test of G_ludcmp and ran it under thread sanitizer. The old version with the pragma reports data races and the fixed version runs clean. The module output is now identical across thread counts with no workaround at all.

The rest of the week went into the column-wise parallelization for r.proj in [#7627](https://github.com/OSGeo/grass/pull/7627) that was suggested to me. The idea is that before loading anything, the code checks whether the next band of output rows fits under the memory cap at full width. If it does, it proceeds row wise like before. But if even a single full width row needs more input than the cap allows, the band gets split into column tiles until each tile fits. Getting this right meant figuring out how to bound which input rows each tile actually needs, which I do by walking the tile edges and back projecting them into the input. I also parallelized the input reading, where each thread gets its own file descriptor so they can read different parts of the input at the same time. Another thing I handled was PROJ transformation objects. They can’t be shared across threads, so each thread now clones its own through a small wrapper I added to the projection library. I was also asked for that wrapper to go into the library properly, and doing it that way also removed a direct PROJ build dependency from both build systems.

Doing column-wise parallelization also extended how many CRS reprojections the code handles, which is from simple reprojections to much harder pairs. There is a wide Lambert Azimuthal job that the row wise version couldn’t run at all under the memory cap. The column wise version finishes it. The output matches the serial module exactly on every projection pair I tested, at 1 through 8 threads with multiple input patterns. The one case still failing at the end of the week was the very hardest kind, maps with a pole inside the output frame.

**What I worked on in week 8:**

*[Week 8 report coming soon]*

</details>

<details markdown="1">
<summary><b>Weeks 5 and 6</b></summary>

**What I worked on in week 5:**

I cleaned up the pytest from [#7534](https://github.com/OSGeo/grass/pull/7534) and it was merged. Right after merging it resurfaced the lu.c bug from issue [#7539](https://github.com/OSGeo/grass/issues/7539), since the test runs cell by cell comparisons that catch a small difference. To keep the CI from erroring out for everyone, I opened a small follow up PR ([#7593](https://github.com/OSGeo/grass/pull/7593)) that skips the affected test until the library fix lands. The skip gets removed once the bug is fixed. For more information on how the bug worked please check the #7539 issue.

Most of the week went into investigating dips in the parallel scaling at certain thread counts. The dips never happened at the same thread count consistently, so that ruled out a code bug tied to a specific configuration. My first guess was that the efficiency cores on my machine were dragging runs down. But testing the timing per thread showed they were not. At the dip, one thread simply runs up to about 2x slower than the rest. Every band waits at a barrier, so that one slow thread holds up the whole band. I ran r.neighbors through the same timing on each thread on the same machine and its threads finished evenly, even though r.param.scale splits rows the same way. I was given a suggestion it could be a frequency or power limit from the performance and efficiency core split. So I logged per cluster frequencies with powermetrics during full sweeps, with 200 second cooldowns between runs. The slow runs actually had the performance cores clocked slightly higher and the efficiency cores less engaged, which is the opposite of what a frequency limit would look like. The slow thread count also moved between runs. I ran other tests throughout the week to try and find a universal reason for why dips happen randomly across different machines. We ended up saying that there might not be a single universal cause. It looks like a minor scheduling imbalance that each machine amplifies differently. The code itself is not the issue, since the output matches serial exactly at every thread count, method, and window size.

**What I worked on in week 6:**

On #7440 I removed the old testsuite and added the pytest with the skip removed to see how CI handles it. I swapped my older benchmark for the script my mentor provided, with the thread pinning commented out. I also rechecked serial speed against the unparallelized version on the same 100 million cell map with window size 31 and cooldowns between runs, and there was no regression.

For the lu.c bug I worked out a plan for a permanent fix. I searched every place G_ludcmp is called across GRASS and the addons, and only four modules use it. R.param.scale, being one of the modules, already had my single threaded workaround, and two others call it from serial code where the race can’t fire. The last one, v.surf.rst, calls it from inside an already parallel region where OpenMP runs the inner loop single threaded anyway. Since no caller actually benefits from the parallel pivot search, and with both Claude’s suggestion and mentor agreeing with the idea, the simplest correct fix was to just remove the pragma call from the library.

I also started on r.proj this week with a row based parallelization. The idea is simple. The output map is a grid of rows, and each thread gets its own set of rows to compute independently. The best way to think about this is like several people painting different horizontal stripes of a wall at the same time. It was memory efficient with a good compute speedup, but the total speedup is limited because raster output writing in GRASS is serial by design. However the compute speedup showed promising results, benchmarks in [#7627](https://github.com/OSGeo/grass/pull/7627). The bigger problem was that the row approach only worked for simple reprojections. For harder projections the output rows need input from all over the source map instead of a neat matching band, so my mentors suggested trying a column-wise approach for those cases. I also addressed mentor feedback and questions within the draft PR comments.

Week 5 taught me more about what happens underneath my code than inside it. I went from guessing that the efficiency cores were the problem to actually logging per cluster CPU frequencies with powermetrics, and I learned that at a barrier the slowest thread sets the pace for the whole band. The harder lesson was accepting that not every performance dip has one clean cause, and that sometimes working with lower level systems brings in a certain level of uncertainty that is dependent machine to machine. For this specific case that uncertainty was also expanded due to the scheduling imbalance.

</details>

<details markdown="1">
<summary><b>Weeks 3 and 4</b></summary>

**What I worked on in week 3:**

While stress testing the parallel r.param.scale I found a small nondeterminism in the output, around one unit in the last place, that only showed up with more than one thread. I traced it to the pivot search in the library function [lu.c](https://github.com/OSGeo/grass/blob/5546e0172d7dad1e3ee248aec3fe00eeebbb3ab3/lib/gmath/lu.c#L57-L65). It runs a parallel loop that writes the shared variables big and imax without any synchronization or lock, so the pivot it picks can change from run to run once multiple threads are used. I reported it to the mentors and it is now tracked in issue [#7539](https://github.com/OSGeo/grass/issues/7539). As a workaround, [#7440](https://github.com/OSGeo/grass/pull/7440) runs just the LU decomposition single threaded, so the module’s output matches serial exactly at every thread count without touching the library.

At mentor’s request I also created [PR #7534](https://github.com/OSGeo/grass/pull/7534), a pytest that generates a small raster and checks r.param.scale results against reference values from the current serial code. I also pushed a gunittest version to #7440 that compares nprocs=1 against nprocs=4 directly.

**What I worked on in week 4:**

This week was more focused on cleanup and review response. I fixed several issues in the pytest from PR #7534 based on mentor feedback and removed the gunittest version from #7440 in favor of keeping just the pytest and a universal benchmark. The rest of the week went into cleaning up the parallel r.param.scale code and addressing review comments on the PR. Meanwhile the mentors ran their own benchmarks on their hardware, with and without pinned threads, so we could compare scaling behavior across different machines.

<img width="600" style="max-width:100%; height:auto;" alt="r.param.scale memory speedup chart, speedup versus number of processing elements for 10, 100, 300 and 1000 MB memory settings" src="/param_scale_memory_speedup.png" />

<img width="600" style="max-width:100%; height:auto;" alt="r.param.scale memory efficiency chart, parallel efficiency versus number of processing elements for 10, 100, 300 and 1000 MB memory settings" src="/param_scale_memory_efficiency.png" />

The biggest lesson from these two weeks was how much of systems engineering is problem solving in code you did not write. A bug can sit in a shared library for years and only surface once you add threads, so chasing a difference in the last decimal place meant reading well below the level I expected to. I learned that it was very crucial to nitpick at every detail and to really verify facts rather than writing things off as simple noise errors. It also reminded me that correctness at scale is not only my module’s problem, and that a clean, well tested PR that proves the old code wrong matters just as much as a fast one.

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
