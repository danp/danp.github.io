---
layout: default
title: My Mastodon setup
date: "2022-11-13"
---

It seems like Twitter may not be around much longer.
Even if does stay around, it's not likely to be a place I want to be anymore.

Like many folks, I decided to dip into the [Fediverse](https://en.wikipedia.org/wiki/Fediverse).
Using [Mastodon](https://joinmastodon.org/) seemed like a good first step.
I set up an account on [mastodon.social](https://mastodon.social/) but quickly realized it was overwhelmed.
There were other servers to join but I wanted to run my own.

My goals:

* have `danp@danp.net` be my handle
* keep danp.net on GitHub pages
* run Mastodon elsewhere (not on danp.net, since it's on GitHub Pages)

The main rub with this is [WebFinger](https://webfinger.net/) which is a big part of Fediverse discovery.

When a handle like `user@example.com` is used, a WebFinger request is made to a URL like:

```
https://example.com/.well-known/webfinger?resource=acct:user@example.com
```

The response is meant to look something like:

```json
{
  "aliases": [
    "https://example.com/@user",
    "https://example.com/users/user"
  ],
  "links": [
    {
      "href": "https://example.com/@user",
      "rel": "http://webfinger.net/rel/profile-page",
      "type": "text/html"
    },
    {
      "href": "https://example.com/users/user",
      "rel": "self",
      "type": "application/activity+json"
    },
    {
      "rel": "http://ostatus.org/schema/1.0/subscribe",
      "template": "https://example.com/authorize_interaction?uri={uri}"
    }
  ],
  "subject": "acct:user@example.com"
}
```

Usually this would change based on the `resource` query parameter.
If I wanted a setup just for my lone user, could I return a static response?

# Setting up Mastodon

First, I got Mastodon set up.
This is not an exhaustive guide to setting up Mastodon,
mostly just bits from my notes along the way.

I saw the [Installing from source](https://docs.joinmastodon.org/admin/install/)
documentation page but it called for installing a bunch of stuff.

After doing some searching for `mastodon docker` I found the project's
[docker-compose](https://github.com/mastodon/mastodon/blob/main/docker-compose.yml)
file so I thought I'd start with that until it didn't work.

That let me mostly skip to [Generating a configuration](https://docs.joinmastodon.org/admin/install/#generating-a-configuration)
with:

``` shell
touch .env.production # so docker stuff runs
docker-compose run -i web bash # builds a bunch
RAILS_ENV=production bundle exec rake mastodon:setup
```

Since I wanted to have my handle be `danp@danp.net` but my instance run at `mastodon.danp.net`
I needed to do the [WEB_DOMAIN wrangling mentioned here](https://docs.joinmastodon.org/admin/config/#web-domain).

The setup process didn't give a way to do that so I nuked the database and redis data after
running the setup process and fixed up the generated config with:

``` shell
LOCAL_DOMAIN=danp.net
WEB_DOMAIN=mastodon.danp.net
```

Then ran `rake db:setup`
[like the setup process does](https://github.com/mastodon/mastodon/blob/87fbd08f74451c18d2fdbef551d5933a78375b63/lib/tasks/mastodon.rake#L437).

With that, I was able to start everything up:

``` shell
docker-compose up -d
```

Next, to get traffic to the instance, I added this to my host's [Caddyfile](https://caddyserver.com/docs/caddyfile):

```
mastodon.danp.net {
        handle /api/v1/streaming* {
                reverse_proxy localhost:4000
        }
        reverse_proxy localhost:3000
}
```

The special bit for the streaming endpoint came later,
after I realized the websockets requests weren't working out.
Once I added it, notifications in the web UI happened instantly!

Then, I added my lone user:

``` shell
RAILS_ENV=production bin/tootctl accounts create danp --email danp@danp.net --confirmed --role Owner
```

Everything seemed to be working!

# But what about WebFinger?

Next, I knew I had to figure out WebFinger requests to danp.net.

I curl'd `https://mastodon.danp.net/.well-known/webfinger?resource=acct:danp@danp.net` and got:

``` json
{
  "aliases": [
    "https://mastodon.danp.net/@danp",
    "https://mastodon.danp.net/users/danp"
  ],
  "links": [
    {
      "href": "https://mastodon.danp.net/@danp",
      "rel": "http://webfinger.net/rel/profile-page",
      "type": "text/html"
    },
    {
      "href": "https://mastodon.danp.net/users/danp",
      "rel": "self",
      "type": "application/activity+json"
    },
    {
      "rel": "http://ostatus.org/schema/1.0/subscribe",
      "template": "https://mastodon.danp.net/authorize_interaction?uri={uri}"
    }
  ],
  "subject": "acct:danp@danp.net"
}
```

So I just [dumped that to a file on my static GitHub Pages repo](https://github.com/danp/danp.github.io/blob/main/static/.well-known/webfinger).

This makes any request to `https://danp.net/.well-known/webfinger` (regardless of query param) return that static response.

Should be fine, right? Everything seemed to be working!

# The octodon.social mystery

I was able to find and follow many folks.
But then I noticed none of my searches for folks on [octodon.social](https://octodon.social/) worked.

This was where having the source came in super handy.

It took a bit of tracing from [the search controller](https://github.com/mastodon/mastodon/blob/87fbd08f74451c18d2fdbef551d5933a78375b63/app/controllers/api/v2/search_controller.rb),
to [the search service](https://github.com/mastodon/mastodon/blob/87fbd08f74451c18d2fdbef551d5933a78375b63/app/services/search_service.rb),
to [the account search service](https://github.com/mastodon/mastodon/blob/87fbd08f74451c18d2fdbef551d5933a78375b63/app/services/account_search_service.rb),
to [the resolve account service](https://github.com/mastodon/mastodon/blob/87fbd08f74451c18d2fdbef551d5933a78375b63/app/services/resolve_account_service.rb),
to [the ActivityPub fetch remote account service](https://github.com/mastodon/mastodon/blob/87fbd08f74451c18d2fdbef551d5933a78375b63/app/services/activitypub/fetch_remote_account_service.rb) and its base class,
but I was able to boil it down to:

``` ruby
actor_url = "https://octodon.social/users/commaok"
x = ActivityPub::FetchRemoteAccountService.new
req = x.build_request(actor_url, Account.representative)
req.perform {|r| p r.body_with_limit }
```

Which let me see that the `Public key not found for key https://mastodon.danp.net/actor#main-key`
error from around
[here](https://github.com/mastodon/mastodon/blob/87fbd08f74451c18d2fdbef551d5933a78375b63/app/controllers/concerns/signature_verification.rb#L91) was being returned.

This indicates octodon.social is running in ["secure mode"](https://docs.joinmastodon.org/admin/config/#authorized_fetch)
which requires signatures which can be verified.

In the code above, `Account.representative` is an instance-level account.
When some requests are made on behalf of the instance,
those requests [are signed](https://docs.joinmastodon.org/spec/security/)
using a key from that account.

The key URI mentioned in the error above is from that account.

When octodon.social saw this key URI, it:

1. requested the data at `https://mastodon.danp.net/actor#main-key`
2. saw `preferredUsername: danp.net` in the body
3. made a WebFinger query for `danp.net@mastodon.danp.net`
4. got a response with `subject: acct:danp.net@danp.net` (no `mastodon` in the hostname)
5. made another WebFinger query for `danp.net@danp.net`

This hit my static WebFinger response and broke.

I fixed this with a little patch to the Account model:

``` diff
diff --git a/app/models/account.rb b/app/models/account.rb
index 3647b8225..72a89dee9 100644
--- a/app/models/account.rb
+++ b/app/models/account.rb
@@ -180,7 +180,11 @@ class Account < ApplicationRecord
   end

   def local_username_and_domain
-    "#{username}@#{Rails.configuration.x.local_domain}"
+    domain = Rails.configuration.x.local_domain
+    if instance_actor? && domain != Rails.configuration.x.web_domain
+      domain = Rails.configuration.x.web_domain
+    end
+    "#{username}@#{domain}"
   end

   def local_followers_count
```

Now the WebFinger query for `danp.net@mastodon.danp.net` in step 3 above returns
a response with `subject: acct:danp.net@mastodon.danp.net` and everything works out.

I wish WebFinger requests could be done in a way that didn't require this.
For example, if it requested paths like this:

``` diff
/.well-known/webfinger/danp.net/danp
/.well-known/webfinger/danp.net/danp.net
```

Then I could place two static files and return the right thing.

Since it uses a query param this isn't possible without some HTTP server intelligence.

# Conclusion

It seems like I've achieved my goals.
My handle is `danp@danp.net`,
danp.net is still on GitHub Pages,
and I have my own instance.

I learned quite a bit about WebFinger and the Mastodon source along the way, too.

If I were doing it again, I might go for s.danp.net ("s" for "social")
or similar instead of trying to use just danp.net.

I also might not have "mastodon" in any hostname.
I realize now that Mastodon is just one way to access the Fediverse.

Still, it's nice to have a setup that could very well outlast Twitter.
