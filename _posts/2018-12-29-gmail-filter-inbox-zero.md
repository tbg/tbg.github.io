---
layout: post
title: "Using Gmail like Inbox (Zero)"
comments: true
permalink: "gmail-filter-inbox-zero"
---

With Google Inbox [shutting down] in March 2019, I made the move over to Gmail
for my work email a few months ago. My volume of non-Github emails is low, so
taming the Github firehose was the crucial point here.  In Inbox, I had a setup
(an admittedly wonky one) that would filter GitHub mails that were likely
relevant to me into categories from which I would "mute out" the threads I
didn't think were relevant to me. That way, I'd look at any issue/PR that I
wasn't going to be involved in once, without having to re-discard the thread
when new messages would arrive over and over again.

This allowed me to be pretty close to [Inbox Zero] in all categories.

Note that Inbox/Gmail is smart about un-muting if the "disposition" changes
(i.e. if you receive a Github notification because you receive all
notifications, then mute, and then get assigned to the issue, the last message
will unmute the thread because the recipient changed).

The way I wanted things organized in Gmail is as follows:

- the inbox receives stuff from the Github firehose that I'll likely want to
  look at
- a Firehose label (skipping the inbox) receives the remainder of the Github
  firehose
- a Talk label (skipping the inbox) receives all non-Github mail.

It's straightforward (in my case) to define the filter rules so that emails go
into the appropriate labels. Unfortunately, muting doesn't work the same way
(except for the inbox): Muted threads will still be sorted into the labels and
there's no way around that (since muting is a per-thread property, but all the
filter gets to look at is the newly arrived message).

So I use an AppScript cronjob that removes messages belonging to muted threads
from my custom labels, which is good enough for me. Occasionally there'll be a
muted thread there, but if I ignore it, it'll go away and, more importantly,
the vast majority of them never even surface on my screen again.

See [here] on how to get this script into the right place, but here it is:

```javascript
function HideMutedThreads() {
  var userLabels = GmailApp.getUserLabels();
  var search = "is:mute {in:inbox " + userLabels.map(function(label) {
    return "label:"+label.getName();
  }).join(" ") + "}";
  
  Logger.log(search);
  
  var threads = GmailApp.search(search).forEach(function(thread) {
    Logger.log(thread.getFirstMessageSubject());

    userLabels.forEach(function(label) {
      thread.removeLabel(label);
    });
    thread.moveToArchive();
  });
}
```

And here are my filters:

> Matches: -from:{notifications@github.com}
Do this: Skip Inbox, Apply label "Talk"

> Matches: from:notifications@github.com -{cc:{author@noreply.github.com review_requested@noreply.github.com comment@noreply.github.com mention@noreply.github.com assign@noreply.github.com} subject:{storage performance perf server engine mvcc bug release roachtest stress teamcity backport}
Do this: Skip Inbox, Apply label "Firehose", Never send it to Spam, Never mark it as important

[shutting down]: https://gsuiteupdates.googleblog.com/2018/09/inbox-by-gmail-shutdown.html
[here]: https://blog.filippo.io/gmail-bot-with-apps-script-and-typescript/
[Inbox Zero]: https://en.wiktionary.org/wiki/inbox_zero
