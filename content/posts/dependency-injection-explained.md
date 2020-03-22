---
title: "Dependency Injection"
subtitle: "Explained"
date: 2020-03-22T17:11:03+01:00
tags: ["DependencyInjection", "Software", "Design"]
draft: true
---

A while ago, I answered several questions on Stackoverflow about Dependency Injection. To get my blog started, I thought to reuse my given answers in this blog post.

DIP means that you program against an abstraction. You invert the kind of a dependency from an implementation to an abstraction.

IOC means that somebody else is responsible for getting the implementation for the given abstraction. Normally the consumer would use the new keyword to get a dependency. With IoC you invert the control, so that the consumer is not responsible for creating the instance anymore.

Dependency Injection and Service Location are a part of Inversion of Control.

Service Location and Dependency Injection is at first for decoupling classes so that they can be easily tested and changed.

When you compare the register and resolve parts of an IoC Container with a Service Locator it seems to be the same.

You can use an IoC Container as a Service Locator, which is considered to be an anti pattern. When you use Service Location you always have to call the Service Locator actively all over your architecture. So you decouple your classes, but on the other hand you couple them all to the Service Locator. Furthermore dependency discovery is more difficult with a Service Locator, because you are hiding dependencies. Whereas with Dependency Injection you make the dependencies “public” by using Constructor Injection.

When you use an IoC Container you use Dependency Injection (Constructor Injection or Property Injection). The IoC Container is now able to resovle the dependency graph by looking at the constructor parameters and create the whole dependency graph. This is called auto-wiring. A Service Locator is not able to auto-wiring dependencies. As I already mentioned, you are not forced to use auto-wiring, you could easily use the IoC container like a Service Locator by simple calling the IoC Container in each class directly, but you should not!