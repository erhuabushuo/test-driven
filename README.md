# 测试驱动开发课程

1. 本地运行 - `bundle exec jekyll serve`
1. 构建 - `JEKYLL_ENV=production bundle exec jekyll build`

## Microservices

- Complexity shifts from the inside (code, vertical stack) to the outside (platform, horizontal stack), managing each dependency, which *can* be good if you have a younger team in terms of developers. Junior developers are free to experiment and muck up smaller apps. You must have solid dev ops support though.
- Less coupling, which makes scaling easier
- Flexible - different apps can have different code bases and dependencies
- Can be slower since multiple requests and responses are often required
- Smaller code base, less coupled, solid API design, not having to understand the full system = easier to read code

### Stateful vs stateless services

- Stateful - databases, message queues, service discovery
- Stateless - apps

Stateful containers should not come down. You should limit the number of these since they are hard to scale.

### What code is common amongst all the services?

Generator for-

1. Auth
1. service discovery
1. RESTful routes
1. Unit and Integration test boilerplate
1. Config (via environment variables)
