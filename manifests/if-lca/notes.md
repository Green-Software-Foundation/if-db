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

**in scope:**

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
              router:
              modem:
              wifi:   
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
  
- users	
  - laptops embodied
  - laptops operational
  - browsing site
  - running iF
  - embodied & operational carbon of internet infra incl. router, modem, wifi repeater


**out of scope**

- food and other general life activities of devs
- any non-essential devices (no spotify, speakers, headphones etc)
- commuting (if any devs use workspaces, for example)
- use of dynamic call backgrounds




## Methodology

## Code storage

### Github

We take the repository size for each active IF repository and multiply it by a coefficient for energy per TB of storage obtained from Cloud Carbon Footprint. Then, we take a grid carbon intensity for the West coast of USA assuming our repositories are stored on Github's servers on the W coast of USA.

The size of the repository can be accessed using a github plugin that hits the `/repos` endpoint.

#### Pipeline

| Action                                             | Plugin type   | Instance name                | Inputs                                 | Outputs             |
| -------------------------------------------------- | ------------- | ---------------------------- | -------------------------------------- | ------------------- |
| Convert duration unit from seconds to hours        | `Coefficient  | `duration-to-hours`          | `duration`, `coefficient`              | `duration-in-hours` |
| Retrieve repo size from Github API                 | `Github`      | `get-github-repo-size-if`    | `repo-name`, `time`                    | `size`, `clones`    |
| Convert repo size unit from GB to TB               | `Coefficient` | `repo-size-unit-gb-to-tb`    | `repo-size`, `coefficient`             | `repo-size-tb`      |
| Multiply storage in TB by coefficient in Watts/TBh | `Multiply`    | `storage-to-energy-in-watts` | `storage-tb`, `watt-hours-per-tb-hour` | `storage-watts`     |
| Convert energy in W to Wh                          | `Multiply`    | `energy-w-to-wh`             | `storage-watts`, `duration-in-hours`   | `storage-wh`        |
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


### Serving website

Serving the IF website comprises several child components. These are:

- storing the static site data on gh pages
- networking-energy-to-serve-static-site-over-the-internet
- gh-pages-builds
- embodied-carbon-for-web-server
- embodied-carbon-for-cdn
- embodied-carbon-for-github-server

We'll treat these individually and then aggregate up to the parent node in IF.



**Storing the static site data on Github pages**

Once the site is built, the static data is stored on servers waiting to transfer the site data to end users. There is an origin server that stores the whole static site, then a content delivery network (CDN) of servers storing a cached fraction of the total static site data for fast delivery.

We determined the total amount of data transferred when a new user loads the site using Google's PageSpeed API.

For the origin server, we applied the CCF's storage coefficient of 1.2 Wh/TBh, multiplying by the total duration in hours, and converting to kWh.

We then assumed the CDN is provided by Fastly - they have a CDN network with 46 Points of Presence (PoP). We do not have precise information about github-pages CDN set up, but we know they used Fastly in the no so distant past, so we assume they do today too. We have not been able to find quantitative estimates of the ratio of origin to PoP storage, only qualitative statements such as "much less" - we take a conservative guess and set the ratio to 10%. Therefore, for CDN storage we take 10% of the storage at the origin server, then multiply by 46 to account for all the PoPs.

The total energy used to store that static site is therefore

`data-transfer size + (46 * (transfer size / 10))`


#### Pipeline

We don't have a plugin for google analytics or google pagespeed api yet, so we will hard code our static site size after manually querying the pagespeed api with curl.

| Action                                      | Plugin type   | Instance name               | Inputs                                  | Outputs                  |
| ------------------------------------------- | ------------- | --------------------------- | --------------------------------------- | ------------------------ |
| Convert duration unit from seconds to hours | `Coefficient` | `duration-to-hours`         | `duration`, `coefficient`               | `duration-in-hours`      |
| Convert static site size in MB to TB        | `Coefficient` | `static-site-size-mb-to-tb` | `static-site-size-in-mb`, `denominator` | `static-site-size-in-tb` |


In manifest format, this pipeline looks as follows:

```yaml
- duration-in-hours
- static-site-size-in-tb
- storage-energy-static-site-in-watts
- storage-energy-in-wh-static-site
- storage-energy-to-kwh-static-site
- storage-energy-per-cdn-node
- storage-energy-for-entire-cdn
- sum-serving-static-site-energy-components
- energy-to-carbon
- sci
```



**Networking energy to serve static site over the internet**

#### Pipeline

| Action                              | Plugin type | Instance name             | Inputs              | Outputs               |
| ----------------------------------- | ----------- | ------------------------- | ------------------- | --------------------- |
| Grab size of source code repository | `Github`    | `get-github-repo-size-if` | `repo-name`, `time` | `repo-size`, `clones` |

In manifest format, this pipeline looks as follows:

```yaml
- static-site-size-in-gb
- network-energy-serving-static-site-per-view
- network-energy-serving-static-site-total
- energy-to-carbon
- sci
```



*********************
SCRATCH NOTES:: WIP!
**********************


    gh-pages-builds:
      pipeline:
        compute:
          - interpolate-power-curve
          - build-stage-cpu-factor-to-wattage
          - build-stage-cpu-wattage-times-build-duration
          - build-stage-cpu-wattage-to-energy-kwh
          - memory-coefficient-correction # memory coefficient in CCF is given /GB/hr - we want to convert to /GB/run-time-duration then x by run duration in s
          - memory-energy-per-build
          - sum-build-energy-components
          - build-energy-total
          - energy-to-carbon
          - sci

    embodied-carbon-for-web-server:
      pipeline:
        compute:
          - embodied-carbon-web-server-origin
          - sci

    embodied-carbon-for-cdn:
      pipeline:
        compute:
          - embodied-carbon-web-server-cdn
          - sci

    embodied-carbon-for-github-server:
      pipeline:
        compute:
          - embodied-carbon-github
          - sci


  - serving website
    - CDN origin
    - CDN PoPs
    - gh-pages hosting
  - serving code via github
    - network transfer of `repo-size` bytes for each clone







### Github actions

We use the Green Coding plugin to measure the energy of our actions. The methodology is explained here: [EcoCI](https://www.green-coding.io/projects/eco-ci/).

Once the app is integrated into our Github Actions we can request data for a date range using an APi which we wrap in an IF plugin.


### WFH

#### Lighting:

Average UK household uses 3941kWh electricity per year, of which 15% is due to lighting (https://www.ovoenergy.com/guides/energy-guides/how-much-electricity-does-a-home-use).

We assume 1/5 of all the rooms in the house are used for WFH purposes, giving `(3941 x 0.15) x 0.2 = 118.29 kWh/yr` for a remote working developer on lighting alone. Scaling to a working day gives `118.29/365 = 0.32 kWh`

A laptop uses approximately 0.05kWh per working day (https://www.ovoenergy.com/guides/energy-guides/how-much-electricity-does-a-home-use).
EDF electricity has grid intensity of 87 gCO2/kWh (2024) (https://www.edfenergy.com/fuel-mix)
So a developer working a normal day will emit `0.05 * 87 =  4.35 gCO2`


A typical UK household uses 11500 kWh of natural gas/year (https://www.ofgem.gov.uk/average-gas-and-electricity-usage). We scale this value down to 1/5 of the total space in the house which is used for home working = `11500 * 0.2 = 2300 kWh/yr`.

We scale the electricity and gas usage by the amount of time covered by each timestep in the manifest.

Natural gas is 201.96 kg Co2 eq / MWh energy (https://ourworldindata.org/grapher/carbon-dioxide-emissions-factor?tab=table)
This equates to `0.20196 kg CO2/ kWh = 201.96 gCO2 / kWh`

So heating a house for WFH purposes is `2300 / 365 / 3 = 2.1 kWh/working day` (assuming 8 hours a day: 8 * 3 = 24)
`2.1 kWh/day * 201.96 = 0.000424gC02/day`



## Developer laptop embodied carbon

embodied carbon for laptops is from https://github.com/rarecoil/laptop-co2e

for a Macbook Pro 14" the embodied carbon is 196.56 kg CO2 eq (i.e. 196560 g) covering a 4 year lifespan. This equates to 0.006210865211549674 g/s.
Say work accoutns for 8 hours a day, 5 days a week, the embodied carbon allocated to work per day is 178.87g/day or 894g /working week.

`embodied + heating + laptop + lighting = total`

`178.87 + 424  + 4.35 + 0.32 = 607.54 g/day`



## Additional screens/keyboards/mouses

- embodied and operational carbon

- monitor embodied is ~526 kgCO2 +/- 130 kgCO2 for Dell P-series including usage across lifespan. Removiong usage (30%) gives embodied carbon of `526*0.7 = 368.2 kgCO2`: https://www.dell.com/en-us/dt/corporate/social-impact/advancing-sustainability/climate-action/product-carbon-footprints.htm#tab0=2
- This assumes 6 year lifespan, so `368/6/365 = embodied-per-day = 0.16 kgCO2 = 160gCO2/day`

- usage is 65W for 8 hours per day = `65*8 = 520Wh = 0.52 kWh/day`
- carbon due to screen usage is `use x carbon-intensity = 0.52 * 87 = 45.24`


## Travel

Train travel is 0.37 kgCO2eq per passenger per 10km (https://www.gov.uk/government/publications/greenhouse-gas-reporting-conversion-factors-2020)
Flying is 1.46 kg per passenger per 10 km (https://www.gov.uk/government/publications/greenhouse-gas-reporting-conversion-factors-2020)
Motorbike is 1.13 kg per passenger per 10 km (https://www.gov.uk/government/publications/greenhouse-gas-reporting-conversion-factors-2020)
Bus is 1.03 kg per passenger per 10 km (https://www.gov.uk/government/publications/greenhouse-gas-reporting-conversion-factors-2020)
Bike is 0 kg Co2eq (https://www.gov.uk/government/publications/greenhouse-gas-reporting-conversion-factors-2020)


## Product disposal

None required - the cost of deleting the IF repo from a local computer is negligible and does not generate any electrical waste.
IF is not tied to any specific hardware that become suseless withouyt IF software being installed on it.
This might not be the case with certain firmware, for example.

## if-docs website

Since writing docs is part of our normal developer work-day, and we typically do it in the same IDE as we use to write code, we rol it in with "development" rather than accounting separately. However, the docs are hosted on n independent repository that has its own storage, traffic and GH actions, plus there is hosting and viewership to account for too.

We here use the CLI tool "greenframe" to analyse the website's carbon emissions: https://docs.greenframe.io/commands
the CLi estimates the usage as: 

12.807 mg eq. co2 ± 3.6% (28.976 mWh)
== 0.012807 g CO2eq

Alternatives include using the GWF CO2js model (which we have a plugin for)




### github storage and traffic
- carbon due to GH storage and downloads
### ci/cd
- carbon for PRs/preview deploys
### hosting/deployment
- carbon to host site on server
### viewers
- carbon due to each end uyser viewing the website on desktop/mobile




## End users


## Other

employee travel
employee commuting
purchased goods and services




 
## TODOs

- create plugin for querying EcoCI data
  - wraps 

```
curl -X 'GET' 'https://api.green-coding.io/v1/ci/measurements?repo=https%3A%2F%2Fgithub.com%2Fgreen-coding-berlin%2Fdjango&branch=main&workflow=usage_scenario.yml&start_date=1720359036238303&end_date=1720360753137587' -H 'accept: application/json'

```
- create plugin that wraps GH metrics API:
```
gh api -H "Accept: application/vnd.github+json" -H "X-GitHub-Api-Version: 2022-11-28"  /repos/Green-Software-Foundation/if
```
