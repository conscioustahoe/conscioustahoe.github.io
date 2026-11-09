+++
title = "Weather Data Lab"
description = "Notes from taking weather station and radar data from raw public archives to a model, and how the same design works at petabyte scale."
template = "article.html"

[extra]
subtitle = "Notes from taking weather station and radar data from raw files to a model."
repo = "https://github.com/conscioustahoe/weather-data-lab"
+++

## Why I did this

I wanted to learn how raw sensor data becomes something a model can learn from. Reading [Natural General Intelligence](https://naturalgeneralintelligence.ai/#observing-system) and [WeatherBench 2](https://arxiv.org/abs/2308.15560) made weather the obvious place to try because the data is public and messy and it goes all the way up to petabytes.

The essay points out that most AI weather models train on ERA5 rather than on observations. ERA5 is a reconstruction that blends measurements with a physics model and doesn't say which values were measured and which were filled in. WeatherBench 2 is built on that same data stored as Zarr in the cloud and its data guide shows how arrays that size are cut up and read.

## The goal

I gave myself one dataset to own from end to end and three questions to answer about it.

> Take four years of hourly data from about 165 California weather stations from raw public archives to a training-ready dataset with a quality verdict on every value. How do I read it correctly? How do I know when a value is wrong? Does cleaning it actually make a model better?

When those had answers I wanted to know how the same ideas hold up with a second kind of sensor and with much more data. So I added radar, rebuilt the pipeline in Ray, Spark and Beam and pushed a 27 GB grid through it. Everything ran on a 16 GB laptop and every plot uses real data. The [notebooks](#the-notebooks) have the detail.

## Getting to know the data

<figure class="wide">
<svg class="fig" viewBox="0 0 640 350" role="img" aria-label="Yearly data volume of Earth observing systems on a log scale. Surface stations make 2.8 TB a year and ground radar 810 TB, against 24 PB for optical satellites.">
<line x1="210.0" x2="210.0" y1="22" y2="324" class="grid"/>
<text x="210.0" y="340" class="tick" text-anchor="middle">100 GB</text>
<line x1="270.0" x2="270.0" y1="22" y2="324" class="grid"/>
<text x="270.0" y="340" class="tick" text-anchor="middle">1 TB</text>
<line x1="330.0" x2="330.0" y1="22" y2="324" class="grid"/>
<text x="330.0" y="340" class="tick" text-anchor="middle">10 TB</text>
<line x1="390.0" x2="390.0" y1="22" y2="324" class="grid"/>
<text x="390.0" y="340" class="tick" text-anchor="middle">100 TB</text>
<line x1="450.0" x2="450.0" y1="22" y2="324" class="grid"/>
<text x="450.0" y="340" class="tick" text-anchor="middle">1 PB</text>
<line x1="510.0" x2="510.0" y1="22" y2="324" class="grid"/>
<text x="510.0" y="340" class="tick" text-anchor="middle">10 PB</text>
<line x1="570.0" x2="570.0" y1="22" y2="324" class="grid"/>
<text x="570.0" y="340" class="tick" text-anchor="middle">100 PB</text>
<text x="200" y="43" class="lab" text-anchor="end">Optical imaging satellites</text>
<rect x="210" y="33" width="322.8" height="14" rx="2" class="bar"/>
<text x="538.8" y="44" class="val">24 PB</text>
<text x="200" y="69" class="lab" text-anchor="end">Radar and altimetry satellites</text>
<rect x="210" y="59" width="287.5" height="14" rx="2" class="bar"/>
<text x="503.5" y="70" class="val">6.2 PB</text>
<text x="200" y="95" class="lab" text-anchor="end">Polar-orbiting satellites</text>
<rect x="210" y="85" width="256.7" height="14" rx="2" class="bar"/>
<text x="472.7" y="96" class="val">1.9 PB</text>
<text x="200" y="121" class="lab strong" text-anchor="end">Ground weather radar</text>
<rect x="210" y="111" width="234.5" height="14" rx="2" class="bar ours"/>
<text x="450.5" y="122" class="val strong">810 TB  · phase 2</text>
<text x="200" y="147" class="lab" text-anchor="end">Geostationary satellites</text>
<rect x="210" y="137" width="218.6" height="14" rx="2" class="bar"/>
<text x="434.6" y="148" class="val">440 TB</text>
<text x="200" y="173" class="lab" text-anchor="end">Trace gas networks</text>
<rect x="210" y="163" width="193.8" height="14" rx="2" class="bar"/>
<text x="409.8" y="174" class="val">170 TB</text>
<text x="200" y="199" class="lab" text-anchor="end">River, snow and soil sensors</text>
<rect x="210" y="189" width="148.6" height="14" rx="2" class="bar"/>
<text x="364.6" y="200" class="val">30 TB</text>
<text x="200" y="225" class="lab strong" text-anchor="end">Surface weather stations</text>
<rect x="210" y="215" width="86.8" height="14" rx="2" class="bar ours"/>
<text x="302.8" y="226" class="val strong">2.8 TB  · phase 1</text>
<text x="200" y="251" class="lab" text-anchor="end">GPS radio occultation</text>
<rect x="210" y="241" width="62.5" height="14" rx="2" class="bar"/>
<text x="278.5" y="252" class="val">1.1 TB</text>
<text x="200" y="277" class="lab" text-anchor="end">Floats, buoys and ships</text>
<rect x="210" y="267" width="52.8" height="14" rx="2" class="bar"/>
<text x="268.8" y="278" class="val">760 GB</text>
<text x="200" y="303" class="lab" text-anchor="end">Weather balloons and aircraft</text>
<rect x="210" y="293" width="16.7" height="14" rx="2" class="bar"/>
<text x="232.7" y="304" class="val">190 GB</text>
<text x="210" y="16" class="tick">data collected per year, log scale</text>
</svg>
<figcaption>How much the world's observing systems collect each year, from the Natural General Intelligence essay. The two I used are highlighted.</figcaption>
</figure>

The first thing I wanted to understand was what a single weather station actually reports. NOAA keeps every airport report as text files and at first it looks like a clean table. Reading it properly showed an hour can hold several kinds of report taken at different minutes so building an hourly table needs a written rule for which one wins. Most airport thermometers report whole degrees Fahrenheit and that makes the tenths in the Celsius column an artefact of unit conversion. NOAA's own quality flags passed a sensor stuck at −39 °C for most of a month and marked four years of a healthy station as erroneous.

That made me wonder what I could compare a thermometer against. I used HRRR, NOAA's 3 km forecast model. Its "analysis" has already been nudged toward these same station reports and comparing the two would be circular so I used its one-hour forecast instead. Station and model mostly agreed and the places they didn't had physical reasons. Coastal stations ran warm because the model's grid box holds sea and high valleys ran cold because cold air pools there at night.

Next I wanted to see how a completely different sensor behaves. Radar is about 300 times bigger than the station data on that chart. A radar file isn't a map but a stack of 360° sweeps at different tilts and the beam climbs as it goes out because the Earth curves away underneath it.

<figure>
<img src="/weather-data-lab/radar-kmux.png" alt="Three radar maps around the Bay Area radar during the October 2021 storm: reflectivity, radial velocity and correlation coefficient. A patch south-west of the radar has low correlation coefficient.">
<figcaption>The Bay Area radar in the October 2021 storm. The patch south-west of the radar looks like light rain on the left and the correlation coefficient on the right shows it's waves on the ocean.</figcaption>
</figure>

About one in eight echoes on the lowest scan were waves rather than rain even in the middle of a major storm. When I matched four years of radar rain against station rain gauges the beam height explained most of the difference. Radar saw about 90% of the gauge rain where its beam passed low over a station and less than half where it passed more than 2 km up. Some places like Bishop behind the Sierra Nevada get almost no radar rain at all.

## Teaching myself to catch bad data

With the data understood the next question was how to tell a wrong value from a strange but real one. A quality check is a classifier and I wanted to measure it like one. Real data has no labels so I measured the two halves separately. For recall I took clean stations and broke them on purpose while keeping an answer key. The faults were the kinds real sensors have: sudden steps, slow drifts, constant biases, stuck values, clocks that were hours off and data that disappears only during storms. For precision I checked 30 real alarms by hand. Every station-hour ends up with a verdict of pass, suspect or quarantine and a quarantine needs two independent kinds of evidence.

<figure>
<img src="/weather-data-lab/susanville.png" alt="Susanville airport temperature against the HRRR model in autumn 2021. The station drops to −39 °C for almost a month while the model shows normal daily cycles.">
<figcaption>A fault NOAA's checks passed: Susanville airport stuck at −39 °C. Most faults are much quieter than this.</figcaption>
</figure>

Loud faults like stuck sensors and spikes were easy to catch. Small level shifts were much harder, with a 2 °C step found almost every time and a 0.5 °C step almost never. A clock eight hours off didn't move the average at all while single hours were off by 10 °C and a station could lose 90% of its storms and still look 95% complete. Most of the real false alarms came from a few kinds of weather like coastal heat waves, Santa Ana winds the model timed an hour off and valley fog. Learning those patterns helped more than tuning thresholds. The checks also found four broken sensors that NOAA had passed. Later radar did the same for rain gauges when Travis Air Force Base reported hours of 100 mm during light drizzle and then measured a real storm correctly three days later.

## Does it help a model?

This was the question I cared about most. If the verdicts are right but don't change what a model learns then they don't matter much. I trained a gradient-boosted model that corrects HRRR at each station on several versions of the same training data and tested every version on a clean 2024.

| Training data | Error at the broken stations (°C) |
|---|---|
| Clean | 1.14 |
| No model, just HRRR | 1.45 |
| Broken | 1.57 |
| Broken with flagged hours removed | 1.52 |
| Broken with the same number of hours removed at random | 1.57 |
| Broken with shifted stretches repaired | 1.17 |

The model trained on broken data was worse than having no model because it learned each broken station's offset and applied it to 2024. Removing the flagged hours barely helped even though they were the right hours since random removal did nothing at all. Most of the damage came from stations reading a degree or two off for months and an hourly check can't see that. What worked was finding the dates where a station's level shifted and correcting the stretch before it. That brought the model back almost to the clean result.

## Turning it into a pipeline

The notebooks worked but I wanted to know what it takes to run the same thing as a real pipeline that can be rerun safely and kept up to date. Each piece of work became a small unit keyed by source and time such as one station-year and it writes its output atomically. A manifest records every unit with a fingerprint of its input file and the version of the code that processed it. With that in place a rerun with nothing changed does nothing, a changed input is redone on its own and the backfill and the incremental update share one plan.

I then ran the same units in three frameworks to see what each one is built for. Ray ran them as tasks about four times faster than a single process and its actors were a natural place to keep one open connection per worker. Spark showed me that comparing each station with its neighbours becomes a join with three shuffles once the data is split across machines even though it's a single line in pandas. Beam showed me what late data does. A reading that arrives after its day was already judged makes that day fire again with a complete new result which only works if downstream writes are keyed by station and day and simply overwrite.

The same thing turned out to happen to the verdicts. When I replayed the checks month by month the way a live system would have run them, 1.7% of verdicts changed once more data had arrived and most of those went from pass to suspect. Verdicts made early in a record are judged against a seasonal baseline that's still being learned which is why verdicts need versions just like the data does.

## At petabyte scale

My data is a few GB. NOAA puts more than 30 PB on public clouds and ECMWF's archive holds about 450 PB mostly on tape. I wanted to understand what actually changes at that size and the short answer is that nothing is ever loaded whole.

### How it's stored

The shape of the data decides the storage. Station data is many short time series that suit columnar files like Parquet. Model output and reanalysis are large regular grids that suit **Zarr** where one n-dimensional array is cut into chunks and each chunk is its own object in a bucket. Radar is a tree of polar sweeps and the newer FM-301 standard stores that as Zarr too.

Archives like ECMWF's MARS keep the original format and index it. You ask for a parameter, date and level and the system works out where it lives on disk or tape. Cloud copies are rewritten as analysis-ready Zarr the way [ARCO-ERA5](https://github.com/google-research/ARCO-ERA5) and WeatherBench 2 are. Two newer tools avoid copying at all. [VirtualiZarr](https://virtualizarr.readthedocs.io/) indexes existing GRIB or NetCDF files once and serves them as Zarr without moving a byte and [Icechunk](https://icechunk.io/) adds git-like snapshots so a dataset can be appended to without readers seeing half-written data.

### How it's analyzed

Analyzing petabytes works the same way at any size. The data is already split into chunks and a cluster of hundreds of machines works on them in parallel. Each machine reads only its own chunks and reduces them to something small like a mean or a count before moving on so its memory never holds more than a few chunks at once. Only those small results get combined and only the final answer comes back to the notebook.

On Spark's original clusters each task ran on the machine whose disk already held its block of data. That's where "move the compute to the data" comes from. In the cloud the data lives in object storage that's separate from the machines. There the same idea means running the cluster in the same region as the bucket and never pulling raw data down to a laptop. From a notebook you open the data lazily and describe the slice you want and only the chunks that hold that slice move.

```python
import xarray as xr
ds = xr.open_zarr("gs://gcp-public-data-arco-era5/ar/full_37-1h-0p25deg-chunk-1.zarr-v3",
                  chunks=None, storage_options={"token": "anon"})          # metadata only
t = ds["2m_temperature"].sel(latitude=37.4, longitude=237.9, method="nearest")
t.sel(time=slice("2023-01-01", "2023-01-02")).load()                     # bytes move here
```

Opening that store took 6 seconds on my laptop and reading two days at one point took 14 seconds. This copy stores one hour of the whole globe per chunk so each hour is a separate read. I wanted to see how much the chunk shape matters so I took four years of HRRR over California that's 27 GB of values and 7.4 GB as compressed Zarr and stored it twice. One copy has map-shaped chunks and the other has time-series-shaped chunks.

| Read | Map-shaped chunks | Time-series chunks |
|---|---|---|
| One map | 0.02 s | 7.7 s |
| Four years at one place | 7.0 s | 0.04 s |

The data and the disk are the same and the difference is 200 to 450 times. That's why large producers keep the same data in more than one layout and pick the chunk shape for the main reader.

Converting between the two layouts is a rechunk which is the array version of a shuffle. My first attempt used Apache Beam's local runner and ran the laptop out of memory since one week of data alone peaked at 3.3 GB. What worked was going through an intermediate layout on disk the way the [rechunker](https://rechunker.readthedocs.io/en/latest/algorithm.html) library does it and that never used more than 2 GB for the whole 27 GB. After that I measured every heavy step on a small slice first and ran it under a hard memory cap.

### The hard parts

These are the problems I'd expect anyone running this at scale to face. Several of them are covered in Martin Kleppmann's *Designing Data-Intensive Applications*.

- **Time.** Event time and arrival time differ, station clocks drift and some ocean moorings deliver their data months late. Late data can be counted and dropped or published as a correction and I chose corrections.
- **Reprocessing is normal.** A new QA rule means rerunning years of data so every step has to be idempotent and recomputable from raw.
- **Formats change.** Model upgrades, new GRIB tables and replaced sensors all change the data under you.
- **Hot spots and skew.** Everyone reads the newest forecast the minute it lands and a storm day of radar is ten times a calm one.
- **Small files.** NEXRAD alone writes about 40,000 files a day and per-file overhead dominates unless files are batched or stored as chunks of one array.
- **Neighbour checks don't shard.** Hourly checks parallelize easily but anything comparing stations needs data from many places at the same time and that means a shuffle.
- **Provenance.** Measured, model-filled and repaired values need to stay distinguishable all the way to the model.

## Architecture

The idea from Kleppmann's book that helped me most is the split between a **system of record** and **derived data**. The raw files are the record and nothing ever edits them. Everything else is derived from them and could be deleted and rebuilt. The derived views are also where the work done when data arrives meets the work done when someone asks for it and choosing their shape is choosing how much of each to do.

<figure class="wide">
<svg class="fig" viewBox="0 0 640 600" role="img" aria-label="Architecture. Raw files from four sources are the system of record. One write path with a manifest turns them into derived views: a station table with verdicts, model values at stations, two Zarr layouts of the model grid, rain at stations and monitoring metrics. Training, analysis, QA and monitoring each read the view shaped for them. A band underneath lists how the pipeline is verified.">
<defs><marker id="ah3" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="ahead"/></marker></defs>
<text x="6" y="16" class="colhead">SYSTEM OF RECORD</text>
<text x="6" y="30" class="tick">raw, never edited</text>
<text x="160" y="16" class="colhead">WRITE PATH</text>
<text x="160" y="30" class="tick">runs when data lands</text>
<text x="306" y="16" class="colhead">DERIVED VIEWS</text>
<text x="306" y="30" class="tick">where writes meet reads</text>
<text x="498" y="16" class="colhead">READ PATH</text>
<text x="498" y="30" class="tick">runs when asked</text>
<rect x="6" y="44" width="138" height="92" rx="6" class="node store"/>
<text x="15" y="61" class="ntitle sm">Stations</text>
<text x="15" y="75" class="nsize">11 GB</text>
<text x="15" y="88" class="nline sm">ISD text, one file</text>
<text x="15" y="101" class="nline sm">per station-year</text>
<path d="M146,90 L158,90" class="edge" marker-end="url(#ah3)"/>
<rect x="6" y="152" width="138" height="92" rx="6" class="node store"/>
<text x="15" y="169" class="ntitle sm">Weather model</text>
<text x="15" y="183" class="nline sm">HRRR GRIB2, two</text>
<text x="15" y="196" class="nline sm">fields read by</text>
<text x="15" y="209" class="nline sm">byte range</text>
<path d="M146,198 L158,198" class="edge" marker-end="url(#ah3)"/>
<rect x="6" y="260" width="138" height="92" rx="6" class="node store"/>
<text x="15" y="277" class="ntitle sm">Radar</text>
<text x="15" y="291" class="nsize">18 GB</text>
<text x="15" y="304" class="nline sm">NEXRAD Level II</text>
<text x="15" y="317" class="nline sm">2,202 volumes</text>
<path d="M146,306 L158,306" class="edge" marker-end="url(#ah3)"/>
<rect x="6" y="368" width="138" height="92" rx="6" class="node store"/>
<text x="15" y="385" class="ntitle sm">Rain mosaic</text>
<text x="15" y="399" class="nsize">35 GB</text>
<text x="15" y="412" class="nline sm">MRMS GRIB2</text>
<text x="15" y="425" class="nline sm">69,983 hourly files</text>
<path d="M146,414 L158,414" class="edge" marker-end="url(#ah3)"/>
<rect x="160" y="44" width="130" height="416" rx="6" class="node"/>
<text x="169" y="61" class="ntitle sm">One plan</text>
<text x="169" y="75" class="nline sm">backfill and live</text>
<text x="169" y="88" class="nline sm">share the code</text>
<text x="169" y="101" class="nline sm"></text>
<text x="169" y="114" class="nline sm">unit: source × time</text>
<text x="169" y="127" class="nline sm"></text>
<text x="169" y="140" class="nline sm">manifest records</text>
<text x="169" y="153" class="nline sm">input fingerprint</text>
<text x="169" y="166" class="nline sm">and code version</text>
<text x="169" y="179" class="nline sm"></text>
<text x="169" y="192" class="nline sm">writes are atomic</text>
<text x="169" y="205" class="nline sm"></text>
<text x="169" y="218" class="nline sm">late data or a fix:</text>
<text x="169" y="231" class="nline sm">rerun those units,</text>
<text x="169" y="244" class="nline sm">publish new version</text>
<text x="169" y="257" class="nline sm"></text>
<text x="169" y="270" class="nline sm">Ray, Spark or Beam</text>
<text x="169" y="283" class="nline sm">schedule the units</text>
<rect x="306" y="44" width="176" height="74" rx="6" class="node"/>
<text x="315" y="61" class="ntitle sm">Station table</text>
<text x="315" y="75" class="nsize">277 MB Parquet</text>
<text x="315" y="88" class="nline sm">flags and verdict</text>
<text x="315" y="101" class="nline sm">beside each value</text>
<path d="M292,81.0 L304,81.0" class="edge" marker-end="url(#ah3)"/>
<rect x="498" y="44" width="136" height="74" rx="6" class="node"/>
<text x="507" y="61" class="ntitle sm">QA and research</text>
<text x="507" y="75" class="nline sm">filter by verdict,</text>
<text x="507" y="88" class="nline sm">station or storm</text>
<path d="M484,81.0 L496,81.0" class="edge" marker-end="url(#ah3)"/>
<rect x="306" y="126" width="176" height="60" rx="6" class="node"/>
<text x="315" y="143" class="ntitle sm">Model at stations</text>
<text x="315" y="157" class="nsize">114 MB Parquet</text>
<text x="315" y="170" class="nline sm">the reference</text>
<path d="M292,156.0 L304,156.0" class="edge" marker-end="url(#ah3)"/>
<rect x="306" y="194" width="176" height="60" rx="6" class="node"/>
<text x="315" y="211" class="ntitle sm">Grid, training copy</text>
<text x="315" y="225" class="nsize">6.9 GB Zarr</text>
<text x="315" y="238" class="nline sm">chunks: 24 h × map</text>
<path d="M292,224.0 L304,224.0" class="edge" marker-end="url(#ah3)"/>
<rect x="498" y="194" width="136" height="60" rx="6" class="node"/>
<text x="507" y="211" class="ntitle sm">Training</text>
<text x="507" y="225" class="nline sm">one hour = 1 chunk</text>
<text x="507" y="238" class="nline sm">version pinned</text>
<path d="M484,224.0 L496,224.0" class="edge" marker-end="url(#ah3)"/>
<rect x="306" y="262" width="176" height="60" rx="6" class="node"/>
<text x="315" y="279" class="ntitle sm">Grid, analysis copy</text>
<text x="315" y="293" class="nsize">9.0 GB Zarr</text>
<text x="315" y="306" class="nline sm">chunks: 4 yr × 16²</text>
<path d="M292,292.0 L304,292.0" class="edge" marker-end="url(#ah3)"/>
<rect x="498" y="262" width="136" height="60" rx="6" class="node"/>
<text x="507" y="279" class="ntitle sm">Analysis</text>
<text x="507" y="293" class="nline sm">one place, 4 years</text>
<text x="507" y="306" class="nline sm">= 1 chunk, 0.04 s</text>
<path d="M484,292.0 L496,292.0" class="edge" marker-end="url(#ah3)"/>
<rect x="306" y="330" width="176" height="60" rx="6" class="node"/>
<text x="315" y="347" class="ntitle sm">Rain at stations</text>
<text x="315" y="361" class="nsize">4.6 MB Parquet</text>
<text x="315" y="374" class="nline sm">radar vs gauge</text>
<path d="M292,360.0 L304,360.0" class="edge" marker-end="url(#ah3)"/>
<rect x="306" y="398" width="176" height="62" rx="6" class="node"/>
<text x="315" y="415" class="ntitle sm">Monitoring</text>
<text x="315" y="429" class="nline sm">one row per</text>
<text x="315" y="442" class="nline sm">monthly run</text>
<path d="M292,429.0 L304,429.0" class="edge" marker-end="url(#ah3)"/>
<rect x="498" y="398" width="136" height="62" rx="6" class="node"/>
<text x="507" y="415" class="ntitle sm">Watching</text>
<text x="507" y="429" class="nline sm">rates vs their</text>
<text x="507" y="442" class="nline sm">own band → alert</text>
<path d="M484,429.0 L496,429.0" class="edge" marker-end="url(#ah3)"/>
<rect x="6" y="478" width="628" height="114" rx="6" class="node verify"/>
<text x="16" y="498" class="ntitle sm">Trust, but verify</text>
<text x="16" y="522" class="nline sm strongish">Recall</text>
<text x="16" y="536" class="nline sm">planted faults</text>
<text x="16" y="549" class="nline sm">with a key</text>
<text x="140" y="522" class="nline sm strongish">Precision</text>
<text x="140" y="536" class="nline sm">real alarms</text>
<text x="140" y="549" class="nline sm">checked by hand</text>
<text x="264" y="522" class="nline sm strongish">Second witness</text>
<text x="264" y="536" class="nline sm">model vs station</text>
<text x="264" y="549" class="nline sm">radar vs gauge</text>
<text x="388" y="522" class="nline sm strongish">Rebuild from raw</text>
<text x="388" y="536" class="nline sm">same input gives</text>
<text x="388" y="549" class="nline sm">the same output</text>
<text x="512" y="522" class="nline sm strongish">Versions</text>
<text x="512" y="536" class="nline sm">old and new</text>
<text x="512" y="549" class="nline sm">side by side</text>
<text x="16" y="575" class="nline sm">Any derived view can be deleted and rebuilt from the system of record.</text>
</svg>
<figcaption>The architecture with this project's real formats and sizes. Each read path uses the view shaped for it.</figcaption>
</figure>

Reading the diagram left to right shows the choices. The write path is one plan for both history and new data so there's no second code path to drift out of sync. The two Zarr copies are the same numbers shaped for two different readers which is the 200 to 450 times difference above. The verdicts sit beside the values in the station table instead of replacing them so anyone can choose a different strictness. The band at the bottom is how I know any of it is right. Recall and precision check the checks, a second source checks each sensor and rebuilding from raw checks the pipeline itself.

## What I took away

I started out thinking of data quality as a cleaning step and I ended up thinking of it as a claim about the data that should be measured like a model, with recall on known faults, precision on real flags and its effect on what's trained downstream. The faults that hurt most were the quiet ones. A second independent source like a weather model or a radar caught what a sensor couldn't show about itself and keeping track of where every value came from mattered as much as the value.

## The notebooks

| # | Notebook | The question it answers |
|---|---|---|
| 01 | [Stations: read and explore](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/01-stations-read-and-explore.ipynb) | what does a station actually report? |
| 02 | [Radar: read and explore](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/02-radar-read-and-explore.ipynb) | how does a radar see and what fools it? |
| 03 | [Matching to a reference](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/03-matching-to-a-reference.ipynb) | what can a thermometer be compared against? |
| 04 | [How data goes wrong](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/04-how-data-goes-wrong.ipynb) | what do real sensor faults look like? |
| 05 | [QA checks and metrics](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/05-qa-checks-and-metrics.ipynb) | how often are the verdicts right? |
| 06 | [Monitoring and verdicts](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/06-monitoring-and-verdicts.ipynb) | do verdicts change over time? |
| 07 | [Pipelines with Ray](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/07-pipelines-with-ray.ipynb) | how do I rerun everything safely? |
| 08 | [Spark and Beam](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/08-spark-and-beam.ipynb) | what is each framework built for? |
| 09 | [Does clean data help the model?](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/09-does-clean-data-help-the-model.ipynb) | does any of this matter to a model? |
| 10 | [Radar vs gauges](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/10-radar-vs-gauges.ipynb) | when do two rain sensors disagree and why? |
| 11 | [Gridded data at scale](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/11-gridded-data-at-scale.ipynb) | what does chunk shape cost? |

<p class="caption">Sources: NOAA <a href="https://www.ncei.noaa.gov/products/land-based-station/integrated-surface-database">ISD</a>, <a href="https://registry.opendata.aws/noaa-hrrr-pds/">HRRR</a>, <a href="https://registry.opendata.aws/noaa-nexrad/">NEXRAD</a> and <a href="https://registry.opendata.aws/noaa-mrms-pds/">MRMS</a> on AWS · Orbuch, <a href="https://naturalgeneralintelligence.ai/">Natural General Intelligence</a> · Rasp et al., <a href="https://arxiv.org/abs/2308.15560">WeatherBench 2</a> · ECMWF, <a href="https://www.ecmwf.int/node/31124">The living archive</a> · <a href="https://github.com/google-research/ARCO-ERA5">ARCO-ERA5</a> · <a href="https://rechunker.readthedocs.io/en/latest/algorithm.html">rechunker</a> · Kleppmann, <em>Designing Data-Intensive Applications</em> (O'Reilly, 2017).</p>
