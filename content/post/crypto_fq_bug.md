+++
date = "2016-08-27T12:02:58+01:00"
draft = true
tags = [ "wifi", "bufferbloat", "ath9k" ]
title = "Solving the crypto-fq bug in ath9k"
description = "How I spent my summer vacation!"
+++

We'd finally hit the holy grail! You can't do much better than this on
wifi, dramatically reducing latency while also improving
throughput. After 5 years of effort, we'd won. It was time to attend a
few conferences in Europe, and take a vacation.

Some of the patches went out... And two of the first reviews came in,
telling us they didn't work. One of our problems has been that we didn't
get *all* the very interrelated patches out, not the whole enchalada,
and thus people have been testing (and often, reverting), individual
patches, in their particular circumstances, where we have been testing
the whole thing, with our limited testbeds, and engrained habits.

Swapping out the engines completely on the airplane, while dealing with
billions of actual users and an incredibly complex codebase, is *hard*.

In particular Felix Feitkau (of the lede-project) reverted the FQ
portion of the patch, which got bandwidth up where it belonged (with
codel engaging), but it cost latency, and we didn't know why it was
needed. We'd actually not expected Felix to come up with a backport
that fast in the first place, there are a good dozen infrastructural
patches needed to be in play, but he managed to do it, find a bug, 
report it, and then work around it, in under a week...

Was the bug in the backport in the mainline code? Was our code
(developed on x86) too slow to run on cheap-ass platforms? What??

Another reporter claimed the patches caused UDP traffic to slow from
800Mbit to 30Mbit in his testbed environment. Let's tackle that one first:

## Getting clobbered by UDP floods

When you flood a device like this you are no longer testing the speed
of the hardware, media or driver, you are testing the speed of the
discard part of the path, rather than anything else. Real networks
don't have floods like this... except that generated by test tools,
broken software and DDOS attacks. A ping -s 1400 -f somewhere sufficies.

Network theorists spend way too much time thinking about circumstances
where everything is behaving nicely and according to spec, and not
enough time thinking about how nasty the real world can be. The need
to discard packets is so bad and so frequent on the Internet itself
that companies like Cloudflare *make a living doing it*, and companies
like Facebook are promoting techniques that can do it at the earliest
moment possible - in the core receive path of the ethernet driver
itself!

After a [long debate](fixme), Eric Dumazet jumped in with [a drastic
patch](fixme) that would drop up to 64 packets in a row from a queue
under a single lock, under this overload condition. That sped up the
code *enormously*.

(he does this all the time)

This right answer is much like what is being proposed in the IETF
[circuit breakers drafts](https://tools.ietf.org/html/draft-ietf-tsvwg-circuit-breaker-15),
except that this one needed to enter the fq_codel qdisc in order to
make it behave sanely on an overload like this. If we ever get around
to issuing a [fq_codel RFC](https://tools.ietf.org/html/draft-ietf-aqm-fq-codel-06).bis,
we'll have to include this very important change.

Lastly, I don't know if this change migrated into the wifi codebase
for fq_codel or not, or got backported into openwrt! I'd burned
out on following all the patch sets at that point, and all I was lugging
was a laptop with a tiny screen I can't see or multitask anything on.

The second problem was even more severe and took over 3 months to find
and more or less fix. Helping find and fix it was basically was how I
spent my summer vacation.

## fq + wifi crypto bug

For starters, we didn't know that crypto was the related problem. It
was the FQ bug - and of all the fq_codel components - the FQ portion
of the algorithm had always given us the least trouble!

Stretched the limits of the possible, and fallen off a cliff somewhere.

Codel wasn't working. The bandwidth of the flows were being regulated
by something else. Some flows would stop entirely until a 250ms
timeout. By shaving all this latency out of wifi it seemed we'd broken
TCP. UDP floods were fine...

What that "something else" was was really hard to find. We explored
TCP send buffering, receive buffering, pacing on or off, disabling the
new ratt code, trying things with OSX...

Felix found several bugs with wifi powersave handling. I'd found [symptoms of these bugs](/post/poking_at_powersave) months earlier but failed to identify the cause. But that wasn't it.

Kathie Nichols brought out her spanking new [TSDE tool](fixme) to analyze the
captures, showing the median delay to be right in the ballpark of
20-30ms.

{{< figure src="" >}}

Matt Mathis suggested

Eric Dumazet thought that [TCP small queues might be the culprit](fixme).

Andrew McGregor blamed an iperf threading problem.

Avery claimed the codel code was already slower than the FIFO queue code.

We tested all these assumptions. Everyone was stumped.

...

Sometimes I think the only thing going for me is a clear goal and
dogged persistence - I don't have the coding chops Eric has, nor the
deep insights into the structure of the ath9k code Felix and Toke
have, nor the experiences and insights into tcp's behaviors Kathie,
Van and Matt have, nor a clear grasp of Minstrel that Andrew and
Thomas have... I just want to end bufferbloat in my lifetime!

And despite that "persistence" - my ADHD has really got out of hand:

I'd got my fingers into too many pies - trying to bring in the ns3
code, dealing with several ietf working groups, consulting for comcast
and google, coping with funding the bufferbloat project one more year,
keeping the website and mailing lists humming - testing the code on
ath10k also - getting more labs setup - building kernels for my
testbed - helping the FCC - I'd devolved into a :gasp: m-m-m-m-manager
- living on what Paul Graham called [the managers schedule](fixme),
one that doesn't know how to solve any technical problems, just *who*
might - and the whole point of me taking this trip was to finalize a
bunch of stuff, get it off my plate, and deal with the burnout! I
needed less stuff to do, so I could go deep on the things blocking
everything else, make an actual technical contribution, and I needed
"just say no" to anything new, for a while... and this was possibly
the last bug blocking a revolution in wifi performance with millions
of essentially broken wifi implementations shipping per day!

"Dave - the HRC campaign is on line 1!"

"Tell them I'm busy and they should just ask QCA to open up the source to the firmware and publish some !@#!@ documentation!"

"They just need your proposal for their first 100 days by tomorrow."

"I'll get back on it! 2PM ok?"

## Anyway...

During the IETF conference in Berlin... we finally gathered together nearly
everyone at a meeting at the [c-base hackerspace](https://www.c-base.org) to have a go.

{{< figure src="https://lh3.googleusercontent.com/hJwVJGSgtMClhfKASLjVAQ5YCO87zaIm9gcZxnmhPNekeWZ9ALa41FHPyyweZEzKWo4j4ImpJg=w4096-h2160-no" link="https://plus.google.com/u/0/107942175615993706558/posts/J7dmBEVJknP" >}}

(from left to right - Tim Shepard, Felix Feitkau, Toke Hoiland-Jorgenson, and Matt Mathis)

I wish I'd taken some more pictures - we had the most people

Felix and Tim had a chance to catch up on the chantx issues, Tom
HuhN(fixme) showed off his minstrel-blues effort to everyone multiple
times, Juliusz

And I went from person to person, analyzing the captures in their
different ways, trying to gain an insight.

I don't remember all that happened that night (beer was present), but
all of a sudden... I started having fun again. Stumping the smartest
people on the planet is inspiring, and knowing that they are just as
stumped as you are AND willing to pitch in, more so.

"Pain shared is reduced, Joy shared, increased", Spider Robinson once wrote.

I blew off a couple meetings, cleared my schedule, and prepared for my
last week with toke in Denmark. Short RTTs between us is a goodness -
when we usually have a 10 hour time difference, now we could cut that
down to ms!

We had one last shot to work closely together, and we'd exausted a
long set of blind alleys.

...

Hacking continued. Felix found a few more bugs.

...

And finally, after everyone doing all this work, after building a
large chart of everything we'd and they'd tested, with known knowns,
the known unknowns, the stuff confirmed and not... Toke and I realized
that having *crypto* on or off was the key variable.

After that the problem became easily repeatable.

Toke found that the wifi queues were emptying 1500 times over the
course of a 30 second test.  He tore apart the pattern of the queue
emptying and filling with a pretty cool custom histogram tool, but got
nowhere. He also found that keeping "some queue" queued up even when
it could fill an entire txop - cutting the size of the txop - cut the
number of fluctuations to 2 per second. This interval was enough to
almost - but not quite entirely - re-enable the codel algorithm, which
depends on delays being persistent.

I kept seeing persistent indications in the aircaps that we were
somehow breaking the block ack window - I'd been seeing these [for
months](fixme) - that the number of packets going into the air was not
necessarily the same as the number that came out - but that's always
the case in wifi - we really need better tools to look at aircaps with
- and doing that analysis by human eyeball and mental pattern
recognition should get delegated to software.

But we still couldn't find it. I gave up and started to spend my
nights talking with one of Toke's roomates, trying to clear my head
and think strategically about the problem, thinking really hard about
giving up entirely and going off to do something easy like
basket-weaving - while he took a deeper and deeper dive, and
periodically popped up to tell me what he'd done, and my overworked
subconcious would then come up with ever crazier ideas to try.  Finally he
too gave up, wrote up his results and [posted to the mailing
list](fixme).

... and a few hours later, Felix [zeroed in on the problem](fixme).

Wifi framing is dependent on 3 numbers being incremented in sequence -
the block ack, the QoS parameter, and the crypto IV. Crypto was a late
addition to the WiFi standard.  The QoS number came as a part of the
802.11e standard, and the Block Ack sequence came in as part of
802.11n.

I *didn't* know this - I thought there were only two! 

We were adding in the IV on the enqueue step, and on the dequeue,
pushing out packets in fq'd order. Most of the time, until bigger
aggregates started forming, the fq'd order would be close to the
enqueue order.

The fix involved basically moving 4 lines of code from one routine to
another, to defer asssigning and incrementing the IV until after the
dequeue. Shazam!

On the same day, the ns3 code got merged into mainline. Finally, we
have a world where not only others can simulate fq_codel behaviors,
but also start figuring out how to apply them to the BQL and WiFi
simulations now therein without needing hardware or deal with the
complexities of the Linux Kernel!

It was a good day.

## Aftermath - the bad

* We should have asked the bug reporters for their exact configuration

Had we done that, we'd have been able to duplicate the circumstances
immediately, and we'd have made a lot more progress long before we
did.

* Debugging with crypto enabled is hard

One reason why toke and I had fallen out of the habit of enabling wpa
is that you can't decrypt the captures unless you capture the moment
in which the crypto session is negotiated. If you don't get that
exchange, the entire capture looks like a pile of garbage - you can't
see the IP headers, you can't see anything but the mac80211 headers

and thus it never occurred to us to try crypto on or off -

The rest of the world enables wpa encryption almost universally now.
Mae Culpa.

Introducing crypto into the mix also increased the probability of
running into other bugs - we know there are many, and here we are,
just trying to hold latencies under load low - and being entirely
focused on that problem.

the test matrix for the

But, um... well, we're going to add crypto to that now.

* Statistical analysis failed us

* We broke everybody's test tools

A lot of good things happened.

## Aftermath - the good

* Kathie improved her TSDE test tools to look at drops and reordering,
as well as adopting infrastructure to parse IPv6. You can clearly see
the carnage, now, with the red marks ind

{{< figure src="/flent/cryto_fq_bug/kathie2.png" >}}

* Simon Wunderlich's lab joined the effort

He did a bunch of tests with 30 stations showing that we'd licked the
airtime fairness problem, fully, and truly. I'm tempted to make the
resulting graph the logo of the make-wifi-fast project.

{{< figure src="/flent/crypto_fq_bug/airtime_plot.png" >}}

He also showed that without fq, with 30 stations running at 1Mbit,
latency still sucked - but we knew that. The bare minium latency under
those circumstances would be close to 500ms, and he measured 1 to 2
seconds, which is more or less within expectations - still horrible -
but that's wifi for you.

But things still worked, and hopefully soon he can move on to testing
higher rates with the same number of stations.

* UNH is interested too

And me, somehow I fit in emotional visits to the Berlin Wall, and
Checkpoint Charlie, got a chance to play some music in a studio in
Bristol, and pushed back some of my incipient burnout.

## Aftermath #2

I didn't expect to have got this far with the OpenWrt side of things in the first place. I'd basically been planning a deployment, at scale, in the yurtlab, in november, where a bunch of other pieces of the project

As it turned out, we'd not solved enough of the problem - only
handling the one crypto case we'd looked at, not TKIP. Johnanas Berg
kicked back the first patch attempts with pithy comments.

Felix found some more bugs.

With enough bugs, all eyeballs are shallow.


https://www.bufferbloat.net/projects/codel/wiki/CakeTechnical/