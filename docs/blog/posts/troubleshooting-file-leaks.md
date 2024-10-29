---
date: 2024-10-17
authors:
    - pablobaxter
draft: true
---

# Detecting File Leak in the Kotlin Daemon

File handle leaks are notoriously difficult to debug, so much so that most of the “fixes” for them are “increase the file descriptor limit”. However, this is not a fix. This is as close as you can get to covering your ears and closing your eyes, then screaming “I DON’T HEAR YOU” to a bug. The file handle leak will still exist, but now it’s ignored.

<figure markdown="span">
  ![img](../../assets/gifs/file-leak.gif){ loading=lazy }
  <figcaption>Seriously, it's exactly this</figcaption>
</figure>

In this case study, I decided to take the approach of finding out why the leak was occurring, and finding the best fix for it.

<!-- more -->

## Signs of a File Leak

The hardest part of debugging an issue is noticing it to begin with and file handle leaks are very hard to notice until an error is thrown, and even then it may not be noticed since the error could be handled by some logic that changes the exception cause or swallows it altogether. So how did I notice the one in the Kotlin daemon? Dumb luck. I had been tracking another file leak in the Gradle daemon, and just got curious about how many files remained open in the Kotlin daemon after compilation. There are some signs that your application may have a file leak, but even these signs are easily missed. A few common signs are:

* Errors relating to files unable to be created
* Any error about a file being deleted (especially on Windows)
* Unable to open files for any reason
* Files having garbage data written or being corrupted

These issues don't always appear on every run, and could be deemed intermittent issues as well. 

## Noticing the File Leak

Following the steps Paul Klauser took to find a memory leak in the Kotlin daemon, I followed the same steps, but attached a file leak detector to it.


> So what's the typical fix?

Well, if you use that Google tool we've all come to rely on, you'll get something along the lines of:

 * "Just increase the file descriptor limit."
 * "Your OS isn't setup properly."
 * "Easy, just use <`insert random shell command here`>."

And all these suggestions boil down to calling `ulimit -n 1000000`.

> Sounds like a solid fix. What's wrong with it?

Well, it's not a "fix". It's more like sweeping the issue under the rug.



File leaks are difficult to trace, though the fixes for them are typically easy. Why are they difficult to trace? It's not always clear where the leak is occurring, and in some cases it's difficult to find that there even is a leak.