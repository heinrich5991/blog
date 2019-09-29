---
layout: post
title: The Anatomy of a One Tick Unfreeze
tags: ddnet
permalink: /blog/one-tick-unfreeze
---

I saw the deepfly bind the other day:

```
bind mouse1 "+fire; +toggle cl_dummy_hammer 1 0"
```

<div style="text-align:center"><iframe width="560" height="315" src="https://www.youtube.com/embed/R5ZKcHXube0?start=21" frameborder="0" allow="autoplay; picture-in-picture" allowfullscreen></iframe></div>

[Deepfly](https://youtu.be/R5ZKcHXube0?t=21) is a technique that allows you to
fly with a tee that's deep frozen, i.e. frozen until it hits an undeep tile.
This means that any input that the deep frozen input gives is discarded, except
when it gets hammered or shot by the unfreeze laser, in which case it is given
a game tick where it can fire a weapon or use its hammer. In the case of
deepfly, you join the game with a dummy, a second tee that is also controlled
by your client. You then hook the dummy, hammer it, causing it to unfreeze and
hammer you back, causing you to generate upward movement.

I heard that you need a reliable connection to the server to manage to make the
deepfly work. This sounded a lot like frame-perfect input being necessary.
I found it a little weird, so I decided to investigate.

One tick unfreezes are part of DDRace and DDNet gameplay. They work reliably
for shotgun, grenade and laser, at the very least since
[39704a04](https://github.com/ddnet/ddnet/commit/39704a04a7c7e4db04d90b13c989ec94b4b42202)
by [Speedy Consoles](https://github.com/Speedy-Consoles). But they've been used
by tees all the way back to the beginning of DDRace. So how do they work and
why?

Currently input and freeze interact as follows:

```
          Tee 1 (weak)            Tee 2 (strong)

         +---------------+       +---------------+       
         | weapon input  |       | weapon input  |       
start -->| if not frozen |    +->| if not frozen |    +-> ... 
         | (non-/auto)   |    |  | (non-/auto)   |    |   ...
         +---------------+    |  +---------------+    |   ...
                |             |         |             |   ...
                v             |         v             |   ...
         +------------------+ |  +------------------+ |   ...
         | unfreeze targets |-+  | unfreeze targets |-+   ... --+
         +------------------+    +------------------+           |
                                                                |
                                                                |
                                                                |
         +---------------+       +---------------+              |
         | weapon input  |       | weapon input  |              |
         | if not frozen |<---+  | if not frozen |<---+   ... <-+
         | (only auto)   |    |  | (only auto)   |    |   ...
         +---------------+    |  +---------------+    |   ...
                |             |         |             |   ...
                v             |         v             |   ...
         +------------------+ |  +------------------+ |   ...
         | unfreeze targets | |  | unfreeze targets | |   ...
         +------------------+ |  +------------------+ |   ...
                |             |         |             |   ...
                v             |         v             |   ...
         +------------------+ |  +------------------+ |   ...
  end <--| check own freeze | +--| check own freeze | +-- ...
         +------------------+    +------------------+    
```

Firstly,
[go through all the tees by client ID](https://github.com/ddnet/ddnet/blob/ef32fc4beda0e02e9f83f5784d3a1e862b424944/src/engine/server/server.cpp#L2022) and
[let](https://github.com/ddnet/ddnet/blob/ef32fc4beda0e02e9f83f5784d3a1e862b424944/src/game/server/gamecontext.cpp#L933)
[them](https://github.com/ddnet/ddnet/blob/ef32fc4beda0e02e9f83f5784d3a1e862b424944/src/game/server/player.cpp#L477)
[fire](https://github.com/ddnet/ddnet/blob/ef32fc4beda0e02e9f83f5784d3a1e862b424944/src/game/server/entities/character.cpp#L702) their automatic
(laser, shotgun, grenade) and non-automatic (pistol, hammer) weapons, [if
they're not frozen](https://github.com/ddnet/ddnet/blob/ef32fc4beda0e02e9f83f5784d3a1e862b424944/src/game/server/entities/character.cpp#L363) by the time
it's their turn. This immediately unfreezes targets of
[l](https://github.com/ddnet/ddnet/blob/ef32fc4beda0e02e9f83f5784d3a1e862b424944/src/game/server/entities/character.cpp#L565)[a](https://github.com/ddnet/ddnet/blob/ef32fc4beda0e02e9f83f5784d3a1e862b424944/src/game/server/entities/laser.cpp#L28)[ser](https://github.com/ddnet/ddnet/blob/ef32fc4beda0e02e9f83f5784d3a1e862b424944/src/game/server/entities/laser.cpp#L69)
and
[hammer](https://github.com/ddnet/ddnet/blob/ef32fc4beda0e02e9f83f5784d3a1e862b424944/src/game/server/entities/character.cpp#L428),
allowing tees with higher client IDs to fire their weapons even if they were
frozen at the start of the tick.

Then, [go through all the tees by weak-strong hook
order](https://github.com/ddnet/ddnet/blob/ef32fc4beda0e02e9f83f5784d3a1e862b424944/src/game/server/gameworld.cpp#L253)[^1], starting with strong hook, and
[let them fire](https://github.com/ddnet/ddnet/blob/ef32fc4beda0e02e9f83f5784d3a1e862b424944/src/game/server/entities/character.cpp#L750) their automatic
weapons if they're not frozen. Unfreeze every hit tee, so that tees later in
the order can also act. Then check whether the tee is on a freeze tile or is
deep frozen and freeze it in that case.

Now, after this explanation, try to figure out how deepfly (hammering your
dummy tee who is deep frozen and who hammers you back, this is usually done via
the deepfly bind shown above). Try to find something strange. ;)

[^1]: Internally, this is entity insertion order, i.e. character spawn order.

---

Well, it turns out that deepfly only works if your dummy has a lower client ID
than you do. If you're reading this blog post right as it came out, try it on
the official servers -- if not, maybe it is fixed in the future and will work.

<div style="text-align:center">
<img src="{{site.baseurl}}/assets/2019-09-28-one-tick-unfreeze-no-unfreeze.gif" alt="Tees unsuccessfully trying to fly out of freeze" />
</div>

Additionally, know that moment when trying to fly up from freeze and the tee
below you is just too dumb to hammer you back? They probably have a higher
client ID than you do and simple **can't** hammer you while they're in freeze.

<div style="text-align:center">
<img src="{{site.baseurl}}/assets/2019-09-28-one-tick-unfreeze-unfreeze.gif" alt="Tees successfully flying out of freeze" />
</div>

---

So, deepfly needs frame-perfect input and only works on tees with lower client
IDs than yourself. I don't consider frame-perfect input to be a good game
design, so I propose fixing that. This would allow players using a normal
client to perform this trick even if they have high ping. While we're at it, we
could also fix the higher-lower client ID thing.

My proposed fix ([#1922](https://github.com/ddnet/ddnet/pull/1922),
[f19220f1](https://github.com/ddnet/ddnet/pull/1922/commits/f19220f1bbbb647954d6544865e3c2a2e9953037))
does this by remembering whether you were frozen last tick and if you were, it
lets you hammer like an automatic weapon, i.e. simply by having held that
button.

What are advantages and disadvantages of this patch?

First, this allows people to do more than they could have done before: Do
deepfly/hammer out of freeze even if you're the one that is frozen and have a
higher client ID than the tee that hammers you. This could give you an
advantage on server ranks.

Second, this allows people to do the deepfly/hammer out of freeze more easily.
However, this is something that was possible before, using frame-perfect input,
an [unlocked scroll wheel](https://youtu.be/aANF2OOVX40) bound to `+fire` or a
nonstandard client.

The second point is also an advantage; it allows people to more easily
experiment with tricks that have already been possible before, but were just
hard to execute without hardware or software support. This might remove some
incentive to use nonstandard clients.

Thanks to [Patiga](https://ddnet.tw/players/Patiga/) and
[Zwelf](https://ddnet.tw/players/Zwelf/) for investigating the intricate
mechanics of one tick unfreezes, deepfly, etc. and testing out my fix for it.
Thanks to [Zeta-Hoernchen](https://ddnet.tw/players/Zeta-Hoernchen/) for
recording the GIFs with me.

PS: This post has rested for almost two months before being published. I guess
I'm not made for blogging. The initial inspiration to publish anything at all
came from my enjoyment of reading the [Dunely
Daily](https://teedune.wordpress.com/), back when it was updated.
