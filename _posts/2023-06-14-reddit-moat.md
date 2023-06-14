---
layout: post
title: "Reddit is OpenAI’s moat"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-06-14
tags: ai, featured
description: Interpreting Reddit's API changes in light of how it serves OpenAI.
---
*Edit: This blew up on [Hacker News](https://news.ycombinator.com/item?id=36325958). An HN mod edited the title to add a ? at the end. It was not me. The answer to any headline with a ? at the end is “no.” I would not own myself like that. Whoever at HN edited it — that's cowardly and lazy. Feel free to argue against the piece on its merits. (Obviously, I don't need to write an article about Hacker News' affiliations.) Edit: another HN mod [shat all over the article](https://news.ycombinator.com/item?id=36328562) and retitled it again. Again, that was not me. Edit: the HN title has been reverted to the original, which I appreciate.*

Much has been written about the Reddit boycott recently, but I have kind of a wild take. A lot of the analyses have examined the issue as if Reddit is an independent company preparing for an IPO. That is, they have examined Reddit's attempts to capture its value as a training corpus or its attempts to show its users more ads. But what if we thought of Reddit as, functionally, subservient to OpenAI?

Now, hear me out. Organizations/people with an interest in OpenAI – like Sam Altman and a16z – have a significant stake in Reddit, and strong ties to the board. Altman himself was on the board until 2022, [participated in the 2014 YC takeover of Reddit](https://blog.samaltman.com/a-new-team-at-reddit), and thus likely has strong influence over the company.

So, if you had control over both OpenAI and Reddit, what would you do?

## Why OpenAI is expanding its moat
OpenAI needs to expand its moat. While it already has a strong moat in the form of a dedicated user base, these are the early days and more capable (or less restricted) LLMs could still take ChatGPT’s lunch (the way that Midjourney took DALLE2’s lunch).

The now-famous [We Have No Moat, And Neither Does OpenAI](https://www.semianalysis.com/p/google-we-have-no-moat-and-neither) makes a strong argument that access to compute is not a good moat and that the biggest threat to Google (and OpenAI) are open-source models.

One way that Altman is aiming to cover OpenAI from threat of open-source models is by lobbying the bajeezus of the US, UK, and China to raise the barrier to entry for upstart AI organizations. This is exactly what his suggestion that AIs should need a license to operate would achieve.

## Training data is a good moat
Similarly, while access to compute is not a moat for developing LLMs, access to high quality data is. And that is where Reddit enters the picture.

There is no question that Reddit is extremely valuable as training data. How often do you append “reddit” to your searches?

*Edit: I mention this later, but I want to make it abundantly clear that Reddit's **future** data is what OpenAI is most concerned about protecting. Existing data has already been scraped.*

It's no secret that [Reddit’s API changes are being driven significantly by the desire to capture the value of its corpus](https://www.theverge.com/2023/4/18/23688463/reddit-developer-api-terms-change-monetization-ai). I think the missing piece, though, is that it doesn’t matter if anyone buys the data or not. The important piece is that it's easiest for OpenAI to get the data (given that companies with co-investors help each other), somewhat harder for Google, and extremely hard for upstarts.

## What's Reddit going to do next?
Of course, the needle that OpenAI must thread is closing off Reddit without killing it. Reddit’s data is valuable *going forward*, and losing the community would destroy the moat. Looking at Twitter, one might say that Twitter remains valuable (as a data set) even with many users gone. This is the gambit: even if Reddit goes nuclear and wrests control of all the subreddits away from the mods, people will *still* come to Reddit to ask and answer valuable questions. Like, do the people who want to ask (or tell others) the best mattress topper in the UK care about Reddit’s internal governance structure?

It seems like a good gambit: locking down Reddit creates a giant moat, while losing it merely fails to create a moat (which is why OpenAI is looking for other moats).

What else might Reddit do next?
* They could buy out more 3rd party clients, which would somewhat appease the community. This would be a terrible move IPO-wise.
* They could extend the June 30th deadline, which I think is likely. This would get rid of 3rd party apps while, perhaps, keeping most of the community intact. Most of Reddit’s current data has been scraped anyway, so the game is to protect Reddit’s data going forward. On the other hand, I may be wrong, but extending this deadline pushes back the IPO, which may not be a very good idea – I don’t know enough about this sort of business decision to say.

I want to address one strong idea that counters my theory: closing off 3rd party API access mostly serves an IPO, not OpenAI. If Reddit merely wanted to restrict the ability to scrape its data, they could have done so without killing off clients – e.g. via licensing deals[1]. However, perhaps if access to training data is seen as an elbows-out brawl, I could see how Reddit would be *extremely* protective of its data. I mean, lyrics websites, map makers, and dictionaries go to great lengths to protect their data. It would not be a giant stretch for Reddit to do so as well.

## Conclusion
I realize that this is a pretty wild theory, but I'm not suggesting a conspiracy. There is no conspiracy: shareholders of two companies could very well use one company to benefit the other. Anyway, I would love to see someone poke holes in this. It's a fun theory, I think.

## Notes
[1] As mentioned by Ben Thompson on a recent Dithering episode.