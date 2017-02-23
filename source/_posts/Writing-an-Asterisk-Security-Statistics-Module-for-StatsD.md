---
title: Writing an Asterisk Security Statistics Module for StatsD
date: 2014-08-12 20:59:57
tags:
    - Asterisk
    - Asterisk 12
    - Tutorial
    - Security
    - StatsD
---

### Houston, We Have a Problem

> "21055537: the number of failed authentication attempts against a public [#Asterisk](https://twitter.com/hashtag/Asterisk?src=hash) 12 server since January 11th" - [@joshnet](https://twitter.com/joshnet)
> 
> — Matt Jordan (@mattcjordan) [July 16, 2014](https://twitter.com/mattcjordan/statuses/489200481160663040)

Yeah... that's not cool.

When [Josh](http://www.joshua-colp.com/) told me how many failed authentication attempts his public Asterisk 12 server was getting, I wouldn't say I was surprised: it is, after all, something of the wild west still on ye olde internet. I enjoy the hilarity of having fraud-bots break themselves on a tide of 401s. Seeing the total number of perturbed electrons is a laugh. But you know what's more fun? Real-time stats.

I wondered: would it be possible to get information about the number of failed authentication attempts in real-time? If we included the source IP address, would that yield any interesting information?

Statistics are fun, folks. And, plus, it is for security. I can justify spending time on that. These people are clearly bad. Our statistics will serve a higher purpose. This is for a good cause.

For great justice!

{% asset_img zerowing.png !!!ZeroWing!!! %}

### Stasis: Enter the message bus

We're going to start off using our [sample module](https://github.com/matt-jordan/asterisk-modules/blob/master/sample_module/res_sample_module.c). In Asterisk 12 and later versions, the [Stasis message bus](http://doxygen.asterisk.org/trunk/stasis.html) will publish notifications about security events to subscribed listeners. We can use the message bus to get notified any time an authentication fails.

### Getting Stasis moving

I'm making the assumption that we've copied our `res_sample_module.c` file and named it `res_auth_stats.c`. Again, feel free to name it whatever you like.

1. First, let's get the right headers pulled into our module. We'll probably want the main header for Stasis, `asterisk/stasis.h`. Stasis has a concept called a [message router](http://doxygen.asterisk.org/trunk/d4/d1b/structstasis__message__router.html), which simplifies a lot of boiler plate code that can grow when you have to handle multiple message types that are published on a single topic. However, in our case, we only have a single message type, [`ast_security_event_type`](http://doxygen.asterisk.org/trunk/d0/d29/include_2asterisk_2security__events_8h.html#88d718027e6e4abe11dcfe17bd267377), that will be published to the topic we'll subscribe to. As such, it really is overkill for this application, so we'll just deal with managing the subscription ourselves. Said Stasis topic for security events is defined in `asterisk/security_events.h`, so let's add that as well. Finally, the payload in the message that will be delivered to us is encoded in JSON, so we'll need to add `asterisk/json.h` too.
```
#include "asterisk/module.h"
#include "asterisk/stasis.h"
#include "asterisk/security_events.h"
#include "asterisk/json.h"
```

2. In `load_module`, we'll subscribe to the [`ast_security_topic`](http://doxygen.asterisk.org/trunk/d8/d30/main_2security__events_8c.html#0db46f92449a8f63b954d776ebead63a) and tell it to call `handle_security_event` when the topic receives a message. The third parameter to the subscription function let's us pass an object to the callback function whenever it is called; we don't need it, so we'll just pass it `NULL`. We'll want to keep that subscription, so at the top of our module, we'll declare a `static struct stasis_subscription *`.
```
/*! Our Stasis subscription to the security topic */
static struct stasis_subscription *sub;
     
...

static int load_module(void)
{
    sub = stasis_subscribe(ast_security_topic(), handle_security_event, NULL);
    if (!sub) {
        return AST_MODULE_LOAD_FAILURE;
    }
    
    return AST_MODULE_LOAD_SUCCESS;
}
```

3. Since we're subscribing to a Stasis topic when the module is loaded, we also need to unsubcribe when the module is unloaded. To do that, we can call [`stasis_unsubscribe_and_join`](http://doxygen.asterisk.org/trunk/dd/d79/stasis_8h.html#221d05010106dcde0cc2e2b43c1a8b67) - the join implying that the unsubscribe will block until all current messages being published to our subscription have been delivered. This is important, as [`unsubscribing`](http://doxygen.asterisk.org/trunk/dd/d79/stasis_8h.html#3498301077e2005383f0d261c471388b) does not prevent in-flight messages from being delievered; since our module is unloading, this is likely to have "unhappy" effects.
```
static int unload_module(void)
{
    stasis_unsubscribe_and_join(sub);
    sub = NULL;
    
    return 0;
}
```

4. Now, we're ready to implement the handler. Let's get the method defined:
```
static void handle_security_event(void *data, struct stasis_subscription *sub,
                                  struct stasis_message *message)
{
     
}
```
    A Stasis message handler takes in three parameters:

    - A `void *` to a piece of data. If we had passed an `ao2` object when we called `stasis_subscribe`, this would be pointing to our object. Since we passed `NULL`, this will be `NULL` every time our function is invoked.

    - A pointer to the `stasis_subscription` that caused this function to be called. You can have a single handler handle multiple subscriptions, and you can also cancel your subscription in the callback. For our purposes, we're (a) going to always be subscribed to the topic as long as the module is loaded, and (b) we are only subscribed to a single topic. So we won't worry about this parameter.

    - A pointer to our message. All messages published over Stasis are an instance of `struct stasis_message`, which is an opaque object. It's up to us to determine if we want to handle that message or not.

5. Let's add some basic defensive checking in here. A topic can have many messages types published to it; of these, we know we only care about `ast_security_event_type`. Let's ignore all the others:
```
static void handle_security_event(void *data, struct stasis_subscription *sub,
                                  struct stasis_message *message)
{
    if (stasis_message_type(message) != ast_security_event_type()) {
        return;
    }
    
}
```

6. Now that we know that our message type is `ast_security_event_type`, we can safely extract the message payload. The [`stasis_message_payload`](http://doxygen.asterisk.org/trunk/dd/d79/stasis_8h.html#f6081f17431b81bc1a010b680517491d) function extracts whatever payload was passed along with the `struct stasis_message` as a `void *`. It is incredibly important to note that by convention, message payloads passed with Stasis messages are immutable. You must not change them. Why is this the case?
    A Stasis message that is published to a topic is delivered to all of that topic's subscribers. There could be many modules that are intrested in security information. When designing Stasis, we had two options:

    - (1) Do a deep copy of the message payload for each message that is delivered. This would incur a pretty steep penalty on all consumers of Stasis, even if they did not need to modify the message data. Publishers would also have to implement a copy callback for each message payload.

    - (2) Pretend that the message payload is immutable and can't be modified (this is C after all, if you want to shoot yourself in the foot, you're more than welcome to). If a subscriber needs to modify the message data, it has to copy the payload itself.

    For performance reasons, we chose option #2. In practice, this has worked out well: many subscribers don't need to change the message payloads; they merely need to know that something occurred.

    Anyway, the code:
```
    struct ast_json_payload *payload;
        
    if (stasis_message_type(message) != ast_security_event_type()) {
        return;
    }
        
    payload = stasis_message_data(message);
    if (!payload || !payload->json) {
        return;
    }
```
    Note that our payload is of type `struct ast_json_payload`, which is a thin ao2 wrapper around a `struct ast_json` object. Just for safety's sake, we make sure that both the wrapper and the underlying JSON object aren't `NULL` before manipulating them.

7. Now that we have our payload, let's print it out. The JSON wrapper API provides a handy way of doing this via `ast_json_dump_string_format`. This will give us an idea of what exactly is in the payload of a security event:
```
static void handle_security_event(void *data, struct stasis_subscription *sub,
                                  struct stasis_message *message)
{
    struct ast_json_payload *payload;
    char *str_json;
    
    if (stasis_message_type(message) != ast_security_event_type()) {
        return;
    }
    
    payload = stasis_message_data(message);
    if (!payload || !payload->json) {
        return;
    }
    
    str_json = ast_json_dump_string_format(payload->json, AST_JSON_PRETTY);
    if (str_json) {
         ast_log(LOG_NOTICE, "Security! %s\n", str_json);
    }
    ast_json_free(str_json);
}
```

### Stasis in action

Let's make a security event. While there's plenty of ways to generate a security event in Asterisk, one of the easiest is to just fail an AMI login. You're more than welcome to do any failed (or even successful) login to test out the module, just make sure it is through a channel driver/module that actually emits security events!

#### AMI Demo

Build and install the module, then:

```
$ telnet 127.0.0.1 5038
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
Asterisk Call Manager/2.4.0
Action: Login
Username: i_am_not_a_user
Secret: nope
    
Response: Error
Message: Authentication failed
     
Connection closed by foreign host.
```

In Asterisk, we should see something like the following:

```
*CLI> [Aug  3 16:00:51] NOTICE[12019]: manager.c:2959 authenticate: 127.0.0.1 tried to authenticate with nonexistent user 'i_am_not_a_user'
[Aug  3 16:00:51] NOTICE[12019]: manager.c:2996 authenticate: 127.0.0.1 failed to authenticate as 'i_am_not_a_user'
[Aug  3 16:00:51] NOTICE[12008]: res_auth_stats.c:60 handle_security_event: Security! {
    "SecurityEvent": 1,
    "EventVersion": "1",
    "EventTV": "2014-08-03T16:00:51.052-0500",
    "Service": "AMI",
    "Severity": "Error",
    "AccountID": "i_am_not_a_user",
    "SessionID": "0x7f50ae8aebd0",
    "LocalAddress": "IPV4/TCP/0.0.0.0/5038",
    "RemoteAddress": "IPV4/TCP/127.0.0.1/57546",
    "SessionTV": "1969-12-31T18:00:00.000-0600"
}
  == Connect attempt from '127.0.0.1' unable to authenticate
```

Great success!

#### Dissecting the event

There's a few interesting fields in the security event that we could build statistics from, and a few that we should consider carefully when writing the next portion of our module. In no particular order, here are a few thoughts to guide the next part of the development:

* There are a number of different types of security events, which are conveyed by the **SecurityEvent** field. This integer value corresponds to the `enum ast_security_event_type`. There are a fair number of security events that we may not care about (at least not for this incarnation of the module). Since what we want to track are failed authentication attempts, we will need to filter out events based on this value.

* The **Service** field tells us who raised the security event. If we felt like it, we could use that to only look at SIP attacks, or failed AMI logins, or what not. For now, I'm going to opt to not care about where the security event was raised from: the fact that we get one is sufficient.

* The **RemoteAddress** is interesting: it tells us where the security issue came from. While we're concerned with statistics - and I think keeping track of how many failed logins a particular source had is pretty interesting - for people using fail2ban, iptables, or other tools to restrict access, this is a pretty useful field. Consume, update; rinse, repeat.

### STATS!

Let's get rid of the log message and start doing something interesting. In Asterisk 12, we added a module, `res_statsd`, that does much what its namesake implies: it allows Asterisk to send stats to a [StatsD](https://github.com/etsy/statsd/) server. StatsD is really cool: if you have a statistic you want to track, it has a way to consume it. With a number of pluggable backends, there's also (usually) a way to display it. And it's open source!

*In the interest of full disclosure, installing statsd on my laptop hit a few ... snags. Libraries and what-not. I'll post again with how well this module works in practice. For now, let's just hope the theory is sound.*

To use this module, we'll want to pull in Asterisk's integration library with StatsD, `statsd.h`.

```
...
#include "asterisk/json.h"
#include "asterisk/statsd.h"
...
```

And we should go ahead and declare that our module depends on `res_statsd`:

```
/*** MODULEINFO
    <support_level>extended</support_level>
    <depend>res_statsd</depend>
***/
```

Since we're not going to print out the Stasis message any more, go ahead and delete the char *str_json. Now that we know we're getting messages, let's filter out the ones we don't care about:

```
static void handle_security_event(void *data, struct stasis_subscription *sub,
                                  struct stasis_message *message)
{
    struct ast_json_payload *payload;
    int event_type;
    
    if (stasis_message_type(message) != ast_security_event_type()) {
        return;
    }
    
    payload = stasis_message_data(message);
    if (!payload || !payload->json) {
        return;
    }
    
    event_type = ast_json_integer_get(ast_json_object_get(payload->json, "SecurityEvent"));
    switch (event_type) {
    case AST_SECURITY_EVENT_INVAL_ACCT_ID:
    case AST_SECURITY_EVENT_INVAL_PASSWORD:
    case AST_SECURITY_EVENT_CHAL_RESP_FAILED:
        break;
    default:
        return;
    }
}
```

Here, after pulling out the payload from the Stasis message, we get the **SecurityEvent** field out and assign it to an integer, `event_type`. Note that we know that the value will be one of the `AST_SECURITY_EVENT_*` values. In my case, I only care when someone:

* Fails to provide a valid account

* Fails to provide a valid password

* Fails a challenge check (rather important for SIP)

So we bail on any of the event types that aren't one of those.

The first stat I'll send to StatsD are the number of times a particular address trips one of those three security events. StatsD uses a period delineated message format, where each period denotes a category of statistics. The API provided by Asterisk's StatsD module lets us send any statistic using `ast_statsd_log`. In this case, we want to just simply bump a count every time we get a failed message, so we'll use the statistic type of `AST_STATSD_METER`.

Using a `struct ast_str` to build up the message we sent to StatsD, we'd have something that looks like this:

```
static void handle_security_event(void *data, struct stasis_subscription *sub,
                                  struct stasis_message *message)
{
    struct ast_str *remote_msg;
    struct ast_json_payload *payload;
    int event_type;
    
    if (stasis_message_type(message) != ast_security_event_type()) {
        return;
    }
    
    payload = stasis_message_data(message);
    if (!payload || !payload->json) {
        return;
    }
    
    event_type = ast_json_integer_get(ast_json_object_get(payload->json, "SecurityEvent"));
    switch (event_type) {
    case AST_SECURITY_EVENT_INVAL_ACCT_ID:
    case AST_SECURITY_EVENT_INVAL_PASSWORD:
    case AST_SECURITY_EVENT_CHAL_RESP_FAILED:
        break;
    default:
        return;
    }
    
    remote_msg = ast_str_create(64);
    if (!remote_msg) {
        return;
    }
    
    ast_str_set(&remote_msg, 0, "security.failed_auth.%s.%s",
    ast_json_string_get(ast_json_object_get(payload->json, "Service")),
    ast_json_string_get(ast_json_object_get(payload->json, "RemoteAddress")));
    ast_statsd_log(ast_str_buffer(remote_msg), AST_STATSD_METER, 1);
    
    ast_free(remote_msg);
}
```

Cool! If we get an attack from, say, 192.168.0.1 over UDP via SIP, we'd send the following message to StatsD:
```
security.failed_auth.SIP.UDP/192.168.0.1/5060
```

Except, we have one tiny problem... IP addresses are period delineated. Whoops.

### (cleaner) STATS!

We really want the address we receive from the security event to be its own "ID", identifying what initiated the security event. That means we really need to mark the octets in an IPv4 address with something other than a '.'. We also need to lose the port: if the connection is TCP based, that port is going to bounce all over the map (and we probably don't care which port it originated from either). Since the address is delineated with a '/' character, we can just drop the last bit of information that's returned to us in the **RemoteAddress** field. Let's write a few helper functions to do that:

```
static char *sanitize_address(char *buffer)
{
    char *current = buffer;
    
    while ((current = strchr(current, '.'))) {
        *current = '_';
    }
    
    current = strrchr(buffer, '/');
    *current = '\0';
    
    return buffer;
}
```

Note that we don't need to return anything here, as this modifies buffer in place, but I find those semantics to be nice. Using the return value makes the modification of the buffer parameter obvious.

That should turn this:

```
IPV4/TCP/127.0.0.1/57546
```

Into this:

```
IPV4/TCP/127_0_0_1
```

Nifty.

Since we used a `struct ast_str *` to build our message, we'll need to pull out the **RemoteAddress** into a `char *` to manipulate it.

**REMEMBER: STASIS MESSAGES ARE IMMUTABLE.**

We're about to mutate a field in it; you cannot just muck around with this value in the message. Let's do it safely:

```
    char *remote_address;
        
    ...
         
    remote_address = ast_strdupa(ast_json_string_get(ast_json_object_get(payload->json, "RemoteAddress")));
    remote_address = sanitize_address(remote_address);
```

Better. With the rest of the code, this now looks like:

```
static void handle_security_event(void *data, struct stasis_subscription *sub,
                                  struct stasis_message *message)
{
    struct ast_str *remote_msg;
    struct ast_json_payload *payload;
    char *remote_address;
    int event_type;
    
    if (stasis_message_type(message) != ast_security_event_type()) {
        return;
    }
    
    payload = stasis_message_data(message);
    if (!payload || !payload->json) {
        return;
    }
    
    event_type = ast_json_integer_get(ast_json_object_get(payload->json, "SecurityEvent"));
    switch (event_type) {
    case AST_SECURITY_EVENT_INVAL_ACCT_ID:
    case AST_SECURITY_EVENT_INVAL_PASSWORD:
    case AST_SECURITY_EVENT_CHAL_RESP_FAILED:
        break;
    default:
        return;
    }
    
    remote_msg = ast_str_create(64);
    if (!remote_msg) {
        return;
    }
    
    service = ast_json_string_get(ast_json_object_get(payload->json, "Service"));
    remote_address = ast_strdupa(ast_json_string_get(ast_json_object_get(payload->json, "RemoteAddress")));
    remote_address = sanitize_address(remote_address);
    
    ast_str_set(&remote_msg, 0, "security.failed_auth.%s.%s", service, remote_address);
    ast_statsd_log(ast_str_buffer(remote_msg), AST_STATSD_METER, 1);
    
    ast_free(remote_msg);
}
```

Yay! But what else can we do with this?

### MOAR (cleaner) STATS!

So, right now, we're keeping track of each individual remote address that fails authentication. That may be a bit aggressive for some scenarios - sometimes, we may just want to know how many SIP authentication requests have failed. So let's track that. We'll use a new string buffer (dual purposing buffers just feels me with ewww), and populate it with a new stat:

```
    service = ast_json_string_get(ast_json_object_get(payload->json, "Service"));
    
    ast_str_set(&count_msg, 0, "security.failed_auth.%s.count", service);
    ast_statsd_log(ast_str_buffer(count_msg), AST_STATSD_METER, 1);
```

Since we're unlikely to get a remote address of count, this should work out okay for us. With the rest of the code, this looks like the following:

```
static void handle_security_event(void *data, struct stasis_subscription *sub,
                                  struct stasis_message *message)
{
    struct ast_str *remote_msg;
    struct ast_str *count_msg;
    struct ast_json_payload *payload;
    const char *service;
    char *remote_address;
    int event_type;
    
    if (stasis_message_type(message) != ast_security_event_type()) {
        return;
    }
    
    payload = stasis_message_data(message);
    if (!payload || !payload->json) {
        return;
    }
    
    event_type = ast_json_integer_get(ast_json_object_get(payload->json, "SecurityEvent"));
    switch (event_type) {
    case AST_SECURITY_EVENT_INVAL_ACCT_ID:
    case AST_SECURITY_EVENT_INVAL_PASSWORD:
    case AST_SECURITY_EVENT_CHAL_RESP_FAILED:
        break;
    default:
        return;
    }
    
    remote_msg = ast_str_create(64);
    count_msg = ast_str_create(64);
    if (!remote_msg || !count_msg) {
        ast_free(remote_msg);
        ast_free(count_msg);
        return;
    }
    
    service = ast_json_string_get(ast_json_object_get(payload->json, "Service"));
    
    ast_str_set(&count_msg, 0, "security.failed_auth.%s.count", service);
    ast_statsd_log(ast_str_buffer(count_msg), AST_STATSD_METER, 1);
    
    remote_address = ast_strdupa(ast_json_string_get(ast_json_object_get(payload->json, "RemoteAddress")));
    remote_address = sanitize_address(remote_address);
    
    ast_str_set(&remote_msg, 0, "security.failed_auth.%s.%s", service, remote_address);
    ast_statsd_log(ast_str_buffer(remote_msg), AST_STATSD_METER, 1);
    
    ast_free(remote_msg);
    ast_free(count_msg);
}
```

### In Conclusion

And there we go! Statistics of those trying to h4x0r your PBX, delivered to your StatsD server. In this particular case, getting this information off of Stasis was probably the most direct method, since we want to programmatically pass this information off to StatsD. On the other hand, since security events are now passed over AMI, we could do this in another language as well. If I wanted to update iptables or fail2ban, I'd probably use the AMI route - it's generally easier to do things in Python or JavaScript than C (sorry C afficianados). On the other hand: this also makes for a much more interesting blog post and an Asterisk module!
