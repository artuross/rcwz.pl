---
title: 'Kubernetes is hard'
date: 2023-03-26T16:21:00+02:00
draft: true
---

Recently, [37signals](https://37signals.com/) unveiled [mrsk](https://github.com/mrsked/mrsk), a tool developed internally to ease with deployments of their services and an accompanying [blog post](https://dev.37signals.com/bringing-our-apps-back-home/) which sparked a lengthy [discussion on Hacker News](https://news.ycombinator.com/item?id=35263285).

## Is Kubernetes hard?

Many commenters in the HN thread argued that Kubernetes is complex and I think they are partially right.

Part of the complexity lies in it's versatility: you can run almost any workload on Kubernetes. Take containers: in Kubernetes you don't run containers on their own. Kubernetes has a concept of `Pods` - a group of one or more containers tighly coupled. But even with `Pods`, you are quite unlikely to run them on their own. You will usually run them via a `Deployment`, `DaemonSet` or a `StatefulSet`. That's before you've even exposed the service to the outside world. That versatility means that resources must be composable. In other words, they must be like Lego, when you can join multiple pieces to build something meaningful.

But that's only part of the story, there's a whole lot of things running in any production-ready cluster that are not required or needed by Kubernetes at all - you could build a minimal Kubernetes cluster and it would work just fine, but you really, _really_ don't want to do that.

## Production is hard

You see, when building a service, one of the many things that you don't want to do is to dissappoint your users. That dissappointment can have multiple forms, but one of the easiest way to do that is to offer an unreliable service. Your service _probably_ doesn't need to be up all the time, you don't even need to target 99.999% uptime, but too much downtime and the service cannot be trusted. User uploads a picture to a service only to find out day later that picture is gone? That will not be remembered fondly.

There's a plethora of reasons why a service may not offer great experience. Maybe you just launched a new product and the service is overwhelmed by signups? That's a good problem to have. More likely, there's a bug that causes panics and the applications crashes. Sometimes it's a configuration issue, other times [a cert expires](https://www.theverge.com/2020/2/3/21120248/microsoft-teams-down-outage-certificate-issue-status).

This brings me to my point: **production is hard**. You need to manage applications, logs, metrics, storage, databases, backups, load balancing, deployments, secrets and hundreds of other things. Even more if you're working with microservices.

You need to do all of these things whether you use Kubernetes or not. With cloud, you get a lot of that for a price and if you're willing to pay that price and accept that part of it comes with a vendor lock-in, you can significantly simplify your stack. But even the best cloud services don't offer what Kubernetes offers: a choice.

With Kubernetes you can evolve your stack. If you're small and don't need autoscaling, you don't need to have HPA. Maybe some of your apps are better scaled vertically? Use VPA. Perhaps you need some advanced scaling capabilities? Use KEDA. Or bring your own.

## Should you use Kubernetes?

No. Just because Kubernetes works for one company doesn't mean it will work for you too. Kubernetes is hard, but that's because production and microservices and software is hard. If you can simplify your workflow with existing solutions, do it, because it will save you time and money. Consider the tradeoffs and make decisions based on that.

If existing solutions are too complex or don't solve your problem, develop your own, just remember there's a cost to it too: now you need to maintain it, build documentation for internal teams, train new people to use it.

I like Kubernetes. I like it a lot. Would I use it for every case? No. Would I consider it for every case? Certainly yes. It gives you so many great things out of the box and you can push it very far. Does that mean you need to run everything on Kubernetes? No.
