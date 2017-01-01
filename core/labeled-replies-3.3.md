---
title: Labeled responses
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Alexey Sokolov"
    period: "2015"
    email: "alexey-irc@asokolov.org"
  -
    name: "James Wheare"
    period: "2016"
    email: "james@irccloud.com"
---

## Notes for implementing work-in-progress version

This is a work-in-progress specification.

Software implementing this work-in-progress specification MUST NOT use the
unprefixed `label` tag name. Instead, implementations SHOULD use
the `draft/label` tag name to be interoperable with other software
implementing a compatible work-in-progress version.

The final version of the specification will use an unprefixed tag name.

## Introduction

This specification adds a new message tag sent by clients and repeated by servers to correlate responses with a specific request.

## Motivation

Certain client actions can result in responses from the server that vary in interpretion depending on how they were triggered, or otherwise lack a robust way to correlate with local state. Clients have historically needed to keep track of additional local state and/or apply comparison heuristics to server responses to correlate these appropriately.

Labeled responses enable a much simpler form of correlation by using a single id attached to a client request and repeated by the server in its response.

Additionally, labeled responses allow bouncers with multiple connected clients to direct responses (such as WHOIS queries or error messages, see examples) to the correct recipient.

## Architecture

### Dependencies

This specification uses the [message tags framework](/specs/core/message-tags-3.2.html) and relies on support for the [`batch`](/specs/extensions/batch-3.2.html) capability which MUST be negotiated at the same time.

### Capabilities

This speciciation adds the `draft/labeled-response` capability.

Clients requesting this capability indicate that they are capable of handling the message tag described below from servers.

Servers advertising this capability indicate that they are capable of handling the message tag described below from clients, and will use with the same tag and value in their response.

Servers MUST reject the capability at negotiation time if the `batch` capability is not also negotiated. Additionally, servers supporting `cap-notify` MUST cancel the capability if `batch` is removed at any time.

### Batch types

This specification adds the `draft/labeled-response` batch type, described below.

### Tags

This specification adds the `draft/label` message tag, which has a required value.

This tag MAY be sent by a client for any messages that need to be correlated with a response from the server.

For any message received from a client that includes this tag, the server MUST include the same tag and value in any response required from this message. Servers MUST include the tag in exactly one message.

If a response consists of more than one message, a `batch` MUST be used to group them into a single logical response. The start of the batch MUST be tagged with the `draft/label` tag. The batch type MUST be one of:

* An existing type applicable to the entire response
* `draft/labeled-response`

If no response is required, an empty batch MUST be sent.

#### Tag value

The tag value is chosen by the client and MUST be treated as an opaque identifier. The client SHOULD NOT reuse a tag value until it has received a complete response for that value from the server.

## Client implementation considerations

This section is non-normative.

In the case of `echo-message` (see example below), a client can use labeled responses to correlate a server's acknowledgment of their own messages with a temporary message displayed locally. The temporary message can be displayed to the user immediately in a pending state to reduce perceived lag, and then removed once a labeled response from the server is received.

When sending messages directed at a client's own nick, `echo-message` will result in duplicate messages being sent by the server, as both sent and received messages. Labeled responses allow clients to deduplicate these messages in one of two ways:

For private messages that match the clients nickname:
1. Ignore labeled messages, and use any unlabeled message as acknowledgment for all sent messages to clear temporary local messages.
2. Ignore unlabeled messages.

Both methods assume that the server will acknowledge all successful messages, or return a labeled error response, but differ in their attitude to to the semantics of sending and receiving.

## Bouncer implementation considerations

This section is non-normative.

Bouncers might choose to restrict routing labeled responses to some or all of their clients. For example, the response to a labeled WHOIS request might only be routed back to the originating client. However, an `echo-message` response might be routed to all clients. The bouncer might also choose to make labeled requests to the server on its own behalf, without routing any response to clients.

To avoid label value clashes from multiple connected clients, a bouncer might choose to modify the client supplied label, for instance, by adding a client id prefix. In this case, the modified label would be returned to its original form before routing the response back to the client.

## Examples

This section is non-normative.

1. `echo-message`


    ```
    Client: @label=pQraCjj82e PRIVMSG #channel :\x02Hello!\x02
    Server: @label=pQraCjj82e :nick!user@host PRIVMSG #channel :Hello!
    ```

2. Failed `PRIVMSG` with `ERR_NOSUCHNICK`

    ```
    Client: @label=dc11f13f11 PRIVMSG nick :Hello
    Server: @label=dc11f13f11 :irc.example.com 401 * nick :No such nick/channel
    ```
    
3. `WHOIS`

    ```
    Client: @label=mGhe5V7RTV WHOIS nick
    Server: @label=mGhe5V7RTV :irc.example.com BATCH +NMzYSq45x labeled-response
    Server: @batch=NMzYSq45x :irc.example.com 311 client nick ~ident host * :Name
    ...
    Server: @batch=NMzYSq45x :irc.example.com 318 client nick :End of /WHOIS list.
    Server: :irc.example.com BATCH -NMzYSq45x
    ```

## Alternatives

For the use case of bouncers directing query responses to the appropriate client, there exists prior art in the form of the znc module [route_replies](http://wiki.znc.in/Route_replies). The complexities and limitations of that module were a primary motivation for the standardised approach described in this specification.