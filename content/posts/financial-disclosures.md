+++
title = "I'm probably on a watchlist now"
date = "2022-08-27T22:23:33-05:00"
author = "Brandon Piña"
authorTwitter = "" #do not include @
tags = ["project","programming"]
keywords = ["twitter"]
description = ""
showFullContent = false
readingTime = true
hideComments = false
draft = false
+++

At the beginning of this year I was itching to start a new project, and found one that sparked my interest. At the time I noticed online that there was interest around tracking the finances of politicians in the U.S. Senate and House of Representatives. The reason being that representatives and senators are privy to and influence economic policy which can and often do impact the stock market. This presents a conflict of interest on their part as at this point they can still participate in the stock market and may pass policies that impact their portfolio favorably. As a result, [concern of insider trading in congress has been growing in recent years](https://prospect.org/power/congress-beats-wall-street-at-its-own-game/). Tracking politicians' finances isn't a new idea, (see the Nancy Pelosi ETF). It seems however, that previous trackers have been focused on specific members. This is why I presume **@PelosiTracker** was banned from twitter. I decided to track all congress' financial disclosures. But where would I get the data?

> On April 4, 2012, President Obama signed into law the Stop Trading on Congressional Knowledge Act of 2012, Pub. L. No. 112-105 (2012) (STOCK Act). Section 6 of the STOCK Act adds new subsection 103(l) to the Ethics in Government Act of 1978, 5 U.S.C. app. 4, § 101 et seq. (EIGA). Effective July 3, 2012, subsection 103(l) of EIGA requires that not later than 30 days after receiving notification of any transaction required to be reported under subsection 102(a)(5)(B) of EIGA, but in no case later than 45 days after such a transaction, a covered employee must file a report of the transaction.

## *Thanks, Obama*

The idea of the STOCK act and EIGA is to keep Congress honest. That's the idea at least. Financial disclosure data is available from the [House Clerk](https://disclosures-clerk.house.gov/PublicDisclosure/FinancialDisclosure) and the [Senate Clerk](https://efdsearch.senate.gov/search/home/) websites. The problem remains on how to get the data from these sites. I used a library called puppeteer to control a headless chromium web browser to scrape the documents from the house disclosure site. This gave me metadata such as Name, Date and Report type as well as a url where I could download a PDF of the submitted report. Now that I have the PDF File, How am I going to post it to twitter? As of now, there is no way to post a PDF as an attachment to a tweet. I need to convert it to JPEG files

## This is where the Magick happens

There is a FOSS tool out there called ImageMagick that can convert images between formats and even do some processing as well. The conversion feature suits my needs perfectly fine. Importantly, we can control it using a library. I found a library with bindings for node unsurprisingly called *imagemagick* and I started merrily converting away. To tell the truth there's not much to say here, despite the fact that without this specific component, this project would have never shipped.


## Posting to Twitter

To get an API key for Twitter, you need to submit an application and furthermore to get write access (which I need to post tweets) I need to submit yet another application. I managed to get write access easily enough, but at first my write access application was denied for reasons unknown. I appealed the decision and managed to get the access I needed to start tweeting.

## Deployment
In a monster fueled fever dream, I managed to get a proof of concept up and running in the space of 24 hours, but there was still the matter of deploying the bot. I was having issues docker-izing the bot as it required dependencies outside the npm ecosystem

Here's the dockerfile I ended up with
```dockerfile
FROM node:16-bullseye-slim
WORKDIR /usr/src/app
COPY . .

# Set imagemagick's cache directory to our mount point so that we don't bloat our docker storage
ENV MAGICK_TMPDIR /usr/src/app/config/cache

# Do stuff that puppeteer requires for some reason
RUN apt-get update \
  && apt-get install -y wget gnupg \
  && wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add - \
  && sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list' \
  && apt-get update \
  && apt-get install -y google-chrome-stable fonts-ipafont-gothic fonts-wqy-zenhei fonts-thai-tlwg fonts-kacst fonts-freefont-ttf libxss1 \
  --no-install-recommends \
  && rm -rf /var/lib/apt/lists/*

# Insall other deps
RUN apt-get update && apt-get install -y imagemagick ghostscript

# Edit ImageMagick PDF Policy for read/write access
RUN sed -i_bak \
's/rights="none" pattern="PDF"/rights="read | write" pattern="PDF"/' \
/etc/ImageMagick-6/policy.xml

# Edit ImageMagick policy for a bigger disk cache
RUN sed -i \
's/domain="resource" name="disk" value="1GiB"/ domain="resource" name="disk" value="10GiB"/' \
/etc/ImageMagick-6/policy.xml

# Limit each instance's thread count to 1
RUN sed -i \
's/domain="resource" name="thread" value="2"/ domain="resource" name="thread" value="1"/' \
/etc/ImageMagick-6/policy.xml

# Give imagemagick more memory
RUN sed -i \
's/domain="resource" name="memory" value="256MiB"/ domain="resource" name="memory" value="2GiB"/' \
/etc/ImageMagick-6/policy.xml

# Install nodejs dependencies
RUN npm ci

# Run app as node
#USER node

RUN mkdir config

RUN mkdir config/img

RUN mkdir config/pdf

RUN mkdir config/cache

CMD ["node","index.js"]
```

As you can see, the dockerfile I ended up with is substantial. Some of the issues I had to contend with were:
* Apt Couldn't find my dependencies in the apt repository
    * This was solved by prepending `apt-get install` commands with `apt-get update`
* Imagemagick requires ghostscript as a dependency
* puppeteer's chromium installation couldn't launch
    * After scouring stackoverflow I found that my docker image was missing necessary resources for chrome to run, like fonts. I ended up installing google chrome via apt to resolve chromium's dependency issues
* Imagemagick didn't have permission to read the filesystem
    * I had to use the `sed` command to edit imagemagick's `policy.xml` to allow it to access to the filesystem so that it could convert pdfs
* Imagemagick uses the filesystem as a cache. This is a problem as if you don't mount imagemagick's cache directory to a docker volume, your `docker.img` size will bloat as pdfs are and converted
    * Imagemagick conveniently allows you to configure it's cache directory using the environment variable `MAGICK_TMPDIR`

## Conclusion
I'm pretty happy with the result I ended up with. I learned a lot about web scraping and more complicated deployments with docker. I had a few regrets after I was done though. For one, I only ended up scraping data from the house clerk site, as at the time I made the false assumption that the house clerk served both senate and house disclosures. Secondly, I threw this project together using plain javascript, with no framework or typescript. The lack of typescript is probably what ended up hurting me the most because when I went back to add support for senate disclosures, I had to try to remember what data lived in my variables and typescript would have made my life much simpler. I'm currently in the process of rewriting this project using `NestJS` which is a framework inspired by angular which makes managing the organization of your code much easier. I can separate out the concerns of components of my project by domain into modules which give me a warm fuzzy feeling inside. Some other improvements I could have made were to first check if an API exists to gather this information before bothering with web scraping, but I was in such a hurry to see it working so I failed to do so. I'm currently having issues scraping the Senate Clerk site so that might be the push I need to go that route.

## TLDR:
I wrote a twitter bot that tracks congressional financial disclosures
* Lessons Learned
    * Before resorting to web scraping, check if an API already exists
    * Typescript is your best friend when you revisit your project a few months after deployment
    * When starting a project, think about organization. Would you benefit from using a framework?
    * Deployment is sometimes harder than development

The bot is currently hosted on a Dell R710 that lives under my bed. You can see the bot at [@CongressionalFD](https://twitter.com/congressionalfd), and you can look at the source code [here](https://github.com/ThatNerdUKnow/Financial-Disclosures)
