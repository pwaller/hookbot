hookbot
-------

Webhooks. But with outbound connections.

Status: beta. Things may change. We may need your feedback to make it work for you.

What is a web hook?
-------------------

A web hook is simply a HTTP post request made to your server when something happens.
For example, github
[can generate webhooks in response to various actions](https://developer.github.com/webhooks/),
such as when a branch is pushed to.

This is useful for creating a continuous deployment infrastructure, which runs tests
or deploys software when it is updated.

## The problem with web hooks

Webhooks require that the receiver of the hook listens on a port accessible
to the publisher, and that the publisher must be configured to publish to
every listener. In a cloud environment this can be inconvenient, as there may be
many receivers interested in an event, and it may not be desirable to poke holes in a
the firewall for them. In a development environment it may not be easy to listen on
a public TCP interface.

Why use hookbot?
----------------

Hookbot is a server which makes it so that you can listen on a public port for
webhooks in one place.

Any applications wanting to listen to the webhooks can make an outbound websocket
connection to a hookbot server, and the hookbot server broadcasts the messages it
receives to any interested listeners.

How do I use hookbot?
---------------------

Hookbot has two types of URL, `https://uri-specific-token@host/pub/<channel>` and `wss://uri-specific-token@host/sub/<channel>`.

Webhooks are broadcast from a HTTP POST to `/pub/` and received by making a websocket
connection to a `/sub/` URI and reading messages from it.

The authentication is a hmac of a single secret with the URI, so a token used for
publishing is different from the one for listening; tokens are also different
between separate channels. The single secret is only known by the server.

Hookbot is an easy-to-run portable go server. It just needs the `HOOKBOT_KEY`
variable setting, for example like so:

```
$ HOOKBOT_KEY=foo hookbot serve
2015/07/22 11:21:58 Listening on :8080
```

Release binaries [are hosted on github](https://github.com/sensiblecodeio/hookbot/releases).

If you're a SensibleCode employee, you can use https://hookbot.scraperwiki.com.

## How do I listen to events

You can listen to events by making a websocket connection to a `/sub/` URL.

This can be done with [`wscat`](https://github.com/pwaller/wscat/releases), for example:

```
$ wscat wss://token@hookbot.scraperwiki.com/sub/foo/bar
EVENT!
```

If you are writing go code, for convenience a
[`listen` package is included](https://godoc.org/github.com/sensiblecodeio/hookbot/pkg/listen).
This includes a `RetryingWatch` function which returns events.

A brief example is below, or [a more complete example](https://github.com/sensiblecodeio/hanoverd/blob/fcfb97f00a4435eb7d420d75e05ecceb88b27e80/main.go#L322)
can be seen in the [hanoverd](https://github.com/sensiblecodeio/hanoverd/) project.

```golang
	finish := make(chan struct{})
	header := http.Header{}
	events, errs := listen.RetryingWatch("wss://hmac-token@hookbot/sub/github.com/repo/sensiblecodeio/hookbot", header, finish)

	go func() {
		defer close(finish)

		for err := range errs {
			log.Printf("Error in hookbot event stream: %v", err)
		}
	}()

	for payload := range events {
		log.Printf("Signalled via hookbot, content of payload:")
		log.Printf("%s", payload)
	}
```


Generating tokens
-----------------

With the exception of recursive pub/sub, every endpoint (URI) has its own unique
token to access it. Tokens can be generated by running, for example:

```
$ HOOKBOT_KEY=foo hookbot make-tokens --url-base https://hookbot.scraperwiki.com /pub/foo/bar
https://2e1150434ba1d8c33bce7c82ee08b5d9850342c7@hookbot.scraperwiki.com/pub/foo/bar
```

The `HOOKBOT_KEY` should be kept secret and the authentication token is derived
from it using a [HMAC](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code).

A token which is valid for a URL ending with a `/` is valid for any URL
beginning with that prefix up to the `/`. For example a token valid for
`/pub/foo/` is valid for `/pub/foo/bar`, `/pub/foo/qux`, etc - but not `/pub/notfoo`.

Query parameters (e.g, ``/foo?query-param`) and hostnames do not contribute to
the MAC, only the path part of the URI.

If your websockets client does not support HTTP authentication, you can pass the
token as a parameter named "auth":

```
https://hookbot.scraperwiki.com/pub/foo/bar?auth=2e1150434ba1d8c33bce7c82ee08b5d9850342c7
```

Wider scope for publication keys
--------------------------------

If you have a valid key for `/pub/foo/`, you can also use that key
to publish to `/pub/foo/bar/baz` etc. Only `/` terminated substrings (and the
original string itself) are considered when looking for a match; so a valid
key for `/pub/foo` is only valid for that particular topic.

Listening to multiple endpoints
-------------------------------

You can listen to all endpoints below a particular point in the hierarchy.
For this, the URI must end with a "/".

```
hookbot make-tokens /sub/foo/
ws://c0..9f@localhost:8080/sub/foo/
```

You will receive the path the message was published on
(i.e. the topic `foo/bar/baz`) followed by a NUL byte, followed by the message.
(Note the absence of a leading `/pub/` or `/sub/`.)

Unsafe URLs
-----------

In addition to `/pub/` and `/sub/`, there is also `/unsafe/pub/` and `/unsafe/sub/`.
The unsafe pub URL can be published to without supplying an authentication token,
but the `/sub/` URI requires the token. It is so-named because attackers can publish
messages to it. This is only reasonable if the authentication is done through
some other mechanism, such as certificates or message signing.

A client is prevented from "accidentally" listening to `/unsafe/` URLs, which could
be dangerous if unintended by requiring that the client supplies an
[`X-Hookbot-Unsafe-Is-Ok: I understand the security implications`](https://github.com/sensiblecodeio/hookbot/blob/03f7430da914ee6bbebfa264ecddc8b683d52a06/pkg/hookbot/auth.go#L71) header. This prevents clients which have not been designed to connect
to an unsafe endpoint from doing so.

Extra metadata
--------------

Sometimes, critical webhook data is not passed in-band in the POST body.
For example, github passes a signing key as a HTTP header, but applications
receiving the webhook might want to know what the key was, as well as the payload.

For this, `/pub/` URLs can be suffixed with `?extra-metadata=github` which causes
hookbot to [construct a new payload](https://github.com/sensiblecodeio/hookbot/blob/03f7430da914ee6bbebfa264ecddc8b683d52a06/pkg/hookbot/hookbot.go#L351-L356)
carrying the additional information.

Using routers to rebroadcast organization-wide webhooks to specific repositories
--------------------------------------------------------------------------------

If you have many projects and want to have different applications listening,
it is tedious and problematic to configure github for each project.
Instead, we can configure
[GitHub's organization level webhook](https://developer.github.com/v3/orgs/hooks/)
once. The problem then is that every project gets details of every other project's
hooks, which would not be ideal. To avoid this, hookbot can rebroadcast messages
from one channel onto other channels.

For example, let's say `sensiblecodeio/hookbot`'s `master` branch is updated.

The `/unsafe/pub/github.com/org/sensiblecodeio?extra-metadata=github` channel receives
an event. In the payload of the event, the target repository is mentioned, so
a router
([such as this github router](https://github.com/sensiblecodeio/hookbot/blob/03f7430da914ee6bbebfa264ecddc8b683d52a06/pkg/router/github/github.go#L192))
can authenticate and rebroadcast the message to `/sub/github.com/repo/sensiblecodeio/hookbot`.

# License

Hookbot is licensed under a BSD-like license.
