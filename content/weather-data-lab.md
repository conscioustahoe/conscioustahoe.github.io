+++
title = "Weather Data Lab"
description = "Taking raw weather station and radar data to a training-ready dataset with a quality verdict on every value, and how the same design works at petabyte scale."
template = "article.html"

[extra]
subtitle = "Raw weather observations, all the way to a model. Then the same ideas at petabyte scale."
repo = "https://github.com/conscioustahoe/weather-data-lab"
+++

Most machine learning weather models never see a raw observation. They train on ERA5, a reconstruction of the atmosphere that a weather centre has already combined, checked and smoothed. Somebody had to decide which measurements to trust. This project is that step, done by hand on a small scale.

I took four years of hourly data from about 165 California weather stations and carried it from raw public archives to a training-ready table with a quality verdict on every value. Then I checked how often the verdicts are right and whether they make a model better. Everything runs on a laptop and every plot is real data. The [notebooks](#the-notebooks) have the detail.

## The data

Earth observation is a patchwork of instruments built over a century. Our two sources sit near the bottom of this chart, which is a good reminder that small data can still be the data that matters. Stations are the ground truth where people live.

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
<figcaption>Yearly data volumes, from Ryan Orbuch's <a href="https://naturalgeneralintelligence.ai/#observing-system">Natural General Intelligence</a> essay. Satellites make about 95% of the bytes.</figcaption>
</figure>

### Weather stations

NOAA's Integrated Surface Database keeps one fixed-width text file per station per year, with a quality code on every value. It looks like a clean table. It isn't.

- **"Hourly" isn't hourly.** Airports send routine, special and synoptic reports, and the routine one is taken a few minutes before the hour at a minute that differs by airport. Building an hourly table needs a written rule for which report wins.
- **The tenths are fake.** Most airport sensors report whole °F. The decimals come from converting to Celsius.
- **NOAA's flags both miss and over-reach.** They passed a sensor stuck at −39 °C for most of a month. They marked four years of a healthy station as erroneous.

### The weather model

To judge a thermometer you need something independent to compare it with. That's HRRR, NOAA's 3 km forecast model, stored as GRIB2 on AWS. I used its 1-hour forecast and not its *analysis*, because the analysis has already been nudged toward these same station reports. Comparing them would be grading a student against their own homework.

Station and model mostly agree, and where they don't there's geography. Bay and coastal stations run warm because the model's grid box holds water. High valleys run cold because cold air pools there at night. Over the marine layer the standard height correction makes things worse, since the air warms with height instead of cooling.

### Radar

A weather radar spins and listens for echoes from rain. NEXRAD's raw Level II files hold one full set of scans every five minutes, about 2.6 GB a day per radar on a stormy day. I read two California radars over five big storms.

<figure>
<img src="/weather-data-lab/radar-kmux.png" alt="Three radar maps around the Bay Area radar during the October 2021 storm: reflectivity, radial velocity and correlation coefficient. A patch south-west of the radar has low correlation coefficient.">
<figcaption>The Bay Area radar in the October 2021 storm. The patch south-west of the radar looks like light rain in reflectivity (left). The correlation coefficient (right) says it's waves on the ocean.</figcaption>
</figure>

- **It sees things that aren't rain.** In the middle of a major storm about 1 in 8 echoes on the lowest scan were sea clutter.
- **The beam leaves the ground.** The Earth curves away beneath it. 150 km out, even a radar scanning flat looks at air 2.4 km up and misses rain that forms below.

### The radar mosaic

MRMS is NOAA's national product. It merges every radar onto a 1 km grid, removes non-weather echo and estimates rain, with and without rain gauges.

- **Radar sees about half the rain.** In an atmospheric river my own radar estimate and MRMS's radar-only estimate agreed with each other, and both were about half of what the gauge-blended product had.
- **A gap can hide inside a mosaic.** The Bay Area radar's data stops for 18 hours during a January 2023 flood storm. The mosaic still looked complete because neighbouring radars covered the area from farther away.
- **The file format lies a little.** MRMS uses NOAA's own GRIB tables, so generic readers call the variable "unknown". And −3 means "no coverage", not rain.

## The pipeline

The design is boring on purpose. Raw files are never modified. Work is split into small units keyed by source and time (a station-year, a radar-day, an hour), and each unit writes atomically, so a job can be killed and rerun any time and only does what's missing. Data goes raw → standardized (SI units, UTC, one row per station-hour) → QA, where every check gets its own column beside the value. Nothing is ever deleted.

The one real ingestion lesson was about connections, not bandwidth. Reading thousands of byte ranges from GRIB files on S3 kept failing when every request opened a new connection. One reused connection per worker fixed it.

## Catching bad data

A quality check is a classifier, but real data has no labels. So I measured its two halves separately. **Recall** comes from faults I planted in clean stations with an answer key: steps, drifts, biases, stuck values, clock offsets and storm-only gaps, at a range of sizes. **Precision** comes from checking 30 real alarms by hand. Every station-hour gets a verdict: *pass*, *suspect* or *quarantine*, and quarantine needs two independent kinds of evidence.

<figure>
<img src="/weather-data-lab/susanville.png" alt="Susanville airport temperature against the HRRR model in autumn 2021. The station drops to −39 °C for almost a month while the model shows normal daily cycles.">
<figcaption>A loud fault NOAA passed: Susanville airport stuck at −39 °C. Most faults are much quieter.</figcaption>
</figure>

- **Loud faults are easy.** Stuck sensors, spikes and clocks off by hours are caught almost every time, within a day or two.
- **Quiet ones are hard.** A 2 °C shift gets pinned to its date almost always, 1 °C about a third of the time, 0.5 °C barely at all. A bias present from day one is close to invisible.
- **An 8-hour clock error doesn't move the average** while single hours are off by 10 °C.
- **Real precision is about 17%** and the false alarms have patterns: coastal heat waves, Santa Ana winds the model times an hour off, and valley fog.
- **NOAA passed four broken sensors** that simple comparisons with a model and neighbours catch.

## Does clean data help a model?

I trained a gradient boosting model to correct HRRR at each station, on 2021–2023, and tested it on 2024. The same model was trained on different versions of the training data, five times each. Errors at the stations I broke:

| Training data | Error (°C) |
|---|---|
| Clean | 1.14 |
| No model, just HRRR | 1.45 |
| Faulty | 1.57 |
| Faulty, bad hours removed by QA | 1.52 |
| Faulty, same number of hours removed at random | 1.57 |
| Faulty, broken stretches repaired | 1.17 |

**Faulty data was worse than no model.** The model learns each station's offset from HRRR, so it learned the broken offsets faithfully.

**Removing bad hours barely helped.** The hours were the right ones (random removal did nothing) but most of the damage came from stations that read a degree or two off for months, which hourly checks can't see.

**Repairing did.** Finding the date a station's level shifted and correcting the stretch before it got the model back almost to clean. The catch is that the repair trusts the newest stretch, which is fine here but not on a station that broke last month.

## At petabyte scale

Our data is a few GB. NOAA puts more than 30 PB on public clouds, and ECMWF's archive holds about 450 PB on tape behind disk, growing by roughly 480 TB a day. The ideas carry over. The hard parts change.

### Where it lives and how it's stored

The shape of the data decides the storage. Station data is many short time series, which suits columnar files like Parquet. Model output and reanalysis are huge regular grids, which suit **Zarr**: one n-dimensional array cut into chunks, each chunk its own object in a bucket. Radar is a tree of polar sweeps, and the new FM-301 standard stores it as Zarr too.

Two newer ideas avoid copying petabytes. **Virtual Zarr** indexes the original GRIB or NetCDF files once and serves them as Zarr without moving a byte. **Icechunk** adds git-like snapshots, so a live dataset can be appended to without readers ever seeing a half-written update.

### Reading it fast from a notebook

You never load a petabyte. You open it lazily, describe the slice you want, and only the chunks that hold it move.

```python
import xarray as xr
ds = xr.open_zarr("gs://gcp-public-data-arco-era5/ar/full_37-1h-0p25deg-chunk-1.zarr-v3",
                  chunks=None, storage_options={"token": "anon"})          # metadata only
t = ds["2m_temperature"].sel(latitude=37.4, longitude=237.9, method="nearest")
t.sel(time=slice("2023-01-01", "2023-01-02")).load()                     # bytes move here
```

That opens 1.3 million hours of global data in 6 seconds. Reading two days at one point then took 14 seconds, because this copy stores one hour of the whole globe per chunk, so every hour is a separate read. A year at one point would take about 40 minutes. **The chunk layout must match the reader.** Training reads whole maps and wants map-shaped chunks. Climate analysis reads long series at a point and wants the opposite. Big producers publish both.

### The hard parts

These are the questions I'd expect anyone designing this to face. Many map straight onto *Designing Data-Intensive Applications*.

- **Time is slippery.** Event time vs arrival time, station clocks that drift, and ocean moorings whose data arrives months late. When is a month "done" enough to judge?
- **Late data rewrites verdicts.** A neighbour check run in March changes when April's data arrives. Verdicts need versions just like data does.
- **Reprocessing is the normal case.** A new QA rule means rerunning years of data, so every stage must be idempotent and recomputable from raw.
- **Formats change under you.** Model upgrades, new GRIB tables and sensor swaps. Readers must survive every version they'll ever meet.
- **Hot spots and skew.** Everyone reads the newest forecast run the minute it lands, and a storm day of radar is ten times a calm one.
- **Small files kill throughput.** NEXRAD alone writes about 40,000 files a day. Per-file overhead swamps the work unless files are batched or stored as chunks of one array.
- **Neighbour checks don't shard.** Hourly checks parallelize trivially. Anything comparing stations needs a window across space and time, which means a shuffle.
- **Provenance has to survive.** Reanalysis blends measured and model-filled values with no label of which is which. Keep observed, inferred and repaired distinguishable all the way to the model.

## Architecture

Here's how I'd build this pipeline for real, at scale and on a live feed.

<figure class="wide">
<svg class="fig" viewBox="0 0 640 760" role="img" aria-label="Pipeline architecture: sources feed an ingest service with a manifest, which writes immutable raw data; standardize and QA stages produce a versioned store that serves training, analysis and monitoring, with human review feeding back into the QA rules.">
<defs><marker id="ah" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="ahead"/></marker></defs>
<rect x="20" y="20" width="600" height="66" rx="6" class="node src"/>
<text x="32" y="41" class="ntitle">Sources</text>
<text x="32" y="60" class="nline">NOAA public buckets: ISD stations, HRRR GRIB2, NEXRAD Level II, MRMS</text>
<text x="32" y="76" class="nline">new files announced as events · history listed for backfill</text>
<path d="M320,86 L320,116" class="edge" marker-end="url(#ah)"/>
<rect x="20" y="118" width="600" height="82" rx="6" class="node"/>
<text x="32" y="139" class="ntitle">Ingest</text>
<text x="32" y="158" class="nline">work units keyed by source × time (station-year, radar-day, hour)</text>
<text x="32" y="174" class="nline">manifest of what's done · retries · atomic writes (.part then rename)</text>
<text x="32" y="190" class="nline">rate limits and connection reuse per worker</text>
<path d="M320,200 L320,230" class="edge" marker-end="url(#ah)"/>
<rect x="20" y="232" width="600" height="50" rx="6" class="node store"/>
<text x="32" y="253" class="ntitle">Raw layer</text>
<text x="32" y="272" class="nline">object store, never modified · checksums · source and fetch time kept</text>
<path d="M320,282 L320,312" class="edge" marker-end="url(#ah)"/>
<rect x="20" y="314" width="600" height="66" rx="6" class="node"/>
<text x="32" y="335" class="ntitle">Standardize  (Ray · Spark · Beam)</text>
<text x="32" y="354" class="nline">SI units, UTC, CF names · one row per station-hour · grids to Zarr</text>
<text x="32" y="370" class="nline">pick the report that wins each hour · keep the original time</text>
<path d="M170,380 L170,410" class="edge" marker-end="url(#ah)"/>
<path d="M470,380 L470,410" class="edge" marker-end="url(#ah)"/>
<rect x="20" y="412" width="290" height="98" rx="6" class="node"/>
<text x="32" y="433" class="ntitle">QA · streaming</text>
<text x="32" y="452" class="nline">per-hour checks as data arrives:</text>
<text x="32" y="468" class="nline">range, stuck, physical, vs model</text>
<text x="32" y="484" class="nline">cheap, embarrassingly parallel</text>
<rect x="330" y="412" width="290" height="98" rx="6" class="node"/>
<text x="342" y="433" class="ntitle">QA · windowed batch</text>
<text x="342" y="452" class="nline">needs space and time windows:</text>
<text x="342" y="468" class="nline">neighbours, shifts, break dating</text>
<text x="342" y="484" class="nline">reruns when late data lands</text>
<path d="M170,510 L250,540" class="edge" marker-end="url(#ah)"/>
<path d="M470,510 L390,540" class="edge" marker-end="url(#ah)"/>
<rect x="20" y="542" width="600" height="66" rx="6" class="node store"/>
<text x="32" y="563" class="ntitle">Versioned store</text>
<text x="32" y="582" class="nline">values + every flag + verdict side by side · nothing deleted</text>
<text x="32" y="598" class="nline">snapshots so a model names the exact data it trained on</text>
<path d="M115,608 L115,638" class="edge" marker-end="url(#ah)"/>
<rect x="20" y="640" width="190" height="66" rx="6" class="node"/>
<text x="32" y="661" class="ntitle">Training</text>
<text x="32" y="680" class="nline">map-shaped chunks</text>
<text x="32" y="696" class="nline">loader feeds the model</text>
<path d="M320,608 L320,638" class="edge" marker-end="url(#ah)"/>
<rect x="225" y="640" width="190" height="66" rx="6" class="node"/>
<text x="237" y="661" class="ntitle">Analysis</text>
<text x="237" y="680" class="nline">time-series chunks</text>
<text x="237" y="696" class="nline">rechunked once offline</text>
<path d="M525,608 L525,638" class="edge" marker-end="url(#ah)"/>
<rect x="430" y="640" width="190" height="66" rx="6" class="node"/>
<text x="442" y="661" class="ntitle">Monitoring</text>
<text x="442" y="680" class="nline">metrics per run</text>
<text x="442" y="696" class="nline">alerts and freshness</text>
<path d="M615,706 C 636,706 636,700 636,690 L636,470 C636,460 630,460 622,460" class="edge dash" marker-end="url(#ah)"/>
<text x="628" y="735" class="elab" text-anchor="end">human review of alarms → corrected rules and labels</text>
</svg>
<figcaption>Every stage is idempotent and keyed by source and time, so backfills and live runs share the same code.</figcaption>
</figure>

The split in the middle is the important bit. Per-hour checks are cheap and run as data streams in. Checks that compare stations or look for shifts over weeks need windows and run as batch jobs that rerun when late data lands. Both write flags into a versioned store, and alarms go to a person whose decisions feed back into the rules.

## What's next

Radar against rain gauges, then the same pipeline rebuilt with Ray, Spark and Beam, and four years of HRRR stored in Zarr in both layouts to measure what layout costs.

## The notebooks

| # | Notebook | What it covers |
|---|---|---|
| 01 | [Stations: read and explore](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/01-stations-read-and-explore.ipynb) | report types, quality codes, gaps |
| 02 | [Radar: read and explore](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/02-radar-read-and-explore.ipynb) | radar geometry, non-weather echo, radar vs mosaic |
| 03 | [Matching to a reference](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/03-matching-to-a-reference.ipynb) | HRRR, GRIB2, forecast vs analysis |
| 04 | [How data goes wrong](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/04-how-data-goes-wrong.ipynb) | planting six kinds of faults |
| 05 | [QA checks and metrics](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/05-qa-checks-and-metrics.ipynb) | checks, verdicts, precision and recall |
| 09 | [Does clean data help the model?](https://github.com/conscioustahoe/weather-data-lab/blob/main/notebooks/09-does-clean-data-help-the-model.ipynb) | the experiment, dating and repairing breaks |

<p class="caption">Sources: NOAA <a href="https://www.ncei.noaa.gov/products/land-based-station/integrated-surface-database">ISD</a>, <a href="https://registry.opendata.aws/noaa-hrrr-pds/">HRRR</a>, <a href="https://registry.opendata.aws/noaa-nexrad/">NEXRAD</a> and <a href="https://registry.opendata.aws/noaa-mrms-pds/">MRMS</a> on AWS · <a href="https://arxiv.org/abs/2308.15560">WeatherBench 2</a> · <a href="https://naturalgeneralintelligence.ai/">Natural General Intelligence</a> · ECMWF, <a href="https://www.ecmwf.int/node/31124">The living archive</a> · Google <a href="https://github.com/google-research/ARCO-ERA5">ARCO-ERA5</a> · <a href="https://icechunk.io/">Icechunk</a> · <a href="https://virtualizarr.readthedocs.io/">VirtualiZarr</a> · Kleppmann, <em>Designing Data-Intensive Applications</em>.</p>
