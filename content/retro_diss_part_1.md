Title: Retrospective Dissertation - Part 1: Introduction
Date: 2020-07-05
Category: retrodiss
Tags: python, weather
Slug: retro-diss-part-1
Author: fma
Summary: What if I were able to apply what I've learned in 5+ years in industry to my dissertation research?

## 0. Introduction
By chance, I happened to stumble across some Python code I wrote back some time
around 2012-2013. It was related to my dissertation research involving probabilistic
forecasting of tornadic activity up to 10 days in advance. I remember setting up
scripts that generated the predictions and sending it off to NOAA/ESRL, where they
ran for over three years as part of an experimental product.

There were some interesting, uh, quirks to that code. I had programmed
mostly in Fortran 77/90 up to that point. It was suggested to me by
my graduate school mentor to learn Python, and I definitely did a great job
programing Fortran in Python. Some other cringe-worthy parts of the code
include:

- Non-descriptive variable names, which I likely inherited from the F77 code
my advisor provided me.
- Not a lot of documentation, either in-line comments or for the functions
themselves. Combined with the non-descriptive variable names, this makes it
really difficult to figure out what the hell is going on (and this is _my_ research).
- I didn't write any tests, and honestly I had no idea that one could write
tests for code back then.
- Loops... lots and lots and _lots_ of loops, despite using `numpy` and having
vectorized computation being an option.

It's been at least six years since I wrote some of that code. Since then, I've worked
in tech as a research meteorologist, quantiative researcher, tech/team lead, and research scientist. I've
learned a whole lot from some very amazing colleagues about writing Python, statistics
and machine learning, how to approach data science projects, reproducibility,
version control (!!!), and making the transition from data science research
to production-based systems. It got me thinking...

...knowing what I know now, how would I tackle the research problem my dissertation
research was based on?

## 1. Oh God, Why?

I'll admit, this sounds a whole hell of a lot more sadistic than ambitious. Graduate
school is rarely a good time for anyone, as any statistic about the impact on one's
mental health during grad school will show. I was no different, and the impacts
still affect me to this day.

However, there are three important reason to me to go through this exercise:

- I'm a huge proponent of open science, and I've long felt that I have
done myself and the field a disservice by not having more of this out in the open.
- I've been _very_ fortunate to have a lot of great mentors and colleagues over
the years. Not everyone is as fortunate, so I'm hoping I can pass along and share
the amazing advice, wisdom, and experiences to others.
- It's important for my mental health to recognize how far I've come since
graduate school. For a long time, I let the habits, fears, and insecurities from
graduate school define me (e.g. no work-life balance, imposter syndrome, etc.). I'm
considering this a kind of exorcising of my grad school demons.

## 2. What Will We Be Doing?

Specifically, we will work towards generating skillful probabilistic predictions
of tornadic activity within a specific radius of a point roughly five days in
advance. We will be collecting/cleaning/aggregating data, performing exploratory
data analysis, engineering features, developing and evaluating model performance. What I **won't** be doing is re-writing my entire dissertation (lol).

Given the nature of this problem, it's important to make note of
what this is and isn't:

- This **IS** a retrospective investigation, so while the end goal is to develop
a model with predictive skill...
- This **IS NOT** a forecast product! You should always refer to the Storm Prediction
Center and your local National Weather Service office for _actual_ forecasts of severe weather.
- No, seriously, **THESE AREN'T FORECASTS, DON'T USE THEM AS SUCH!**

## 3. Tools We'll Be Using

We are going to be using two main datasets:

- **GEFS Reforecast Dataset**: Reforecasts are retrospective weather forecasts
that use a "frozen" numerical weather prediction (NWP) model. Using the same parameterizations and
data assimilation methods, this keeps the systematic biases from the NWP model the
same across the entire dataset. With a large enough number of these retrospective
forecasts, we have a better chance of sampling less common events.

- **SPC Tornado Reports**: The Storm Prediction Center (SPC) provides information about
severe weather events in the form of "storm reports." These reports mostly require
volunteer reports (especially in the case of severe hail and wind events), but
also come in the form of damage surveys after severe weather events.

The exploratory data analysis, feature engineering, and model development
will all be done in Python, with Jupyter notebooks provided in the associated
GitHub repo.
