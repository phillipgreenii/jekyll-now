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

# My first source (`interval`)

The power of callbags comes from composing very simple instances into a larger chain.  The problem I am going to work through is a count down timer.  My plan is to build the chain one callbag at a time starting from the beginning and working through to the end.  
 ```typescript
import {Factory as CallbagFactory } from 'callbag';

export const interval: CallbagFactory = (period) => (type, sink) => {
  if (type !== 0) return;

  let i = 0;
  setInterval(() => {
    sink(1, i++);
  }, period);

  sink(0, (t,d) => {})
}
```

`Factory` from `callbag` is how you create the initial source.  The previous code can be used as follows:

```typescript
const c: Callbag = interval(1000);

c(0, (type, payload) => {
  if(type === 1) {
    console.log(`It's the final cound down: ${payload}`);
  }
});
```
`interval()` returns a callbag.  The callbag needs initialized, which happens when it is called with the first parameter (type) set to `0`.  The second paramater is the talkback function.  Every period (1000ms in this instance), the interval callbag will call the talkback function with the `type` set to `1` and the `payload` set to an ever increasing value.  If you check the console, will see something similar to the following:

```
It's the final cound down: 0
It's the final cound down: 1
It's the final cound down: 2
It's the final cound down: 3
It's the final cound down: 4
...

It's the final cound down: 10
...
```
Yay, my first working callbag.  It isn't perfect, but it is a start.  There is two problems.  The most obvious is that is counts up and not down.  This problem will be resolved later by creating my first operator.  The more subtle problem, which needs to be resolved now, is how to stop the callbag.  As it is written now, it will continue calling the talkback forever.  `interval` needs updated to allow `setInterval()` to be cleared.

```typescript
import {Factory as CallbagFactory } from 'callbag';

export const interval: CallbagFactory = (period) => (type, sink) => {
  if (type !== 0) return;

  let i = 0;
  const id = setInterval(() => {
    sink(1, i++);
  }, period);

  sink(0, (type) => {
    // handle END type
    if (type === 2) {
      if(id) clearInterval(id);
    }
  })

}
```

Now the callbag from `interval()` supports cancelling.  This can be done as follows. 

```typescript
let canceller;
const c: Callbag = interval(1000);
c(0, (type, payload) => {
   // on init, grap the canceller to use later
   if (type === 0) {
     canceller = payload;
   } 
  if (type === 1) {
     console.log(`It's the final cound down: ${payload}`);
  }
});

setTimeout(() => {
  canceller(2);
}, 5000);
```

The talkback from the `interval` callbag will cancel if passed the `type` of `END` (`2`).  It is passed in when `type` is `0`, we just need to grab it (`canceller`).  The previous code will cancel the callbag after `5000ms` which results in the following:

```
It's the final cound down: 0
It's the final cound down: 1
It's the final cound down: 2
It's the final cound down: 3
It's the final cound down: 4
```

This is my first fully functioning callbag.  For a more official version of `interval`, please use [callbag-interval][callbag-interval].

# My first operator (`map`)

In the previous section, I pointed out that my count down timer actually counts up.  My choice to resolve this problem is by implementing an operator known as `map`.  An operator takes a callbag and returns a callbag.  When working with callbags, most of the time will be spent chaining operators together.  You will start with some code which generates some kind of payload which will be manipulated and transformed through various number of operations and finally end in some sink.  For this particular use case, I'm going to use the `map` operator to offset the values in such a way that it counts down.  The logic is `(i) => 4 - i`, which translates as follows

|`i`|`(i) => 4 - i`|
|---|--------------|
|0|4|
|1|3|
|2|2|
|3|1|
|4|0|

`map` can be implemented as follows

```typescript
import { Callbag, Operator as CallbagOperator } from 'callbag';

export const map: CallbagOperator = (mapFn: Function) => (callbag: Callbag) => (type, sink) => {
  if (type !== 0) return;
  
  callbag(0, (t, d) => {
    if(t === 1) {
      sink(t, mapFn(d));
    } else {
      sink(t, d);
    }
  });
}
```

The operator callback only responds to initialize (`0`), in which it initializes the upstream callback.  This initialization is bascially a pass through.  Calls from downstream to the operator will flow upstream.  As values come from upstream, the will be passed downstream, with the exception that when the `type` is 1, then `mapFn` is applied first.


Its talkback passes along all calls to the sink, which is the downstream callback.  If the type is `1`, then it will call the passed in function.  This is the mapping.

```typescript
let canceller;
// the chaining works inside out, so the values from interval will flow into map
const c: Callbag = map((i) => 4 - i)(interval(1000));
c(0, (type, payload) => {
   if (type === 0) {
     canceller = payload;
   } 
  if (type === 1) {
     console.log(`It's the final cound down: ${payload}`);
  }
});

setTimeout(() => {
  canceller(2);
}, 5000);
```

`map()` is called with the mapping function.  The result of this is an operator which accepts The result of `interval()` (which is a callback).  The composed callback is now stored in `c`.  If you were to compare this code with the previous example, only the creation of the callback changed.  The canceller is now the talkback from `map`, not `interval`, but because of the implementation of `map`, the cancel is passed through to `interval`.

That's it.  I have my first operator.  

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
* Callbags
  * [Interval][callbag-interval]

[callbag]: <https://github.com/callbag/callbag> "Callbag Specification"
[callbag-getting-started]: <https://github.com/callbag/callbag/blob/master/getting-started.md> "Getting Started: Creating your own utilities"
[compare-callbags-rxjs]: <https://egghead.io/articles/comparing-callbags-to-rxjs-for-reactive-programming> "Comparing Callbags to RxJS for Reactive Programming"
[callbag-interval]: <https://github.com/staltz/callbag-interval/>
