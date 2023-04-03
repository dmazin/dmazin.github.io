---
layout: post
title: "On OpenAI's Terrible Arguments for Opaqueness"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2023-04-03
tags: ai, featured
description: Explore the crucial need for transparency in AI research, particularly at OpenAI, as the author argues that public welfare must take precedence over business interests and safety concerns.
---
Unlike the slew of AI letters released in the past couple weeks, you will find no signatures of respected researchers under this letter. I am just a guy. But I have to get on the soap box and say my piece.

We must force AI research groups to be transparent. We should demand the same transparency we expect from public research.

If you are not aware, almost all the research into large language models, image generation, and other forms of disruptive AI are being done behind closed doors by for-profit consumer technology companies. While the teams do, technically, publish PDFs resembling academic papers, these are more like product announcements: they lack key details like model architecture, parameter size, and, crucially, training sources. 

Before I continue, it's important to establish my biases. I, for one, am excited about LLMs (large language models), especially code generation. It has made me far more productive and ambitious than ever before. I hope it can do the same for others. However, I am also wary of the coming changes. Five whole years ago, in [Pink Lexical Slime](/2017/12/12/pink-lexical-slime.html), I wrote: 
> We are increasingly able to bless AI with more and more agency (or more precisely, what we believe *passes* for agency). […] We must approach AI breakthroughs with skepticism. Not only does technology wield great power over us, it advances at an inhuman pace. Unless we anticipate its coming challenges, we will realize them only in hindsight.  

I believe that the lack of transparency in AI seriously hampers our ability to anticipate the coming challenges. What's more, it prevents us from building the tools we need to deal with misuse of AI.

The transparency issue is not limited to OpenAI – and we should demand it from all the big consumer tech companies driving disruptive AI research – but I am going to pick on them. They are the clear leaders, and they claim moral high ground. I feel a little bad, because they seem like good folks. But OpenAI was supposed to be the non-profit that made AI research serve the people, and they violated that promise horribly.

By the way, for a cogent argument why OpenAI is now a consumer technology company, see [Ben Thompson's “The Accidental Consumer Tech Company”](https://stratechery.com/2023/the-accidental-consumer-tech-company-chatgpt-meta-and-product-market-fit-aggregation-and-apis/).

The reasons OpenAI gives for its lack of transparency are two-fold: business and safety.

## Public Welfare Must Take Precedence Over OpenAI's Business
Let's start with the reason of business. Getting to where OpenAI is today took billions of dollars, talent, and work. Understandably, OpenAI wants to recoup its investment. At the end of the day, though, public welfare should take precedence over OpenAI's liabilities. If necessary, the public should pay to spin out, and continue to fund, OpenAI's research division. This would hamper the now-thriving consumer business (precluding them from first access to future models), but tying the research division to the consumer business is untenable if it means lack of transparency.

I say this as someone who enjoys OpenAI's products! I want to continue enjoying those products. I believe that transparency would not actually preclude them from making great products, but merely from maximizing profit.

## OpenAI's Terrible Safety Argument
Then, there is the reason of safety. I will dwell much longer on it. OpenAI's argument goes something like this. We are on track to achieving AGI (artificial general intelligence), and we're closer than anyone else. As soon as AGI is achieved, something immediate and massive will happen. Also, we take alignment (making sure that AI objectives line up with our own) seriously. So, by keeping things closed, we ensure that AGI is well aligned.

First, let's just set aside AGI completely. It's a topic with so many unknowns, plus, the pre-AGI models are disruptive enough on their own. And, OpenAI's argument kind-of takes the singularity as a given, which is extremely controversial.

Leaving AGI aside, OpenAI's central safety argument becomes weak. To his credit, Sutskever admits that the safety argument is not that strong for the current models, and their primary motivation for keeping the models closed is business. Still, OpenAI usually gives safety as the reason for keeping their models closed, so let's examine it.

OpenAI's team is, clearly, the current world leader. They also take safety seriously. Their products – DALLE, GPT, ChatGPT – are powerful and disruptive. Therefore, by keeping these products closed, they ostensibly ensure their safety.

What happened with DALLE2, though? When OpenAI launched DALLE2, they kept it closed so that they could enforce safety rules. It wasn't long before other models – with far less focus on safety – became capable.

LLMs are far harder to develop, train, and deploy than image generation. Still, multiple teams have already developed powerful LLMs. I believe that extremely sophisticated, unaligned, LLMs – BadChatGPT – are inevitable. The compute necessary is not enough of a detriment for a sufficiently motivated group of people, and it seems that the secret sauce is not that secret.

Because OpenAI keeping things closed won't prevent BadChatGPT, societal benefits to opaqueness disappear. The best we can do is to deeply understand the models so that we can prepare ways to deal with BadChatGPT. And OpenAI's lack of transparency, obviously, stands in the way.

## AGI Is Not Immune to Multiple Discovery
Let's deal with the singularity and AGI, after all.

First, OpenAI's safety argument depends heavily on believing in the singularity. While the singularity has its advocates, a reasonable person should not buy it. For a really good explanation why, see [Ted Chiang's “Why Computers Won't Make Themselves Smarter”](https://www.newyorker.com/culture/annals-of-inquiry/why-computers-wont-make-themselves-smarter).

Second, if/when OpenAI comes close to achieving AGI, someone will too. Nothing is immune to multiple discovery (the effect where the same thing tends to get invented multiple times, independently). Even something as novel as calculus was invented twice.

The most important point is the same from above: transparency would help us prepare for AGI, while opaqueness would bring no benefit.

## Research Transparency Is Only Part Of It
In this essay, I have focused on transparency. I believe this alone will be a major struggle. However, I have not even touched on transparency among the people/entities who use AI: that is its own ball of ethics.

Speaking of ethics, I know that I will disappoint some by focusing on safety, rather than ethics, in my arguments. However, I hope that this is alleviated by the fact that I am fighting for the same thing: namely, forcing AI research teams to prioritize public welfare over profits by making their research transparent.
