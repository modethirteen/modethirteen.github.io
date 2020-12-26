---
title: Impact Mapping with Graphviz
description: I took a collaborative product planning technique and of course had to formalize it with code...
thumbnail: gaming_example.png
date: 2019-10-16 13:12:10
updated: 2020-11-30 21:45:00
tags:
    - Product Management
    - Programming
---

<!-- markdownlint-disable no-inline-html -->
![Image](0_e8YPUHOfr9xyFlkO.jpg)<span class="caption">Image: [Patrick Perkins](https://unsplash.com/@stay_in_touch)</span>
<!-- markdownlint-enable no-inline-html -->

So I've turned on _terminator vision_ lately while I try to locate the absolute unicorn of opportunities to prioritize in our product backlog. I've turned this into a borderline obsession now, after reading [Kathy Sierra's Badass: Making Users Awesome](https://www.goodreads.com/book/show/24737268-badass). What is the absolutely smallest improvement that can have the greatest impact?

No, really, WHAT IS IT?!

A colleague of mine recently turned me onto a process that potentially gets me closer to that answer: [Impact Mapping](https://www.impactmapping.org). I'll let the creator, [Gojko Adzic](https://gojko.net/about), explain the process in detail [in his highly recommended book](https://www.impactmapping.org/book.html). The key factor here is that the collaborative process involves _building a graph_ or a _tree_ from which the smallest yet most impactful items can be recognized. In other words, you can visualize what is valuable and very importantly, things that are _not_. Suddenly, everyone sees the loudest and most highly paid individual's "must-have" idea orphaned on a dead tree branch with wilting leaves. The concept of graphing value as dependencies (because I live for {% post_link 'automocking-dependencies' 'scalable dependency management' %}) is highly appealing to the engineer in me since my IDE these days is Google Slides 🙄.

Without getting too deep into the [methodology of drawing impact maps](https://www.impactmapping.org/drawing.html), let's state that the map starts with a measurable product or company objective. The next level down includes the _actors_: the direct practitioners or indirect influencers that affect this goal, what actions they perform or impact that is created through their behavior, and what we need to deliver to enable (or in some cases secure against) those impacts.

Great - here is a possible impact map (using an example from the book), with success measurements, in [Graphviz-compatible](https://graphviz.org) `dot` format:

```js
graph {

    // draw the map from right to left
    graph [rankdir=RL, nodesep=.5, ranksep=.5];

    // Measureable Objective(s)
    "More Players" [style=filled, fillcolor=chartreuse]
    subgraph cluster_Measurements {
        node [ shape=record ]
        "More Players" {
            "What: Active monthly count"
            "Where: Game database"
            "Current: 350k"
            "Target: 800k"
            "Stretch Goal: 1M"
        }
    }

    // Associate actors with objectives
    subgraph cluster_Actors {
        node [style=filled, fillcolor=cyan]
        "More Players" -- {
            "Players"
            "Internal"
            "Advertisers"
        }
    }

    // What are all the impacts that an advertiser can make?
    subgraph AdvertiserImpacts {
        node [style=filled, fillcolor=cornflowerblue]
        "Advertisers" -- {
            "Publishing our banners"
            "Bulk invitations"
        }
    }

    // What are all the impacts that internal staff can make?
    subgraph AdvertiserImpacts {
        node [style=filled, fillcolor=brown1]
        "Internal" -- {
            "Organize PR event"
            "Engaging our network"
        }
    }

    // What are all the impacts that a player can take?
    subgraph PlayerImpacts {
        node [style=filled, fillcolor=deepskyblue]
        "Players" -- {
            "Posting"
            "Recommending"
            "Inviting friends"
        }
    }

    // Which deliverables exist to organize events?
    subgraph AdvertiserOrganizeEventOpportunities {
        "Organize PR event" -- "Invites"
    }

    // Which deliverables impact the player posting experience?
    subgraph PlayerPostingOpportunities {
        "Posting" -- "Content to post about"
        "Content to post about" -- {
            "Levels"
            "Achievements"
            "Weekly competitions"
            "Tournament ending"
        }
    }

    // Which deliverables impact the player invitation experience?
    subgraph PlayerInviteFriendsOpportunities {
        "Inviting friends" -- {
            "Semi-automated invites"
            "Incentives"
            "Personalization"
            "More compelling product"
            "Viral content"
        }
        "Incentives" -- {
            "Chips"
            "Recognition for inviting lots of friends"
        }
        "Personalization" -- {
            "My tournaments"
            "My table"
            "My events"
        }
        "More compelling product" -- {
            "Rebranding games"
            "Website optimized for new users"
        }
    }
}
```

...and here is the rendered graph. The `dot` file can be checked into a project repository as source code, to be updated if new information is received or requirements change.

![Image](graph_lr.png)

Obviously, more work can go into arranging the components for a more pleasing layout, such as the example provided by the author:

<!-- markdownlint-disable no-inline-html -->
![Image](gaming_example.png)<span class="caption">Image: [https://impactmapping.org](https://www.impactmapping.org/example.html)</span>
<!-- markdownlint-enable no-inline-html -->

...but I hope to continue exploring this Graphviz use case to further the application of structured data visualization to product management tooling.
