---
title: "War Story: The HID PID Bug"
data:
    tags: ["warstory"]
---

Lessons:
- Don't let seniority get in the way of doing what's right
- Look to the systemic cause of a problem
- Design APIs for the [Pit of Success](https://philosopherdeveloper.com/posts/pit-of-success.html)
- Technical debt incurs risk, and not just cost.

Looking back, I appreciate how much trust my first job out of college gave me.
Within a year of being there, I was neck deep refactoring one of the hairiest
pieces of (involved in a three-process concurrent handshake).  I was trying to
reduce the scope of the state in the code so I could see how to introduce my
first big project: support for a piece of hardware that communicates as a
[HID](https://en.wikipedia.org/wiki/Hardware_interface_design) device.

We had an existing stack for detecting and talking to hardware but it was all
coupled together, meaning I couldn't get it to talk to our new piece of
hardware.  I created a new path for detecting and talking to these HID devices
and all was right in the world.

A while later, we got word that a major university was bringing up their
student lab and our software was busted.  Somehow a legacy child device was in
our hardware database without its parent, breaking the schema, preventing any
further writes to the database.

That new HID code path I mentioned?  It was in a place that couldn't go through
our fancy new database abstraction which would enforce schema validation.  That
combined with the fact that I was doing a lookup of the device by PID (Product ID), without
the context of the VID (Vendor ID) or Bus.  Each bus (PCI, USB), defines its own standard
for assigning out VIDs.  Each vendor is then responsible for assigning out
PIDs, in the context of their VID.  So in this case, a HP USB Mouse had the
same PID as our legacy child device, causing my new detection path to insert it
into the database, without any safety checks to fail it out.

We switched the code to only doing lookups by the triple Bus, VID, PID, patched
6 versions of our software, and got on our way.

Once we realize this, we remembered getting vague, non-reproducible reports
from within the company over the last couple of months.

So what system failures led up to this?

- We had an API that only took PID, assuming Bus and VID were already verified, making this non-obvious when new code paths were added.
- We delayed getting rid of those old database access code paths.  Even 10
  years later, and with a PR ready to go before I left that company, the code
  path is still not updated.

Many years later, I found myself in a battle over this.  We had a new
cross-company hardware API.  We wanted to use it to support creating mock
devices for low cost validation.  Everyone wanted it to just accept PID because
"we'll support a third-party device in this API" (we would) and "but if we do,
people can specify it then" (notice this is the systemic failure that led to my
bug).  I had to fight an uphill battle, repeatedly like it was [Groundhog's
Day](https://en.wikipedia.org/wiki/Groundhog_Day_(film)), against multiple
engineers with double my seniority until they gave up and acquiesced.  Even
with us having seen this problem once before showing it was creating a trap for
future developers, they still wanted something "convenient" for themselves.
