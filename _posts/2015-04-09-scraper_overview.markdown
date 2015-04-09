---
layout: post
title:  "Scraper: Overview"
date:   2015-04-09 06:09:00
categories: scraper node meteor phantomjs cheeriojs nginx
---
Web Scraper
===========
Over the past few months, I've been working on building a [web scraper](http://en.wikipedia.org/wiki/Web_scraping)--an aplication that can read websites, extracting structured data from the websites, and storing the data for later retrieval.  The primary use case I have for a web scraper is to retrieve corporate financials and stock options contract prices.  All of this is an attempt to implement the work I did for my Master's dissertation, where I attempted to identify mis-valued corporations, and to take maximal advantage of their impending price corrections.

Overview
--------
I've been responsibile for the design and maintenance of web scrapers at past corporations I've worked with.  My goal is to pull the best ideas from these scrapers, add in a few ideas I've had for improvements, to build a world-class open-source web scraper.  My aim is to build a web scraper capable of scraping any page, even those that use Javascript to manipulate the DOM and those that require complex login procedures prior to accessing the desired pages.  The web scrapers should be allowed to run on remote networks as part of a syndicate of scrapers.  The syndicate of scrapers should know about one another, in order to prevent excessive traffic at the target site.  The scrapers should be able to be controlled centrally, but also act completely autonomously in the event of any outages that might occur with the syndicate controller.  In general, throughout the design, the aim is to have no single failure point.

Each scraper should be able to have complex rules for scraping.  They should be able to validate their output to make sure the data was scraped properly.  They should be able to perform statistical analysis over historical scrapes to ensure that the current data does not have any suspicious data that could indicate either scraping errors or invalid data published by the scraped site.  They should be able to tell when data has changed from a prior scrape.  If a website is expected to be updated at a specific time, the syndicate of scrapers should allow for coordinated requests to the scraped site so that the data can be scraped and interpreted as soon as the data is published, all without presenting excessive traffic to the scraped site.

Scrapers should be able to update one or more remote data servers.  The role for the data servers is to collect, hold, and organize the data for users.  These remote data servers should be able to handle arbitray data types without requiring extensive setup prior to scraping.  The remote data servers should be able to talk to one another, independently of the syndicate of scrapers, to federate data sets among servers.  A side effect of this data federation is that new remote data servers should be able to sync historical data from other servers. 

The process of building a new scraper, or maintaining an existing scraper, should be simple and easy.  Small and simple changes in the HTML and CSS of the scraped site can cause scrapers to fail, so fixing them should be simple an easy.  This, combined with automated data integrity checks, should allow for scrapers to be maintained easily and efficiently, with minimal down time when scrapers fail.

In addition to the functional requirements for the project, I also have a non-functional requirement that is purely personal.  My intent is to use this project to develop proficiency with Javascript, Node, and many of the leading-edge technologies that are being developed in the Javascript community.  This list of technologies includes: [Meteor](http://meteor.com/), [Node](https://nodejs.org/) & [IO.js](https://iojs.org/en/index.html), [MongoDB](https://www.mongodb.org/), [Flux](https://facebook.github.io/react/docs/flux-overview.html), [React](https://facebook.github.io/react/), [Express](http://expressjs.com/), [Angular](http://expressjs.com/), and many more.

Early Tech Choices
------------------
I have an alterior motive with my non-functional requirement to learn Javascript.  Based upon my experience with scrapers in the past, I've found that some of the emerging web technologies present difficulties for standard web scraping tools like [wget](https://www.gnu.org/software/wget/) and [curl](http://curl.haxx.se/).  Namely, many modern websites make extensive use of Javascript to manipulate the DOM before presenting content to users.  As a result, a web browser with a fully functioning Javascript interpreter is needed.  In my experience in the past, I knew this could be done with [Selenium's WebDriver](http://www.seleniumhq.org/projects/webdriver/), powering an actual instance of IE, Firefox, Chrome, or Safari to retrieve the websites.  I also knew that the resulted in a fragile collection of technologies, resulting in less reliable performance than I would get with alternate technologies.  I figured that the emergence of [Google's V8 Javascript engine](https://code.google.com/p/v8/) and [Node.js](https://nodejs.org/) might provide server-side technologies to scrape websites, avoiding the complexities of the WebDriver approach.  

As it turns out, there are a number of Javascript libraries that make web scraping easier in Javascript than any other compteting language/environment I have found.  [Cheerio.js](https://github.com/cheeriojs/cheerio) provides the ability to search and manipulate the DOM, extracting stuctured data easily.  [Phantom.js](http://phantomjs.org/) provides a fully functioning web browser, allowing Javascript-heavy websites to be scraped without resorting to WebDriver.  

The web scraper syndicate controllers and the data servers require a graphical user interface, as these are the two primary points of user interaction with the system.  I had recently built a website for a financial management firm using [Meteor](https://www.meteor.com/), and found that the resulting website felt more like a "fat client" application than a web browser application.  I also was happy with my productivity with Javascript and the Meteor stack.  Even though I was a relative novice with the technologies, the project still moved at a quick pace.  None the less, as that project grew in size, I found myself increasingly aware of how little I knew about modern Javascript (or, more accurately, how fast the Javascript community has been moving since the introduction of V8 & Node).

This lead me to my choice for Meteor for the client-facing portions of the application.  My hope is to build a GUI that encourages social interaction between users regarding web scrapers and scraped data.  I envision a web site where users can request or suggest sites to be scraped, with a community of developers willing and able to build the scraping logic.  In short, I felt a social news website would serve as an excellent foundation for this.  Discussion threads should feature prominently.  Users should be able to upvote or downvote requests and submissions, so that the most important ones would receive the most attention from the community.  I was familiar with [Telescope](http://www.telescopeapp.org/) (and [Microscope](https://github.com/DiscoverMeteor/Microscope)), built by Tom Coleman and Sacha Grief, and used as the foundation for their [Discovering Meteor](https://www.discovermeteor.com/) book.  I felt this would make an excellent starting point.

The various servers would require extensive server-to-server communication.  Each syndicate of web scrapers require server-to-server communication between the scrapers and the syndicate controller.  After the structured data is extracted, the web scrapers require server-to-server communication to update the data servers with the latest data.  For this purpose, I have selected a number of technologies that can work--[Meteor's DDP](https://www.meteor.com/ddp), a message queue like [0MQ](http://zeromq.org/), and a [REST API](http://en.wikipedia.org/wiki/Representational_state_transfer).  Since this is a learning project for me, I decided that I would implement all three.  Besides, the REST API would allow me to utilize any language for the scraping servers, allowing me to incorporate "legacy" scrapers built I have encountered, using technologies like wget, curl, Java, and Python.  Given the feature set of DDP and REST, a message queue-based solution wasn't necessary.  A message queue-based solution might provide additional robustness in the vent of outages, and could allow for flexible deployment-time decisions for routing that can be tremendously useful towards the goals of scalability and fault tolerance. 

Deployment Technologies
-----------------------
I knew from my past experience that I would want a front web server like [nginx](http://nginx.org/) to handle HTTPS and to allow for multiple instances of Meteor or Node so that one can make full utilization of all of the CPU cores available to them (V8/Node is inherently single-threaded; to make use of multiple cores on servers, one must run multiple instances of V8/Node, with a front web server maintaining sticky sessions). 

I also would like a solution that allows for flexible deployment.  In the early stages of development, I would prefer to deploy to one of the cloud computing services, like [AWS](http://aws.amazon.com/) or [Digital Ocean](https://www.digitalocean.com/).  Over time, I'd like to have the option of deploying to self-managed servers.  Additionally, testing and development should not be too onerous.  Luckily, over the past few years a very interesting technology has been created to meet this purpose: [Docker](https://www.docker.com/).  Docker allows for the building of application "containers" that can be easily deployed in test environments, to cloud computing services, and to self-managed production hardware.  Furthermore, Docker provides additional security features that limit and protect applications from accessing the environment of other applications on the same server. 

Concerns
--------
I do have some concerns about Meteor that I hope to address during this development.  First, the choice of MongoDB worries me--I have read numerous articles analyzing the performance and reliability of MongoDB.  Second, Meteor provides a lot of coding "magic" that obfuscates the inner workings, and can make it difficult to track down odd rendering bugs in my own code.  In particular, I have had difficulties understanding when and how Meteor reacts to data changes, resulting in unnecessary multiple screen redraws.  However, I feel that both of these issues can be overcome in this project.  For instance, I could use a traditional database as the data store of record, with MongoDB acting as a bit of a "slave", cloning the data in the data store of record.  As for Meteor, I feel that as I become more experienced with the entire development stack, my understanding of the internals of Meteor will grow so that I will no longer consider it a "black box".

The Full Stack
--------------

## Syndicate Controller
* [Meteor](https://www.meteor.com/)
  * [V8](https://code.google.com/p/v8/) + [Node](https://nodejs.org/)/[IO.js](https://iojs.org/en/index.html)
  * [MongoDB](https://www.mongodb.org/)
  * [Blaze](https://www.meteor.com/blaze)
* [Telescope](http://www.telescopeapp.org/), with modifications
* [Docker](https://www.docker.com/)

## Web Scraper
* [V8](https://code.google.com/p/v8/) + [Node](https://nodejs.org/)/[IO.js](https://iojs.org/en/index.html)
* [Phantomjs](http://phantomjs.org/)
* [Cheeriojs](https://github.com/cheeriojs/cheerio)
* [Spookyjs](https://github.com/SpookyJS/SpookyJS)
* [REST API](http://en.wikipedia.org/wiki/Representational_state_transfer)
* [Docker](https://www.docker.com/)

## Data Server
* [Meteor](https://www.meteor.com/)
  * [V8](https://code.google.com/p/v8/) + [Node](https://nodejs.org/)/[IO.js](https://iojs.org/en/index.html)
  * [MongoDB](https://www.mongodb.org/)
  * [Blaze](https://www.meteor.com/blaze)
* [PostgreSQL](http://www.postgresql.org/)
* [Experiment](https://github.com/reactjs/react-meteor) with [ReactJS](https://facebook.github.io/react/) as a replacement for Blaze for constructing the View
* [REST API](http://en.wikipedia.org/wiki/Representational_state_transfer)
* [Docker](https://www.docker.com/)
