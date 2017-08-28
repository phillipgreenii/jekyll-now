---
layout: post
title: Thinking of computers in terms of Rube Goldberg machines
date: '2015-04-08T23:47:00.001-05:00'
author: Phillip Green II
tags:
- computers
- rube goldberg machine
modified_time: '2015-04-08T23:47:00.001-05:00'
---

How do computers work?  They are machines and work similar to any other machine.  Start with some phenomenon of physics that works in a particular way and build a small component utilizing concept.  Next, conceptually higher components are built with the lower conceptual components.  This process continues until the machine is capable of performing the desired task.

Computers work from how electrons flow through semiconductors.  Instead of starting there, think about a [Rube Goldberg machine][wiki-rube-goldberg-machine].  Any machine could work, but Rube Goldberg machines are fun and very easy to visualize.  Start with one that can be elaborate as desired, but has two properties.  First, the Rube Goldberg machine begins with a rolling ball.  Lastly, the final step is to produce a distinct sound; perhaps a low note played with a tuba.  Remember, the details of how the rolling ball triggers the sounds is arbitrary.  It can be simple or complicated.

{insert image of rgm}

Most Rube Goldberg machines are built in a certain way that once they run, they are done.  They need lots of time and patience to reset them to fire again.  That won't work for out implementation; a third property is added to the Rube Goldberg machine. Not only will it start with a rolling ball and end with a noise, it must be able to reset itself.  How this happens is not important.  The idea is that once the final step is complete and the noise happens, a ball could be placed at the start and the machine will work over and over again.

A computer does more than one thing.  To model this, there needs to be multiple Rube Goldberg machines.  For this implementation, the assumption is there will be four.  Each machine will have the same three properties: They will all begin with a rolling ball, end with a distinct noise, and reset themselves.  The difference between the four Rube Goldberg machines is the final noise and the middle bit.  One Rube Golberg machine may primarily be built with lincoln logs, while the next using legos, wood, or paper.  As stated before, how the rolling ball ultimately causes the noise to happen doesn't matter.  

{insert image of rgms}

To connect the four Rube Goldberg machines together we need another machine.  It could also be a Rube Goldberg machine, but it will have marginally different properties.  It will begin with a rolling ball and will be resetable, but it will make no noise.  Instead, if a small ball rolls into it, the ball will be redirected to the first of the noise making Rube Goldberg machines.  If a larger ball is rolled, then it will be redirected to the second of the noise making Rube Goldberg machines.  This pattern continues for the two remaining noise making Rube Goldberg machines and increasingly larger balls.  When everything is built, 4 different sized balls could be rolled into the first machine and it will trigger one of the four noise making Rube Goldberg machines.  


{insert image of rgms with redirector}

In order for the next part to work, we need to add a fourth property to the noise making Rube Goldberg machines.  This property states that the noise making Rube Goldberg machines must complete within ten seconds.  For all four, the time from the ball rolling, to the noise being generated, and the reset must be less than ten seconds.

Why does the timing matter?  The last machine being added will contain a timer.  That timer will fire every ten seconds.  This last device is composed of two parts.  The first is a large tube that will hold balls of varying sizes in such a way that the order is maintained.  The second part is a release mechanism that fires every ten seconds.  In short, every ten seconds a ball (from the tube) will be released.


{insert image of rgms with redirector and timer}

With everything attached together, the whole machine works as follows.  A person decided which balls and in which order will be added to the long tube of the first machine.  From that point on, every ten seconds, one of the balls will be released.  It will roll into the next machine which redirects the ball to one of the noise making Rube Goldberg machines and finally, noise will be heard.  Different initial ball orders will generate different songs.

How does this relate back to a computer?  The tube defines a sequence of balls, just as in a computer a program is a sequence of operation codes (opcodes). Each ball (opcode) triggers a specific noise making Rube Goldberg machine (operation). The triggering happens in the Rube Goldberg machine with the machine that does the rolling ball redirection, but in a computer the circuit is called a control unit.  The same principals apply, instead of enabling machines, the control unit enables circuits.  In the Rube goldberg implementation there was a timer that controls the start of everything, in a computer it is called a clock.  They both do the same thing by keeping everything in sync. Lastly, the different operations on a computer are analogous to the noise making Rube Goldberg machines. In the Rube Goldberg machines, the operations available were four different noises.  In typical a computer, these operations could be "add", "rotate", or "clear".  Just as in the Rube Goldberg implementation, the complexity of how of different operations work will vary widely.   Typically, "add"ing is far more complicated than "clear"ing.  The associations just described are summerized in the following table:

|Rube Goldberg implementation|Computer|
|----------------------------|--------|
|Rolling balls|Operation Codes (opcodes)|
|Tube of balls|Program|
|Timer|Clock|
|Rolling Ball Redirection machine|control unit|
|Noise Making Rube Goldberg machine|operation|

{add diagram of rgm components and computer components}

## References
* [Rube Goldberg machine][wiki-rube-goldberg-machine]
* [Central Processing Unit (CPU)][wiki-central-processing-unit]

[wiki-rube-goldberg-machine]: <https://en.wikipedia.org/wiki/Rube_Goldberg_machine> "Wiki: Rube Goldberg machine"

[wiki-central-processing-unit]: <https://en.wikipedia.org/wiki/Central_processing_unit> "Wiki: Central Processing Unit (CPU)"
