---
layout: post
title: DoDonBotchi Sitrep
---

## Overview

After [looking closer into the internals of DoDonPachi][1] and researching more
about the current state of deep learning, my plans for a DoDonPachi AI have
become much clearer and are, in large parts, already implemented. This post is
to document the current plans and state of the project.

Like all the cool kids are doing right now, DoDonBotchi will rely on an
artificial neural network that got trained to play DoDonPachi well. In that
area, the current state of the art seems to point fairly definitively towards
using [Google's TensorFlow][2] library, combined with [Keras][3], which is
mainly a wrapper around neural network libraries providing a much more
intuitive interface for them. In our case, we'd use Keras with the TensorFlow
backend, of course.

On the learning side, things get more fuzzy. Machine learning generally works
better when the underlying algorithm can learn from samples of data. In the
context of DoDonBotchi, this would mean showing the AI data of expert players
making it through DoDonPachi to have it learn from examples how to play the
game. The problem here is availability: There's no easily available data set of
DoDonPachi playthroughs to train an AI on. There are YouTube videos of World
Record runs, of course, but processing the various videos to be in a digestible
format for an AI seems infeasible for me, especially given the wildly varying
quality of those videos. On top of that, such training requires *massive*
amounts of data, which, I fear, is simply not available.

When it comes to learning from scratch, I've zero'd in on two methods:
[Reinforcement Learning][4] and [NeuroEvolution][5], both of which are able to
learn by basically trying random things and picking up what are good things to
do based on some feedback. In case of reinforcement learning, it's a reward for
each action performed and, in case of neuroevolution, it's a fitness function
that gets used to select best neural networks in a genetic algorithm evolving
them. I've decided to try the reinforcement learning approach first, mostly
based on gut feeling and there being an easy-to-use library called
[Keras-RL][6], which implements a lot of the techniques used in reinforcement
learning, ready to be used for Keras models -- work smart not hard. The
algorithm behind neuroevolution isn't too complicated, though, so once
reinforcement learning is set up and running, I might have a go at
neuroevolution, too.

## Implementation

DoDonBotchi is mainly implemented in Python, but will need a Lua-based
component to interface with [MAME][7]. As mentioned in the overview, the deep
learning will rely on Keras and the Keras-RL library offering reinforcement
learning mechanisms.

### DoDonPachi & OpenAI Gym

The [OpenAI Gym][8] is a collection of AI challenges used for benchmarking and
competition. It covers various problems, but, most importantly, also includes a
set of [Atari 2600 games][9], where the goal is for an AI to perform well.
Obviously analogous to what I'm doing, just focused on Atari games. Since
there are a lot of resources on how to train neural networks for OpenAI Gym
environments and one of our main libraries, Keras-RL, is even built around
using environments with an OpenAI-Gym-like API, I've decided that implementing
an OpenAI Gym environment to play DoDonPachi would be the best way to go. This
sounds more fancy than it actually is, in the end, because making an OpenAI Gym
environment mainly boils down to offering the following methods and fields:

* `action_space`: The space of possible actions to perform in the environment.
* `observation_space`: The space of possible things to happen in the
  environment.
* `reset()`: Reset the environment to an initial state
* `step(action)`: Perform an action in the environment and return the resulting
  observation
* `render()`: Draw a frame of the environment. Optional, as not every
  environment has visual output.

There are more details, but they are trivial. Above are the essential things
needed to make an OpenAI Gym environment. Within our context, this is achieved
as follows.

#### Action space

Our action space will consist only of controller inputs. The AI is not able to
do anything a human wouldn't be able to do with a controller at an actual
DoDonPachi cabinet. An action, then, is a summary of each input's current
state. The easiest way to do this would be a simple bit vector where each bit
indicates whether a button at the corresponding position is pressed or not. For
example, assuming an input order of `UDLR123S` -- representing up, down, left,
right, button 1, button 2, button 3, and start -- a bit vector `10011000` would
represent pressing up, right, and 1. However, in this system, it's possible to
press opposing directions at the same time. Up and down, for example, can both
be pressed, which is not possible on real hardware and thus shouldn't be
possible for our AI. Therefore, directional inputs along the same axis will be
combined into one, resuling in an input map of `VH123S`, where `V` and `H`
stand for vertical and horizontal movements respectively. Bits alone aren't
enough here, as there need to be at least three states. For the directional
inputs, there needs to be the option of no movement, negative movement (left,
down), and positive movement (up, right). An input vector would look more like
this: `22100`, again for pressing up, right, and 1.

Additionally, and this is a more domain-specific thing, the buttons 3 and start
do nothing for our AI; 3 simply does nothing and the AI is never in a state
where the start button has meaning. These inputs won't even be an option for
DoDonBotchi. Additionally, I made the decision to simply prevent the AI from
using any bombs whatsoever. More on the motivation behind this later, but this
also means that the 2 button isn't available to the AI, leaving only vertical
input, horizontal input, and whether or not 1 is pressed: `VH1`. The action
space is therefore every combination of `(0-2, 0-2, 0-1)`.

#### Observation space

As outlined in the previous post on [how far I got into reverse engineering
DoDonPachi][1], the data I was able to get at are the current lives, bombs,
score, and visible sprites. A possible observation simply aggregates all of
these into a single vector:

    (bombs, lives, score, sprite_1, sprite_2, ..., sprite_n)

Where each sprite consists of five values: `Sprite ID`, `X`, `Y`, `Width`,
`Height`. These sprites are written as five values directly in the vector, not
as additional sub-vectors. DoDonPachi's hardware is limited to `1024` sprites,
so our observations have a dimension of `1 + 1 + 1 + 1024 * 5 = 5123`. It's
worth noting that, of course, there aren't always `1024` sprites on screen. The
observation vector will be padded with `(0, 0, 0, 0, 0)` sprites to make up for
missing ones. Additionally, sprites will be sorted along their position to make
similar arrangements of them on screen result in actually similar observations.

#### reset()

Our environment is mainly executed in MAME. A reset happens at the start of
each episode to always start it off at the initial state. DoDonBotchi's
`reset()` function starts MAME with our DoDonBotchi plugin used to communicate
with MAME, the DoDonPachi rom to load, and an initial save state the AI always
starts from. The MAME plugin is written such that, after booting up, it sleeps
for a few frames and then waits for further instructions from the host process
by pausing the emulation.

#### step(action)

The step function expects a member of our action space and simply sends it to
MAME. Our plugin then decodes corresponding inputs and triggers them within the
emulation. All of this is done in lockstep: The DoDonBotchi MAME plugin always
pauses the emulation waiting for inputs, upon receiving them changes button
states accordingly, then continues the emulation for `n` frames, pauses it,
collects the current observation, sends it back to the host process, and waits
for more input. This makes the AI resistant to going out of sync due to
processing time. It could think for years before deciding on an input and the
emulation would remain consistent.

Besides performing an action, giving feedback about it is the most important
part of the step function. Its return values have to include:

* `observation`: The state observed after performing the action.
* `reward`: A number rewarding or punishing the AI depending on what the action
  achieved.
* `done`: Boolean indicating whether this episode has ended in this step.
* `aux`: An additional dictionary of information. Can be anything and isn't
  meaningful to the AI, but can be used for debugging.

These values are returned as a tuple and then used by our reinforcement
learning library to train the AI on when to perform which action. Currently,
the reward is calculated simply by how much the score has increased. An episode
ends when a life is lost.

#### render()

Normally, OpenAI Gym environments have an optional visualisation the user can
toggle by calling `render()` after each step. However, in our context this is
hard to achieve, as you can't just boot MAME up to render a single frame of the
emulation -- it's all or nothing. As such, `render()` does nothing in our
environment and whether or not to render video output is specified at the start
of the emulation.


### EXY

![EXY]({{ site.baseurl }}/public/img/EXY.png){:.center-image}

The EXY module handles creation and training of the neural networks used to
play DoDonPachi. I'm far from an expert on DoDonPachi lore, but EXY, as far as
I know, is one of the element dolls from future games that basically evolves
into a computer virus that travels back to the past to try and prevent
DoDonPachi from happening, which includes defeating Colonel Longhena piloting
Hibachi. Therefore, I named the module centred around the actual neural network
training after her.

Implementation of this module is fairly straightforward, as it mainly relies on
setting up a Keras model, which is well documented throughout the internet, and
hooking it up to a Keras-RL agent for reinforcement learning. As the Keras-RL
project isn't fully implemented yet, its documentation is a bit lacking, but
there are examples on how each of the agents can be used in [Keras-RL's Github
repository.][10] What's more important to note is that EXY's setup and training
code needed to be easy to change. This is my first deep learning project and
I'm likely to get things wrong, so reworking the code to address issues as they
pop up is crucial. I won't go into details how, but EXY is written to easily
switch towards another neural network model or reinforcement learning agents,
policies, and memories.

EXY also hooks into Keras-RL's training process to keep track of performance.
This is mainly done to keep a leaderboard of best trials so far which get
written out to disk as that data comes in. This makes it easy to track how well
the neural net is doing just by looking at a file.

#### Captures

Besides tracking leaderboards, EXY is written to record every single run of
DoDonPachi the learning procedure performs. This is done through [MAME's input
recording][11] feature at its heart, which makes it possible to dump every
input made while playing a game to a file, which can then be played back at
will. During playback one can then also render out a video of the emulation as
an `.avi` file. Of course, this would be possible without prior input
recordings, too, but MAME's input recordings a very small files, whereas the
rendered videos tend to be huge. Keeping an input recording of each run doesn't
take up a lot of space. I can then use EXY's leaderboard data to look for the
best runs and render out videos for them. I already wrote code to automatically
render out the `n` best runs first as `.avi` files, which are then re-encoded
as much smaller `.mp4` files. Here's a video of the very first run:

<iframe
    width="560"
    height="315"
    src="https://www.youtube.com/embed/7NAcBQQI0aM"
    frameborder="0"
    allow="autoplay; encrypted-media"
    allowfullscreen>
</iframe>
*'s goin' nuts*

Since then, I also wrote a little plugin to visualise inputs during recordings,
you can see that in the evaluation section below.

At this point, I want to give a shoutout to the MAME developers, because it has
worked flawlessly with these fairly unorthodox things I'm doing with it. I was
afraid the input recording system would neglect inputs I trigger from Lua. This
wasn't the case. I was afraid running the emulation at faster than real time
would break input recordings played back in real time. Also didn't happen. It
would've been plausible for the input recordings to not work when started from
a certain save state, but they did. I also expected running the emulation with
no video or audio output could also break input recordings. Nope, they were
fine. Even more surprising, you can render out `.avi` videos just using the
command line and not having a MAME window with video/audio. There are probably
more things like this, but overall I'm saying that, despite lacking
documentation in many areas, MAME is a really robust piece of software. Kudos
:)

## Evaluation

Currently, EXY uses a simple [DQNAgent][12] set up like the Atari 2600 example
from [Keras-RL's example section][10]. After roughly eight hours of training,
it didn't perform much better than that very first, purely random, run you see
above:

<iframe
    width="560"
    height="315"
    src="https://www.youtube.com/embed/urdYtQbJu10"
    frameborder="0"
    allow="autoplay; encrypted-media"
    allowfullscreen>
</iframe>

As I wrote above, I've decided to disable bombs entirely for the AI. You can
see why, now. I realise eight hours of training isn't very much, but I would
expect some improvement, at least. What actually seems to happen is that the AI
spams bombs and some times gets lucky enough to get past the first mini boss
using that. Limiting the AI to only shooting and not the cheap screen clears
bombs provides will force it to actually learn the game. DoDonPachi also
rewards having surplus bombs with higher score modifiers, so you're encouraged
to not use bombs, anyway. Additionally, as you can see in that video, the AI
also dies thrice, until it actually game overs. This also lets the AI do too
many mistakes, preventing it from learning proper play. I've since changed the
criterion for ending an episode to be a single death.

Overall, I also feel like the reward function used in our environment is too
simple. While our overall goal is to reach a high score, it intuitively makes
sense to base rewards around that. However, with our possible actions, it
becomes hard to clearly map input actions to score rewards, since usually you
only get points for killing an enemy a short while after actually pressing the
buttons, and there are components of the score system that don't even
correspond to certain inputs, like the chaining system. These aren't reflected
in this simple metric and the disconnect between actions and rewards make it
hard for reinforcement learning to "connect the dots."

## Future work

I have some ideas on how to address these problems. Simple ones, like
preventing the AI from using bombs at all, are already mentioned above. The
more complicated one is rewriting  the reward function. My current idea is to
instead focus the reward on properties that help the AI *in the moment*. For
example, to encourage the AI to doge bullets, the reward could factor in the
current distance to the closest bullet. To reward actually shooting at enemies,
the reward could factor in the amount of our ship's shots currently headed in
the direction of an enemy. To reward chaining, the reward should include the
current chain level. And so on. Coming up with these ideas for rewards is the
easy part, however. What's harder is getting the information to actually
calculate them. With my current observations, it's not obvious how to determine
the distance to the nearest bullet, for example, because I only have sprite
data available to me. It's not even obvious which of the sprites is your own
ship. To make any more sophisticated reward function possible, I need to get
more information about the current game state. How to achieve that, I'm not
sure, but I'll write about it on here as soon as I figure it out.

[1]: {% post_url 2018-05-01-dodonpachi-reverse-engineering %}
[2]: https://www.tensorflow.org/
[3]: https://keras.io/
[4]: https://en.wikipedia.org/wiki/Reinforcement_learning
[5]: https://en.wikipedia.org/wiki/Neuroevolution
[6]: https://github.com/keras-rl/keras-rl
[7]: http://mamedev.org/
[8]: https://gym.openai.com/
[9]: https://gym.openai.com/envs/#atari
[10]: https://github.com/keras-rl/keras-rl/tree/master/examples
[11]: https://90sarcadegames.blogspot.de/2013/06/tutorial-recording-videos-of-games.html
[12]: https://keras-rl.readthedocs.io/en/latest/agents/dqn/
