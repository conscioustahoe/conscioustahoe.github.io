+++
title = "Weather Data Lab"
description = "Learning how raw sensor data becomes something a model can learn from, with four years of California weather stations, radar and a weather model."
template = "article.html"

[extra]
subtitle = "Learning how raw sensor data becomes something a model can learn from."
repo = "https://github.com/conscioustahoe/weather-data-lab"
+++

## Why I did this

I read Ryan Orbuch's essay [*Natural General Intelligence*](https://naturalgeneralintelligence.ai/#observing-system) about how much data we collect about the Earth. Most of it comes from physical sensors measuring nature and I got curious about how that data becomes something we can learn from. That led me to [WeatherBench 2](https://arxiv.org/abs/2308.15560) and a lot of science and terms I didn't know so I decided to learn it by building something.

## The goal

I gave myself one dataset to own from end to end. Four years of hourly readings from about 165 California weather stations would go from NOAA's raw public archives to a training-ready table with a quality verdict (pass, suspect or quarantine) on every value. Then I'd find out how often those verdicts are right and whether they actually make a model better.

When that worked I added radar as a second and very different sensor and pushed a 27 GB grid of the HRRR weather model through the same pipeline to see what changes with scale. Everything ran on a 16 GB laptop.

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
<figcaption>How much the world's observing systems collect each year (from the Natural General Intelligence essay). The two I used are highlighted.</figcaption>
</figure>

I started with the smallest stream on that chart. NOAA keeps every airport weather report as text and an hour can hold several of them taken at different minutes so even "the temperature at 2 pm" needs a written rule. NOAA's own quality flags were the bigger surprise. They passed a sensor stuck at −39 °C for most of a month and marked four years of a healthy station as erroneous.

That raised the obvious question of what to compare a thermometer against. I used the HRRR weather model's one-hour forecast and not its analysis because the analysis has already been nudged toward these same stations. Where the two disagreed there was usually geography behind it. Coastal stations ran warm because the model's grid box holds sea and high valleys ran cold because cold air pools there at night.

Radar was the second sensor and about 300 times bigger. Its beam climbs as it goes out because the Earth curves away underneath it. It also hears more than rain.

<figure>
<img src="/weather-data-lab/radar-kmux.png" alt="Three radar maps around the Bay Area radar during the October 2021 storm: reflectivity, radial velocity and correlation coefficient. A patch south-west of the radar has low correlation coefficient.">
<figcaption>The Bay Area radar in the October 2021 storm. The patch south-west of the radar looks like light rain on the left and the correlation coefficient on the right shows it's waves on the ocean.</figcaption>
</figure>

Matching four years of radar rain against station gauges showed how much the beam height matters. Radar saw about 90% of the gauge rain where its beam passed low over a station and less than half where it passed more than 2 km up.

## Teaching myself to catch bad data

A quality check is a classifier and I wanted to measure it like one. Real data has no labels though. For recall I broke clean stations on purpose and kept an answer key (steps, drifts, biases, stuck values, clocks hours off and gaps only during storms). For precision I checked 30 real alarms by hand.

<figure>
<img src="/weather-data-lab/susanville.png" alt="Susanville airport temperature against the HRRR model in autumn 2021. The station drops to −39 °C for almost a month while the model shows normal daily cycles.">
<figcaption>A fault NOAA's checks passed: Susanville airport stuck at −39 °C. Most faults are much quieter than this.</figcaption>
</figure>

Loud faults were easy. A 2 °C step was found almost every time and a 0.5 °C step almost never. Most real false alarms were weather (coastal heat waves, Santa Ana winds the model timed an hour off and valley fog) and the checks found four broken sensors NOAA had passed.

## Does it help a model?

I trained a gradient-boosted model that corrects HRRR at each station on several versions of the same training data and tested every version on a clean 2024.

| Training data | Error at the broken stations (°C) |
|---|---|
| Clean | 1.14 |
| No model, just HRRR | 1.45 |
| Broken | 1.57 |
| Broken with flagged hours removed | 1.52 |
| Broken with the same number of hours removed at random | 1.57 |
| Broken with shifted stretches repaired | 1.17 |

Broken data made the model worse than having no model because it learned each broken station's offset faithfully. Removing the flagged hours barely helped since most of the damage came from stations reading a degree or two off for months. What worked was finding the dates where a station's level shifted and correcting the stretch before it.

## Designing it for petabytes

My data is a few GB. I wanted to think through how the same pipeline would work if it had to take in every radar, station and model run in the country with years of history behind them. I haven't run anything at that scale so this is how I'd expect to build it and where I'm unsure. It follows one radar file from the moment it lands to the moment a model reads it.

### Getting data in

<figure class="wide">
<svg class="fig" viewBox="0 0 640 400" role="img" aria-label="A new radar file lands in a public bucket. The bucket sends a new-object event into a queue. A backfill job puts events for past files into its own lower-priority queue. The planner reads each event, turns it into a unit keyed by radar and hour and checks the manifest to see whether that unit is already done.">
<defs><marker id="ah6" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="dhead"/></marker></defs>
<path d="M18,47 L18,97 A35.0,7 0 0 0 88,97 L88,47" class="shape"/>
<ellipse cx="53.0" cy="47" rx="35.0" ry="7" class="shape"/>
<text x="53.0" y="120" class="dlabel" text-anchor="middle">NEXRAD bucket</text>
<text x="53.0" y="134" class="dlabel" text-anchor="middle">on AWS</text>
<rect x="170" y="58" width="110" height="28" class="shape"/>
<line x1="192.0" y1="58" x2="192.0" y2="86" class="shape"/>
<line x1="214.0" y1="58" x2="214.0" y2="86" class="shape"/>
<line x1="236.0" y1="58" x2="236.0" y2="86" class="shape"/>
<line x1="258.0" y1="58" x2="258.0" y2="86" class="shape"/>
<text x="225" y="50" class="dlabel" text-anchor="middle">event queue</text>
<path d="M92,72 L166,72" class="dedge" marker-end="url(#ah6)"/>
<text x="129" y="64" class="dlabel strong" text-anchor="middle">new object</text>
<rect x="340" y="50" width="100" height="44" rx="14" class="shape"/>
<text x="390.0" y="77.0" class="dtitle" text-anchor="middle">Planner</text>
<path d="M284,72 L336,72" class="dedge" marker-end="url(#ah6)"/>
<path d="M540,47 L540,97 A35.0,7 0 0 0 610,97 L610,47" class="shape"/>
<ellipse cx="575.0" cy="47" rx="35.0" ry="7" class="shape"/>
<text x="575.0" y="120" class="dlabel" text-anchor="middle">manifest</text>
<path d="M444,64 L536,64" class="dedge" marker-end="url(#ah6)"/>
<text x="490" y="56" class="dlabel strong" text-anchor="middle">already done?</text>
<path d="M536,82 L444,82" class="dedge" marker-end="url(#ah6)"/>
<text x="490" y="96" class="dlabel" text-anchor="middle">no</text>
<line x1="129" y1="78" x2="129" y2="150" class="dthin"/>
<text x="18" y="166" class="dtitle sm">Event</text>
<text x="18" y="183" class="dmono">bucket:</text>
<text x="74" y="183" class="dmono">unidata-nexrad-level2</text>
<text x="18" y="197" class="dmono">key:</text>
<text x="74" y="197" class="dmono">.../KMUX20240205_101000_V06</text>
<text x="18" y="211" class="dmono">size:</text>
<text x="74" y="211" class="dmono">9.1 MB</text>
<rect x="18" y="300" width="110" height="40" rx="14" class="shape"/>
<text x="73.0" y="325.0" class="dtitle" text-anchor="middle">Backfill</text>
<text x="73" y="356" class="dlabel" text-anchor="middle">lists 2021–2024</text>
<rect x="170" y="306" width="110" height="28" class="shape"/>
<line x1="192.0" y1="306" x2="192.0" y2="334" class="shape"/>
<line x1="214.0" y1="306" x2="214.0" y2="334" class="shape"/>
<line x1="236.0" y1="306" x2="236.0" y2="334" class="shape"/>
<line x1="258.0" y1="306" x2="258.0" y2="334" class="shape"/>
<text x="225" y="298" class="dlabel" text-anchor="middle">backfill queue</text>
<path d="M132,320 L166,320" class="dedge" marker-end="url(#ah6)"/>
<path d="M284,320 L330,320 L330,104 L346,96" class="dedge" marker-end="url(#ah6)"/>
<text x="336" y="290" class="dlabel strong" text-anchor="start">lower priority</text>
<line x1="390" y1="98" x2="390" y2="150" class="dthin"/>
<text x="360" y="166" class="dtitle sm">Unit</text>
<text x="360" y="183" class="dmono">source:</text>
<text x="418" y="183" class="dmono">nexrad</text>
<text x="360" y="197" class="dmono">radar:</text>
<text x="418" y="197" class="dmono">KMUX</text>
<text x="360" y="211" class="dmono">hour:</text>
<text x="418" y="211" class="dmono">2024-02-05T10</text>
<text x="360" y="225" class="dmono">code:</text>
<text x="418" y="225" class="dmono">radar-v4</text>
<text x="360" y="239" class="dmono">input:</text>
<text x="418" y="239" class="dmono">fingerprint 9f3c…</text>
<path d="M560,236 L612,236" class="dedge" marker-end="url(#ah6)"/>
<text x="586" y="228" class="dlabel strong" text-anchor="middle">to workers</text>
<line x1="540" y1="236" x2="556" y2="236" class="dthin"/>
<text x="18" y="390" class="dlabel">About 160 radars × 10 volumes an hour ≈ 40,000 files a day.</text>
</svg>
<figcaption>A new radar file becomes a unit of work. Backfill takes its own lower-priority queue to the same planner.</figcaption>
</figure>

Each of the roughly 160 US radars writes a new file every 4 to 10 minutes into NOAA's public bucket on AWS. The bucket announces every new file as a notification so nothing has to poll for it. I'd point those notifications at a queue that sits between files arriving and files being processed. When a storm triples the rate of files the queue just gets longer for a while instead of overwhelming the workers. A message that fails can be retried.

A planner reads from the queue and groups files into units of work. For radar a unit might be one radar for one hour. Before running a unit the planner asks the manifest whether it has already been done. The manifest is a small database table with one row per unit. It records which input files the unit used, which version of the code ran it, whether it finished and where its output went. If the same unit already finished with the same inputs and code it's skipped and that makes a file announced twice harmless. When the code changes the manifest also says exactly which units have to run again.

Workers pick up the units, read the files straight from the bucket and write their output. They'd run in the same region as the bucket so reading is fast and cheap. History goes in the same way. A backfill job lists the old files and sends the same kind of messages so four years of history runs through exactly the same code as a file that landed a minute ago. I think it needs its own lower-priority queue though. Otherwise a four-year backfill would sit in front of live data for days.

### Laying it out

<figure class="wide">
<svg class="fig" viewBox="0 0 640 420" role="img" aria-label="Raw files stay untouched. A worker turns each one into rows in a Parquet table partitioned by year and station and, for model grids, into Zarr chunks of 24 hours by a map tile. A sample station-hour row shows the value, NOAA's quality code and the original report time.">
<defs><marker id="ah7" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="dhead"/></marker></defs>
<text x="6" y="18" class="colhead" text-anchor="start">STATIONS</text>
<path d="M6,34 L48,34 L58,44 L58,98 L6,98 Z" class="shape"/>
<path d="M48,34 L48,44 L58,44" class="shape"/>
<text x="6" y="114" class="dmono" text-anchor="start">raw/isd/2024/</text>
<text x="6" y="128" class="dmono" text-anchor="start">72290023188.csv</text>
<rect x="146" y="46" width="112" height="40" rx="14" class="shape"/>
<text x="202.0" y="71.0" class="dtitle" text-anchor="middle">Standardize</text>
<path d="M62,66 L142,66" class="dedge" marker-end="url(#ah7)"/>
<text x="102" y="58" class="dlabel strong" text-anchor="middle">one unit</text>
<path d="M346,41 L346,91 A37.0,7 0 0 0 420,91 L420,41" class="shape"/>
<ellipse cx="383.0" cy="41" rx="37.0" ry="7" class="shape"/>
<text x="383" y="114" class="dmono" text-anchor="middle">standardized/</text>
<text x="383" y="128" class="dmono" text-anchor="middle">year=2024/</text>
<text x="383" y="142" class="dmono" text-anchor="middle">72290023188.parquet</text>
<path d="M262,66 L342,66" class="dedge" marker-end="url(#ah7)"/>
<text x="302" y="58" class="dlabel strong" text-anchor="middle">atomic write</text>
<line x1="420" y1="66" x2="452" y2="66" class="dthin"/>
<text x="456" y="30" class="dtitle sm" text-anchor="start">One row</text>
<text x="456" y="47" class="dmono" text-anchor="start">station:</text>
<text x="536" y="47" class="dmono" text-anchor="start">72290023188</text>
<text x="456" y="61" class="dmono" text-anchor="start">time:</text>
<text x="536" y="61" class="dmono" text-anchor="start">2024-01-01T01Z</text>
<text x="456" y="75" class="dmono" text-anchor="start">temp:</text>
<text x="536" y="75" class="dmono" text-anchor="start">14.4 °C</text>
<text x="456" y="89" class="dmono" text-anchor="start">dew point:</text>
<text x="536" y="89" class="dmono" text-anchor="start">8.9 °C</text>
<text x="456" y="103" class="dmono" text-anchor="start">NOAA code:</text>
<text x="536" y="103" class="dmono" text-anchor="start">5</text>
<text x="456" y="117" class="dmono" text-anchor="start">reported:</text>
<text x="536" y="117" class="dmono" text-anchor="start">00:51Z, FM-15</text>
<text x="6" y="176" class="dlabel" text-anchor="start">Partitioned by year then station: a year of ~3,000 US stations is ~3,000 files,</text>
<text x="6" y="191" class="dlabel" text-anchor="start">not millions of tiny ones.</text>
<line x1="6" y1="206" x2="634" y2="206" class="dthin"/>
<text x="6" y="230" class="colhead" text-anchor="start">GRIDS</text>
<path d="M6,248 L48,248 L58,258 L58,312 L6,312 Z" class="shape"/>
<path d="M48,248 L48,258 L58,258" class="shape"/>
<text x="6" y="328" class="dmono" text-anchor="start">HRRR GRIB2,</text>
<text x="6" y="342" class="dmono" text-anchor="start">one file an hour</text>
<rect x="146" y="260" width="112" height="40" rx="14" class="shape"/>
<text x="202.0" y="285.0" class="dtitle" text-anchor="middle">Rechunk</text>
<path d="M62,280 L142,280" class="dedge" marker-end="url(#ah7)"/>
<text x="102" y="272" class="dlabel strong" text-anchor="middle">a month</text>
<path d="M346,277.5 L416,277.5 L416,347.5 L346,347.5 Z" class="shape"/>
<path d="M346,277.5 L377.5,246 L447.5,246 L416,277.5" class="shape"/>
<path d="M416,277.5 L447.5,246 L447.5,316 L416,347.5" class="shape"/>
<path d="M346,277.5 L354.4,277.5 L354.4,347.5 L346,347.5 Z" class="hl"/>
<path d="M262,280 L342,280" class="dedge" marker-end="url(#ah7)"/>
<text x="302" y="272" class="dlabel strong" text-anchor="middle">write region</text>
<text x="384" y="362" class="dmono" text-anchor="middle">grids/t2m.zarr</text>
<text x="456" y="250" class="dtitle sm" text-anchor="start">One chunk</text>
<text x="456" y="267" class="dmono" text-anchor="start">shape:</text>
<text x="528" y="267" class="dmono" text-anchor="start">24h×512×512</text>
<text x="456" y="281" class="dmono" text-anchor="start">size:</text>
<text x="528" y="281" class="dmono" text-anchor="start">≈ 25 MB</text>
<text x="456" y="295" class="dmono" text-anchor="start">per year:</text>
<text x="528" y="295" class="dmono" text-anchor="start">≈ 4,400</text>
<text x="456" y="309" class="dmono" text-anchor="start"></text>
<text x="528" y="309" class="dmono" text-anchor="start">per variable</text>
<text x="6" y="392" class="dlabel" text-anchor="start">The US grid is 1,799 × 1,059 points so one day of one variable is 12 tiles:</text>
<text x="6" y="407" class="dlabel" text-anchor="start">big enough for object storage and small enough to read on its own.</text>
</svg>
<figcaption>Raw files stay as delivered. Points become a partitioned table and grids become chunks.</figcaption>
</figure>

What a worker writes depends on the shape of the data. A station file becomes rows in a table. Each row is one station-hour with the value in SI units and UTC along with NOAA's quality code and the time the report was really taken. The table is stored as Parquet in a folder per year with one file per station inside it. A year of about 3,000 US stations is then one folder of about 3,000 files with thousands of rows each. That keeps common questions to a few files and avoids millions of tiny ones.

Model grids go into Zarr. Zarr is a format for large arrays that cuts an array into chunks and stores each chunk as its own compressed object, with a small JSON file describing the whole array. For HRRR I'd try chunks of one day by a 512 × 512 tile (about 25 MB each). That should be big enough for object storage to handle efficiently and small enough to fetch on its own. The raw files are never edited so every one of these layers can be rebuilt from them if a rule or a bug changes.

### When data arrives late

<figure class="wide">
<svg class="fig" viewBox="0 0 640 330" role="img" aria-label="A QA run on 3 January judges an hour as pass with three of five neighbours reported and training run A uses it. Two days later the late neighbour files arrive, the neighbour check reruns and the 5 January run marks the same hour suspect. Training run B uses that one. Both stay reproducible.">
<defs><marker id="ah8" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="dhead"/></marker></defs>
<text x="6" y="26" class="dtitle sm" text-anchor="start">QA run on 3 Jan</text>
<text x="6" y="43" class="dmono" text-anchor="start">station:</text>
<text x="102" y="43" class="dmono" text-anchor="start">72290993115</text>
<text x="6" y="57" class="dmono" text-anchor="start">hour:</text>
<text x="102" y="57" class="dmono" text-anchor="start">2024-01-02T08Z</text>
<text x="6" y="71" class="dmono" text-anchor="start">neighbours:</text>
<text x="102" y="71" class="dmono" text-anchor="start">3 of 5 reported</text>
<text x="6" y="85" class="dmono" text-anchor="start">verdict:</text>
<text x="102" y="85" class="dmono" text-anchor="start">pass</text>
<path d="M276,37 L276,79 A32.0,7 0 0 0 340,79 L340,37" class="shape"/>
<ellipse cx="308.0" cy="37" rx="32.0" ry="7" class="shape"/>
<text x="308.0" y="102" class="dlabel" text-anchor="middle">qa/ runs</text>
<path d="M214,56 L272,56" class="dedge" marker-end="url(#ah8)"/>
<text x="420" y="26" class="dtitle sm" text-anchor="start">QA run on 5 Jan</text>
<text x="420" y="43" class="dmono" text-anchor="start">station:</text>
<text x="516" y="43" class="dmono" text-anchor="start">72290993115</text>
<text x="420" y="57" class="dmono" text-anchor="start">hour:</text>
<text x="516" y="57" class="dmono" text-anchor="start">2024-01-02T08Z</text>
<text x="420" y="71" class="dmono" text-anchor="start">neighbours:</text>
<text x="516" y="71" class="dmono" text-anchor="start">5 of 5 reported</text>
<text x="420" y="85" class="dmono" text-anchor="start">verdict:</text>
<text x="516" y="85" class="dmono" text-anchor="start">suspect</text>
<path d="M416,56 L344,56" class="dedge" marker-end="url(#ah8)"/>
<path d="M285,160 L321,160 L331,170 L331,216 L285,216 Z" class="shape"/>
<path d="M321,160 L321,170 L331,170" class="shape"/>
<text x="308" y="234" class="dlabel" text-anchor="middle">late neighbour files</text>
<path d="M308,156 L308,108" class="dedge" marker-end="url(#ah8)"/>
<text x="316" y="138" class="dlabel strong" text-anchor="start">rerun the neighbour check</text>
<circle cx="70" cy="156" r="7" class="shape"/>
<path d="M70,163 L70,186 M59,172 L81,172 M70,186 L61,202 M70,186 L79,202" class="dedge"/>
<text x="70" y="218" class="dlabel" text-anchor="middle">training run A</text>
<text x="70" y="232" class="dlabel" text-anchor="middle">uses the 3 Jan run</text>
<circle cx="560" cy="156" r="7" class="shape"/>
<path d="M560,163 L560,186 M549,172 L571,172 M560,186 L551,202 M560,186 L569,202" class="dedge"/>
<text x="560" y="218" class="dlabel" text-anchor="middle">training run B</text>
<text x="560" y="232" class="dlabel" text-anchor="middle">uses the 5 Jan run</text>
<path d="M70,150 L70,100" class="dedge dash" marker-end="url(#ah8)"/>
<path d="M560,150 L560,100" class="dedge dash" marker-end="url(#ah8)"/>
<text x="6" y="296" class="dlabel" text-anchor="start">Nothing is overwritten. Each QA run is written beside the last one</text>
<text x="6" y="311" class="dlabel" text-anchor="start">so both models can be traced to the exact verdicts they trained on.</text>
</svg>
<figcaption>Late neighbour files change a verdict. Both QA runs survive.</figcaption>
</figure>

From what I read not everything arrives on time. Some ocean buoys are only read when a ship visits and NOAA notes that values and their quality flags in its station archive can change as late data comes in ([GHCN-Daily](https://webarchive.library.unt.edu/web/20201218051500/https://www.ncdc.noaa.gov/ghcn-daily-description)). I'm not sure how often that happens for the stations I used. If it does it matters most for checks that compare a station with its neighbours. A QA run on 3 January might judge an hour when only three of five neighbours have reported and call it a pass. Once the other two arrive the comparison changes and the run on 5 January marks the same hour suspect. When I replayed my own checks month by month (notebook 06) 1.7% of verdicts changed like this. So I wouldn't overwrite QA results. Each run writes its results beside the last one and each training run records which QA run it used. A model can then always be traced back to the exact verdicts it trained on.

### Reading it back

<figure class="wide">
<svg class="fig" viewBox="0 0 640 290" role="img" aria-label="A training job reads one hour across the whole map from the map-chunked copy in one chunk. An analyst reads four years at one place from the time-series copy in one chunk. Reading each from the other copy would touch hundreds of chunks.">
<defs><marker id="ah9" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="dhead"/></marker></defs>
<circle cx="44" cy="50" r="7" class="shape"/>
<path d="M44,57 L44,80 M33,66 L55,66 M44,80 L35,96 M44,80 L53,96" class="dedge"/>
<text x="44" y="112" class="dlabel" text-anchor="middle">training job</text>
<text x="110" y="58" class="dlabel strong" text-anchor="middle">one hour</text>
<path d="M66,72 L162,72" class="dedge" marker-end="url(#ah9)"/>
<text x="110" y="92" class="dmono" text-anchor="middle">1 chunk</text>
<text x="110" y="106" class="dmono" text-anchor="middle">0.02 s</text>
<path d="M170,72.0 L250,72.0 L250,152.0 L170,152.0 Z" class="shape"/>
<path d="M170,72.0 L206.0,36 L286.0,36 L250,72.0" class="shape"/>
<path d="M250,72.0 L286.0,36 L286.0,116 L250,152.0" class="shape"/>
<path d="M170,72.0 L179.6,72.0 L179.6,152.0 L170,152.0 Z" class="hl"/>
<text x="226" y="172" class="dlabel" text-anchor="middle">map-shaped chunks</text>
<path d="M380,72.0 L460,72.0 L460,152.0 L380,152.0 Z" class="shape"/>
<path d="M380,72.0 L416.0,36 L496.0,36 L460,72.0" class="shape"/>
<path d="M460,72.0 L496.0,36 L496.0,116 L460,152.0" class="shape"/>
<path d="M380,104.0 L460,104.0 L460,115.2 L380,115.2 Z" class="hl"/>
<text x="436" y="172" class="dlabel" text-anchor="middle">time-series chunks</text>
<circle cx="610" cy="50" r="7" class="shape"/>
<path d="M610,57 L610,80 M599,66 L621,66 M610,80 L601,96 M610,80 L619,96" class="dedge"/>
<text x="610" y="112" class="dlabel" text-anchor="middle">analyst</text>
<text x="552" y="44" class="dlabel strong" text-anchor="middle">4 years at</text>
<text x="552" y="58" class="dlabel strong" text-anchor="middle">one place</text>
<path d="M588,72 L500,72" class="dedge" marker-end="url(#ah9)"/>
<text x="552" y="92" class="dmono" text-anchor="middle">1 chunk</text>
<text x="552" y="106" class="dmono" text-anchor="middle">0.04 s</text>
<text x="330" y="204" class="dlabel strong" text-anchor="middle">from the other copy</text>
<text x="330" y="220" class="dmono" text-anchor="middle">783 chunks · 7.7 s  /  1,461 chunks · 7.0 s</text>
<text x="6" y="256" class="dlabel" text-anchor="start">Both copies hold the same numbers. Each reader gets the layout that makes its question one read</text>
<text x="6" y="272" class="dlabel" text-anchor="start">and the work runs in the same cloud region as the bucket so only answers leave it.</text>
</svg>
<figcaption>The same numbers in two layouts. Each reader's question is one read in its own copy.</figcaption>
</figure>

Different readers ask different questions. A training job reads whole maps one hour at a time while an analyst might want four years at a single place. With chunks shaped like maps the training job's hour is one chunk but the analyst's four years is over a thousand. With chunks shaped like long time series it's the other way round. I measured this on four years of HRRR over California stored both ways and the difference was 200 to 450 times for the same data on the same disk. So I'd keep a copy for each kind of reader. Either way nothing gets loaded whole. A cluster in the same region as the bucket reads only the chunks a question needs and reduces them to something small like a mean before sending it back.

There's a lot here I'm still learning. I don't know yet when a day of data should count as final, how to tell a quiet format change at the source from a real change in the weather or how fast chunked data can be shuffled to keep a training job busy.

## What I took away

Every number in this project had a reason behind it that lived in the domain: why a station runs warm, why radar misses the rain under its beam, why a verdict changes a month later. I keep running into how much there is to know before the data makes sense and each answer opened two more questions. That's the part I want to keep going deeper on, both the science of these natural systems and how to engineer for them.

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

<p class="caption">Sources: NOAA <a href="https://www.ncei.noaa.gov/products/land-based-station/integrated-surface-database">ISD</a>, <a href="https://registry.opendata.aws/noaa-hrrr-pds/">HRRR</a>, <a href="https://registry.opendata.aws/noaa-nexrad/">NEXRAD</a> and <a href="https://registry.opendata.aws/noaa-mrms-pds/">MRMS</a> on AWS · Orbuch, <a href="https://naturalgeneralintelligence.ai/">Natural General Intelligence</a> · Rasp et al., <a href="https://arxiv.org/abs/2308.15560">WeatherBench 2</a> · <a href="https://www.weather.gov/asos/FAQ.html">NWS ASOS</a> · <a href="https://herbie.readthedocs.io/en/stable/user_guide/background/model-info/hrrr.html">HRRR run lengths</a> · <a href="https://rechunker.readthedocs.io/en/latest/algorithm.html">rechunker</a>.</p>
