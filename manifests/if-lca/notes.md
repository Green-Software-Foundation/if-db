# Impact Framework Life Cycle Assessment

## Useful links

[GHG spec for IT](https://ghgprotocol.org/sites/default/files/GHGP-ICTSG%20-%20ALL%20Chapters.pdf)


## Setup

Clone and install necessary dependencies, including IF, Github plugin and EcoCI plugins.

```
git clone https://github.com/Green-Software-Foundation/if
git clone https://github.com/Green-Software-Foundation/if-github-plugin
git clone https://github.com/Green-Software-Foundation/if-eco-ci-plugin

>>>>

cd if-github-plugin
npm i
npm run build
npm link

cd if-eco-ci-plugin
npm i
npm run build
npm link

cd if
npm i && npm link if-eco-ci-plugin if-github-plugin

```

## LCA Boundary

The following tree diagram shows the components that will be considered in this LCA. it also shows the grouping and nesting of components as they will be organized int he manifest file. For any element with children, we will receive a carbon value that aggregates the carbon values of all the children.


**In scope:**

```
- Code storage	
  - gh-storage-if
  - gh-actions-if
  - gh-storage-if-docs
  - gh-actions-if-docs
  - gh-storage-if-data
  - gh-actions-if-data
  
- servers	
  - serving website
    - CDN origin
    - CDN PoPs
    - gh-pages hosting
  - serving code via github
    - network transfer of `repo-size` bytes for each clone
  
- SaaS	
  - googlemeet
    - daily standup
    - monthly IF call
    - weekly office hours

  - slack
    - work hours daily
            end-user-internet-infra:
            children:
              router
              modem
              wifi 

- developers	
  - lighting
  - heating
  - monitor embodied
  - monitors operational
  - laptops embodied
  - laptops operational
  - travel
  - open source developers
  - routers, modems, wifi embodied & operational
  
- users	
  - laptops embodied
  - laptops operational
  - browsing site
  - running iF
  - embodied & operational carbon of internet infra incl. router, modem, wifi repeater
```

**Out of scope**

```
- food and other general life activities of devs
- any non-essential devices (no spotify, speakers, headphones etc)
- commuting (if any devs use workspaces, for example)
- use of dynamic call backgrounds
```


## Methodology

## Code storage

### Github

We take the repository size for each active IF repository and multiply it by a coefficient for energy per TB of storage obtained from Cloud Carbon Footprint. Then, we take a grid carbon intensity for the West coast of USA assuming our repositories are stored on Github's servers on the W coast of USA.

The size of the repository can be accessed using a github plugin that hits the `/repos` endpoint.

#### Pipeline

| Action                                             | Plugin type   | Instance name                | Inputs                                 | Outputs             |
| -------------------------------------------------- | ------------- | ---------------------------- | -------------------------------------- | ------------------- |
| Convert duration unit from seconds to hours        | `Coefficient` | `duration-to-hours`          | `duration`, `coefficient`              | `duration-in-hours` |
| Retrieve repo size from Github API                 | `Github`      | `get-github-repo-size-if`    | `repo-name`, `time`                    | `repo-size-gb`, `clones`    |
| Convert repo size unit from GB to TB               | `Coefficient` | `repo-size-unit-gb-to-tb`    | `repo-size-gb`, `coefficient`             | `repo-size-tb`      |
| Multiply storage in TB by coefficient in Watts/TBh | `Multiply`    | `storage-to-energy-in-watts` | `storage-tb`, `watt-hours-per-tb-hour` | `storage-power-w`     |
| Convert energy in W to Wh                          | `Multiply`    | `energy-w-to-wh`             | `storage-power-w`, `duration-in-hours`   | `storage-wh`        |
| Convert energy in Wh to kWh                        | `Coefficient` | `energy-wh-to-kwh`           | `storage-wh`, `coefficient`            | `storage-kwh`       |
| Convert energy to carbon                           | `Multiply`    | `energy-to-carbon`           | `storage-kwh`, `grid-carbon-intensity` | `carbon`            |

In manifest format, this pipeline looks as follows:

```yaml
- duration-to-hours
- get-github-repo-size-if
- repo-size-unit-gb-to-tb
- storage-to-energy-in-watts
- energy-w-to-wh
- energy-wh-to-kwh
- energy-to-carbon
```

#### Constants and coefficients

- `grid-carbon-intensity`: 765.5
- `watt-hours-per-tb-hour`: 1.2
- `coefficient`:
  - `duration-in-hours`: 1/3600
  - `energy-wh-to-kwh`: 0.001
  - `storage-in-gb-to-tb`: 0.001 


This pipeline will be repeated identically for each IF repository:

- if
- if-docs
- if-data

The only change required to rerun the pipeline for the different repositories is to change the `repo-name` parameter passed to the `if-github-plugin` plugin.


#### Assumptions

- Github servers are on W Coast USA and grid intensity is 765.49 (from Watt-time API in July 2024).
- The CCF coeffcient for relating TB storage to energy is appropriate ([link](https://www.cloudcarbonfootprint.org/docs/methodology/#storage))
- The repository size really represents the storage space on Github's ervers (the files are not compressed, or duplicated across multiple servers for redundancy)


#### Data

For this component we need data for Github storage size at daily timesteps. The Github API only provides access to historical data for the most recent month.


## Servers

### Serving source code from Github

The number of times the repo is cloned is available from the Github API, which we wrap in the `if-github-plugin`. We calculate the energy required to transfer this amount of data over the internet and multiply that by the number of clones in each timestep. 

We only do this for IF, as clones of our other repositories are so rare.


##### Pipeline

| Action                                                  | Plugin type   | Instance name               | Inputs                                                | Outputs                        |
| ------------------------------------------------------- | ------------- | --------------------------- | ----------------------------------------------------- | ------------------------------ |
| Grab size of source code repository                     | `Github`      | `get-github-repo-size-if`   | `repo-name`, `time`                                   | `repo-size`, `clones`          |
| Multiply site size by N clones                          | `Multiply`    | `total-data-transferred-gb` | `repo-size`, `clones`                                 | `total-data-transferred-in-gb` |
| Multiply data transferred by network energy coefficient | `Coefficient` | `network-energy-kwh`        | `total-data-transferred-gb`, `kwh-per-gb-transferred` | `energy`                       |
| Convert energy to carbon                                | `Multiply`    | `energy-to-carbon`          | `storage-kwh`, `grid-carbon-intensity`                | `carbon`                       |

In manifest format, this pipeline looks as follows:

```yaml
- get-github-repo-size-if
- total-data-transferred-gb
- network-energy-kwh
- energy-to-carbon
```

#### Constants and coefficients

- `grid-carbon-intensity`: 765.5
- `kwh-per-gb-network`: 0.001 ([CCF](https://www.cloudcarbonfootprint.org/docs/methodology/#networking))


#### Assumptions

- The US grid carbon intensity value is appropriate for the Github server
- The CCF networking coefficient is appropriate for converting data transferred to energy consumed
- The number of clones of our ancillary repositories is negligible (observations show 0 clones in past month)

#### Data

We want repo size and number of clones per day for a month. We cna only get one month's worth of data using the GHithub API.


### Serving website

Serving the IF website comprises several child components. These are:

- storing the static site data on gh pages
- networking-energy-to-serve-static-site-over-the-internet
- gh-pages-builds
- embodied-carbon-for-web-server
- embodied-carbon-for-cdn
- embodied-carbon-for-github-server

We'll treat these individually and then aggregate up to the parent node in IF.


### Storing the static site data on Github pages

Once the site is built, the static data is stored on servers waiting to transfer the site data to end users. There is an origin server that stores the whole static site, then a content delivery network (CDN) of servers storing a cached fraction of the total static site data for fast delivery.

We determined the total amount of data transferred when a new user loads the site using Google's PageSpeed API.

For the origin server, we applied the CCF's storage coefficient of 1.2 Wh/TBh, multiplying by the total duration in hours, and converting to kWh.

We then assumed the CDN is provided by Fastly - they have a CDN network with 46 Points of Presence (PoP). We do not have precise information about github-pages CDN set up, but we know they used Fastly in the no so distant past, so we assume they do today too. We have not been able to find quantitative estimates of the ratio of origin to PoP storage, only qualitative statements such as "much less" - we take a conservative guess and set the ratio to 10%. Therefore, for CDN storage we take 10% of the storage at the origin server, then multiply by 46 to account for all the PoPs.

The total energy used to store that static site is therefore

`data-transfer size + (46 * (transfer size / 10))`


#### Pipeline

We don't have a plugin for google analytics or google pagespeed api yet, so we will hard code our static site size after manually querying the pagespeed api with curl.

| Action                                                     | Plugin type   | Instance name                                   | Inputs                                                                                  | Outputs                                      |
| ---------------------------------------------------------- | ------------- | ----------------------------------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------- |
| Convert duration unit from seconds to hours                | `Coefficient` | `duration-to-hours`                             | `duration`, `coefficient`                                                               | `duration-in-hours`                          |
| Convert static site size in MB to TB                       | `Coefficient` | `static-site-size-mb-to-tb`                     | `static-site-size-in-mb`, `denominator`                                                 | `static-site-size-in-tb`                     |
| Convert static site size in TB to energy in W              | `Multiply`    | `storage-energy-static-site-at-origin-in-watts` | `static-site-size-in-tb`, `watt-hours-per-tb-hour`                                      | `storage-energy-static-site-at-origin-watts` |
| Convert storage energy in W to Wh                          | `Multiply`    | `storage-energy-static-site-at-origin-to-wh`    | `storage-energy-static-site-at-origin-watts` , `duration-in-hours`                      | `storage-energy-static-site-at-origin-wh`    |
| Convert storage energy in Wh to kWh                        | `Coefficient` | `storage-energy-static-site-at-origin-to-kwh`   | `storage-energy-static-site-at-origin-wh`, `coefficient`                                | `storage-energy-static-site-at-origin-kwh`   |
| Calculate storage energy in each CDN PoP                   | `Divide`      | `storage-energy-static-site-at-cdn-node-kwh`    | `storage-energy-static-site-at-origin-kwh`, `denominator`                               | `storage-energy-static-site-at-cdn-node-kwh` |
| Calculate storage energy across whole CDN                  | `Coefficient` | `storage-energy-static-site-whole-cdn-kwh`      | `storage-energy-static-site-at-cdn-node-kwh`, `coefficient`                             | `storage-energy-static-site-whole-cdn-kwh`   |
| Sum storage energy at origin and storage energy across CDN | `Sum`         | `sum-static-site-server-energy-components`      | `storage-energy-static-site-across-cdn-kwh`, `storage-energy-static-site-at-origin-kwh` | `energy`                                     |
| Convert energy to carbon                                   | `Multiply`    | `energy-to-carbon`                              | `storage-kwh`, `grid-carbon-intensity`                                                  | `carbon`                                     |

In manifest format, this pipeline looks as follows:

```yaml
- duration-in-hours
- static-site-size-mb-to-tb
- storage-energy-static-site-at-origin-in-watts
- storage-energy-static-site-at-origin-to-wh
- storage-energy-static-site-at-origin-to-kwh
- storage-energy-static-site-at-cdn-node-kwh
- storage-energy-static-site-whole-cdn-kwh
- sum-static-site-server-energy-components
- energy-to-carbon
```


#### Coefficients

- `grid-carbon-intensity`: 765.5
- `watt-hours-per-tb-hour`: 1.2
- `static-site-size-in-mb`: 1.4
- `denominator`:
  - `storage-energy-static-site-at-cdn-node-kwh`: 10
  - `static-site-size-mb-to-tb`: 1000000
- `coefficient`:
  - `storage-energy-static-site-at-origin-to-kwh`: 0.001
  - `storage-energy-static-site-whole-cdn-kwh`: 46


### Networking energy to serve static site over the internet

#### Pipeline

| Action                                             | Plugin type   | Instance name                                  | Inputs                                                               | Outputs                                             |
| -------------------------------------------------- | ------------- | ---------------------------------------------- | -------------------------------------------------------------------- | --------------------------------------------------- |
| Convert static site size from Mb to GB             | `Coefficient` | `static-site-size-in-gb`                       | `static-site-size-in-mb`,  `coefficient`                             | `static-site-size-in-gb`                            |
| Calculate networking energy to load site per visit | `Multiply`    | `network-energy-serving-static-site-per-view`  | `static-site-size-in-gb`, `kwh-per-gb-network`                       | `network-energy-serving-static-site-per-view-kwh`   |
| Multiply energy per visit by number of visits      | `Multiply`    | `network-energy-serving-static-site-total-kwh` | `network-energy-kwh-serving-static-site-per-view-kwh`, `site-visits` | `energy` |
| Convert energy to carbon                           | `Multiply`    | `energy-to-carbon`                             | `storage-kwh`, `grid-carbon-intensity`                               | `carbon`                                            |

In manifest format, this pipeline looks as follows:

```yaml
- static-site-size-in-gb
- network-energy-serving-static-site-per-view
- network-energy-serving-static-site-total
- energy-to-carbon
```

#### Coefficients

- `grid-carbon-intensity`: 765.5
- `kwh-per-gb-network`: 0.001
- `denominator`:
  - `storage-energy-static-site-at-cdn-node-kwh`: 10
- `coefficient`:
  - `static-site-size-mb-to-gb`: 0.001



### gh-pages-builds






## SaaS


### Conference calls

We assume some standard config for a conference call. The variable that changes is the numbner of participants. These calls are all assumed to be 30 minutes long.

Standups typically have 4 participants.
Monthly IF call now typically has ~8 participants.
Weekly office hours have one average 3 participants.
Weekly 1-1s between R&D lead and ED. 2 participants.
Weekly 1-1s between R&D lead and PM. 2 participants.
Ad-hoc calls between devs: 3 participants.

Instead of massively lengthening the manifest by including components for each type of call, we bundle them all together by summing the number of participants across the different calls and pretending they are all the same call in each week. This leads to the same final carbon value, but we just communicate it as "call carbon" rather than breaking it down further into individual call types. We can always copme back and decompose this later if we decide we want to be more granular. Therefore, `n-participants` will be equal to 12 and there will be one 30 min call with 12 participants each week.

The method used to calculate the carbon emissions for the call is as follows:

**Network**

We use a coefficient (0.07 kWh/GB) to convert the amount of data transferred over the network to energy in kWh. This coefficient was taken from Orbinger et al 2021.

The amount of energy transferred was assumed to be constant throughout the call. We used data from this report, selecting the transfer rate for group calls streaming at 720p resolution (1.35 GB/hr). this was scaled down to per minute by dividing by 60 (0.0225 GB/min).

Therefore, in each minute of the call, the energy used is:

`energy = 0.07 * 0.0225`

Then we convert this energy to carbon by multiplying by the grid carbon intensity.

Since the coefficient, grid carbon intensity and the data transfer rate are constant across all timesteps, the carbon emissions are uniform across the time series. Adding real observations of data transfer rates would be a very useful improvement to the manifest!

**Server**

This blog describes an investigation into the server config and CPU load for conference calls. They actually looked into Jitsi, which they assume to be an analog for Zoom.

Jitsi engineers did performance testing of Jitsi Meet. They showed that a quad-core Intel ® Xeon® E5-1620 v2 @ 3.70GHz CPU easily accommodates a 33 participant call using just 20% of the CPU capacity. This processor has a TDP of 130 W.

This is enough information to use the Teads power curve method to estimate energy used by the CPU in kWh. We can also look for a cloud instance that uses that processor and has some fairly substantial memory available - such as the AWS c5.2xlarge - let's assume that's what Zoom uses and look up the manufacturer's value for embodied carbon (0.0043 kgCO2/hour).

**Operational carbon**

The well-known Teads power curve pipeline was used to estimate energy from CPU utilization. This involves the following steps:

interpolate the power curve for the CPU utilization at each timestep, yielding the "cpu-factor"
multiply the processor TDP by the cpu-factor to yield power
apply time and unit conversions to yield energy in kWh
multiply by grid carbon intensity to yield carbon emissions per timestep

**Embodied carbon**

We use the manufacturer's embodied carbon value for a cloud instance we think is a decent analog for the Zoom server that serves the call. We then scale it for the proportion of the server lifespan accounted by the call (30 mins out of 4 years).

Climatiq estimates `0.0043 kg/Co2e/hr == 4.3g/hr` for this instance `4.3g/hr = 0.0716g/min`

assume zoom use their servers effectively and would not have a dedicated server for this call that only uses 20% of the cpu/mem.

`== 0.0716/5 gCO2/min == 0.00352 gCO2/min`

So this can be hardcoded as a default in the time series.

**User**

Embodied carbon

Each user is assumed to be using a Macbook Pro 13 (15"). It's embodied carbon is 159 kg CO2e according to the manufacturer. We scale this value by the proportion of the device lifespan (4 years) accounted for by the call (30 mins). The calculated value is multiplied by the number of participants for each day there is a call.

Operational carbon:

The well-known Teads power curve pipeline was used to estimate energy from CPU utilization. This involves the following steps:

interpolate the power curve for the CPU utilization at each timestep, yielding the "cpu-factor"
multiply the processor TDP by the cpu-factor to yield power
apply time and unit conversions to yield energy in kWh
multiply by grid carbon intensity to yield carbon emissions per timestep
The calculated value is multiplied by the number of participants.


#### Pipeline

**Network energy to transfer call data**

| Action                                              | Plugin type | Instance name                                                 | Inputs                                     | Outputs                  |
| --------------------------------------------------- | ----------- | ------------------------------------------------------------- | ------------------------------------------ | ------------------------ |
| Calculate energy from data transferred              | `Multiply`  | `networking-energy-to-transfer-call-data-to-each-participant` | `data-transferred`, `kwh-per-gb-network`   | `energy-per-participant` |
| Multiply data transferred by number of participants | `Multiply`  | `energy-for-data-transfer-to-all-call-participants`           | `energy-per-participant`, `n-participants` | `energy`                 |
| Convert energy to carbon                            | `Multiply`  | `energy-to-carbon`                                            | `energy`, `grid-carbon-intensity`          | `carbon`                 |

In manifest format, this pipeline looks as follows:

```yaml
- network-energy-to-transfer-call-data
- multiply-energy-by-participants
- energy-to-carbon
```


**Operational carbon of the remote server**


| Action                                                     | Plugin type   | Instance name                   | Inputs                                 | Outputs              |
| ---------------------------------------------------------- | ------------- | ------------------------------- | -------------------------------------- | -------------------- |
| Interpolate CPU power curve                                | `Interpolate` | `interpolate-power-curve`       | `cpu-util`                             | `cpu-factor`         |
| Apply cpu-factor to processor TDP                          | `Multiply`    | `cpu-factor-to-power-w`         | `cpu-factor`, `thermal-design-power`   | `cpu-power-w`        |
| Convert power to energy                                    | `Multiply`    | `cpu-power-to-energy-ws`        | `cpu-power-w`, `duration`              | `cpu-energy-ws`      |
| Convert energy to unit of kWh                              | `Divide`      | `cpu-energy-to-kwh`             | `cpu-power-Ws`, `denominator`          | `cpu-energy-kwh`     |
| Convert memory coefficient from CCF to unit of GB/duration | `Coefficient` | `memory-coefficient-correction` | `duration`, `coefficient`              | `memory-coefficient` |
| Apply memory coefficient to memory utilization             | `Multiply`    | `memory-util-to-energy-kwh`     | `memory-gb`, `memory-coefficient`      | `memory-energy-kwh`  |
| Sum energy components                                      | `Sum`         | `sum-energy-components-calls`   | `memory-energy-kwh`, `cpu-energy-kwh`  | `energy`             |
| Convert energy to carbon                                   | `Multiply`    | `energy-to-carbon`              | `storage-kwh`, `grid-carbon-intensity` | `carbon`             |

In manifest format, this pipeline looks as follows:

```yaml
- interpolate-power-curve
- cpu-factor-to-power-w
- cpu-power-to-energy-ws
- cpu-energy-to-kwh
- memory-coefficient-correction
- memory-energy
- sum-energy-components
- energy-to-carbon
```



**Operational carbon of the user devices**

This is identical to the operational carbon for the server except for the TDP of the processor and the grid-carbon-intensity given..


| Action                                                     | Plugin type   | Instance name                     | Inputs                                 | Outputs              |
| ---------------------------------------------------------- | ------------- | --------------------------------- | -------------------------------------- | -------------------- |
| Interpolate CPU power curve                                | `Interpolate` | `interpolate-power-curve`         | `cpu-util`                             | `cpu-factor`         |
| Apply cpu-factor to processor TDP                          | `Multiply`    | `cpu-factor-to-power-w`           | `cpu-factor`, `thermal-design-power`   | `cpu-power-w`        |
| Convert power to energy                                    | `Multiply`    | `cpu-power-to-energy-ws`          | `cpu-power-w`, `duration`              | `cpu-energy-ws`      |
| Convert energy to unit of kWh                              | `Divide`      | `cpu-energy-to-kwh`               | `cpu-power-Ws`, `denominator`          | `cpu-energy-kwh`     |
| Convert memory coefficient from CCF to unit of GB/duration | `Coefficient` | `memory-coefficient-correction`   | `duration`, `coefficient`              | `memory-coefficient` |
| Apply memory coefficient to memory utilization             | `Multiply`    | `memory-util-to-energy-kwh`       | `memory-gb`, `memory-coefficient`      | `memory-energy-kwh`  |
| Sum energy components                                      | `Sum`         | `sum-energy-components-calls`     | `memory-energy-kwh`, `cpu-energy-kwh`  | `energy`             |
| Convert energy to carbon                                   | `Multiply`    | `energy-to-carbon`                | `storage-kwh`, `grid-carbon-intensity` | `carbon`             |
| Multiply carbon by number of participants                  | `Multiply`    | `multiply-carbon-by-participants` | `carbon`, `n-participants`             | `carbon`             |

In manifest format, this pipeline looks as follows:

```yaml
- interpolate-power-curve
- cpu-factor-to-power-w
- cpu-power-to-energy-ws
- cpu-energy-to-kwh
- memory-coefficient-correction
- memory-energy
- sum-energy-components
- energy-to-carbon
- multiply-carbon-by-participants
```

**Embodied carbon of the remote server**

There is no pipeline to execute for this component, as we calculated the `carbon` offline and simply add it to the time series for the days the calls happened.


**Embodied carbon of the user devices**

There is no pipeline to execute for this component, as we calculated the `carbon` offline and simply add it to the time series for the days the calls happened.



#### Coefficients

**Network energy to transfer call data**

- `grid-carbon-intensity`: 765.5
- `kwh-per-gb-network`: 0.001
- `n-participants`: 12
  

**Operational carbon of the remote server**

- `grid-carbon-intensity`: 765.5
- `denominator`:
  - `cpu-energy-to-kwh`: 3600000
- `coefficient`:
  - `memory-coefficient-correction`: 0.000006533333
  

**Operational carbon of the user devices**

- `grid-carbon-intensity`: 265.5
- `denominator`:
  - `cpu-energy-to-kwh`: 3600000
- `coefficient`:
  - `memory-coefficient-correction`: 0.000006533333
- - `n-participants`: 12

#### Data

This requires daily-interval time series data. Not all days will have calls. The manifest aggregation feature will handle aggregating calls over the month.


### Slack

Slack is constantly performing operations in the background if the app is running, even if it is not actively being used. we use Slack for our day-to-day communications. We estimate that each user sends around 20 slack messages per day,a nd hasd the application running in the background throughout the working day.

We borrow data from the [Greenspector messaging app carbon estimate](https://greenspector.com/en/direct-messaging-business-apps/#methodology) in this LCA. Note that these values were measured using a Samsung S7 mobile - we are simply assuming that the devices being used by our developers to run Slack are the same. Specifically, we account for:

```
Launching the app: 0.039 gCO2e
Opening a chat: 0.01 gCO2e
Sending a text message: 0.058 g CO2e
Sending an image: 0.051 g CO2e
Sending attachment: 0.057 gCO2e
```

We assume that each developer does the following each day:

```
Launch app
Open a chat
Send 20 text messages
Send one attachment
Send one image
```

We also assume that the app is running for 7 working hours each day, "idling" in the background. [Greenspector](https://greenspector.com/en/direct-messaging-business-apps/#methodology) observed that Slack used 26 gCO2e/day in the background.


We'll simply add these values to each daily timestep for each developer on the team.


## Developers

- developers	
  - lighting
  - heating
  - monitor embodied
  - monitors operational
  - mouse embodied
  - laptops embodied
  - laptops operational
  - travel
  - open source developers
  - routers, modems, wifi embodied & operational


### Lighting:

Average UK household uses 3941kWh electricity per year, of which 15% is due to lighting (https://www.ovoenergy.com/guides/energy-guides/how-much-electricity-does-a-home-use).

We assume 1/5 of all the rooms in the house are used for WFH purposes, giving `(3941 x 0.15) x 0.2 = 118.29 kWh/yr` for a remote working developer on lighting alone. Scaling to a working day gives `118.29/365 = 0.32 kWh`

A laptop uses approximately 0.05kWh per working day (https://www.ovoenergy.com/guides/energy-guides/how-much-electricity-does-a-home-use).
EDF electricity has grid intensity of 87 gCO2/kWh (2024) (https://www.edfenergy.com/fuel-mix)
So a developer working a normal day will emit `0.05 * 87 =  4.35 gCO2`

We will add this to each timestep for each developer.

#### Pipeline

There is no pipeline to execute for this component, as we calculated the `carbon` offline and simply add it to the time series for the days the calls happened.


### Heating

A typical UK household uses 11500 kWh of natural gas/year (https://www.ofgem.gov.uk/average-gas-and-electricity-usage). We scale this value down to 1/5 of the total space in the house which is used for home working = `11500 * 0.2 = 2300 kWh/yr`.

We scale the electricity and gas usage by the amount of time covered by each timestep in the manifest.

Natural gas is 201.96 kg Co2 eq / MWh energy (https://ourworldindata.org/grapher/carbon-dioxide-emissions-factor?tab=table)
This equates to `0.20196 kg CO2/ kWh = 201.96 gCO2 / kWh`

So heating a house for WFH purposes is `2300 / 365 / 3 = 2.1 kWh/working day` (assuming 8 hours a day: 8 * 3 = 24)
`2.1 kWh/day * 201.96 = 0.000424gC02/day`

We will add this to each timestep for each developer.

#### Pipeline

There is no pipeline to execute for this component, as we calculated the `carbon` offline and simply add it to the time series for the days the calls happened.


## Developer laptop embodied carbon

embodied carbon for laptops is from https://github.com/rarecoil/laptop-co2e

for a Macbook Pro 14" the embodied carbon is 196.56 kg CO2 eq (i.e. 196560 g) covering a 4 year lifespan. This equates to 0.006210865211549674 g/s.
Say work accoutns for 8 hours a day, 5 days a week, the embodied carbon allocated to work per day is 178.87g/day or 894g /working week.

We add this for each developer for each day.

#### Pipeline

There is no pipeline to execute for this component, as we calculated the `carbon` offline and simply add it to the time series for the days the calls happened.



## Additional screens/keyboards/mouses

- embodied and operational carbon

- monitor embodied is ~526 kgCO2 +/- 130 kgCO2 for Dell P-series including usage across lifespan. Removiong usage (30%) gives embodied carbon of `526*0.7 = 368.2 kgCO2`: https://www.dell.com/en-us/dt/corporate/social-impact/advancing-sustainability/climate-action/product-carbon-footprints.htm#tab0=2
- This assumes 6 year lifespan, so `368/6/365 = embodied-per-day = 0.16 kgCO2 = 160gCO2/day`

- usage is 65W for 8 hours per day = `65*8 = 520Wh = 0.52 kWh/day`
- carbon due to screen usage is `use x carbon-intensity = 0.52 * 87 = 45.24 gCO2e/day`

We'll add this to each developer's time series for each day.

#### Pipeline

There is no pipeline to execute for this component, as we calculated the `carbon` offline and simply add it to the time series for the days the calls happened.



## Travel

Train travel is 0.37 kgCO2eq per passenger per 10km (https://www.gov.uk/government/publications/greenhouse-gas-reporting-conversion-factors-2020)
Flying is 1.46 kg per passenger per 10 km (https://www.gov.uk/government/publications/greenhouse-gas-reporting-conversion-factors-2020)
Motorbike is 1.13 kg per passenger per 10 km (https://www.gov.uk/government/publications/greenhouse-gas-reporting-conversion-factors-2020)
Bus is 1.03 kg per passenger per 10 km (https://www.gov.uk/government/publications/greenhouse-gas-reporting-conversion-factors-2020)
Bike is 0 kg Co2eq (https://www.gov.uk/government/publications/greenhouse-gas-reporting-conversion-factors-2020)

We will add conference travel for each developer using these coefficients.

#### Pipeline

There is no pipeline to execute for this component, as we calculated the `carbon` offline and simply add it to the time series for the days the calls happened.


## Product disposal

None required - the cost of deleting the IF repo from a local computer is negligible and does not generate any electrical waste.
IF is not tied to any specific hardware that become suseless withouyt IF software being installed on it.
This might not be the case with certain firmware, for example.
