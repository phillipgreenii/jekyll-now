---
layout: post
title: Introduction to Callbag
date: '2018-04-28T11:55:00.001-05:00'
author: Phillip Green II
tags:
- javascript
- callbag
modified_time: '2018-04-28T11:55:00.001-05:00'
---

# Origin

Recently, I saw the following tweet:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Not too long ago, I reached out to Andre about an idea I had to rearchitect RxJS around ideas inspired by Callbags. There is a public experimental branch any RxJS repository, and 9 or 10 page design document that will be public shortly. <a href="https://t.co/A8fCr1STok">https://t.co/A8fCr1STok</a></p>&mdash; Ben Lesh 🛋️👑🔥 (@BenLesh) <a href="https://twitter.com/BenLesh/status/987234002254745600?ref_src=twsrc%5Etfw">April 20, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

I had not heard of [callbags][callbag], so I clicked through to see what they are about.  This post is my process through re-implementing various parts of the callbag specification so that I have a better understanding of what they are and how they work.

# Research

First, I started with the post referenced in the tweet: [Comparing Callbags to RxJS for Reactive Programming][compare-callbags-rxjs].  This was a great introduction for me as I have previous experience with RxJS.  If you haven't used RxJS, I would still recomend it because it reads easier than the [full specification][callbag], which is what I read next.  What jumps out at me as first is the smallness of the specification.  There is a simple `README.md` and `types.d.ts` for support with Typescript.  The rest of the files are overhead, except [getting-started.md][callbag-getting-started].  Be sure to read `getting-started.md`, because I missed it at first and it was a great help. 

At this point, I felt I had become familiar enough to attempt writing some code.


# My first source
__TODO__

# My first operator
__TODO__

# My first chain
__TODO__

# My first sink
__TODO__

# Putting it all together
__TODO__

## References
* [Callbag][callbag]
* [Getting Started: Creating your own utilities][callbag-getting-started]
* [Comparing Callbags to RxJS for Reactive Programming][compare-callbags-rxjs]

[callbag]: <https://github.com/callbag/callbag> "Callbag Specification"
[callbag-getting-started]: <https://github.com/callbag/callbag/blob/master/getting-started.md> "Getting Started: Creating your own utilities"
[compare-callbags-rxjs]: <https://egghead.io/articles/comparing-callbags-to-rxjs-for-reactive-programming> "Comparing Callbags to RxJS for Reactive Programming"
