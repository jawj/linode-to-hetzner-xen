Migrating from Linode to Hetzner
================================

Why?
----

I've had Linodes running for 4 years. I like Linode, and there are some **good reasons to stay**.

* Fast network, and decent disk I/O performance for a VPS. CPU power isn't bad either.
* Extremely reliable — in the London and East Coast US data centres, and in my experience.
* Regular service upgrades — disk space, CPU, and most recently RAM — for everyone, including current customers.
* Famously responsive and helpful support. (Which I barely ever need).
* Easy-to-use web interface and web-based console.
* Up-to-date and pretty wide selection of distros.

But recently, **I've been thinking about moving services to Heroku or Hetzner**. 

* Heroku should be dearer but easier. Moving to Heroku would mean largely giving up on sysadmin, allowing it to take more of my money and not much of my time. 
* Hetzner should be cheaper but (a bit) more time-consuming. Moving to Hetzner would mean _embracing_ sysadmin, continuing to become better at it, and allowing it to take a little more of my time and less of my money.

**Linode feels in some ways like an unhappy compromise** between these two options. I have to do almost as much sysadmin as I would for a dedicated Hetzner box running Xen. And I have to pay not so far off what it would cost me to give up sysadmin almost entirely on Heroku.


Then, there are **a couple of specific reasons** I might switch away from Linode.

* **Memory.** My Linodes run various web apps which are basically interfaces to a Postgres database. Performance is thus constrained more-or-less exclusively by RAM. As long as my database fits in disk cache, things are snappy. As soon as Postgres has to start hitting the disk, we have a problem. The recent RAM doubling is a big help here. However, cost per GB of RAM is still pretty high, which means I'm running with less than I'd like.
* **Security.** The incidents of the past few days have cast some doubt on Linode's security competence and transparency. This is a serious worry.

There are also some specific reasons that have been keeping me off Heroku and Hetzner.

* Heroku's **not-so-intelligent routing** shenanigans have slightly tarnished their appeal. 
* With Hetzner, my main worry is **disk failure**. Sure, you get two disks and can set them up in RAID1 with `mdadm`. But identical disks are liable to fail close together, and I don't look forward to using `mdadm` in anger with the clock ticking and my data at risk. Linode and Heroku deal with redundant storage transparently, and I think I trust them to do it right.

I still haven't decided what to do in the long run. If I were going to use Hetzner for production, I'd go for one of the machines with 16GB or 32GB ECC memory. But meanwhile, I got a 4GB server for about €25/month in Hetzner's ongoing auctions, and have been experimenting with moving Linodes to my own Xen setup on this dedicated machine.


How?
----

What's really nice about this, in theory, is that you can install Xen on your Hetzner box, then **transfer your Linode disk images directly** and boot them up with minimal downtime and few additional deployment steps.

(I realise that this isn't such an advantage for everyone: if you have some sort of push-button deployment set-up, say with Chef or Puppet, moving hosts is probably no big deal. To date, though, I haven't invested the time and effort to do things this way. I have a small number of systems, of varying ages, manually set up and configured. One of these days, I'll get round to learning Chef, Puppet, or similar. But even when I do, it's not going to help much with these legacy systems. Hence, this direct transfer of disk images is really appealing to me.)

However, it turns out that the **Xen documentation, official and otherwise, is less than ideal**. It is, variously, non-existent, inconsistent, contradictory, and just plain bad. It thus took me a few days' wrestling to make the transfer from Linode to Hetzner work, for reasons mostly to do with networking. And I therefore thought these steps might be worth sharing.

So, with that out the way, let's (proceed to the instructions)[instructions.md].
