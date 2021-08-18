+++
author = "Meyer"
categories = ["thoughts", "philosophy"]
date = 2020-08-22T23:47:54Z
description = ""
draft = false
image = "https://images.unsplash.com/photo-1462331940025-496dfbfc7564?ixlib=rb-1.2.1&q=80&fm=jpg&crop=entropy&cs=tinysrgb&w=2000&fit=max&ixid=eyJhcHBfaWQiOjExNzczfQ"
slug = "randomness"
summary = "The ultimate test of humanity will be our ability to accept the randomness of our universe."
tags = ["thoughts", "philosophy"]
title = "Randomness and the human condition"
math = true

+++


This is a (slightly edited) version of my favorite college supplemental essay which I wrote in response to one of the Yale prompts:

> What is a random number?
> 
> It would seem like a question with an obvious, striking answer: a random number is a number that cannot be predicted. But clearly, any particular number cannot be random; the number 42 immediately follows the number 41 and proceeds 43. Give me a tape measure and I can find 42 inches, a calculator and I can multiply six by seven. Thus, randomness is not an attribute inherent to numbers, yet somehow there are still “random” numbers, implying a degree of uncertainty which differentiates them from other numbers.
>
> This conundrum puzzled me because it strikes at the core of cryptography and computer science. Computers are supposed to be the ultimate calculators, capable of finding answers to the most difficult math problems but unable to spontaneously generate information. Pseudo-random numbers are a critical part of protecting passwords in a database, but they are still not random. Indeed, a calculated number cannot be random because by its nature it is derived from other numbers, which themselves are not random. Instead, we find clever ways to manipulate numbers to give them an appearance of randomness; we find algorithms whose outputs are nearly impossible to correlate with their inputs.
>
> Yet somehow, the universe is capable of generating randomness ad nauseam; from quantum particles to radioactive decay, there is no shortage of unpredictability in nature. I am fascinated by how we might incorporate our knowledge of the natural world to create computer programs that more accurately represent the world as it is: random.

The problem of randomness underpins so many different aspects of our daily lives, yet we rarely stop to consider the broader implications of living in a random world. In fact, I would contend that we have constructed entire philosophies and religions to justify our existence as something more than the random product of natural selection on a planet that by unfathomable luck is habitable to organic life.

The notion of randomness is so difficult to accept because it runs counter to our everyday experiences. A few weeks ago, I was playing a game (via Zoom, of course) wherein one player is randomly selected each round to be the "card tzar." After four rounds, only two people had been selected. Always a sucker for a good statistics problem, I quipped, "we should start writing down who's been chosen so we can tell who is most likely to be next." There is a problem with my logic that was quickly pointed out to me by my much smarter classmates on the Zoom call: if the selection process is taken to be random (which, for all practical purposes, it is), then the outcome of past selections has no bearing on the outcome of future selections.

I sought to ascribe order where there is none because it seemed unlikely that those two players would each be chosen twice before anyone else had been selected once. In fact, we can calculate the probability of such an event using the [binomial distribution](https://en.wikipedia.org/wiki/Binomial_distribution). Assume there were five people on the call (persons A-E) and define a "successful" sample to be one where person A or B gets chosen. What I witnessed was four consecutive successful draws.

$$ P(X=4)={4 \choose 4} * \left(\frac{2}{5}\right)^4 * \left(1-\frac{2}{5}\right)^0 = 2.56 \\% $$

There was just a 2.56% chance of the observed selections! That's so... _unlikely_. The logical fallacy to which I fall victim on a daily basis is called the **gambler's fallacy**:

> "The gambler's fallacy... is the erroneous belief that if a particular event occurs more frequently than normal during the past it is less likely to happen in the future (or vice versa), when it has otherwise been established that the probability of such events does not depend on what has happened in the past."
> ([Wikipedia](https://en.wikipedia.org/wiki/Gambler%27s_fallacy))

Which brings me back to my original discussion regarding our understanding of human existence. We have evolved over millions of years to efficiently capture the patterns in our environment; that is essentially the entire function of the brain. When our ancestors looked to the heavens, they saw the same stars move across the night sky on a regular basis. From that recurring observation, they formulated different explanations for the seemingly haphazard spattering of dots littering the night sky. We overlaid pictures of animals with the assumption that their existence on Earth necessarily implies some transcendental significance to their figure. They were seen as divinely ordained because of the sheer unlikelihood of their existence otherwise, which means we should expect to see their same form repeated even at the scale of galaxies; there _must be_ something inherently important about a fish that would land them on Earth, and some stars roughly resemble a fish... _Pisces_!

We simply cannot comprehend the incredible odds that were overcome to facilitate our existence, so we fall victim to the gambler's fallacy: _because we exist and the odds of us existing are so low, there must have been a higher being that created us_. We forget that the observation of an event (our existence) does not change the random nature of the process that yielded such an event.

As I ponder the existential nature of randomness, it is hard to avoid the sense of nihilism that creeps into any philosophical consideration, taken to its logical conclusion. If our existence is the result of a random process, does that make it meaningless? Not necessarily. "Meaning" is a construct of our minds meant to serve an evolutionary purpose: to keep us alive. Without meaning, there would be no point in living. Without living, our species would no longer be the fittest, hence we would not meet Darwin's criterion of being well-adapted to the environment. We have evolved neurochemical pathways to facilitate gratification, happiness, and satisfaction; and why shouldn't we? Do we really need a specific _reason_ for our existence, or can it suffice to say that we _do_ exist, and live with the humility to know that our continued existence is not guaranteed?

When confronted with the immense challenges of the twenty-first century, from climate change to global pandemics, we cannot take the preservation of our species for granted simply because we were lucky enough to exist in the first place. Our current ideology, guided by a statistical and logical fallacy, is like playing a game of Russian roulette with the idea that because we have not lost yet, we will never lose. The ultimate test of our species—the human experiment with intelligence—will be our ability to accept the randomness of our universe.

