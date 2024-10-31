
## Prerequisites

BLOOM model carbon estimate: https://arxiv.org/pdf/2211.02001
Carbon emissions and large neural network training: https://arxiv.org/pdf/2104.10350

## Training 

### GPT3 (not Chat-GPT3)

GPT3 is a general model, not designed to return human-like answers, plus it is a huge model. Chat GPT is much smaller because it is more specialized, and it is optimized for human responses. Chat-GPT was tuned from GPT3.

Hardware configurations are secret - they only expose very general information, including that it was trained on Azure using Nvidia GPUs.

However, [Microsoft announced](https://news.microsoft.com/source/features/ai/openai-azure-supercomputer) in May 2020 a new supercomputer for AI training. It had > 285000 CPU cores, 10000 GPUs and 400 Gbit/s network. In top 5 of the largest supercomputers, which means it must have 100 Petaflop peak performance. The press release mentions being idally suited to LLM training.

An OpenAI paper on LLMs "language models are few shot learners" revealed that "all models were trained on V100 GPUs on a high bandwidth cluster provided by Microsoft."

We also know that training also happens exclusively on the GPUs, via CUDAs deep neural network library. The CPUs are just for support.

At the time the supercomputer was built, the Volta range of GPUs was the only available chip that could provide the required performance.


### Chat GPT3

We know, from OpenAI, Chat-GPT3 was tuned from a model in the GPT3.5 series in 2022. We also know that, again, it was trained on microsoft hardware.

In 2021 Microsoft announced the availability of A100 GPUs. Since the A100 delivers a substantial performance improvement over the V100, and the model was trained *after* these GPUs became available to Azure customers, we can assume these chips were used to train Chat-GPT3. 

NVIDIAs A100 chip is hugely more performant for tensor operations, even comapred to the V100. The economic decision would not be to replace all 10,000 GPUS from V100 to A100, as the performance increase negates the need for so many and the new model is smaller anyway.

Megatron-Turing NLG supercomputer was announced as a partnership between Nvidia and Microsoft just before Chat-GPT3 was trained. It contains 4480b A100 GPUS. It is possible that this specific machine was used, buit even if not, it is probably a good analog for whatever hardware was used.

So in summary, it seems reasonable to assume GPT3 was trained on 10,000 V100s and Chat-GPT3 was trained on 4480 A100s.


### Links

Some links supporting this logic:

[Youtube video](https://www.youtube.com/watch?v=4q9-yf1eU8c)
[The Verge](https://www.theverge.com/2023/3/13/23637675/microsoft-chatgpt-bing-millions-dollars-supercomputer-openai)
[Bloomberg](https://www.bloomberg.com/news/articles/2023-03-13/microsoft-built-an-expensive-supercomputer-to-power-openai-s-chatgpt?sref=ExbtjcSG)
[Microsoft News](https://news.microsoft.com/source/features/ai/how-microsofts-bet-on-azure-unlocked-an-ai-revolution/)
[Nvidia news](https://nvidianews.nvidia.com/news/nvidia-microsoft-accelerate-cloud-enterprise-ai)



## Inference

Hardware requirements for inference might exceed that for training.

MS/Nvidia already announced the models are running on Azure servers. A single Nvidia DGX or HGX A100 server is sufficient to run the model but not at the scale of 100 M users/day.

[Semianalysis](https://www.semianalysis.com/p/the-inference-cost-of-search-disruption) did a very deep dive and suggested 3617 HGX A100 servers required to serve inference. We'll assume this is correct.


## Embodied carbon

```
Some thoughts on embodied carbon attribution.

If we calculate the embodied carbon of the hardware that was used for training, and then bundle it with operational carbon before dividing by queries per day, we end up double counting embodied emissions. I've seen this method used in several places, and started doing it myself, but some vague alarm bell was ringing in my head and I sat down to deal with it and realised the following:

Why?

Because training was finite, and we don't know at t[N] how many requests we'll accumulate by t[M]. The aforementioned methodology effectively attributes an amount of carbon per query that's held constant forever, regardless of how many requests are made - eventually, we'll account for more carbon than was ever actually emitted during the actual training phase. 

Take a simplified example - if we emit 10g C during training, and then calculate that this equates to 1gC/query, and use this constant in perpetuity. Well, after 11 requests, we'll have added 11g of carbon to our balance sheet despite the real training onyl ever emitting 10. We'd have to have some condition in our calculation algorithm that stopped adding the embodied carbon to the C per query once we'd accountedf for 10 g. But then, it's not logical to say "all queries before X date cost Y carbon, and all queries after X date cost Z carbon". 

Instead, one option is to calculate a dynamic value for g/query that uses the *cumulative* number of queries as the denominator, not the queries per unit time. This gradually amortizes the embodied carbon so that as more and more queries are made, each individual query accounts for a smaller portion of the total, and we never account for more than was actually emitted.

The problem with this is that we heavily bias the carbon allocation to the early users. At t=0, we use, say 10 queries as the denominator and get a value of 1g/query. We've already accounted for all the carbon! The next day, we do the same, having observed another 10 queries, and we see 10g/20 = 0.5g/query. but we are back to our original problem of adding to a carbon defecit that was paid back on day 1! This problem resolves when we are doing these calculations retrospectively, because we can evenly distribute the burden over the total queries to date, but then we have the issue that the C per query decreases over time as you accumulate more total queries - it's ok in theory, but unless we have a fixed end point, we never settle on a final number.

So what else could we do?

We could fix a time window over which we want to amortize the total carbon emitted during the training phase. After that time window expires, we consider the carbon to be accounted for and simply zero the balance, assuming it all to have been accounted for already.

Let's say we want to account for it all over 1 year. Then we distribute the total carbon over, say, 365 equal intervals, and then calculate a dynamic carbon per query based on the actual, observed number of queries observed on a given day and the daily carbon allocation (total/365). The nuance here is that the carbon price is dynamic depending on the number of queries, and the choice of time window is arbitrary. There's also a risk that it relies on at least 1 query per day in order to account for all the carbon - if any day has zero requests there will be unallocated carbon in the system.

Another option could be to establish a decay curve for the carbon, so that instead of expiring on some given day, the burden decreases according to some linear or non-linear function, until eventually the carbon allocated per query becomes negligible, and we can decide what negligible means based on how much of the original carbon emission has been allocated. This seems uneccessarily complex, and also vulnerable to unusual effects.

**Chosen solution**

In this estimate, I amortized the embodied carbon due to training over the time beetween Chat-GPT being released using GPT3, and Chat-GPT being upgraded to use GPT4, as this is the time period of heaviest use for the actual model setup we've modelled here. That time span was about 4 months, between [November 2022 and March 2023](https://www.searchenginejournal.com/history-of-chatgpt-timeline/488370/).

This means the embodied carbon for the GPT-3 training and Chat-GPT training phases are *both* scaled to a daily value using 120 days as the denominator, and then carbon is allocated per query by dividing that daily amount of embodied carbon by queries per day.

This only makes sense for the hardware that was *only* used for training. The time window for the hardware being used to serve queries should be the lifespan of the hardware, or if the hardware is used for less than the declared lifespan, the time it was actually in operation for (*if* the hardware is theh used for other purposes - if it is binned, then account for it all). If the application is moved to new hardware, the embodied carbon of that new hardware should be accounted for and amortized over *its* lifespan.

```


Nvidia do not publish embodied carbon values for their chips, but GPUs are well known to be dirty and energy intensive to produce. We will have to look for analogs from other manufacturers and accept there's some uncertainty. 

Dell are one of few companies that publish embodied carbon values for some of their servers. We can look for a Dell server that has approximate specs to a HGX A100 from Nvidia.

The challenge was therefore to find an AI-specific rack server with GPU support, and *also*  reports embodied carbon estimates. I found the PowerEdge R740xd. This supports up to three double-width 300W GPUs, such as the A100 or V100.

The reported embodied carbon value for this server is [9180 kgCO2e](https://i.dell.com/sites/csdocuments/CorpComm_Docs/en/carbon-footprint-poweredge-r740xd.pdf), i.e. 9.18 tonnes per server, for a full 4 year lifecycle.

Given that we identified 10,000 GPUs were needed for training GPT3 and 4480 were used to train Chat-GPT3.

The A100 and V100 GPUs are 300W so each server can accommodate three. So the total number of servers required to deliver the GPT3 and Chat-GPT3 training is:


```
n-servers:

(10000 + 4800)/3
== 4933
```


Multiply that by the embodied carbon of each server:

```
4933 * 9.18
== 45284.9 tonnes embodied carbon
```

Therefore, I estimate the embodied carbon of the GPT-3 and Chat-GPT3 training hardware to be ~45285 T. 

This breaks down to about 30600 T for the GPT-3 model training and 14685 T for Chat-GPT tuning.

But this is the embodied carbon assuming the entire product lifespan was dedicated to the training task. This can be scaled down by the time taken to actually complete the task, assuming the servers were repurposed afterwards.

GPT-3 took about 34 days to train on a 10,000 GPU system, according to [researcher estimates](https://lambdalabs.com/blog/demystifying-gpt-3). The 4-year lifespan is 1460 days, so the ratio of training time to lifespan is 

```
34/1460 = 0.01382
```

So let's scale the embodied carbon of the training hardware for GPT-3 by 0.01382:

```
30600*0.01382 = 423 T
```

For Chat-GPT I have not been able to find any quantitative estimates of the training time, only qualitative suggestions such as "several months". We can assume the hardware was dedicated during this time. Let's say it's 4 months - accepting that this is a guess. The final value for embodied carbon is pretty sensitive to this estimate - it does have an appreciable impact whether the true number is e.g. 3 months, or 6 months.

4 months out of 4 years = 4/ 48 = 0.0833, so the embodied carbon we attribute to the Chat-GPT training is:

```
14685*0.0833 = 1223 T
```


Lets assume the same type of servers are used for inference too - we identified that 3617 servers are required to handle 100 M users. This equates to:

```
3617*9.18
== 33205 T
```

Another 33205 T of embodied carbon for the inference stage.

So to sum up the embodied carbon for training and inference for Chat-GPT, we have:

```
423 + 1223 + 33205
= 34851 T
```

As explained earlier, we want to fix a time window of 4 months (120 days) over which to amortize this carbon. So we will account for the 34851 T of carbon in 120 individual timesteps.






We can further distribute this embodied carbon value down to daily timesteps to make it easier to aggregate with operational carbon estimates and to normalize to daily interactions data that are available online.

34851 T per 4 months equates to 311 T /day.

Chat-GPT was estimated to have served around [10 million queries per day](https://nerdynav.com/chatgpt-statistics/) during launch week, and this value may have increased since then due to the growing popularity of the tool, meaning the embodied carbon per request is

```
311/10000000
== 3.11e-5 T C02e / query
== 0.0311 kg CO2e / query
== 31 g/query
```


## Operational carbon

Now let's look at operational carbon:

#### Operational carbon for training phase

We assume V100 processors for GPT3 training and A100 chips for Chat-GPT3 training.
We assume the processors have a TDP of 300W. We also assume the developers are running the chips as efficiently as possible (i.e. that they are all sized for 100% utilization), meaning we simply multiply 400 W by the number of processors.

There were 10000 GPUs used to train GPT-3, and it took 34 days. We assume a data centre PUE of 1.1, and a grid carbon intensity of 390 g/kWh.

So we can do the following:

```
(((300*10000)*(34*24))*1.1)
= 2692800000 Wh, or 2692800 kWh or 2692.8 MWh
= 2692800 kWh * 390 = 1050192000 g CO2e
or 1050192 kg or 1050.19 T
```

This is in line with the results published [here](https://arxiv.org/pdf/2104.10350)who used 14 days training time and 300W TDP per GPU, estimating 552 T CO2e emitted during training.

Scaling this down to per day, and then per query, we get:

```
1050 / 34 / 10000000
= 3.088e-06 T CO2e/query
= 3.088 g CO2e per query
```



Let's do the same for ChatGPT training.

There were 4800 GPUs used to train Chat-GPT, over a period of 4 months (120 days) so we can do the same:

```
(((300*4800)*(120*24))*1.1)   // 120 days in hours
= 4561920000 Wh
= 4561920 kWh or 4561.92 MWh energy used

4561920 * 390 = 1779148800 g CO2e
or 1779.15 T
```

Scaling this down to per day, and then per query, we get:

```
1779 / 120 / 10000000
= 1.4825e-6 T CO2e/query
= 1.4825 g CO2e per query
```


So the total energy used to train both GPT-3 and Chat-GPT was:

```
4561.9 + 2692.8 = 7254.7 MWh energy
```

The total Carbon was:

```
1779.15 +  1050.19 = 2829 TCO2e
```

And the total Carbon per query is:

```
3.088 + 1.4825 =  4.571 gCO2e/query
```


Remember, this is only the operational carbon used *in training the models* that is allocated to each query, not the operational carbon associated with the query itself. That still needs to be calculated. 

So far we have calculated the embodied carbon of training Chat-GPT and GPT3, plus accounted for the energy used during the model training. We then scaled that carbon to a daily value, and then by the queries per day, to allocate an amount of carbon per query. however, what this represents, is the "background" carbon - the carbon that has already been emitted just to establish the system you can send queries into. it doesn't yet account for the carbon emitted during the actual act of querying the model. 


#### Operational carbon for inference


Turns out this is quite straightforward, assuming [Semianalysis](https://www.semianalysis.com/p/the-inference-cost-of-search-disruption): estimate of 3617 HGX A100 servers (28.936) required to serve inference are correct.

Each of these servers contains 8 A100 GPUs, meaning there are 3617 * 8 = 28936 GPUs serving queries.

In this case, we can assume they are all being fully utilized and are running constantly.
This means we can do:

```
400W * 28936 = 11574400 W power
```


Then turn this into an amount of energy used in a day:

```
11574400 * 24
== 277785600 Wh per day
== 277785.6 kWh per day
```

We use the same grid carbon intensity as before (390 g/kWh) to yield:

```
277785.6 * 390 = 108336383.99 g CO2e per day
== 108336.38 kg CO2e per day
```

Now let's scale by the number of queries:

```
108336383/10000000
= 10.83 g CO2e/query
```


## Total carbon

`31 + 4.571 + 10.83 =  46.4 g CO2e/query`

The total carbon emission for Chat-GPT is estimated at ~46 g/query, assuming 10M queries/day.

This equates to a total carbon emission of 453 T CO2e per day, including embodied and operational carbon for the training and inference phases. The training phase includes both GPT-3 and Chat-GPT models, since Chat-GPT couldn't exist without GPT-3.

[These folks](https://smartly.ai/blog/the-carbon-footprint-of-chatgpt-how-much-co2-does-a-query-generate) estimated 4.32gCO2e/query, seemingly just for the operational carbon component, which is surprisingly similar to our estimate (4.571 g/query). However, it's an order of magnitude less than our totalcarbon estimate which also includes embodied carbon and both training and inference phases, and also includes GPT3 training as a prerequisite for Chat-GPT.

Excluding the GPT-3 model from the analysis altogether, but including embodied carbon for the training and inference server, operational carbon for the training and operational carbon for serving queries, Chat-GPT3 alone emits ~16 g/query. This comprises 1.02g embodied carbon from training servers, 2.74 g embodied carbon for inference servers, 1.48 g/query operational carbon from training and 10.83 g/query operational carbon for inference.

This suggests that the the carbon emissions of the training pahse are smaller than for inference, even accounting for the GPT-3 model training.
