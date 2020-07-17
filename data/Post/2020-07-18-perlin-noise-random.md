---
title: Biased Randomness
slug: perlin-noise
header: web-dev
summary: I used Perlin noise to skip state tracking for randomized failures over time, find out how.
author: piper
published: 2020-07-18
---

Last weekend, I joined in the GMTK Game Jam 2020. It was a milestone of sorts
for me and for ppb. It's actually the first time I participated in a game jam
_and actually submitted a game._ I participate a lot mostly on the "I'll do what
I can" basis, and have never had a thing assembled enough to be happy to submit.
For ppb, this represented the first known use of it in a game jam, and let me
tell you: it went very well.

You can check out my entry below:

<iframe src="https://itch.io/embed/694670" height="167" width="552" frameborder="0"><a href="https://pathunstrom.itch.io/smugglers-run">Smuggler&#039;s Run by Piper Thunstrom</a></iframe>

The theme of the jam was "Out of Control", and as a developer very focused on
user experience, I knew I wanted the state of being out of control to be part
of the challenge, but not to take away reasonable controls completely.

Thus spawned the theme of shock mines that make your controls unreliable.

That started as a raw chance to reverse your controls. The following code
is basically what it looked like:

    import random
    
    control = True
    fail_chance = .3

    roll = random.random
    if control and not roll < fail_chance or not control and roll < fail_chance:
        activate_control(control)

This is nice and simple to build, but there's one problem:

We roll every simulation frame.

When you do that, one frame the roll will be under our fail_chance, and the next
it might be higher. There's not enough change per frame for that to really feel
like something failed to the end user. So I needed another solution.

I considered, for a while, having a chance for it to fail, then a time tracker
until it was good again, but that would be a lot of effort and debugging. So I
continued to work on other bits of the game, but had this problem in mind.

I then remembered that I've been doing research on
[normalized transforms](https://www.youtube.com/watch?v=mr5xkf6zSzk)
(you might know them as tweens or easing functions) and I realized I could apply
the same strategy of normalizing them to fake my random.random.

With that idea in hand, I reached for Perlin noise. A quick summary for those
new to Perlin noise: it's a form of gradient noise on an arbitrary number of
axes. For our purposes, no matter how many dimensions you run it in, you get
back a value in approximately the (-1: 1) range. You can
[read more here](https://eev.ee/blog/2016/05/29/perlin-noise/).

Okay, two tools in hand, how do we apply this?

For each control the player has access to, I defined a separate 1 dimensional
Perlin noise generator. It looks like this:

    main_random = PerlinNoiseFactory(1)
    retro_random = PerlinNoiseFactory(1)
    left_random = PerlinNoiseFactory(1)
    right_random = PerlinNoiseFactory(1)
    rot_left_random = PerlinNoiseFactory(1)
    rot_right_random = PerlinNoiseFactory(1)

Now each of those variables is a function that takes a single input, and outputs
a value between -1 and 1. It's not _quite_ random.random, but we can work with
this.

Since each one of these items ended up being its own state, I made a dataclass
to track them:

    @dataclass
    class Component:
        control_name: str
        operable: bool = True
        damage: int = 0
        CONFIG_MAX_DAMAGE = 100
        randomizer: Callable[[float], float] = lambda _: random()

So here you can see the previous version of the randomizer: Take a single input,
output a value from 0 to 1. Again, this _works_ it just doesn't feel quite as
nice.

So the final step to setting this up, is changing the component randomizer based
on the individual controls:

    components = {
        "forward": Component(
                "forward", 
                damage=CONFIG_STARTING_DAMAGE, 
                randomizer=lambda x: (main_random(x) + 1) / 2),
        "backwards": Component("backwards", damage=CONFIG_STARTING_DAMAGE, randomizer=lambda x: (retro_random(x) + 1) / 2),
        "left": Component("left", damage=CONFIG_STARTING_DAMAGE, randomizer=lambda x: (left_random(x) + 1) / 2),
        "right": Component("right", damage=CONFIG_STARTING_DAMAGE, randomizer=lambda x: (right_random(x) + 1) / 2),
        "rotate_left": Component("rotate_left", damage=CONFIG_STARTING_DAMAGE, randomizer=lambda x: (rot_left_random(x) + 1) / 2),
        "rotate_right": Component("rotate_right", damage=CONFIG_STARTING_DAMAGE, randomizer=lambda x: (rot_right_random(x) + 1) / 2)
    }
    
Here's the initialization of the components. Note that the component name and
the key are the same, and I'd probably find a nicer way to handle this in the
future.

The important piece, though, is the randomizer function:

Our randomizer now takes one input, passes that to our `main_random` function
(remember, it's between -1 and 1), then normalizes it by adding one and dividing
by 2. So the output of this randomizer has the same range as our original one,
but because it's perlin noise, it has some unique properties when applied over a
continuous sequence. A continuous sequence like, say, time.

    def control_active(self, component, controls, now):
        _random = component.randomizer(now / 5)
        malfunction = _random < component.damage / component.CONFIG_MAX_DAMAGE
        control_val = getattr(controls, component.control_name)
        return (control_val and not malfunction) or (not control_val and malfunction)

Here it is, the actual implementation. I pass in the value from
`time.perf_counter` for this frame, and I divide that by 5 (which stretches the
gradient out) and compare that random value to the percentage chance of a
malfunction, calculated here as the percentage damage the component happens to
have.

What this does is if the gradient of the Perlin noise goes low, it keeps going
low for a period of time, and then rises again, based on the offsets of the
underlying noise implementation. Instead of a bunch of state managing when and
how things fail, we just let our noise function operate on time and give us that
behavior for us. And by dividing or multiplying our input, we can adjust the
intervals.

So try out some biased randomness in your next project, it's surprisingly simple
to replace the built in randomizers and you get very different flavor from
something like a noise function versus than you would get from a uniform random
function.
