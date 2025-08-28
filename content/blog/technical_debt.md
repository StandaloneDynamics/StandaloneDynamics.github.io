---
title: "Technical Debt And Project Management"
date: 2025-06-15T18:26:02+02:00
draft: false
---

## Introduction

I've been working on an experimental project and as I was working through it, I started saying things like, "I need to come back to this later" or adding `TODOs` all over the place, even though I'm not planning on taking the project further it really got me thinking about technical debt in software why it's not addressed sooner.


## I've Seen This Movie
I've been working as a software developer for 10+ years and have joined projects that are in various stages. One thing that always pops up is developers mentioning that they need some time to clean things up but that there is never time to actually do so.

This generally indicates that new features are prioritized over maintenance. One might ask, well why isn't it built right the first time. In many instances this can be achieved, especially if the business problem has been clearly defined and the appropriate time is allocated. It's always a good feeling to build out a feature and deploy it knowing that it has been well crafted and works exactly as expected.

The reality is that a lot of business problems are not clearly defined or well understood until development starts taking place or that changes are needed once the feature has been deployed as it has not met business expectations. 

There is also the issue of allocating enough time to build out features due to deadlines or other factors that are not within the teams control.


## Consequences

Technical debt is one of those things that slowly creeps up on teams, until it suddenly appears when a major feature is required that touches various aspects of the codebase. At this stage it is just too big to address and forces developers to work around it. leading to more technical debit, this tends to significantly reduce the quality of the software and increase frustration among the developers.

The product itself tends to suffer as less features can be implemented eroding the trust from various client and stakeholders. 


## What can be done

When projects are initiated, maintenance of the product should be part of the planning process, which means that time and budget should already be allocated to just perform code cleanups and address lingering issues. We know these issue will occur so it makes sense to account for these immediately. 

When new features are proposed I think it would be worthwhile to spend extra time with clients or the deign team to talk through the feature and make sure it has been fully specified and thought through. A lot of times there is a back and forth between clients and developers during development  because of unforeseen scenarios or issues such as missed compliance or 3rd party dependencies.

Taking the extra time to think through a feature, which might be slow at first, will greatly reduce the debt as the developers will know exactly what to build and how to build it.


The really tricky aspect is getting buy in from stakeholders from the get go. This is were I think project managers should step in and explain to them why its vitally important to start planning for maintenance and how it will benefit the project in the long term and why allocating more time it worth it.

By having these maintenance windows in place, this will allow developers to better manage any issues that have crept up during the course of development and keep the debt in check.


## When Should Maintenance Occur

So when should these maintenance windows occur? I'd push for a time period say every 2-3 months, which is about after every 4-6 sprints, assuming each sprint is about 2 weeks. A lot of work can be done during that time especially among big teams, so having that window after those sprints seems like a good compromise.

How long is the window? I think a sprint, max, for each window will suffice, this assumes that the maintenance windows are been carried out, skipping the windows kind of defeats the purpose as this will increase the debt and make it much harder to contain the window to just a sprint. 


All aspects of our lives from exercising, plumbing, road works require some sort of regular maintenance, software should be no different.
