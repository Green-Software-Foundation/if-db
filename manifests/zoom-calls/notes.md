# How we calculated the SCI score for a Zoom call

## Introduction

We used the Impact Framework to calculate the Software Carbon Intensity (SCI) for a hypothetical 30 minuite Zoom call with 5 participants.

This is a first attempt that we would love people to help us improve. Don't be shy to criticize the design decisions made here - even better if you raise a PR to improve the modelling. The idea underpinning Impact Framework is radical transparency made real using manifest files - all our working out is presented specifically so you can come along and refine it with us!

We would particularly benefit from updating these hypothetical time series with real measurements for CPU and memory utilization and data transfers.

## Scope

We'll set up a hypothetical call with 5 participants and set the call length to 30 minutes. We'll track the call metrics at 1 minute time intervals.

We want to compute total carbon emissions and an SCI score. The functional unit for the SCi score is the number of participants, so we will yield an SCI unit of gCO2e/participant.

```
SCI = (operational carbon + embodied carbon) / n-participants
```

The following components are in scope for the embodied carbon:
- user devices
- zoom server

The following devices are in scope for the operational carbon

- transferring call data over the internet
- operational carbon, derived from CPU and memory utilization, for end user devices
- operational carbon, derived from CPU and memory utilization, for Zoom server
  

Out of scope:

- data center operational carbon (lighting, heating, etc)
- data center embodied carbon (building)
- powering end user screens
- powering cameras, speakers and microphones



## Calculations

**Note** the functional unit is participants, not participants/minute. This means that we want to allocate the *total carbon* for the full 30 minute call to each participant. Therefore, our aggregation should **sum** across each time series AND **sum** across components. The alternative would be to use the average method to aggregate within the time series and then sum across them, giving the SCI in units of gCO2e/participant/min.


### Network

We use a coefficient (0.07 kWh/GB) to convert the amount of data transferred over the network to energy in kWh. This coefficient was taken from [Orbinger et al 2021](https://www.sciencedirect.com/science/article/abs/pii/S0921344920307072).

The amount of energy transferred was assumed to be constant throughout the call. We used data from [this report](https://tactiq.io/learn/zoom-data-usage-facts), selecting the transfer rate for group calls streaming at 720p resolution (1.35 GB/hr). this was scaled down to per minute by dividing by 60 (0.0225 GB/min).

Therefore, in each minute of the call, the energy used is:

```
energy = 0.07 * 0.0225
```

Then we convert this energy to carbon by multiplying by the grid carbon intensity.

Since the coefficient, grid carbon intensity and the data transfer rate are constant across all timesteps, the carbon emissions are uniform across the time series. Adding real observations of data transfer rates would be a very useful improvement to the manifest!


### Server

[This blog](https://trembit.com/blog/how-much-does-it-cost-to-maintain-zoom-servers-infrastructure/) describes an investigation into the server config and CPU load for conference calls. They actually looked into Jitsi, which they assume to be an analog for Zoom.

Jitsi engineers did performance testing of Jitsi Meet. They showed that a quad-core Intel ® Xeon® E5-1620 v2 @ 3.70GHz CPU easily accommodates a 33 participant call using just 20% of the CPU capacity. This processor has a TDP of 130 W.


This is enough information to use the Teads power curve method to estimate energy used by the CPU in kWh. We can also look for a cloud instance that uses that processor and has some fairly substantial memory available - such as the AWS c5.2xlarge - let's assume that's what Zoom uses and look up the manufacturer's value for embodied carbon (0.0043 kgCO2/hour).

#### Operational carbon

The well-known Teads power curve pipeline was used to estimate energy from CPU utilization. This involves the following steps:

- interpolate the power curve for the CPU utilization at each timestep, yielding the "cpu-factor"
- multiply the processor TDP by the cpu-factor to yield power
- apply time and unit conversions to yield energy in kWh
- multiply by grid carbon intensity to yield carbon emissions per timestep

#### Embodied carbon

We use the manufacturer's embodied carbon value for a cloud instance we think is a decent analog for the Zoom server that serves the call.
We then scale it for the proportion of the server lifespan accounted by the call (30 mins out of 4 years).

[Climatiq](https://www.climatiq.io/data/emission-factor/49b1f8c1-e6e0-4952-b871-3cd70f8e52e0) estimates 0.0043 kg/Co2e/hr == 4.3g/hr for this instance
4.3g/hr = 0.0716g/min

assume zoom use their servers effectively and would not have a dedicated server for this call that only uses 20% of the cpu/mem.

== 0.0716/5 gCO2/min
== 0.00352 gCO2/min

So this can be hardcoded as a default in the time series.


### User

#### Embodied carbon

Each user is assumed to be using a Macbook Pro 13 (15"). It's embodied carbon is 159 kg CO2e according to the manufacturer.
We scale this value by the proportion of the device lifespan (4 years) accounted for by the call (30 mins).
We add one-minutes' worth of this embodied carbon to each timestep.

The calculated value is multiplied by the number of participants.

#### operational carbon

The well-known Teads power curve pipeline was used to estimate energy from CPU utilization. This involves the following steps:

- interpolate the power curve for the CPU utilization at each timestep, yielding the "cpu-factor"
- multiply the processor TDP by the cpu-factor to yield power
- apply time and unit conversions to yield energy in kWh
- multiply by grid carbon intensity to yield carbon emissions per timestep

The calculated value is multiplied by the number of participants.
