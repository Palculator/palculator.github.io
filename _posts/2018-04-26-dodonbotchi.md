---
layout: post
title: DoDonBotchi
categories: sitrep dodonbotchi
date: 2018-04-26 21
---

The project I'm currently working on will be an experiment in writing *some
algorithm* to go for highscores in [DoDonPachi][1]. I emphasised "some 
algorithm" there to highlight that I'm not sure what it'll actually end up
being. It might be something that's just brute force, something smarter than
that, or something closer to AI. I'll document approaches in here as I evaluate
them.

My current goal is to set up a genetic algorithm that tries to evolve input
sequences using the score as a fitness function. It would produce a random
sequence of inputs, launch MAME & play through the input sequence, analyse what
happened to evaluate a sequence's fitness. Like any genetic algorithm, it would
be done with a population of `n` input sequences, the fittest of which would
then crossed over and mutated to create a next generation of input sequences
to test. After a (hopefully) finite amount of time, it would converge to inputs
that achieve high scores.

The main reason I'm starting with this is simple: expertise. The work is
similar to what I'm doing in my master thesis right now and I know how to get
it going fairly quickly. I could then buy a Raspberry Pi as a tiny compute
server to keep evolving better inputs.

Execution time is the biggest problem in this method. MAME can be launched such
that it runs at whatever speed the CPU can handle, but even if a run takes only
ten seconds, it will still accumulate to take ages. An idea to cut them down I
have had is to train a machine learning model of which input sequence performed
how well and, once it has achieved a certain accuracy, use it to predict which
inputs are too garbage to even attempt. It might help -- I'll find out -- but
it's new terrain for me as I'm fairly unexperienced in machine learning. I know
the gist, but, for example, I'm not even sure how I'd apply it to a data set
where each observation has variable length, like an input sequence. We'll see!

Once this is set up and running, I can keep it going on my Pi while I look into
other things that require more work from *me*. I'll at least start reverse
engineering DoDonPachi to the point of having structured information about the
game state, from simple measurements like score and lives to current enemy data
with their locations relative to the ship, and so on. That will be a hassle,
but it will be interesting to dissect an arcade game like this. As far as I'm
aware, it's also mostly unknown information, so documenting the internals of
DoDonPachi like this has some informational value, too.

I won't go on for too long as there are enough things to do before even
thinking further. What I want to finish with is a list of general requirements
I have about the project:

 * 2-ALL the game.
 * Get competitively high scores.
 * Produce valid MAME input files that can be played back on official
   builds/roms.
 * Have a way to visualise the progress of either AI or whatever method is
   being used.
 * Not tinker with the game in any way whatsoever; all interactions must be
   done through simulated controller inputs.

[1]: https://en.wikipedia.org/wiki/DoDonPachi
