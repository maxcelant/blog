---
title: Go Proxy Project - 12/26/25
---

#### Go Proxy Project — 12/26/25
I'm working on my own dynamically configurable reverse HTTP proxy in Go. It's been a passion project that I've been sitting on for awhile and finally decided to give more attention to. This blog will serve to give updates on the progress of the project and some of the challenges that I'm facing.

At a high level, the idea is to allow the user to send the proxy configuration updates which consist of apps with routes and clusters. Allowing them (or some sort of controller), to dynamically update upstreams.

I've decided to go with an decouple API design. So `App`, `Route`, and `Cluster` live as independent objects that are tied and validated separately. My reasoning is two-fold: I wanted to simplify the API from a users perspective and I wanted to allow a more modular proxy configuration.

The registry got a very large face lift. It now has independent thread-safe caches for each kind, as well as a hostname reverse indexer so that we can quickly locate an app by hostname. The current drawback is the reverse index is 1:1, which means you can't have multiple apps with the same hostname, but maybe that's a good thing? We can loop back some other time.

I fully built out a relationship graph data structure to encourage the independence of the objects and their mappings. I'm still working through what the best way to validate those objects during the request flow. Ensuring relationships on every single request seems burdensome and computationally slow when it's already been confirmed multiple times. It could be worth using a faster caching strategy between config updates.

#### Future Plans
- Continue to flesh out the request flow; Primarily the relationship mapping strategy.
- Start working on the configuration engine, ensuring consistency in routing while also having dynamic updates to the config.
