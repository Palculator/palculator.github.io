---
layout: post
title: More DoDonPachi reverse engineering
---
## On-screen game objects

As explained in the [last post about DoDonBotchi progress][1], I needed to find
out more about DoDonPachi's memory layout to get more information about the
game objects the AI has to interact with. My last [reverse engineering post][2]
explained how I got to the data needed to read which sprites are currently
being rendered, including their position, sizes, and a sprite ID. However,
there was no pattern behind these sprite IDs; enemies, HUD, debris flying off
things being hit, were all bundled together with only the sprite ID to tell
them apart. Looking at the IDs themselves and which sprites they are on screen,
I could not determine a real pattern to them. I resigned to manually
classifying them into their types. For that, I wrote a simple MAME plugin,
which, whenever it encounters an unclassified sprite on screen, highlights that
sprite, puts it at a fixed position on screen, and prompts me for what it is. It
looked like this:

![sprite classifier]({{ site.baseurl }}/public/img/sprite_classifier.png){:.center-image}

Each new sprite would get a coloured border at its actual location on screen
with a copy of it rendered in the lower portion of the screen. The command line
DoDonBotchi is launched from then prompts for sprite classifications using the
tool [fzf][3]. This worked fairly well, and I'm writing about it here since it
was nifty and required a decent amount of work, but, unfortunately, I quickly
grew tired and came to terms with the reality that going through potentially
`2^32` sprites to classify them was too much work. Instead, I went back to look
at DoDonPachi's RAM to maybe figure out a better way to get information about
game objects.

I started off dumping the memory regions `$100000 - $200000` and `$400000 -
$500000` to files that I could read through more easily. These regions would
include the RAM and sprite RAM of the machine respectively.  Thanks to my
earlier work on sprite classification, I knew which sprite IDs belong to
enemies. My gut feeling was that these sprite IDs have to come from somewhere,
so I looked for an enemy sprite currently contained in sprite RAM and
cross-referenced it with the contents of the normal RAM. This revealed a memory
region that, at the very least, contained sprite IDs. Staring at this region
for a while and observing how it changes frame-to-frame, I've inferred the
following layout:

![ram objects]({{ site.baseurl }}/public/img/ddonpach_ram_objects.png){:.center-image}

Starting at address `$104AF6`, each enemy on screen takes up 32 bytes. The
first few words are what matter to us, and are highlighted in the image above.
The first one seems to be some object ID, followed by the sprite ID we already
know from sprite RAM. After those, there's the position of the object. Not
actually its pixel-position on screen, but rather its pixel-position
multiplied by 64. I'm presuming this is to allow for sub-pixel movements (and
to confuse me.) The position is followed by a size-word, which works just like
the one for the sprite: Each byte multiplied by 16 is the object's width and
height in pixels. For the rest of the 32 bytes, I wasn't able to figure out
what they mean. Some seem to work like timers, decrementing each frame and
such, but I didn't look much further.

This data alone doesn't help much; I already had positions and sizes and some
way to identify objects, after all. The issue was knowing what type of object
it is and the only improvement on that front is that the object ID is 16 bit,
whereas the sprite ID is 32 bit, so fewer options to classify. However, looking
at the surrounding regions in RAM, there are several ones that follow the
layout above, but contain objects of different types. Enemies are in one
region, bullets in another, bonus items in another, and so on. To find all
enemies, one just needs to scan that region and knows the object found there is
an enemy. Same for the others. The regions I found/looked for are:

* Own shots: `$102EF6 - $1038F6`
* Enemies: `$104AF6 - $106836`
* Bullets: `$106CB6 - $108CB6`
* Power ups: `$10A9F6 - $10AB16`
* Bonuses: `$10AB36 - $10BDF6`

This solves the problem of object classification: Scan each region and group
them up as types of whichever region they're from. I've written rendering code
to highlight each object and their type, it looks like  this:

<iframe
    width="720"
    height="960"
    src="https://www.youtube.com/embed/YBrB51pCnzo"
    frameborder="0"
    allow="autoplay; encrypted-media"
    allowfullscreen>
</iframe>


## Additional properties

The most important game object is missing in the regions listed above: Your own
ship. Luckily, that was easy to find using the same method from earlier.
Looking for the sprite ID of the ship in sprite RAM reveals the ship's location
in regular RAM, including its position. It follows the layout described above,
but is only one such object. It can be found starting at address `$102C86`.

Additionally, for sake of rewarding combos, I looked for data related to those.
These were easy enough to find using the cheat methods described in the last
post. What I found was the current combo timer at `$1017D1` and the current
chain count at `$1017EA`.

## Reward function

Armed with this new data I've formulated a new reward function that tries to
reward the neural net for shooting at enemies, avoiding bullets, and keeping up
a combo. More specifically, the reward for an action is the sum of the
following:

* `1 / (smallest distance of own shot to an enemy)`
* `(smallest distance of an enemy shot to our ship) / 400`
* `(current combo timer) / 55`

The first component ranges from 0 to 1, approaching 1 as the distance of your
own shots to an enemy ship gets smaller. The second component also ranges from
0 to 1, approaching 1 as the player ship moves away from enemy bullets. The
division by 400 seems arbitrary, but it's the longest distance two on-screen
objects can have on DoDonPachi's 240x320 screen. Finally, the combo timer's
highest value is 55, so dividing it by 55 also forces it to be between 0 and 1,
approaching 1 the higher the combo timer is. This leads to a reward from 0 to
3, a perfect one being 3.


## Early results

Hooking up this reward function and new data to the neural net didn't improve
much, unfortunately. Below is a video of the AI after eight hours of training.
However, there is still work to be done on the format of observations.
Currently, game objects get listed in sequence in the observations, sorted by
their position. This ends up with objects that are technically in the same
position being shifted around within the observation a lot, introducing a lot
of noise for the neural net. I call these "early results" because of that: I
know there's still work to be done and the new reward function alone wouldn't
have solved everything.

<iframe
    width="720"
    height="960"
    src="https://www.youtube.com/embed/GenHTeGw1Eo"
    frameborder="0"
    allow="autoplay; encrypted-media"
    allowfullscreen>
</iframe>

As you can see, it's still randomly going apeshit. Initially, I was impressed
by it chaining the first group of tanks nicely, but looking at the scores
DoDonBotchi got over time confirms that it's mostly random and there's no
upwards trend. Don't even need linear regression to see it:

![score over time]({{ site.baseurl }}/public/img/score_over_time.png){:.center-image}

I'll write a proper progress report once I've finished the work on the
observations and got more meaningful results from those.

[1]: {% post_url 2018-05-09-dodonbotchi-sitrep %}
[2]: {% post_url 2018-05-01-dodonpachi-reverse-engineering %}
[3]: https://github.com/junegunn/fzf
