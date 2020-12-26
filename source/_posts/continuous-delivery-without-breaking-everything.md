---
title: Continuous Delivery Without Breaking Everything
description: What is CD and, especially, what is it not?
thumbnail: 0_hIc0MCrGilAtZXcO.jpg
date: 2014-11-10 10:16:15
updated: 2020-11-27 12:36:15
tags:
    - DevOps
---

<!-- markdownlint-disable no-inline-html -->
![Image](0_hIc0MCrGilAtZXcO.jpg)<span class="caption">Image: [Science in HD](https://unsplash.com/@scienceinhd)</span>
<!-- markdownlint-enable no-inline-html -->

Let's start with one key fact: Continuous Delivery, though expanded from Continuous Integration, is _not_ simply automating your code _deployment_ pipeline like you automated your build architecture. Continuous Delivery is a shared business process, a contract between different members of product delivery to move software through stages to someone who will benefit, _without blocking it_ or at best _avoiding unreasonably long delays_.

{% blockquote Bill Gates %}
Automation applied to an inefficient operation will magnify the inefficiency.
{% endblockquote %}

Where were we ten years ago? At your organization, it probably looked something like this: long product release cycles leading to long development and testing cycles with lots and lots of value bundled up in massive code drops.

![Image](0.png)

...followed by many smaller point releases to fix all the things that broke when you finally received validation from the real world - not just from internal testing or beta users. I'm sure it was _really fun_ going back over six months of code to find the one change that broke the entire release 🙃.

This was [MindTouch](https://mindtouch.com) prior to 2012. We delivered software packages, available for system administrators to download and patch or upgrade our software running in data centers. Software-as-a-Service (SaaS) delivery became a near-term goal for us, as supporting our customers indirectly through their IT departments was becoming unmaintainable. We wrote the software, so we knew how best to deploy and run it!

However, long six-month development cycles were unsuitable for SaaS delivery and also under-leveraged one of the key business-benefits of SaaS: you can deliver value to your customers immediately.

<!-- markdownlint-disable no-space-in-emphasis -->
Much of our software {% post_link automocking-dependencies 'was not in an ideal state' %} for the "high-speed, always merge to the main branch, the main branch is deployed to customers" style of software delivery. As a result, we approached SaaS development and delivery in a more _cautious_, though not entirely pure _continuous_ model:
<!-- markdownlint-enable no-space-in-emphasis -->

![Image](3.png)
![Image](4.png)

This is our current [GitHub](https://github.com) workflow. Yes, I know: Feature branches are not _continuous_. However, this is a transitional state for a codebase, which was until very recently tested end-to-end within six-month cycles, while its stabilized to be continuously delivered _whenever it needs to be_. For the time being, we've chosen weekly releases as our target. The immediate benefit achieved is a much smaller delta between changes resulting in quicker defect fixes (or rollbacks if necessary).

<!-- markdownlint-disable no-inline-html -->
![Image](1.png)<span class="caption">Rollbacks require that there is a previous feature branch or tag that we can rollback to - seemingly the antithesis of a continuous delivery process</span>
<!-- markdownlint-enable no-inline-html -->

Technically speaking, the long-term goal is to deliver value when it's needed and not let the tools dictate when we _can deliver_.

<!-- markdownlint-disable no-inline-html -->
![Image](2.png)<span class="caption">A single branch that always flows into live production requires confidence that the process will be resilient as possible and can quickly deliver corrections to defects as well as new features or value</span>
<!-- markdownlint-enable no-inline-html -->

Test your Continuous Delivery process with humans first and make sure it works for how your organization needs to reliably deliver software today.
