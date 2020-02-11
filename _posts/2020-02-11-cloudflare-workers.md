---
title: 'Getting Started with Cloudflare Workers'
date: '11-02-2020 12:00'
published: true
categories:
  - 'Blog'
  - 'DevOps'
tag:
  - 'Cloudflare'
  - 'Cloud'
---

I have recently had the need to redirect a few subdomains to another domain, due to old links still being used by public individuals.

I wanted a solution that would not require running another web server, so I started looking into a few different options.

1. Cloudflare Page Rules

While Page Rules do want I want, they were either not flexible enough (could only target a single subdomain), and would be relatively expensive for the number I would need.

2. Cloudflare Workers

Cloudflare Workers are pretty cool. They allow you to run JS code at the Cloudflare edge. They are basically functions-as-a-service, which are super fast and customizable.
  
I was able to set up a function in less than 5 minutes that allows me to redirect any subdomain to another domain.
  
The function I am using looks like this:
  
```js
async function handleRequest(request) {
  return Response.redirect(someURLToRedirectTo, code)
}
addEventListener('fetch', async event => {
  event.respondWith(handleRequest(event.request))
})
/**
 * Example Input
 * @param {Request} url where to redirect the response
 * @param {number?=301|302} type permanent or temporary redirect
 */
const someURLToRedirectTo = 'https://homelab.blog'
const code = 302 // Temporary Redirect
```

Basically, it handles requests by sending a 302 Temporary Redirect to my current blog page.

Additionally, the function is super fast. It seems significantly faster than if I were to run a webserver to handle the redirect myself.

It is also using the [routes](https://developers.cloudflare.com/workers/about/routes/) function. You still have to define a DNS name for the URLs you want to redirect, and I currently just CNAME them to the workers.dev hostname of the worker.