---
title: Kubernetes is hard
date: 2023-03-26T16:21:00+02:00
hn_link: https://news.ycombinator.com/item?id=35331887
---

Recently, [37signals](https://37signals.com/) unveiled [mrsk](https://github.com/mrsked/mrsk), a tool developed internally to ease with deployments of their services and an accompanying [blog post](https://dev.37signals.com/bringing-our-apps-back-home/) which sparked a lengthy [discussion on Hacker News](https://news.ycombinator.com/item?id=35263285).

## Is Kubernetes hard?

Many commenters in the HN thread argued that Kubernetes is complex and I think they are partially right.

Part of the complexity lies in it's versatility, after all, you can run almost any workload on Kubernetes. That versatility means that resources must be composable. In other words, they must be like Lego, when you can join multiple pieces to build something meaningful. Take containers: in Kubernetes you don't run containers on their own. Kubernetes has a concept of `Pods` - a group of one or more containers tighly coupled. But even with `Pods`, you are quite unlikely to run them on their own. You will usually run them via a `Deployment`, `DaemonSet` or a `StatefulSet`. That's before you've even exposed the service to the outside world.

But that's only part of the story, there's a whole lot of things running in any production-ready cluster that are not required or needed by Kubernetes at all - you could build a minimal Kubernetes cluster and it would work just fine, but you really, _really_ don't want to do that.

## Production is hard

You see, when building a service, one of the many things that you don't want to do is to dissappoint your users. That dissappointment can have multiple forms, but one of the easiest way to do that is to offer an unreliable service. Your service _probably_ doesn't need to be up all the time, you don't even need to target 99.999% uptime, but too much downtime and the service cannot be trusted. User uploads a picture to a service only to find out day later that picture is gone? That will not be remembered fondly.

There's a plethora of reasons why a service may not offer great experience. Maybe you just launched a new product and the service is overwhelmed by signups? That's a good problem to have. More likely, there's a bug that causes panics and the applications crashes. Sometimes it's a configuration issue, other times [a cert expires](https://www.theverge.com/2020/2/3/21120248/microsoft-teams-down-outage-certificate-issue-status).

This brings me to my point: **production is hard**. You need to manage apps, logs, metrics, storage, databases, backups, load balancing, deployments, secrets and hundreds of other things. Even more if you're working with microservices.

You need to do all of these things whether you use Kubernetes or not. With cloud, you get a lot of that for a price and if you're willing to pay that price and accept that part of it comes with a vendor lock-in, you can significantly simplify your stack. That's what we did before Kubernetes was _the cool thing_ and it worked just fine.

Forget the hype, there's a reason why Kubernetes is being adopted by so many companies. It allows dev teams to not worry about all these things; all they must do is to write a simple YAML file. More importantly, teams no longer need to ask DevOps/infra folks to add DNS entry and create a Load Balancer just to expose a service. They can do it on their own, in a declarative manner, _if_ you have an operator do to it.

## Should you use Kubernetes?

With Kubernetes the "burden" of maintenance of the clusters is on DevOps/infra teams. Is it going to work for every company? No. Does it make sense if you have 3 microservices? Maybe. Maybe not. If you reach a certain scale then go for it. Maybe your use case is simple enough that adopting Kubernetes is too costly.

Adopting Kubernetes means that you have a unified dev experience that you can adapt as you grow. Mix and match operators to create powerful automations when you need them. Consider autoscaling: what is the right way to autoscale your services? Should you scale it horizontally, vertically? Maybe the service should scale with open connections? Maybe you don't need autoscaling at all.

I like Kubernetes. I like it a lot. Would I use it for every case? No. Would I consider it for every company? Probably. It gives you so many great things out of the box and you can push it very far. Does that mean you need to run everything on Kubernetes? No.
