---
title: "Technical Debt And Project Management"
date: 2025-06-15T18:26:02+02:00
draft: true
---

## Introduction

I've been working on an experimental project and as I was working through it, I started saying things like, "I need to come back to this later" or adding `TODOs` all over the place, even though I'm not planning on taking the project further it really got me thinking about technical debt in software why its not addreessed sooner.

## I've Seen This Movie
I've been working as a software developer for 10+ years and have joined projects that are in various stages. One thing that always pops up is developers mentining that they need some time to clean things things up but that time is never really allocated.

This indicates that new features are prioritized over maintenance. One might ask, well why isn't it built right the first time. In many instances this can be achived espcially if the business problem has been clearly defined and the appropiate time is allocated. For small tasks this is generally not an issue and once its written, that code is never looked at as it does what it needs to do and does it well.

The reality is that business problems are not clearly defined and time estimation tricky.

I'm going to try and explain how the debt occurs and how it can be managed as the project goes along.



## Business Logic

When a business requirement(feature) comes in, they are generally a lot of questions about that features, particulary the goal and the workflow of that feature and as that feature is discussed, scenarios that have not been thought of tend to appear, not just scenarios but also legal and compliance hurdles that have not been addressed. At this stage not code has been written, so no technical debt. 

What does tend to happen is developers get asked what can they get started with and how long it will take while the rest of the requirements are gathered. Its very difficult to start developing when there are gaps in logic/requirements, some of those incomming requirements could involve some database changes, which would then already require some refactoring and we all know once a developer provides an estimation it mysterisoly becomes a target date. 

More often that not business already has set milestones as to when they would like certain things complete and those milestones are not always flexible 

There is generally a push back from developers noting that its better to work on something else that is ready for dev whilst waiting for the complete requirements but that is rarely the case.

So now we have unclear specifications and timelimits that are tight, this generally results in the developer just trying to get the work complete as best as he can. Now the developer(dev) might be able to write this part of the code well, but as soon as the full requirements come in, timelines will need to change as the dev did not consider the workflows of the updated requirements.

This does not generally sit well with clients and takes a crafty project manager to get the clients on board


## What Can Be Done

So what can we do to allviate this. I think business and project managers need to be better informed as to how technical debt can negitily affect the business especially in the long term, this will then allow them to better prepare and better plan software developments.


