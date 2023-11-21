# Subscriptions 应用设计

## 功能概括

负责管理资源的集合。对于新来的用户会默认有一个 `Default Subscription`，这个 Subscription 会包含所有的资源。用户可以创建新的 Subscription，然后将资源添加到 Subscription 中。用户可以在不同的 Subscription 中添加相同的资源，但是一个资源只能属于一个 Subscription。

Subscription 是为了解决：假如没有 Subscription，那么资源是属于 App 的。而用户将很难看到自己的资源。对于用户的直觉来讲，资源是自己的，因此我们将诞生一个中间层：Subscription：也就是资源的集合。用户可以创建多个 Subscription，然后将资源添加到 Subscription 中。这样用户就可以将资源分门别类的管理了。而 App 本身对于 Subscription 来讲，也具有一定的权限。例如 App 可以增删改查 Subscription 中的资源。

## 数据库

Subscriptions 的数据库中只下面几个表：

* Subscriptions，用于存储
