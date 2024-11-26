---
title: "Don't Bring Your Work MacBook To The Apple Store"
date: 2024-11-26T16:28:36-07:00
draft: false
---

It all started when I plugged in my MagSafe charging cable and saw that it was
blinking orange. I looked at my Mac and noticed it was not charging at all.
Rebooting didn't help, or using a different charging cable.

So I decided to go to the Apple Store. I could have brought it to the IT
department at work instead, but then I would have to setup a loaner machine
while I waited for the repair. I was working remotely out of state anyway;
plus, since it was just a broken charging port, I figured it would be an easy
fix. Right?

Right?

# First Visit: Bad Omens

The first warning sign of trouble that I should have noticed was when I brought
in the machine: they said they needed to restart it to run a diagnostic, and
they asked me for the firmware password to unlock the bootloader. I told them
that the firmware password was controlled by JAMF, and I would need to ask
someone at the office to send me the latest version of the password (it was
outside of office hours).

The Apple rep told me that they could order the part for me anyway so it's
ready when I came back in. I shrugged but I knew that would be heading back
home by the time the part arrived. I could still charge my computer with a
USB-C cable, so I put it off until later.

# Second Visit: The Monster Awakens

I spoke to the company IT department and asked how I can get the JAMF password.
They said that it rotates every 15 minutes, so someone would need to be on hand
to respond to my request for it during my visit with Apple.

I decided to wait until a Friday afternoon to go to the store because I could
drop it off for the weekend and pick it up after a day or two without losing
the ability to work. I made the mistake of choosing the Apple Store that was 25
minutes away because I thought I would only have to be there twice.

When I brought the machine, they were able to run the diagnostic using the
password I got from my colleague. Of course, like the previous store, they had
to order the new charging port so I would have to bring my device back in a few
days.

# Third Visit: The Horror Grows

I dropped off my laptop at the Apple Store and make my way back home, satisfied
that this slightly annoying problem will be alleviated.

However, the next day I receive a strange email from the Apple Store stating
the following:

> We have some important information about the service of your product. Please
> get in touch as soon as possible—you can contact us at <phone number>.

That seemed a little less than ideal, but I wasn't too worried. I called the
number and found an automated message directing me to various lines. I
mentioned that I was returning a call from the Genius Bar and I was transferred
to another line.

The line rang for 5 minutes straight and then hung up on me. There was no hold
music, just ringing. Then click and the call was over.

Confused, I called the number again. Directed to the repair department. Same
response. A third time, after 20 minutes of trying. No difference.

I called the automated number again and this time asked for the Sales
department. I was immediately sent to a real person, although the Sales
department was apparently centrally located and not connected to my particular
Apple Store. I explained to the Sales rep what happened and how I was trying to
call back the Genius Bar at a particular store location, and they very kindly
helped connect me to someone over there via a direct line of some sort. I was
very grateful as this was not in their job description but they chose to help
me out anyway.

Getting in touch with the Genius Bar, what did I find out? Yup, they needed to
run a final diagnostic to hand over the computer and therefore they needed the
JAMF password again. Of course.

# Fourth Visit: A Creeping Sensation

I spend another 25 minutes going to the Apple Store on Monday morning, ready to
pick up my computer. I had informed the IT team that I would need the password
again so I just had to wait for work hours to begin.

I got the password and had them reboot the machine to run the diagnostic. They
plugged in the charging cable and I noticed it was still blinking orange like
it had before. That did not seem like a good sign. Did the repair technicians
fail to even attempt to charge the device after replacing the cable?

The diagnostic did not pass. The technician told me that the real issue was
almost certainly with the logic board, whose contact point with the charging
port may have shorted out or something.

Well, great.

They said they would order the new logic board, and I would just have to come
in again. They said that the NVME drive is soldered to the board and will be
replaced as well, meaning I need to make sure I backup everything off the
device.

So I thanked them and left to head into work.

# Fifth Visit: Something Lurking Just Ahead

I wait for another Friday afternoon, after preparing my backups, and leave the
laptop for repair once again. I was ready to have this over with.

On Saturday, I once again receive the dreaded email:

> We have some important information about the service of your product. Please
> get in touch as soon as possible—you can contact us at <phone number>.

This time I know what to expect. I don't bother calling. I just have to come in
during work hours and get the password again to run the diagnostic once more.
So I wait patiently.

# Sixth Visit: Absolute Terror

Another Monday morning and another 25 minutes travel to the Apple Store for my
ritual encounter with the repair team.

However, this time they did not run the diagnostic. Instead, they told me
something unusual. Apparently, according to the repair technician who was
working on the board, they are not capable of replacing a logic board that is
protected by a firmware password.

They said that the firmware password would need to be removed, which means that
it will need to be essentially unrenrolled from JAMF.

This was a non-starter.

Somehow neither the technician who ran the diagnostic and ordered the part nor
the technician who accepted my dropped off machine had understood that this was
the case.

I knew immediately at that point that I had to give up with the Apple Store.
This whole mess probably should have gone through the IT RMA process the entire
time and I was hopelessly naive.

So the lesson is thus: Don't do what I did. Don't try to bring a managed
computer to the Apple Store for repair.

