---
layout: post
title: Handling cookies with Fetch's credentials
slug: fetch-credentials
tags: ['fetch', 'javascript', 'async', 'cookies', 'express', 'node']
status: draft
blockRobots: true
---

Fetch has a `credentials` option that can be used to send credentials to servers. It has three possible values — `omit`, `same-origin`, and `include`.

- What does each of these three values do?
- Does Fetch send cookies to specific servers only?
- Does Fetch send specific cookies only?

I couldn't find answers to these questions online so I began experimenting. I want to document my findings and experiments for people who have the same questions.

<!-- more -->

## Summary of my findings <!-- omit in toc -->

Here's a quick summary of what I found:

- If you set `credentials` to `same-origin`: Fetch will send 1st party cookies to its own server. It will not send cookies to other domains or subdomains.
- If you set `credentials` to `include`: Fetch will continue to send 1st party cookies to its own server. It will also send 3rd party cookies set by a specific domain that domain's server.
- `Access-Control-Allow-Credentials` is not required to send 3rd party cookies between domains and subdomains. It is only required to send 3rd party cookies between domains.

If you don't understand the difference between 1st party and 3rd party servers — including how to set them — consider reading [my article on the sameSite attribute](/blog/samesite-cookies). where I dive deeper into this topic.

Note: Fetch always sends `Authorization` headers if you include it (assuming `Access-Control-Allowed-Headers` contains `Authorization`). The `credentials` value doesn't affect whether Fetch sends authorization headers ([unlike what is mentioned on MDN](https://github.com/mdn/content/issues/13063#issuecomment-1046550394)).

## Sites vs Origins<!-- omit in toc -->

We have to be careful about the difference between [sites](https://developer.mozilla.org/en-US/docs/Glossary/Site) and [origins](https://developer.mozilla.org/en-US/docs/Glossary/Origin) when we work with cookies. Cookies are set across sites — which can be defined by registrable domain names.

- Sites are only defined by domain names. Subdomains are considered to be the same site.
- Origins are defined with schemes (`http` or `https`), domains, and ports. Changing any of these values is considered a change in origin.

We are working with sites when we're testing Fetch's `credentials` property. Take note!

## Github Repository<!-- omit in toc -->

I included a [Github repository](https://github.com/zellwk/demos/tree/main/fetch-**credentials**) so you can test the findings below. I'll explain the steps to use the Github repository in each of the tests.

If you open up the Github repository, you will find three folders — Host, Subdomain, and External.

- Host serves a website at a domain — used for both tests
- Subdomain serves a subdomain of the Host — only used for subdomain test
- External is the external website — only used for cross-site test

Here are the consistent things you will find in all three folders

- You will find a link pointing to `/set-cookies`. Use this to set cookies for each of these sites.
- You will find an `iframe`. This sets cookies from the other domains for you. You can remove this, but you'll have to set the cookies from those domains manually.
- You will find some buttons to trigger a Fetch request.

<figure role="figure">
  <img src="/images/2022/fetch-credentials/demo-picture.png" alt="" loading="lazy">
</figure>

For `/set-cookies`, we set three different cookie — `None`, `Lax`, and `Strict`. Each folder sets the cookies for their respective folders so we know which cookies came from where.

The code for `/set-cookies` looks like this:

```javascript
// In Host
app.use('/set-cookies', async (req, res) => {
  res.cookie('None', 'Host', { sameSite: 'none', secure: true })
  res.cookie('Lax', 'Host', { sameSite: 'lax' })
  res.cookie('Strict', 'Host', { sameSite: 'strict' })
  res.send('Host cookies set.')
})

// In Subdomain
app.use('/set-cookies', async (req, res) => {
  res.cookie('None', 'Subdomain', { sameSite: 'none', secure: true })
  res.cookie('Lax', 'Subdomain', { sameSite: 'lax' })
  res.cookie('Strict', 'Subdomain', { sameSite: 'strict' })
  res.send('Subdomain cookies set.')
})

// In External
app.use('/set-cookies', async (req, res) => {
  res.cookie('None', 'External', { sameSite: 'none', secure: true })
  res.cookie('Lax', 'External', { sameSite: 'lax' })
  res.cookie('Strict', 'External', { sameSite: 'strict' })
  res.send('External cookies set.')
})
```

## Testing how Fetch Credentials work <!-- omit in toc -->

We'll go into each of these tests next and I'll explain what to look out for.

- [Testing credentials between domain and subdomain](#testing-credentials-between-domain-and-subdomain)
- [Testing credentials between two domains](#testing-credentials-between-two-domains)

### Testing credentials between domain and subdomain

To test `credentials` between a domain and a subdomain, we have to serve them up. The easiest way to do it is through `lvh.me`. (I don't know what `lvh.me` stands for, but it's a godsend for testing subdomains on localhost).

Here's the basic idea:

- We serve Host on `localhost:3000`
- We serve Subdomain on `localhost:4000`
- We can navigate to `lvh.me:3000` to view Host.
- We can navigate to `subdomain.lvh.me:400` to view Subdomain
- We send a request to `lvh.me:3000` to reach the Host
- We send a request to `subdomain.lvh.me:4000` to reach Subdomain

You probably have figured out by now that `lvh.me` maps any domain (or subdomain) back to localhost. 😉

Here are my findings.

**Fetch from Domain with `credentials: include`**

- To Domain's server: Domain's `strict` and `lax` cookies got sent to the server
- To Subdomain's server: Subdomain's `strict` and `lax` cookies are sent to the server

**Fetch from Subdomain with `credentials: include`**

- To Subdomain's server: Subdomain's `strict` and `lax` cookies are sent to the server
- To Domain's server: Domain's `strict` and `lax` cookies are sent to the server

**Fetch from Domain with `credentials: same-origin`**

- To Domain's server: Domain's `strict` and `lax` cookies sent to the server
- To Subdomain's server: No cookies sent to the server

**Fetch from Subdomain to Domain** with `credentials: same-origin`

- To Subdomain's server: Subdomain's `strict` and `lax` cookies are sent to the server
- To Domain's server: No cookies are sent to the server

No `none` cookies are sent in these tests because we're using a `http` scheme with `lvh.me`. You can, if you want to, use a `https` scheme by [configuring your server to use SSL](/blog/serving-https-locally-with-node).

I'm working on a video of the actual tests since it's really hard to explain this in words.

### Testing credentials between two domains

We need to use two domain addresses if we want to test how `credentials` work across two domains.

- `localhost` is considered one domain
- We need another served with another ip address

The simplest way is to serve a domain with your LAN IP address. I managed to figure out my LAN IP address with [http-server](https://www.npmjs.com/package/http-server) on your computer.

```shell
npm install -g http-server
```

```shell
http-server
```

<figure role="figure">
  <img src="/images/2022/fetch-credentials/http-server.png" alt="" loading="lazy">
</figure>

You can pick any value from this list except for `127.0.0.1` because `127.0.0.1` is synonymous with `localhost` in web development. I picked `192.0.2.2` for my tests.

There's a complication here: We want to test whether `credentials` can send cookies across different websites, but the only type of cookies that can be set from a server to another website are cookies with the `sameSite` attribute set to `none`. These cookies also require a `secure` attribute.

You can find out more about the sameSite attribute [here](/blog/samesite-cookies).

Long story short: We need `https` to test whether cookies can be sent across two different domains on a local environment.

I chose to do this:

- Serve Host on `http` — because `localhost` is considered secure in web development.
- Serve `192.0.2.2` with `https` — because we need `https` to test cookies across different sites.

These are configured in both Host and External. If you run `npm run server` on both of these folders, you should see the following logs:

- Host will be served with `localhost:3000`
- External can be reached at `192.0.2.2`. You have to replace this IP address with one that you got from `http-server`. (I didn't include the dynamic IP address automatically in the logs because I don't know how to do it yet).

<figure role="figure">
  <img src="/images/2022/fetch-credentials/run-server.png" alt="" loading="lazy">
</figure>

Note: You need to create your own SSL certs and place them in the `certs` folder for the `https` scheme to work. I wrote about how to do this [another article](/blog/serving-https-locally-with-node), but the basic idea is to use [mkcert](https://github.com/FiloSottile/mkcert).

One more thing. The `Access-Control-Allow-Credentials` is required for any cross-site requests to work, so we need to set this headers on both Host and External.

```js
// CORS headers on Host
app.use((req, res, next) => {
  res.setHeader('Access-Control-Allow-Origin', 'https://192.0.2.2')
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST')
  res.setHeader('Access-Control-Allow-Credentials', true)
  next()
})

// CORS headers on External
app.use((req, res, next) => {
  res.setHeader('Access-Control-Allow-Origin', 'http://localhost:3000')
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST')
  res.setHeader('Access-Control-Allow-Credentials', true)
  next()
})
```

Here are my findings:

**Fetch from Host with `credentials: include`**

- To Host's server: Host's `strict` and `lax`, and `none` cookies are sent to the server
- To External's server: External's `none` cookies are sent to the server

**Fetch from External with `credentials: include`**

- To External's server: External's `strict` and `lax`, and `none` cookies are sent to the server
- To Host's server: Host's `none` cookies are sent to the server

**Fetch from Host with `credentials: same-origin`**

- To Host's server: Host's `strict` and `lax`, and `none` cookies are sent to the server.
- To External's server: No cookies are sent to the server.

**Fetch from External with `credentials: same-origin`**

- To External's server: External's `strict` and `lax`, and `none` cookies are sent to the server.
- To Host's server: No cookies are sent to the server.

I'm working on a video of the actual tests since it's really hard to explain this in words.

That's it! I hope this sheds some light for people who're researching Fetch's `credentials` option.
