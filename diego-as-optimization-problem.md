---
layout: post
title: "App Placement in Cloud Foundry Diego: A Classical Optimization Problem"
date: 2014-06-30 22:48
comments: true
categories: 
---

I work on the Cloud Foundry [Diego](https://github.com/cloudfoundry-incubator/diego-release) team, a project that started as a rewrite of one of the existing Cloud Foundry components, namely the DEA, in Go (hence "Diego"), but evolved into a ground-up rearchitecture of the core service. Our team's fearless leader [Onsi Fakhouri](https://twitter.com/onsijoe) recently [gave a talk](https://www.youtube.com/watch?v=1OkmVTFhfLY) explaining the Diego rewrite.  Seriously, watch that talk, it might literally be *the* best engineering talk you'll ever see.

Onsi talks about the problems associated with having a single component, namely the [Cloud Controller](https://github.com/cloudfoundry/cloud_controller_ng), responsible for knowing about all the apps currently running on all the application executors, known as [Droplet Executation Agents](https://github.com/cloudfoundry/dea_ng) or DEAs for short, and orchestrating the placement of newly started apps on a DEA.  I want to introduce some of the key players in the Diego architecture and explain the new approach that Diego takes to solve the app placement problem, and to illustrate all this I thought it'd be fun to model the app placement problem as a sort of classical **constrained optimization problem**.

<!--more-->

In what follows, I actually describe a slightly idealized version of Diego. For instance, I take into account the desire to balance apps across multiple clusters of hosts (which are usually physically isolated from one another in some way) to ensure higher availability, which Diego doesn’t currently do. I’ll refer to these clusters as Availability Zones (AZs) to borrow a term from AWS, but everything that follows is IaaS-agnostic.

## Introducing Diego's "Auction"

Most decision problems in optimization research owe their complexity to the size of the input space.  The hard part is searching this space for an input which optimizes some objective function, or comes close enough, and doing so in a reasonable amount of time.  The approach to these sorts of problems is to find faster and more efficient algorithms.

The app placement problem is rather different.  The solution space is rather small -- even in a large production Cloud Foundry deployment there may be only 50 application execution VMs to choose from.  Most of the complexity here is due to the fact the problem is (a) distributed, and (b) asynchronous.  Because it’s a distributed problem we have to take into account that the inputs, objective function, and constraints may be coming from different sources, and that these data need to be communicated between sources in a machine-readable way over network and messaging protocols such as HTTP and [NATS](https://github.com/apcera/gnatsd). Because it’s an asynchronous problem, where we may be simultaneously be attempting to place 20 instances of the same app, we have to take into account that between the time system realizes it needs to make a decision about placing an app instance, and the time it takes action on a decision its made, the input it based its decision on may have changed -- indeed, we’d otherwise end up placing all 20 instances on the same execution VM!

Instead of the old model where the Cloud Controller tries to maintain its own copy of the state of the world, we want to gather state from the appropriate sources of truth at decision-time. And we’re going to want to have some confidence that our decision is stable in light of recent changes in the state of the world that may have occurred while we were gathering the state and making the decision.  This is accomplished in Diego by holding an “auction”, where application execution VMs “bid” to host an app, and the strength of their bid is determined by several factors, for instance the amount of free memory they have.  Let’s understand the auction a little better by looking at the players involved and their responsibilities.

### The Cast of Characters in a Diego Auction

* [**The App Manager**](https://github.com/cloudfoundry-incubator/app-manager): sits behind the user-facing API, and handles requests to start apps.  It knows things about desired apps, such as how many instances of the app are desired to start, where the source code for the app lives (in some blobstore), what the compute requirements are for an instance of the app in terms of memory, disk, and so on.  It (eventually) causes app instances to be run by requesting an "auction" to be held where machines bid to run the app.
* [**The Auctioneer**](https://github.com/cloudfoundry-incubator/auctioneer): watches for "start-auction" requests for desired app instances, and then holds the auctions.  It gathers "bids" and then chooses a "winner", and the winner is then responsible for running the app.
* [**The Executors**](https://github.com/cloudfoundry-incubator/executor): form a pool of processes, each running on a separate VM, with each one capable of running multiple apps in isolated containers within its VM.
* [**The Reps**](https://github.com/cloudfoundry-incubator/rep): represent the Executors, one Rep for one Executor.  They participate in the auctions, providing bids requested by the Auctioneer on behalf of their Executors.  A bid encapsulates relevant information about an Executor, such as what apps its currently running, and how much more free memory it can allocate to new containers.

(For more, check out the [Diego Design Notes](https://github.com/cloudfoundry-incubator/diego-design-notes))

So the problem that we’re going to model as a classical optimization problem is that of choosing the winning Rep bid for a particular auction.

## A Concrete Use Case

Let’s use the example of a user pushing a new app.  The user wants to have three instances of it running, so that it’s highly available. Ideally, these instances would be balanced across different Executors running in different AZs.  If they all end up in the same AZ, or worse, on the same Executor, then the high availability you’d expect to achieve by running 3 instances is a lot more brittle.

So if we’re to balance instances across AZs, how should we do it?  It can’t be as simple as saying, “put instance 0 in the first AZ, instance 1 in the next AZ, etc.” since that’ll clearly skew app placement towards the first AZs and away from the last AZs (in whatever ordering you have for your AZs).

We might try to pick an AZ at random, and then pick an Executor in that AZ at random.  But in addition to having their app instances balanced across AZs, the user wants to ensure a certain amount of memory and disk are allocated to each running instance. What if the Executor we chose doesn’t have the resources?  Worse, what if the AZ we chose doesn’t have any Executors with enough resources?

So when placing an app instance, we want to place it in an AZ that isn’t already running any instances of our app, or if we’ve asked for more instances than there are AZs, we want to make sure that when placing a particular instance, we place it in an AZ that isn’t already running too many other instances of our app.  In addition to that, we want to choose an AZ, and an Executor in that AZ, that has enough free compute resource to run the instance.  Finally, if there are several candidate Executors, it’s better to go with the one with the most free resources so as to better balance the load across Executors. 

Whether it’s AZs or Executors, balancing the load intelligently hedges against the risk associated with any one of the AZs or Executors going down.
The App Manager is going to take data about an `AppInstance` and trigger a "start-auction", which will include an **objective function** and **constraints** specific to that `AppInstance`.  The Auctioneer will start the auction and gather `RepBid`s from the Reps.  These `RepBid`s will constitute the **inputs** for the objective function, and the Auctioneer will find the input which minimizes the objective function amongst those that satisfy the constraints.

## The Optimization Problem: Deciding a Diego Auction Winner

Now let's actually formally define the problem, the data structures involved, and see what pieces are already in place in Diego that implement solutions and measure the quality of solutions.

### Data Structures

```go
type AppInstance struct {
    RequiredDiskMB      uint
    RequiredMemoryMB    uint
    AppID               uint
    InstanceNumber      uint
    TotalInstances      uint
    AppSourceBlobID     string
    Stack               string
}

type RepBid struct {
    AvailableMemoryMB   uint
    AvailableDiskMB     uint
    TotalMemoryMB       uint
    TotalDiskMB         uint
    RunningAppIDs       []uint
    AvailZoneNumber     uint
    CachedBlobIDs       []string
    Stack               string
}           
```
    
### Problem

Choose the "highest-scoring" Rep to run a particular `AppInstance`:

$$ \argmax _r f_{\mathrm{AM},ai}(r) $$

where $f_{AM,ai}$ is the objective function for the `AppInstance` provided by the $\mathrm{AM}$ (App Manager), and $r$ ranges over all the `RepBid`s collected by the Auctioneer from the Reps.

### Constraints

A Rep's bid is only feasible if its Executor has enough memory and disk, and is running the same stack (e.g. Lucid 64-bit) as required for the `AppInstance`:

$$r.AvailableMemoryMB \geq ai.RequiredMemoryMB $$
$$r.AvailableDiskMB \geq ai.RequiredDiskMB $$
$$r.Stack = ai.Stack $$


### Actual Problem

The **actual** problem is to express an objective function that captures the business requirements, and then find a near-optimal solution efficiently.  Furthermore, any solution must take into account that the data in a Rep's bid can change in the course of an auction it's participating in, since it may be participating in multiple simultaneous auctions.  For instance, a Rep might report having 4096MB free memory simultaneously in many different auctions.  If it wins one of the auctions and agrees to start a 512MB app instance, then its original bid in any of the auctions that haven't been decided yet is based on old information, since it now only has 3584MB free.

### Meta-Constraints

The objective function $f_{\mathrm{AM},ai}$ itself must be expressible in terms of $ai$ and $r$ in a fairly simple way.  Here are some meta-constraints on the expression used to define the function by allowing only the following operations:

* **Attributes:** Access public attributes of $r$ and $ai$.
* **Arithmetic:** $+, \times, \div, \mathrm{mod}$ where the modulus for $\mathrm{mod}$ must be a constant, not a computed value.
* **Counting:** `count(x, xs)` which counts how many times `x` occurs in the slice `xs`.

It's not terribly important what the exact constraints are, they just need to be strict enough to ensure that a very simple DSL will suffice to express the objective function.  Nobody wants to be sending arbitrary expressions involving $ai$ and $r$ from the App Manager to be `eval`ed on the Auctioneer.

### A Proposal for the Objective Function

$$ f_{\mathrm{AM},ai}(r) = (ai.InstanceNumber + ai.AppID + r.AvailZoneNumber)(\mathrm{mod}\  \# AZs) + 1$$
$$ +\ \alpha \cdot \mathrm{count}(ai.AppSourceBlobID, r.CachedBlobIDs)$$
$$ +\ \beta \cdot (r.AvailableMemoryMB / r.TotalMemoryMB)$$
$$ +\ \gamma \cdot (r.AvailableDiskMB / r.TotalDiskMB) $$
$$ +\ \delta \cdot (1 - \mathrm{count}(ai.AppID, r.RunningAppIDs) / ai.TotalInstances) $$

where $\# AZs$ is the total number of AZs, and $\alpha, \beta, \gamma, \delta$ are non-negative weighting constants that add up to $1$.

Let me explain:

1. The first line deserves some detailed explanation.  
  *  For starters, let’s ignore the ai.AppID term.  Then the expression that remains **ensures that each instance of an app gives preference to a different AZ**. For example, suppose there are four AZs, and so each `RepBid` has an `AvailZoneNumber` of $1$, $2$, $3$, or $4$.  Then for instance $0$ the first term (ignoring $ai.AppID$) is maximized when the `Rep` is on AZ $3$.  For instance $1$, it’s maximized when `r.AvailZoneNumber` is $2$.  For instances $3$ and $4$, the preferred AZ will be $1$ and $4$, respectively.  And then it repeats.  So if an instance $4$ is requested, it’ll prefer to be on AZ $3$, just like instance $0$.
  * This expression doesn’t merely pick out a preferred AZ for each instance, it determines a complete preference ranking of all the AZs.  Instance $0$ prefers AZs $3$, $2$, $1$, and $4$, in that order.  That way if there are no `RepBid`s coming from a preferred AZ, there is some deterministic way to decide amongst the rest.
  * Recall earlier that wanted to avoid making the same AZ placement for instance 0 of every app, since that AZ would quickly become overwhelmed.  This is where $ai.AppID$ comes in.  The $ai.AppID$ term effectively shifts the AZ-preference order for each `AppInstance` by $ai.AppID\ (\mathrm{mod}\ \# AZs)$.  Assuming that $ai.AppID$ are sufficiently randomly distributed, we’d expect that a quarter of all applications’ $0^{\mathrm{th}}$ instance will prefer AZ $1$, and likewise a quarter for AZs $2$, $3$, and $4$.
 * Finally, since the result of the modulo is an integer, and then we add $1$, and then the rest adds up to $\alpha + \beta + \gamma + \delta = 1$ at best, we’re assured that balancing instances across AZs is the "most-significant bit"", and is given highest consideration compared to any other feature of a `RepBid`, such as free memory.  
2. The second line gives some weight to an **Executor that has a cached copy of the app's source** locally.  The $\mathrm{count}$ expression on that line will only ever evaluate to $0$ or $1$.  
3. The third line considers the **percentage of free memory on an Executor**.  
4. And the fourth considers the **percentage of free disk**.  
5. The fifth line takes into account **how many instances of the app the Executor is already running**.  So, for instance, if two Reps are "tied" before considering this criteria, both have the same percentage of Memory and Disk free, but one of them is already running many instances of the app, it would be preferable to put the current instance in question on the other Executor.

Note: to implement that function in Golang most things will have to be typecast to `float64`.  Not only for the division, turns out Go wants floats for [$\mathrm{mod}$](http://golang.org/src/pkg/math/mod.go) as well.


Solutions to this optimization problem have to be evaluated based on time taken, CPU and memory cost, number of network calls, how well-balanced it places apps, and all in the context of being run multiple times concurrently.

## What Diego Already Does, and Where It's Headed

Diego already runs auctions.  Not only that, the [auctionrunner](https://github.com/cloudfoundry-incubator/auction/tree/master/auctionrunner) package already defines multiple ways to run an auction.  There are auctions such as [randomAuction](https://github.com/cloudfoundry-incubator/auction/blob/master/auctionrunner/random.go) which optimizes for performance but obviously does miserably at choosing a near-optimal, feasible `RepBid`.  Other auctions make tradeoffs between quality of the `RepBid` it chooses, total number of communications, etc.  Any one of these auctions can be run in a multi-round format, where a winner is tentatively selected but multiple rounds are played, to adjust for the fact that a Rep's bid can quickly become invalid if it's participating in multiple auctions.

What's currently missing is the elaboratness of the objective function.  Diego does not take AZ placement or cached blobs into account, as though the first two lines on the right-hand side of the equation weren't there.  It also weighs free memory, disk, and currently running apps equally, as though $\beta = \gamma = \delta = 1/3$).

Moreover, the objective function is not currently something dynamic.  In fact, the Auctioneer has no notion of an objective function, and Rep bids aren't structured objects.  The Reps compute the scores and just give those scores directly to the Auctioneer, and the formula for computing scores is [hard-coded](https://github.com/cloudfoundry-incubator/auction/blob/master/auctionrep/auction_rep.go#L238-L247).

AZ balancing is [in the backlog](https://www.pivotaltracker.com/story/show/70403224)!  [Another cool feature](https://www.pivotaltracker.com/story/show/70403278) we have planned is periodic rebalancing based on CPU usage (for starters).  Placing apps in a balanced manner when starting new instances is all well and good, but over time, app placement may become less balanced.  For instance, if an AZ goes down, all the instances on it will be evacuated and re-run in different AZs.  When that AZ comes back up, we’d like to gradually move some instances back there.

## How to Evaluate and Prosose Solutions for Diego

Diego has a [Simulator](https://github.com/cloudfoundry-incubator/auction/tree/master/simulation)!  You can run the simulator on its own, and it also runs as part of the test suite.  You can play with all sorts of different variables in the system.  For instance, you can obviously run different types of auctions (random vs. pick-the-best).  But you can also play around with different communication modes, e.g. in-process vs. NATS, to see how performance is affected when the communication between processes is "realistically" handled by a central message bus.  

Simulation runs are accompanied by a report that will show up in your browser.  Reports include statistics, so you can evaluate things such as total number of communications and standard deviation in the number of apps placed on each Executor, and visualizations, so you can see how well the apps were balanced or how many rounds were required to pick an auction winner. Here’s what the output looks like in the browser and the terminal:

<div style="margin:auto"><a href="/images/simulation_browser.png" rel="shadowbox"><img src="/images/simulation_browser.png" width="40%" height="40%" style="vertical-align: middle"></a>
<a href="/images/simulation_terminal.png" rel="shadowbox"><img src="/images/simulation_terminal.png" width="40%" height="40%" style="vertical-align: middle"></a></div>

