COM 274, Linux Kernel Internals: Data Structures for Open Computing
===================================================================
# Project #4 (Bonus): Bring Your Own Sabotage

[Sergei Prokofov's](https://raw.githubusercontent.com/rachmiroff/images/refs/heads/main/auca/com-274/spring-2026/ll-project/proc.jpg) [third](https://github.com/toksaitov/vma-hlist-project) sabotage was a financial disaster. The doubly-linked list became singly-linked and disappeared into the scheduling noise. The buddy bitmap became an extent list and accidentally _outperformed_ the original on sparse filesystems. The VMA tree gained a hash index and ran straight into a Maple Tree from a future kernel that did the job better. Three subsystems, three swings, three misses. The Linux kernel is too big for one villain.

So Sergei is hiring. His next campaign is decentralized: an army of apprentices, each picking their own subsystem, each writing their own patch. You are his next intern. You will pick a subsystem he has not yet tried, identify a data structure that can be plausibly sold as an upgrade and is actually a downgrade, implement it behind a `CONFIG_*` flag, benchmark the damage, and present the case to him in person.

In this work, you will

* Choose your own kernel subsystem and your own data-structure swap.
* Write the cover story, a sales pitch plausible enough to slip past code review.
* Implement the swap behind a `CONFIG_*` flag.
* Design and run a benchmark that exposes the slowdown.
* Defend the work in person with a short 5 minute presentation.

## Bonus, No Submission Portal

This is a bonus project. There is no GitHub Actions check, no automated grader, no GitHub Classroom URL, no commit hash to submit to the LMS. Come to the last class and present your work to the instructor and the other students. If you do not plan to defend your project, please still attend the session to see other projects and support your peers.

## What to Do

Read [Instructions.md](Instructions.md). It contains the rules, the defense format, and three example sabotages.

## Additional Information

### Web Resources

* [Linux Cross Reference](https://elixir.bootlin.com/linux/v6.8/source)
* [virtme-ng Documentation](https://github.com/arighi/virtme-ng)
* [LWN.net Kernel Index](https://lwn.net/Kernel/Index/)
* [Phoronix, Inspirational Linux-related News](https://www.phoronix.com)

### Books

* _Understanding the Linux Kernel, Third Edition by Daniel P. Bovet and Marco Cesati_
* _Linux Kernel Development, Third Edition by Robert Love_
