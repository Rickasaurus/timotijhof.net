---
layout: post
title: Wikipedia's JavaScript initialisation on a budget
excerpt: <p>This week saw the conclusion of a project I've been shepherding since September of last year. The goal was for the initialisation code for our JavaScript pipeline to fit within a budget of 28 KB.</p>
tags: wikipedia performance javascript
plainwhite:
  og_title: Wikipedia's JavaScript on a budget
  og_image_url: /assets/attachments/2019_wikipedia_figure1_chart.png
  og_image_alt: Chart going downwards
  og_image_align: aside
  original_url: https://phabricator.wikimedia.org/phame/live/7/post/175/wikipedia_s_javascript_initialisation_on_a_budget/
  original_label: wikimedia.org
---

This week saw the conclusion of a project that I've been shepherding on and off since September of last year. The goal was for the initialisation of our asynchronous JavaScript pipeline (at the time, 36 kilobytes in size) to fit within a budget of 28 KB.

![Chart showing a decline in Startup manifest size from 36.2 kilobytes in 2018 to just under 28 KB in September 2019](/assets/attachments/2019_wikipedia_figure1_chart.png "From 36.2 KB to 27.2 KB"){:style="max-height:370px" class="md-box"}
{:class="md-block"}

The above graph shows the transfer size over time. Sizes are after compression (i.e. the net bandwidth cost as perceived from a browser).

In total, the year-long effort is saving 4.3 terabytes a day of data bandwidth for our users' page views.

-------

## How we did it

The startup manifest is a difficult payload to optimise. The vast majority of its code isn't functional logic that can be optimised by traditional means. Rather, it is almost entirely made of pure data. The data is auto-generated by ResourceLoader and represents the registry of module bundles. ([ResourceLoader](https://www.mediawiki.org/wiki/ResourceLoader/Architecture) is the delivery system Wikipedia uses for its JavaScript, CSS, interface text.)

This registry contains the metadata for all front-end features deployed on Wikipedia. It enumerates their name, currently deployed version, and their dependency relationships to other such bundles of loadable code.

I started by identifying code that was never used in practice ([task #202154](https://phabricator.wikimedia.org/T202154 "Reduce registry overhead (Audit modules 2018).")). This included picking up unfinished or forgotten software deprecations, and removing unused compatibility code for browsers that no longer passed our [Grade A](https://www.mediawiki.org/wiki/Compatibility#Browsers) feature-test. I also wrote a [document about Page load performance](https://www.mediawiki.org/wiki/Wikimedia_Performance_Team/Page_load_performance). This document serves as reference material, enabling developers to understand the impact of various types of changes on one or more stages of the page load process.

## Fewer modules

Next was collaborating with the engineering teams here at Wikimedia Foundation and at Wikimedia Deutschland, to identify features that were using more modules than is necessary. For example, by bundling together parts of the same feature that are generally always downloaded together. Thus leading to fewer entry points to have metadata for in the ResourceLoader registry.

Some highlights:

* Editing product team (WMF):<br>The WikiEditor extension has 11 fewer modules now. Another 31 modules were removed in UploadWizard.
* Language product team (WMF):<br>Combined 24 modules of the ContentTranslation software.
* Reading product team  (WMF):<br>Combined 25 modules in MobileFrontend.
* Community Wishlist team (WMDE):<br>Removed 20 modules from the RevisionSlider and TwoColConflict features.

Last but not least, there was the Wikidata client for Wikipedia. This was an epic journey of its own ([task #203696](https://phabricator.wikimedia.org/T203696 "Reduce the number of modules that WikidataClient registers.")). This feature originally had a whopping 248 distinct modules registered on Wikipedia page views. The magnificent efforts of Amir Sarabadani **removed over 200 modules**, bringing it down to 42 today.

The bar chart above shows small improvements throughout the year, all moving us closer to the goal. Two major drops stand out in particular. One is around two-thirds of the way, in the first week of August. This is when the aforementioned Wikidata improvement was deployed. The second drop is toward the end of the chart and happened this week – more about that below.

-------

## Less metadata

This week's improvement was achieved by two holistic changes that organised the data in a smarter way overall.

First – The [EventLogging](https://www.mediawiki.org/wiki/Extension:EventLogging) extension previously shipped its schema metadata as part the startup manifest. Roan Kattouw ([@Catrope](https://twitter.com/Catrope)) refactored this mechanism to instead bundle the schema metadata together with the JavaScript code of the EventLogging client. This means the startup footprint of EventLogging was reduced by over 90%. That's 2KB less metadata in the critical path! It also means that going forward, the startup cost for EventLogging no longer grows with each new event instrumentation. This clever bundling is powered by ResourceLoader's new [Package files](https://www.mediawiki.org/wiki/ResourceLoader/Package_modules) feature. This feature was expedited in February 2019 in part because of its potential to reduce the number of modules in our registry. Package Files make it super easy to combine generated data with JavaScript code in a single module bundle.

Second – We shrunk the average size for each entry in the registry overall ([task #229245](https://phabricator.wikimedia.org/T229245 "Reduce the size of version hashes.")). The startup manifest contains two pieces of data for each module: Its name, and its version ID. This version ID previously required 7 bytes of data. After thinking through the [Birthday mathematics problem](https://en.wikipedia.org/wiki/Birthday_problem) in context of ResourceLoader, we decided that the probability spectrum for our version IDs can be safely reduced from 78 billion down to "only" 60 million. For more details see [the code comments](https://github.com/wikimedia/mediawiki/commit/9f516f1d3b6ab6a4f1bb7e385c93e4d9bccb46d7#diff-57e85f8b8063990fa5b0e2d2f0d25f8e), but in summary it means we're saving 2 bytes for each of the 1100 modules still in the registry. Thus reducing the payload by another 2-3 KB.

Below is a close-up for the last few days (this is from synthetic monitoring, plotting the decompressed size):

![Line graph showing a sudden drop in Startup JS size from 55.6KB to 52.8KB](/assets/attachments/2019_wikipedia_figure2_synth.png "From 55.6KB to 52.8KB (decompressed)"){:class="md-box"}

The change was detected in ResourceLoader's synthetic monitoring. The above is captured from the [Startup manifest size dashboard](https://grafana.wikimedia.org/d/BvWJlaDWk/startup-manifest-size?orgId=1&from=1568439360000&to=1568680200000) on our public Grafana instance, showing a **2.8KB** decrease in the uncompressed data stream.

With this week's deployment, we've completed the goal of shrinking the startup manifest to under 28 KB. This cross-departmental and cross-organisational project reduced the startup manifest by **9 KB** overall (net bandwidth, after compression); From 36.2 kilobytes one year ago, down to 27.2 KB today.

We have around 363,000 page views a minute in total on Wikipedia and sister projects. That's 21.8M an hour, or 523 million every day ([User pageview stats](https://stats.wikimedia.org/v2/#/all-projects/reading/total-page-views/normal%7Cbar%7C2-year%7Cagent~user%7Cmonthly)). This week's deployment saves around 1.4 terabytes a day. In total, the year-long effort is saving 4.3 terabytes a day of bandwidth on our users' page views.

-------

## What's next

![Percentage of bundle metadata size, by component. 26% is for MediaWiki core's bundles, 12% for ContentTranslation bundles, 7% for VisualEditor, 5% for Wikidata.](/assets/attachments/2019_wikipedia_figure3_pie.png "Percentage of bundle metadata size, by component"){:style="max-height:280px"}
{:class="md-block md-block-right"}

It's great to celebrate that Wikipedia's startup payload now neatly fits into the target budget of 28 KB – chosen as the lowest multiple of 14KB we can fit within subsequent [bursts of Internet packets](https://tylercipriani.com/blog/2016/09/25/the-14kb-in-the-tcp-initial-window/) to a web browser.

The challenge going forward will be to keep us there. Over the past year I've kept a very close eye ([spreadsheet](https://docs.google.com/document/d/1SESOADAH9phJTeLo4lqipAjYUMaLpGsQTAUqdgyZb4U/edit)) on the startup manifest — to verify our progress, and to identify potential regressions. I've since automated this laborious process through a public [Grafana dashboard](https://grafana.wikimedia.org/d/BvWJlaDWk/startup-manifest-size).

We still have many more opportunities on that dashboard to improve bundling of our features, and (for Wikimedia's Performance Team) to make it even easier to implement such bundling.

– Timo Tijhof

-------

_Further reading:_
* [Metrics & Perf reports](https://performance.wikimedia.org/), on performance.wikimedia.org
* [ResourceLoader Architecture](https://www.mediawiki.org/wiki/ResourceLoader/Architecture), on mediawiki.org.
* [The 14KB Initial Window](https://tylercipriani.com/blog/2016/09/25/the-14kb-in-the-tcp-initial-window/), by Tyler Cipriani.
