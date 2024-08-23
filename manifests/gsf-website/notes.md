# How we calculated the SCI score for the GSF website

## Introduction

We used the Impact Framework to calculate the Software Carbon Intensity (SCI) for [greensoftware.foundation](https://greensoftware.foundation) in gCO2eq/visit.

There are several tools out there that measure the carbon emissions of websites, such as:

[Beacon](https://digitalbeacon.co/) uses SWD model
[Website Carbon Calculator](https://www.websitecarbon.com/how-does-it-work/) uses SWD under the hood
[Ecograder](https://ecograder.com/how-it-works) uses SWD via CO2js

All of these use the Sustainable Web Design model, which is a top-down calculation that scales down whole-internet-scale measurements of energy usage based on the amount of data transferred by a website. This is a common approach to measuring website carbon emissions, and there are good reasons why. However, the top down approach has substantial limitations (see https://davidmytton.blog/approaches-to-calculating-network-website-energy-and-carbon/ for a good explanation).

For this reason, we wanted to try to build our own estimate using a bottom-up approach. This allowed us to choose what components to include, fine-control over the assumptions we made, and more opportunities to include real observations of our systems.

These notes will explain the approach taken, the design decisions and assumptions.

You are encouraged to take the manifest file, do a better job and raise a PR!


## About the GSF website

The GSF website is developed using Docusaurus, a React framework. The source code is hosted on Github and it is built and served using Netlify.

You can see the site at https://greensoftware.foundation and the github at https://github.com/Green-Software-Foundation/gsf-website

## Application Boundary

We include the following components in our calculations:

- Github storage
- Netlify builds
- Netlify static site storage
- Storage across content delivery network
- Data transferred over network
- End users viewing site in browser
- Embodied carbon of Github server (for storing ans serving source code)
- Embodied carbon of static site servers (for storing and serving static site)
- Embodied carbon of end user devices (for viewing content)


## Calculations

### Github storage

We first used the Github API to determine the storage size for the source code repository and the number of times the repository has been cloned in the most recent month. IF has a plugin that calls this API, here: https://github.com/Green-Software-Foundation/if-github-plugin.

The repository size value returned by the Github API plugin is in Megabytes. We first convert this to TB by multiplying by a factor of 0.000001. Then we multiply the size in Terabytes by a constant 1.2 W/TB for storage energy obtained from Cloud Carbon Footprint (https://www.cloudcarbonfootprint.org/docs/methodology/#storage) to yield power in Watts. We then convert W into Wh by multiplying by the storage duration in hours, and into kWh by dividing by 1000.

This yields the energy, in kWh, consumed by storing our source code on Github.


## Netlify builds

We first looked at our Netlify stats to see how many times the website was built in the most recent month.
We had to establish some heuristics to approximate the hardware used to build the site, in order to then estimate the energy.

As of 2022 netlify allow up to 8GB memory and 4 cpu cores to be allocated to a build: https://www.netlify.com/blog/up-to-40-faster-build-times/

We don't know the proportion of these resources we actually use, but we can assume it's not all of it.
Building the site on my local machine required 100% of 6 CPU cores for about 4 seconds, and 1.5% available meemory (32GB * 0.015 = 0.48 GB). Let's assume on a machine with 4 cores, it will run for 50% longer - i.e. 6 seconds using 4 CPUs at 100% utilization.

We can then work out the energy consumed during this process using the processor power curve.

We'll found a 4CPU, 8GB Azure VM and assumed it was what was used for our website builds (Azure A4 v2). We looked up the TDP of the processor used in that VM instance in our `if-data` repository (205 W - the max from these possible processors: Intel® Xeon® Platinum 8272CL,Intel® Xeon® 8171M 2.1 GHz,Intel® Xeon® E5-2673 v4 2.3 GHz,Intel® Xeon® E5-2673 v3 2.4 GHz). 

We interpolated the standard CPU power curve provided here: https://medium.com/teads-engineering/building-an-aws-ec2-carbon-emissions-dataset-3f0fd76c98ac to obtain the "tdp-factor" for a cpu at 100% capacity.

We then multiplied this by the processor's actual TDP, yielding the power used by the CPU during the build, in W.
This was then multiplied by the build duration to yield energy, and then scaled to units of kWh. 

We then accounted for the memory used durign the build process, which we estimated to be 0.48 GB. We multiplied this by a memory-energy factor published by Cloud Carbon Footprint (0.000392 kWh/GBh). We scaled the coeffcient to be in units of kWh/GBs and then multiplied by our build duration to yield kWh/build due to memory.

The sum of the CPU energy and memory energy gave the total build energy in kWh.

This was then multiplied by the number of times the build process was executed in the most recent month, giving the energy used to build the static site across the previous month, in units of kWh.


## Static site storage

Once the build process is finished, the static site is stored on servers waiting to transfer the site data to end users. There is an origin server that stores the whole static site, then a content delivery network (CDN) of servers storing a fraction of the total static site data for fast delivery.

We determined the total amount of data transferred when a new user loads the site using Google's PageSpeed API.
For the origin server, we applied the CCF's storage coefficient of 1.2 Wh/TBh, multiplying by the total duration in hours, and converting to kWh.

We then assumed the CDN is provided by Fastly - they have a CDN network with 46 Points of Presence (PoP). We have not been able to find quantitative estimates of the ratio of origin to PoP storage, only qualitative statements such as "much less" - we take a conservative guess and set the ratio to 10%. Therefore, for CDN storage we take 10% of the storage at the origin server, then multiply by 46 to accoutn for all the PoPs.

The total energ used to store that static site is therefore

`data-transfer size + (46 * (transfer size / 10))`


### Data transferred over network

To determine the energy used to transfer the site data between the servers and the end users, we simply multiply the number of site visits (obtained from Google analytics) by the volume of data transferred and a coefficient in kWh/GB transferred obtained from Cloud Carbon Footprint (https://www.cloudcarbonfootprint.org/docs/methodology/#networking).


### End users viewing site in browser

We were not able to devise a better approach than using the SWD coefficient provided here: https://sustainablewebdesign.org/estimating-digital-emissions/#toc-2

The coefficient we used was 0.081 kWh/GB. We multiplied this by the size of the static site and the number of visits in the most recent month to yield an estimate of the total energy used to view the site in the browser.


### Embodied carbon

#### Static site server

We don't know what hardware is used to serve the site, so we'' make some educated guesses.

Lets' say the origin server meets the minimum requirements recommended for a Wordpress web server.
https://kinsta.com/blog/wordpress-server-requirements/

- Disk space: At least 1 GB - 1 x SSD- let's assume 1GB.
- RAM (Random Access Memory): At least 512 MB 
- CPU (Central Processing Unit): At least 1.0 GHz 1 x CPU

This matches the definition of a "baseline" server according to the CCF embodied carbon calculations, for which they assume 1M g of emissions (1 tonne).

Let's also use the same specs for the CDN PoP servers, of which there are 46.

We scale the server's total embodied emissions (1M g CO2eq) by the portion of the server's lifespan we are accountable for. We put the whole lifespan of the server at 5 years, and our accountable period as 1 month.

To scale by this, we divide 1 day (our timestep resolution) in seconds by 4 years in seconds:  

`86400 / 145152000 = 0.000595`

Our website uses only 0.8 MB of server storage space for the static site, which is 0.000008% of the total (1GB).
We do not scale further by CPU or RAM usage because we cannot get good estimates for this.

`1000000 * 0.000595 * 0.0008 = 0.476 gCO2eq` for the origin server

So we end up with 0.000476 g for the origin server, and 0.0000476g per CDN node (10% of origin).
There are 46 CDN servers, so `0.0476 * 46 = 2.1 g CO2e` for the whole CDN network.

The embodied carbon of the static site servers is therefore `0.476 + 2.1 = 2.576 gCO2eq `  for each day.


#### Github server

Now for Github, whose servers are much larger. We use the Github server to store and serve our source code.

Let's just create a big server that we assume Github to be using. To spec this server we'll look at a storage focused option from Azure, since Microsoft own Github.

The largest storage-optimized VM on Azure is the L series - we use this as the base from which we build our assumed Github server.

"Ls-series instances are storage optimized virtual machines for low latency workloads such as NoSQL databases (e.g. Cassandra, MongoDB, and Redis). Ls-series offer up to 32 CPU cores, using the Intel® Xeon® processor E5 v3 family with 8-GiB random-access memory (RAM) per core, and from 768 GB to 6 TB of local solid state drive (SSD) disk. This instance provides Premium Storage support."

6TB is clearly far too little to account for all of Github. Let's say the real server has 30TB of storage. We saw online that the top 400,000 repositories account for 14TB of storage, and that the Arctic storage vault accounted for 21 TB, so 30TB seems realistic. It's arbitrary anyway, because CCF assumes each SSD to have a constant embodied carbon regardless of the size, and there are single SSDs that offer 50 TB storage and greater.

However, for Github it seems like there would need to be more CPUs than are supported on the greatest L series consumer VM, so let's increase the max in the same ratio as we have for SSDs. 

L-series max SSD = 6TB, we are using 30TB, an increase of 5x. Let's multiply the max CPUs by 5 too, giving `32 * 5  = 160 CPUs`. L-series allow 8GB RAM per core, so let's borrow that too: `8GB * 160 cores = 1280 GB RAM`. This is pretty large, let's go with it until we have some better information.

Now we can use the CCF method to estimate embodied carbon for this hypothetical server:

```
1000000 gCO2e for a baseline server with 1 CPU and 1 SSD
159 additional CPUs = 159 * 100000g = 15900000 g
total mem = 1280 GB RAM 
emissions due to additional mem = (1280 - 16) * ( ((533 / 384) * 16)/ 16) = (1264 * 22.208)/16 = 1754.431 kg = 1754431 g

SSD = 30TB - according to CCF an SSD has a constant embodied carbon regardless of size, so 1 * 100000 

sum components:
= 1000000 + 15900000 + 1754431 + 100000
= 18754431 g 
= 18754.431 kg C
```

So this calculation yielded an estimated embodied carbon value of 18754431 gCO2eq for 30TB of storage.

Now we can scale this according to the portion of that storage we actually use.

The `greensoftware.foundation` repo has a storage size of 0.00090896 GB. The storage ratio is therefore `0.00090896/30000 = 3.0298666666666667e-08`.

Now we can scale the total embodied carbon for the Github server by that ratio to get the embodied carbon we are accountable for:

`18754431*3.0298666666666667e-08 = 0.56 gCO2eq`


now let's scale that for the duration of each timestep:

We have daily timesteps, so let's make it

1 day = `86400` seconds

lifespan of the server = 4 years

4 years is `116121600` seconds

the time ratio is therefore `86400/116121600 = 0.0007 gCO2e / day`

so our daily ration of embodied carbon is `0.56 * 0.0007 = 3.92e-04 gCO2e`

so we use this value in each 1 day timestep in our manifest



## End user devices:

Google Analytics reveals an 90/10 split between laptop and mobile users. We will set a constant embodied carbon value for a representative laptop and mobile, then scale them by the user ratio and the time spent on our website. 

We'll assume mobile visitors are all using an I-phone 13 Pro. We'll assume laptop users are all using a Macbook Pro 13" M1.

Apple report 64kg carbon footprint per iphone 13 pro, of which 10kg is due to usage, leaving 54kg embodied emissions. https://8billiontrees.com/carbon-offsets-credits/carbon-footprint-of-iphone/

A Macbook Pro 13" M1 (2020, 256GB SSD)has 149.85kg embodied emiussions (https://github.com/rarecoil/laptop-co2e).

According to Google Analytics, our visitors spend 40 seconds on average browsing our site.
We can scale the device lifespan by 40s to get embodied emissions for our application.

```
lifespan of 4 years = 116121600 s
40s out of 4 years = 3.4446649029982366e-07
54000 g Co2 * 3.4446649029982366e-07 = 0.018 g per mobile user for a 40s visit
149000 * 3.4446649029982366e-07 = 0.05g per macbook user for a 40 s visit
```

Now, take weighted average (assuming users are 90/10 mobile vs laptop): 

```
((4*0.05)+0.018) / 5
== 0.0436 g embodied per user 
```

We multiply this by the new-user ratio (0.8) and add this to each timestep.

```
0.0436 * 0.9 = 0.0392
```

However, we do not want this value to be divided by our number of users yet, because these are already per-user values, so we will have to multiply by number of users in a separate step to get total embodied carbon of all new users so that the division by n-users during the SCI calculation gets us back to per user, rather than `per user/n_users`.



## Calculate total carbon

Having now calculated energy in kWh for all the components within our application boundary we can sum all those energy values together and multiply by the carbon intensity of the grid (g/kWh) to calculate the carbon emitted by our energy consumption.

We use the global average carbon intensity, because Google Analytics shows our users to be globally distributed.


## Calculate SCI

We currently have a problem with our SCI calculation - we effectively have to do it manually either by summing the average SCI values across each component, or by dividing the aggregated total carbon value by the number of timesteps per component. The reason is that we have to choose one aggregation method for the SCI parameter that applies to both time aggregation and component aggregation.

Let's explore a bit deeper:

The SCI value is a rate.

If we have a functional unit of e.g. visits, and we have values for that per timestep, then we can gather an SCI score per timestep by doing `carbon/visits`. This is what our SCI plugin does in each timestep, in each component.

However, now let’s say we want to do this over three components.

We have per-timestep SCI in units of gCO2e/visit in each of three components to aggregate up to a single value.

We *don’t* want to sum across time, because what we end up with is not SCI - we’ll end up with an inflated rate that doesn’t represent the actual rate at any point during our times series, but instead a spuriously high one.

E.g. if you did 60 mph for an hour, you would cover 60 miles and your average speed would be 60 mph and your max speed would also be 60 mph, but if we added up the speed of your car measured every minute for an hour long journey, we’d end up saying you went 3600 mph. We're effectively doing this with SCI.
So instead, we actually want to set the aggregation method to `avg`, or we want to add a normalization step where we do generate a time-totalled SCI by setting the aggregation method to `sum`  but then we divide by number of time-steps retroactively (which ends up being the exact same thing).

No problem, then, for calculating the average SCI per component, but now we want to aggregate across components. Now we really *DO* want to sum the SCI values together to yield one overarching value for the whole tree, but oh dear we already set our aggregation method to `avg`. So we can only get a spuriously LOW estimate in the top level aggregation because we are forced to average where we want to sum.

So, because we have a single aggregation method that covers both time and component aggregation, and we can’t do operations over values after they are aggregated - we can’t calculate SCI in a multi-component manifest. We can only ever massively overestimate or massively underestimate.

What we really want to achieve, is:

- to generate a total carbon value that’s aggregated across all the timesteps and all the tree components and THEN divide by the functional unit (meaning an operation has to be done on the aggregated values)

OR (better)

- enable averaging for time aggregation and summing for component aggregation, and vice-versa.


We'll address this as an urgent priority in next week's development tasks, but for now we can just manually sum the averaged SCI values across the tree components,

```
0.976+0.0001786+0.000689+0.41869+0.001388+0.0000495+0.00001735+0.0000255+0.0000643
== 1.39710225 gCO2e/visit
```

## Assumptions

- We assume every user loads every page on the site, whereas in reality they only visit a page or two. Accounting for this will reduce the estimated SCI score.
- We assume the global average grid carbon intensity
- We assume Github servers are large azure machines - they probably aren't, and they probably have methods to build/serve sites more efficiently than we assume in our calculations
- We assume all our non-mobile users are on full size laptops, not tablets or cheaper/smaller laptops with lower embodied emissions
- We assume the CDN is provided by Fastly, with 46 PoPs, and that each PoP stores 10% of the origin node storage
- We've been quite harsh in the embodied carbon calculations - assigning 100% of the device emissions to the application for the duration that the device is being used to access our site - in reality that device is probably doing many other things at the same time and could justifiably be scaled down further.


## Insights

We compare reasonably well to other websites. The global average web page produces about 0.8 g of C per visit. Our SCI score for the GSF website is 1.39 g CO2e/visit, but we have erred on the side of overestimating values that we couldn't directly measure and been very comprehensive in the factors we included. The embodied carbon for end user devices is by far the main contributor to the overall SCI score by a very large margin - the SCI score excluding the end user device embodied carbon is ~0.4 gCO2e/visit.

For comparison, using the SWD model via CO2js forn the GSf site yielded an estimate of 0.447 g per visit, about one third of what we estimate using the SCI via IF.
