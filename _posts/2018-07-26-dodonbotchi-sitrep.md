---
layout: post
title: Current DoDonBotchi Sitrep
---

It's been a while since I've had the chance to write up current progress, mostly due to only having limited time to even work on DoDonBotchi and even less to write about it. But here goes!

## Scrapping Neural Nets

As mentioned in the last post, I was at a point where I got structured data regarding current enemies on screen, bullets being fired, the position of the ship, and so on. I've been using this information to construct an artificial frame that simplifies the current game state to its most essential components. See the clip below for an idea of how this worked:

<video class="center-image" src="{{ site.baseurl }}/public/img/artificial_state_frame.mp4" autoplay loop></video>

Relevant entities all get rendered into a frame like this removing most of the noise raw frames would have. I had hoped this would help accelerate learning. However, for whatever reason, reinforcement learning with the [Keras-RL][1] library failed to produce noteworthy results. After almost a week of training, which included more than 30,000 episodes, it could not even beat the mid-boss of the first stage. It was crap. I was using the same model [Mnih et al., 2015][2] used for Atari reinforcement learning. In their research, it worked beautifully. The artificial frames I composed looked fairly similar to Atari games to begin with, but, to make it even more like one, I even changed the rendering to produce different shapes based on entity type:

<video class="center-image" src="{{ site.baseurl }}/public/img/ddonpach_atari.webm" autoplay loop></video>

Because of the similarity of input, I had hoped to get similar results to the Atari paper, but this wasn't the case. I'm sure it messed something up and I could've continued tuning the model to work, but I got tired after a while and decided to move on from the idea of neural nets entirely.

## Custom AI

Instead, the next idea was to simply write a custom AI specific to DoDonPachi. Thanks to the work with Neural Nets, I already had code in place where the game pings out its current state to my Python host process and waits for inputs to press. I just needed to figure out a way to determine those inputs.

This turned out to be hard.

Initially, the idea was to just have the ship always shoot and move away from bullets. Maybe the AI having perfect knowledge and super-human reaction time would be enough. What ended up happing, however, was that whenever a bullet approached the ship, it would move away in the same direction as the bullet as any other direction would've moved it closer to it. This went on until it reached the border and either was lucky enough to be out of the way or not. I've then expanded this approach to infer bullet trajectories and project their future position. Instead of moving away from the bullet, it would move away from the bullet *and where the bullet will be*. This still didn't work out, as now it suffered from the same issue as before, just moving away a bit further. I then tried using the trajectories not to predict position, but having the ship move completely away from any bullet heading its direction by calculating the line from bullet to the edge of the screen and having the ship maximise distance to each of these lines. This worked somewhat well, but broke as soon as there were many bullets on screen which made it too hard to maximise distance to all edge trajectories. Unfortunately, I didn't keep any media of this method, so this text will have to suffice.

After concluding this method a failure, I switch to calculating a heat map of the environment where temperature corresponds to danger. The ship picked the coolest point close to some fixed point (usually in the centre of the lower quarter) and always tried to stay there. If that point became too hot, it picked a different one. I called this the "MDP" — Most Desirable Point. When moving towards the point, the ship picked the direction which moves it to the coolest point closest to the MDP. Additionally, the border of the heat map was always heated up to discourage moving too close to the border and, when a power up was on screen, the MDP was chosen to be the coolest point closest to the power up. This worked okay, see the clip below:

<video class="center-image" style="max-width: 100%;" src="{{ site.baseurl }}/public/img/heatmap.webm" autoplay loop></video>

You can see the MDP and the current ship position visualised in the heat map. While this was more promising, it also failed to clear the levels with no misses and was never able to beat the second boss. Its most successful run can be seen below:

<iframe
    width="720"
    height="960"
    src="https://www.youtube.com/embed/_aBAka_uaqA"
    frameborder="0"
    allow="autoplay; encrypted-media"
    allowfullscreen>
</iframe>

This was the most promising method so far. However, as I was spending more and more time on this, I got a little frustrated and started to look into solutions that might be more stupid, but which require less time on my part.

## Bruteforcing

Simply brute forcing the game was the easiest (and dumbest) method, but it worked. It did a 2-ALL with A-L and scored enough to land on spot #2 on the global leaderboards.

To implement bruteforcing, I made the MAME plugin I wrote for DoDonBotchi save state every `n` frames. My algorithm then performs a random sequence of inputs for `n` frames and checks if the ship has died. If it did, it reloads the state. If not, it continues with a new save state. That's it. You can see this at work below:

<video class="center-image" src="{{ site.baseurl }}/public/img/brute.webm" autoplay loop></video>

As this clip shows, it's easy for the ship to get stuck in situations where `n` frames of input aren't enough to avoid death. To prevent this, the bruteforcing algorithm is smart enough to backtrack after all possible inputs have been enumerated. This is visible in the clip. The drawback is that often times the ship got stuck in some situation is takes ages to backtrack out of. The first boss is a good example of this. When it does it spin-up move and the ship had flown too close to the boss already, it needed to backtrack a lot of frames to get to a point where it's no longer in range for the upcoming spin-up move. For reference, I made the ship shoot all the time and reduced all possible inputs to the eight possible directions to move in. This means an input is an octal number, an an input sequence of `n` frames has `8^n` possible options. All these options need to be enumerated before the decision to backtrack is made. However, despite this, DoDonBotchi was able to bruteforce the game in around six hours of real time which, rendered as a video played back at the game's natural speed, was around three hours:

<iframe
    width="720"
    height="960"
    src="https://www.youtube.com/embed/YFw57XM_Yxw"
    frameborder="0"
    allow="autoplay; encrypted-media"
    allowfullscreen>
</iframe>

This run beat both loops with the A-L ship and got a final score of 282,008,400. This would place it as second best on the leaderboard over at the [Shmups Forum][3].

The big remaining problem is that I haven't found a way to re-create only the successful inputs to create an `.inp` file of that run. I can deduce which inputs were the successful ones, but the way I'm using MAME is apparently not deterministic. It *should* be, but due to all the save stating and loading, the frame timings tend to be a bit off during brute forcing and ultimately, the replay of brute force inputs goes out of sync after a while.

It's also not good enough. Merely never dying and always shooting is not enough to beat the highest known score, but it's also not even close to the ultimate goal of 999,999,990.

## Evolution

My next idea is to not brute force, but rather try to evolve the next input sequence. I've had the idea of evolving input strings since the beginning, but I could never think of a way to deal with the variable length required to beat the game. I can revise this to work somewhat like the bruteforcing and only focus on evolving the next `n` frames. Whatever was evolved before that would not be touched again. This has its drawbacks with regards to global maxima, of course, but it's a more feasible solution and, hopefully, always maximising something like the next 60 frames or so for the highest score will get us to the final goal.

A bigger issue on the horizon is that, to remove the issue of frame timings messing up replays, the genetic algorithm should replay the inputs already set in stone when trying to evolve the next window of `n` inputs. This will take ages, but seems to be the only way to guarantee deterministic playback with DoDonBotchi. Once that is a given, I can finally produce an `.inp` file.

[1]: https://github.com/keras-rl/keras-rl
[2]: https://www.cs.toronto.edu/~vmnih/docs/dqn.pdf
[3]: https://shmups.system11.org/viewtopic.php?t=56826