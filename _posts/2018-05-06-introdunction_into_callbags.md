---
layout: post
title: Introduction to Callbags
date: '2018-05-06T18:32:00.001-05:00'
author: Phillip Green II
tags:
- javascript
- callbag
modified_time: '2018-05-06T18:32:00.001-05:00'
---

# Origin

Recently, I saw the following tweet:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Not too long ago, I reached out to Andre about an idea I had to rearchitect RxJS around ideas inspired by Callbags. There is a public experimental branch any RxJS repository, and 9 or 10 page design document that will be public shortly. <a href="https://t.co/A8fCr1STok">https://t.co/A8fCr1STok</a></p>&mdash; Ben Lesh 🛋️👑🔥 (@BenLesh) <a href="https://twitter.com/BenLesh/status/987234002254745600?ref_src=twsrc%5Etfw">April 20, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

I had not heard of [callbags][callbag], so I clicked through to see what they are about.  This post is my process through re-implementing various parts of the callbag specification so that I have a better understanding of what they are and how they work.

# Research

First, I started with the post referenced in the tweet: [Comparing Callbags to RxJS for Reactive Programming][compare-callbags-rxjs].  This was a great introduction for me as I have previous experience with RxJS.  If you haven't used RxJS, I would still recommend it because it reads easier than the [full specification][callbag], which is what I read next.  What jumps out at me as first is the smallness of the specification.  There is a simple `README.md` and `types.d.ts` for support with Typescript.  The rest of the files are overhead, except [getting-started.md][callbag-getting-started].  Be sure to read `getting-started.md`, because I missed it at first and it was a great help. 

At this point, I felt I had become familiar enough to attempt writing some code.

# My first source (`interval`)

The power of callbags comes from composing very simple instances into a larger chain.  The problem I am going to work through is a countdown timer.  My plan is to build the chain one callbag at a time starting from the beginning and working through to the end.  
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
    console.log(`It's the final countdown: ${payload}`);
  }
});
```
`interval()` returns a callbag.  The callbag needs initialized, which happens when it is called with the first parameter (type) set to `0`.  The second parameter is the talkback function.  Every period (1000ms in this instance), the interval callbag will call the talkback function with the `type` set to `1` and the `payload` set to an ever increasing value.  If you check the console, will see something similar to the following:

```
It's the final countdown: 0
It's the final countdown: 1
It's the final countdown: 2
It's the final countdown: 3
It's the final countdown: 4
...

It's the final countdown: 10
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
     console.log(`It's the final countdown: ${payload}`);
  }
});

setTimeout(() => {
  canceller(2);
}, 5000);
```

The talkback from the `interval` callbag will cancel if passed the `type` of `END` (`2`).  The talkback is passed when the callbag is initialized, we just need to grab it (`canceller`).  The previous code will cancel the callbag after `5000ms` which results in the following:

```
It's the final countdown: 0
It's the final countdown: 1
It's the final countdown: 2
It's the final countdown: 3
It's the final countdown: 4
```

This is my first fully functioning callbag.  For a more official version of `interval`, please use [callbag-interval][callbag-interval].

# My first operator (`map`)

In the previous section, I pointed out that my countdown timer actually counts up.  My choice to resolve this problem is by using an `operator` known as `map`.  An `operator` takes a callbag and returns a callbag.  When working with callbags, most of the time will be spent chaining operators together.  For this particular use case, I'm going to use the `map` operator to apply `(i) => 4 - i` to offset the values in such a way that it counts down. 

|`i`|`(i) => 4 - i`|
|---|--------------|
|0|4|
|1|3|
|2|2|
|3|1|
|4|0|

`map` is an operator that will apply the specified function to each payload as it flows through the operator.  It can be implemented as follows

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

The operator callback only responds to initialize (`0`), in which it initializes the upstream callback.  This initialization is basically a pass through.  Calls from downstream to the operator will flow upstream.  As values come from upstream, the will be passed downstream, with the exception that when the `type` is 1, then `mapFn` is applied first.

```typescript
let canceller;
// the chaining works inside out, so the values from interval will flow into map
const c: Callbag = map((i) => 4 - i)(interval(1000));
c(0, (type, payload) => {
   if (type === 0) {
     canceller = payload;
   } 
  if (type === 1) {
     console.log(`It's the final countdown: ${payload}`);
  }
});

setTimeout(() => {
  canceller(2);
}, 5000);
```

`map()` is called with the mapping function.  The result of this chain is an operator which accepts the result of `interval()` (which is a callbag).  The composed callback is now stored in `c`.  If you were to compare this code with the previous example, only the creation of the callback changed.  The canceller is now the talkback from `map`, not `interval`, but because of the implementation of `map`, the cancel is passed through to `interval`.

That's it.  I have my first operator.  For a more official version of `map`, please use [callbag-map][callbag-map].

# My second operator (`take`)

`map` from the previous section fixed the data which was emitted, but the code to create and then cancel the callbags is very difficult to understand.  The `map` operator did a great job of encapsulating functionality, which is the opposite of what is happening with `canceller`.  Can the cancelling logic be encapsulated into an operator?  Yes, I could take the timer and put it into an operator which cancels the upstream callbag after a fixed period of time.  However, that doesn't feel like the correct solution for this situation.  I did that because I was still learning.  In truth, I don't like dealing with multiple uses of `setInterval` because they could become out of sync.  I think an easier solution would be to count how many messages come from interval and then stop.  This is what the `take` operator does:

```typescript
import { Callbag, Operator as CallbagOperator } from 'callbag';

export const take: CallbagOperator = (n: number) => (source: Callbag) => (type, sink) => {
  if (type !== 0) return;

  let count = 0;
  let sourceTalkBack;
  function protectedTalkBack(t, d) {
    if (count < n) sourceTalkBack(t, d);
  }
  source(0, (t, d) => {

    if (t === 0) {
      // on initialize, remember the original talkback
      sourceTalkBack = d;
      // but register with the protectedTalkBack
      sink(t, protectedTalkBack);
    } else if (t === 1) {
      // only process the data if the count hasn't be achieved
      if (count < n) {
        sink(t, d);
        count++;
        // once count has been achieved, end both upstream and downstream
        if (count === n) {
          sink(2);
          sourceTalkBack(2);
        }
      }
    } else {
      sink(t, d);
    }
  });
}
```

`take`'s implementation is more complicated, but we can step through it. It initializes by passing through the downstream talkback, however, it wraps it first.  If the count has been achieved, then it doesn't pass along the messages.  As `DATA` (`1`) messages come in, the count is checked and if the max hasn't happened, it passes the message through.  It will also increment the counter and check if the max count (`n`) has happened.  When the max has occurred, then the `END` (`2`) message is sent to both upstream and downstream callbags.

Adding in `take` simplifies the code:

```typescript
const c: Callbag = map((i) => 4 - i)(take(5)(interval(1000)));
c(0, (type, payload) => { 
  if (type === 1) {
     console.log(`It's the final countdown: ${payload}`);
  }
});
```

`interval` sends messages to `take`.  They pass through until the limit occurs, then it cancels everything.  Until the cancellation happens, the data is mapped into the values to be displayed.

For a more official version of `take`, please use [callbag-take][callbag-take].

# My first sink (`forEach`)

The innards of `callbag`s shouldn't be exposed outside of implementation of sources, operators, and sinks.  Right now my main code is very callbag specific.  To correct this, I'll create a sink called `forEach`  which calls a function for each data message.

```typescript
import { Callbag } from 'callbag';

export const forEach = (fn: Function) => (source: Callbag) => {
  let talkBack;
  source(0, (t, d) => {
    if (t === 1) fn(d);
  });
}
```

There isn't much here, which is good. If the message is for data, then it calls the function with the data.  By having, `forEach`, the usage code becomes trivial:

```typescript
forEach((data) => console.log(`It's the final countdown: ${data}`))(map((i) => 4 - i)(take(5)(interval(1000))));
```

For a more official version of `forEach`, please use [callbag-for-each][callbag-for-each].

## Caveat

I wanted to point this out for completeness.  If you look at the actual implementation of `forEach`, it looks like the following

```typescript
import { Callbag } from 'callbag';

export const forEach = (fn: Function) => (source: Callbag) => {
  let talkBack;
  source(0, (t, d) => {
    if (t === 0) talkBack = d;
    if (t === 1) fn(d);
    if (t === 1 || t === 0) talkBack(1);
  });
}
```
The extra code is to support pulling data.  Callbags are either listenable or pullable.  Pullable callbags will only send the next data message when told to.  The extra code will send those pull messages after each received message.  For upstream callbags which are listenable, they will simply ignore the pull request.


# Putting it all together

The code works, however it is difficult to read with the compositions.  Everything is on a single line and it reads right-to-left.  The fix for this is to use a helper function called `pipe`:

```typescript
export function pipe(...operators) {
  let r = operators[0];

  for (let i = 1; i < operators.length; i++) {
    r = operators[i](r);
  }

  return r;
}
```

`pipe` takes a variable number of arguments which consists of a source, a various number of operators, and finally a sink.  It allows my code to be written as follows:

```typescript
pipe(
  interval(1000),
  take(5),
  map((i) => 4 - i),
  forEach(data => console.log(`It's the final countdown: ${data}`))
)
```

For a more official version of `pipe`, please use [callbag-pipe][callbag-pipe].


# Summary

Overall I really like callbags.  I think it is a great concept and I look forward to digging into more in the future.  If you also enjoy them, checkout [Callbag Basics][callbag-basics] and please thank [@andrestaltz].

All of the code in this post can be viewed at [Stackblitz][stackblitz-introduction-into-callbags].

# References
* [Callbag][callbag]
* [Getting Started: Creating your own utilities][callbag-getting-started]
* [Comparing Callbags to RxJS for Reactive Programming][compare-callbags-rxjs]
* Callbags
  * [interval][callbag-interval]
  * [map][callbag-map]
  * [take][callbag-take]
  * [forEach][callbag-for-each]
  * [pipe][callbag-pipe]
* [Callbag Basics][callbag-basics]
* [Code available on Stackblitz][stackblitz-introduction-into-callbags]
* [Inspiration][final-countdown]

[callbag]: <https://github.com/callbag/callbag> "Callbag Specification"
[@andrestaltz]: <https://twitter.com/andrestaltz> "Author of Callbag Specification"
[callbag-getting-started]: <https://github.com/callbag/callbag/blob/master/getting-started.md> "Getting Started: Creating your own utilities"
[compare-callbags-rxjs]: <https://egghead.io/articles/comparing-callbags-to-rxjs-for-reactive-programming> "Comparing Callbags to RxJS for Reactive Programming"
[callbag-interval]: <https://github.com/staltz/callbag-interval/>
[callbag-map]: <https://github.com/staltz/callbag-map/>
[callbag-take]: <https://github.com/staltz/callbag-take/>
[callbag-for-each]: <https://github.com/staltz/callbag-for-each/>
[callbag-pipe]: <https://github.com/staltz/callbag-pipe/>
[callbag-basics]: <https://github.com/staltz/callbag-basics> "Basic callbag factories and operators to get started with"
[stackblitz-introduction-into-callbags]: <https://stackblitz.com/edit/phillipgreenii-callbags?file=index.ts> "Code from Introduction to Callbags on Stackblitz"
[final-countdown]: <https://www.youtube.com/watch?v=9jK-NcRmVcw> "Europe - The Final Countdown (Official Video"

