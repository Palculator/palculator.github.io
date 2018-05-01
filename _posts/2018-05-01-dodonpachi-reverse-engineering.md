---
layout: post
title: DoDonPachi reverse engineering
---

I finally got the chance to poke around DoDonPachi a bit to see which
information I can easily access. This was mostly exploratory, as I didn't have
clear plans for what kind of AI [DoDonBotchi][1] would be going for and which
information it would need.

## MAME's debugger

Inspecting DoDonPachi's memory was, thankfully, made easy by [MAME's excellent
debugging][2] tools. They offer a basic disassembler (even decrypting the code
from the ROM), memory viewers, setting break- and watch points to pause
execution on certain instructions and memory accesses, stepping through
instructions with registers and RAM changing in real time, and, finally, a
cheat search that works similar to Cheat Engine. If you've never used Cheat
Engine, it basically lets you scan memory for addresses that contain certain
values and then narrow down that search by further filtering candidate
addresses when values change. For example, searching for where the current bomb
count is stored in DoDonPachi went something like this:

1. Initialise cheat search with `cheatinit ub`, searching for unsigned bytes in
   all memory
2. Search for addresses containing the initial bomb count with `cheatnext eq,3`
3. Play a bit and use a bomb
4. Now search for the addresses from the previously-found list that became 2:
   `cheatnext eq,2`
5. Repeat until it's narrowed down to one address

That's how the current bomb count was easily determined to be stored at
`$102CB0`. This was the easiest thing to find, but more on the harder ones
later.

The debugging interface looks like this:

![mame debugger]({{ site.baseurl }}/public/img/mame_debugger.png)

It looks confusing because it is, but you get the hang of it.

## Current game state

There are a few easy-to-find properties of the game state that are helpful to
know for any AI and were relatively easy to find. One example above is the
current bomb count. The same approach can be used to determine variables like
the amount of lives left, and the amount of credits inserted. These are all
single-byte values and can be found at the following addresses:

* Bombs: `$102CB0`
* Lives: `$101965`
* Coins: `$1013AB`

Other things like the current chain count would probably be just as easy to
find, but I haven't required those yet.

The most important value is conspicuously absent from this list, as you
probably noticed: Score. I wasn't able to find the score using the method above
because of two reasons. Firstly, the score is actually a composite of two
values: Your actual "score" you accumulated by playing and the amount of game
overs so far as the last digit. A score of `32512`, for example, wouldn't be a
single value `32512` in memory, but rather one value `3251` and another value
`2` representing two continues after a game over. This is why your score is
always a multiple of ten at the start of the game: no continues used. Secondly,
the score isn't actually stored in the decimal representation you see on
screen. It's kept in a weird mix of hexadecimal and decimal. To make this
clear, let's go back to the example of `32512`. We drop the `2` because it's a
separate value and end up with `3251`. DoDonPachi's RAM stores this as `$3251`,
meaning it's a hexadecimal value `$3251` where each digit represents the
respective digit of the decimal value. I don't know why it is stored that way;
the only reason I can think of is it being easy to map digits to sprites on
screen: Just use the four bits at the respective position. However,
manipulating the score becomes harder, as adding `10,000` to the score with a
simple addition now doesn't work. Whatever the reasons, the score is saved as a
32bit unsigned integer at location `$10161E`. Transforming it to its correct
representation can be done as follows:

{% highlight lua %}
weird = mem:read_u32(0x10161E)
score = 0
score = score + (weird % 16) * 1
weird = math.floor(weird / 16)
score = score + (weird % 16) * 10
weird = math.floor(weird / 16)
score = score + (weird % 16) * 100
weird = math.floor(weird / 16)
score = score + (weird % 16) * 1000
weird = math.floor(weird / 16)
score = score + (weird % 16) * 10000
weird = math.floor(weird / 16)
score = score + (weird % 16) * 100000
weird = math.floor(weird / 16)
score = score + (weird % 16) * 1000000
weird = math.floor(weird / 16)
score = score + (weird % 16) * 10000000
{% endhighlight %}

Silly, and the actual code I use does this in a `for` loop, of course, but it
gets the job done.

## Sprite locations

Score and lives would suffice to start a very simple AI that just tries inputs
until it doesn't die and achieves a high score. However, that would likely take
ages and boil down to simple brute force -- not fun. The AI needs to know
information about which objects, including the player-controlled ship itself,
are currently at what positions. I've had several ideas on how to obtain that
information, such as trying to read the map data from RAM, which should contain
which enemy is supposed to be where, finding the memory locations for enemies
currently loaded into the level, and so on. The only one I was able to pull
off, so far, was reading the regions of memory that place sprites on screen,
since every enemy, bullet, the player ship, etc. would naturally have a
corresponding sprite rendered. I got this idea from poking around in [MAME's
source code][3] for the CAVE arcade hardware. It reveals that DoDonPachi's
memory contains a region called `spriteram.0` starting at `$400000` and going
on for `$40FFFF` bytes. Take a look at a portion of this region during gameplay:

![ddonpach sprite ram]({{ site.baseurl }}/public/img/ddonpach_sprite_memory.png)

There's a clear pattern here: Five words (word size is 16bit) followed by three
words that are just zero. Furthermore, stepping through the game frame by frame
while keeping an eye on these values revealed that the third and fourth values
only ever change by small amounts. This gave me the intuition that they're the
sprite's coordinates, since they are also relatively small values that lie
within the CAVE system's resolution. Sure enough, when drawing a little 1px
point at positions indicated by the third and fourth word in each row, it ended
up at the lower left corner of each sprite.

The other pattern is that the fifth word seemed to always be composed of two
bytes, each of which is lower than 16. To find out what it does, I simply
started overwriting it with other values. It turned out that this word is used
to scale sprites. A value of `$0101` renders a sprite at `16x16` pixels,
`$0102` stretches it to `16x32`, and `$0202` doubles the size to `32x32`. This
was enough information to start visualising entities on screen:

![ddonpach sprite_outlines]({{ site.baseurl }}/public/img/ddonpach_sprite_outlines1.png)![ddonpach sprite_outlines]({{ site.baseurl }}/public/img/ddonpach_sprite_outlines2.gif)

As you can see, this includes sprites used for the HUD, but the AI will just
have to figure out what's noise and what's data.

What's left are the first two words. Intuitively, there needs to be some
information on which sprite on actually render, and I've concluded those first
two words to represent that: Sprite IDs. I assume they are indices on the
sprite atlas of the game, but I haven't confirmed that yet. The only other
observation I have to offer is that the last byte of the second word seems to
be a frame count for animations. If I put text for the sprite ID next to each
bounding box, it becomes apparent that the last two digits step through frames
of an animation:

![ddonpach_sprite_id]({{ site.baseurl }}/public/img/ddonpach_sprite_id.gif)

Apologies for the rotated text, but MAME's Lua functions to render text on
screen does not account for the orientation of the cabinet.

As I said, for now I consider this enough information for the AI. The
DoDonBotchi MAME plugin reads the entire sprite memory, parses the data as
outlined above, and writes it out to the DoDonBotchi server via a standard TCP
socket. This sounds like a lot of work, but DoDonPatchi never renders more than
1024 sprites, so essentially it boils down to reading 10kb of memory, which is
fast enough. This upper limit also helps a lot regarding machine learning,
since an observation can now be understood to be 1024-dimensional, padding it
out with zeros if fewer sprites than that are on screen.

## Future work

It would be helpful to find the data used for enemies in the game logic, not
just for rendering. The information on how many points an enemy rewards, how it
affects the chain multiplier, its actual hitbox, and so on have to be in memory
somewhere. There's a [fantastic blog post by someone much more talented in this
reverse engineering work than me][4] which provides some cheat files that
enable DoDonPachi's debug menu. Most importantly, it contains an object
spawner. However, I've yet to get it to actually work. I can get into the debug
menu, do things like switching levels or pausing the game, but the object
spawner does not seem to work, even for the known object IDs listed in that
blog post. If I ever do get it working, I could observe the code run to spawn
an object, where its data gets read from and where it gets written to and
probably figure out more about the internals of the game. Hopefully I'll figure
it out down the line.

I also found a talk about [reverse engineering arcade games][5] where the
speaker uses the [Radare2][6] disassembler for Super Street Fighter 2X and
visualises the call graph of subroutines in a much more intuitive way:

![radare2]({{ site.baseurl }}/public/img/radare2.png)

Unfortunately, I could not get radare2 to work the way it is shown in that
video. Even built from the official source, it fails to recognise the Motorola
68000 architecture used by DoDonPachi (and 2X in that video, as a matter of
fact) meaning I couldn't analyse the code like this yet. Once I do, I can get a
better grasp of DoDonPachi's code and how to get more information.

If anyone out there can help, hit me up! This has been my first reverse
engineering project, so there's a lot to learn, I'm sure.

The code reading all this data from memory is available [here.][7]


[1]: {% post_url 2018-04-26-dodonbotchi %}
[2]: http://docs.mamedev.org/debugger/index.html
[3]: https://github.com/mamedev/mame/blob/master/src/mame/drivers/cave.cpp
[4]: http://sudden-desu.net/entry/dodonpachi-debug-tools-level-select-and-more
[5]: https://www.youtube.com/watch?t=44m15s&v=88PQs5ZWUCM
[6]: https://github.com/radare/radare2
[7]: https://github.com/Signaltonsalat/dodonbotchi/blob/master/dodonbotchi/plugin/dodonbotchi_ipc/init.lua
