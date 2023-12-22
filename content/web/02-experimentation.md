---
title: 2. Short guide to experimentation
images: []
---

# 2. Short guide to experimentation

Recently I've been involved in running an experiment at work and decided to dive a bit deeper into all the different parts of how to get it set up. You can find a lot of articles online which talk very vaguely about how to run an experiment. This post is meant to be a cheat sheet of sorts to give you a step by step of what is needed, but at the same time it won't cover the topic very in depth. For the coding and instrumentation part I will focus on .NET as it's my preferred tech stack but with a little bit of research you should have no problem finding the libraries for your preferred stack.

## What is experimentation and why do we care?

If I'm running a small website and decide to change something in the UI for example, do I care how it affects users it looks better to me? Probably no. But if I'm running a business and care about how many people execute some action on my site and what contributes to it then it may be a bit more important.

So say, you have a checkbox in a sign-up form where the user can opt-in to additional marketing communication. Right now it just has some basic text next to it which is there to announce what this checkbox is and some legally required formula. We want to make it look nicer by putting it on different background color and adding an envelope icon.

The question is, how is it going to affect the users. Are they going to click the checkbox more, or less? To answer the question we first have to measure this engagement somehow and then compare the numbers between old and new UI.

Without heading towards statistics too fast, let's talk about a high level expectation for an experiment. We want to check if the change in UI affects the user action. What if there is other factors that affect the user action? Do users subscribe to mailing more if they fill out the form in the morning and less in the evening? Do users who's name start with 'M' are more into mailing offers? Or maybe there are small differences in how the page is rendered in different browsers on or mobile that would affect this?

An A/B test tries to compare two page variants where only 1 factor, our feature, is the differing factor. This way, if we can prove a statistically significant difference in the measured user action across the page variants, it means it wasn't influenced by other factors and it's only the feature that has an impact.

## What do we need to conduct this experiment?

1. Partition users into two equal groups (50/50). One group will see the old UI and one the new UI.
2. Show the right UI variant to the given group.
3. Log the signal of whether user interacted with the UI to a data store.
4. Perform data analysis using statistical methods to make conclusions which UI is better at making users engage with it.

## Feature management

We need to put some code into our website which will check whether the feature is on or off, or whether to use value X or Y as part of the feature evaluation. The most basic thing you can do is just hardcode a constant and check if it's value is true or false. And with each deployment of the website you change what is being shown.

```csharp
const bool isNewUIEnabled = true;
if (isNewUIEnabled) {
    //...
} else {
    //...
}
```

Now, this has two downsides:

1. You can't modify the value when the service is running. Depending on your set up the deployment can take a long time.
2. You can't present the 2 UI during the same time frame.

For the experiment quality, you generally want to run your two variants at the same time. This way you can account for situations like I mentioned earlier - what if users are more into subscribing on Monday mornings. So let's look at alternative systems.

The `Microsoft.Extensions.Configuration` solves the issue (1) by allowing you to specify config sources that can be modified at runtime. From simple json files to integrations with Azure, AWS and other configuration providers. There's not much in the open source for fully fledged solutions here, but at the same time it's not hard to build something yourself to suit your needs - [example](https://github.com/matjazbravc/Custom.ConfigurationProvider.Demo).

But we still need to solve for (2). There is a decent framework for .NET [Microsoft.FeatureManagement](https://github.com/microsoft/FeatureManagement-Dotnet) and you can read about how to get started with it in this wonderful (but a little old) [series by Andrew Locke](https://andrewlock.net/series/adding-feature-flags-to-an-asp-net-core-app/). There are other solutions like [this library with ASP.NET UI](https://github.com/Odonno/FeatureManagement.UI) and now I'm looking forward for open source integrations with the new [Microsoft.Extensions.Options.Contextual](https://github.com/dotnet/extensions/tree/main/src/Libraries/Microsoft.Extensions.Options.Contextual).

Regardless of which lib you go with, you want to make your experiments sticky to a browser session or user identifier (for authenticated experiences) to ensure consistency (imagine the UI changes when you refresh the page - yuck!).

You want to end up with 50% of users getting the feature variant A and 50% getting the variant B. I recently learned it's very important to have a **very** consistent 50/50 split. Otherwise you can get into a Signal Ratio Mismatch - which means more users see one of the variants than the other. What's the big deal? It can mean that another factor is affecting the situation, and remember, we said we want our feature to be the only varying factor. There's a cool [paper on the topic](https://dl.acm.org/doi/10.1145/3292500.3330722).

Once you have code in place, the feature configured to be shown to the user groups, make sure it actually works and looks as expected before launching the experiment. A very useful thing is to have a feature flag override via query parameters or a cookie.

## Telemetry

I generally split the telemetry into three categories: diagnostics, health metrics, analytics. You use diagnostics to debug your service if somethings doesn't work, you display health metrics on a dashboard to see if everything is working well, and with analytics logs you store them somewhere for further processing to generate business relevant metrics.

For experiments we focus on analytics logs and we will care about a few things:

1. Session or User identifier - the unit of our measurements
2. Where they selected for the experiment? Was it shown to them?
3. Did they complete the action (or in general what actions did they execute in case the experiment affects some other action)?
4. When did stuff happen?
5. Other metadata to account for additional factors (e.g. country, browser, age group, etc)

Make sure to prepare questions for your experiment upfront so that you know what needs to be collected. It's really not great when you decide mid experiment you're missing some additional context. Example questions:

1. If we show the new UI, does the rate of submitting the form change?
2. If we show the new UI, do more people subscribe for mailing?
3. If the user visited our site before in the last month, are they much more likely to subscribe with the new UI (bigger % difference in comparison to first time visitors)?

Where do we put this telemetry? It will depend on your website's traffic size and budget. There's plenty Big Data storage solutions but the easiest is just to log it to a SQL database. Choose a flexible schema which allows you to add more analytics telemetry signals over time to build a rich picture of the user behavior.

Quick note: you may need to inform your user that you're collecting data and for what purpose. Experimentation and service improvement is generally acceptable under "required" data, while marketing personalization is usually optional and requires user consent. Be careful with user data. It's a good idea to store analytics telemetry separately from your users' personal data.

Once you've collected 'raw' telemetry you may want 'cook' it, by transforming it a bit. Suppose we have one table for page views, one table for button clicks on page, one table for the end state of submitted forms. You would aggregate those to get a single row for the user/session with all the context: `[who, page, variant, submitted, xBtnClickCount, xFinalState]`.

## Analysis

You can do some basic analysis yourself with ad hoc queries and often you will get a decent picture of how the experiment went, but there's a lot of statistics and complexity that needs to be accounted for in order to "get it right". One way is to learn more about Data Science, another is to find existing solutions which can connect to your database and give you the insights.

One of the nice open source projects I found is [GrowthBook](https://docs.growthbook.io/). It comes with a [statistics engine](https://docs.growthbook.io/statistics/overview) for data analysis. It also has feature flag management, but through its custom SDK instead of the popular .NET frameworks (I might try to integrate it with contextual one day). It can be [self hosted with Docker](https://docs.growthbook.io/self-host).

## Read more

1. [Practical Guide to Controlled Experiments on the Web](https://ai.stanford.edu/~ronnyk/GuideControlledExperiments.pdf)
2. [O'Reilly - Experiment!: Website conversion rate optimization with A/B and multivariate testing](https://learning.oreilly.com/library/view/experiment-website-conversion/9780133040067/)
3. [The Open Guide to Successful AB Testing](https://docs.growthbook.io/open-guide-to-ab-testing.v1.0.pdf)
4. [Growth Book Bayesian Statistics Engine](https://docs.growthbook.io/assets/files/GrowthBookStatsEngine-d239dd518fdfa7198be46489bb65b8e3.pdf)
5. [Microsoft Experimentation Platform](https://www.microsoft.com/en-us/research/group/experimentation-platform-exp/)
