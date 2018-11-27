# Activity over Email

Historically the primary content part in an email message ([RFC 5322][]) is
usually plain text or HTML.
A message thread consists of these content parts presented to the user in order
received.
ARFE adds [Activity Streams 2.0][] as an option for primary content parts.
Activity Streams parts provide many options for presenting content via
activity types such as announcements, photo albums, calendar
events, invitations, likes, etc.

TODO: How does Activity Streams compare to formats such as iCal, or Google's travel itinerary thing?

Activity Streams provide a view of a sequence of interactions.
In other uses Activity Streams are computed from an external source of truth,
such as a database.
In Activity over Email the email messages transporting activities are
themselves the source of truth.

## Referring to Activities

[Activity Streams 2.0][] requires that an activity objects and actors specify
an ID which takes the form of a URI.
In a typical Activity over Email exchange activity objects will be content
parts in email messages.
URIs pointing to these objects are constructing using `mid:` URIs as specified
by [RFC 2392][].
URIs pointing to actors should be constructed using the `mailto:` scheme.

## Metadata Calculation

Activity Streams includes its own specification for metadata that is also
specified by email such as authorship, and the time when the activity was
created.
Email metadata is expected to be present in typical cases,
but messages will not always contain Activity Streams content parts.
In addition email metadata such as sender can be verified via [DMARC][].
For these reasons metadata in email message headers is to be considered the
source of truth when computing metadata of an activity.
Activities may include additional metadata that is not given by email message
headers.
Activities should not include metadata that contradicts or overlaps metadata in
an email message header.

TODO: There is nuance to this. Details like actor, audience, and secondary
audience can be taken from email metadata; but it might be useful to allow
properties like `inReplyTo` where an activity can specify objects with greater
precision than an email message can.

## MIME Type

The content type for a content part containing an Activity Streams document
should be given as `application/ld+json`,
as opposed to the more specific type, `application/activity+json`,
given in the Activity Streams specification.
The reason for this is so that activity data and ARFE data can be combined into
a single document in a single message content part.

## Compatibility with Classic Email

Classic Email clients are not able to display Activity Streams content.
Fortunately the MIME format allows for fallback representations for any content
part.
Any important information in activity data should be made accessible to Classic
Email users somehow.

Any message that uses Activity Streams for its primary content should also
include a fallback plain text or HTML representation, or both.
Fallback representations may be customized by the author of the message.

For example, a `Like` activity could be represented by fallback content such as
"+1", "👍", or "Keep up the good work!".

Content that cannot be displayed inline in a user-friendly way should be
included as an attachment.
It is recommended that the activity object link to the URI of the attachment
part so that the object is accessible to both Activity Streams-aware clients
and to Classic Email clients.

## Attachments

In Classic Email if a message has a `multipart/mixed` type the first nested
part is considered to be primary content,
and the remaining parts are presented as attachments.
Activity Streams allows new options:
the primary part can specify how other parts are presented.
For example an activity have the type "Create" and link to a PDF content part
as its object.
In cases like this the PDF should be placed in the MIME tree so that it is
presented as an attachment in Classic Email clients;
but Activity over Email clients should only display attachments that are
explicitly listed in the activity in cases where the primary content of an
email message is an `Activity`.
This is partly to make it possible to use ARFE to modify the attachments
presented with a message.

In cases where the primary content of an email message is not an `Activity` the
client should infer an `Activity` that includes a list of attachments based on
non-initial parts under a `multipart/mixed` part,
as Classic Email clients do.

## Retrieving Conversation History, and Quoting

A client can infer which portions of conversation history each conversation
participant has had access to by examining `From:`, `To:`, and `Cc:` headers in
existing messages in the conversation.
The author of a message may assume that participants are able to retrieve prior
messages,
and therefore is not required to quote previous content when replying to
a conversation.

If it is determined that some participant does not have access to the full
conversation history then a message author should either quote or link to
messages to fill in the gaps.

It is important that the author does not quote messages from a conversation
branch that any participants in the current branch are excluded from!

TODO: Discuss branches in more detail.

Quoting in ARFE is entirely unlike quoting in Classic Email.
In the ARFE model quoted messages are intended to be processed by machine.
Relevant metadata is computed from email headers,
content parts must appear in the context of their original message to compute
correct URIs,
and digitally-signed parts require a MIME structure to preserve cryptographic
verifiability.
For the full [RFC 5322][] content of each quoted message is included,
which each message appearing in its own content part.
These content parts should be grouped under a `multipart/related` part which
indicates that they are not intended to be presented directly to the user.
An ARFE-enabled client should parse quoted messages and use them to construct
the interpretation of a conversation if the client does not have copies of
those messages from another source.

Messages that are delivered by an email provider can be verified (to some
extent) with [DMARC][].
Quoted messages do not have the same verifiability.
A client should indicate to the user cases where information comes from quoted
messages,
and should specify who provided those messages.
This requirement can be skipped in cases where content is signed by a trusted
key.

If messages are accessible via URL then a client may include links to message
instead of embedding original message content.
Links are given in a references list in an activity.

TODO: Is there Activity Streams support for references?
Or should this be an ARFE feature?

A participant may send a message with a special activity type to indicate
a request for missing context.

TODO: This activity type needs to be specified.

## PGP and S/MIME

Users are encouraged to sign messages,
and clients are encouraged to attempt to verify signatures.

As is mentioned on the section on quoting,
a quoted message should include sufficient context to verify any signed content.

## Order of Presentation

Certain activities provide an explicit ordering.
For example a comment may be `inReplyTo` a specific object.

For other cases the client should consider the hierarchy of references in
`References:` and `In-Reply-To:` headers in messages where presentational
elements originate
(the message containing the initial revision of the corresponding resource).

If order cannot be inferred unambiguously by either of the above methods then
presentational elements should be further ordered by time received as reported
by the user's email service.

## References

- [Activity Streams 2.0][]
- [DMARC][]: Domain-based Message Authentication, Reporting and Conformance
- [JSON-LD][]
- [RFC 2392][]: `mid:` and `cid:` URN schemes for email messages and content parts
- [RFC 5322][]: Internet Message Format, A.K.A. Email

[Activity Streams 2.0]: https://www.w3.org/TR/activitystreams-core/
[DMARC]: https://tools.ietf.org/html/rfc7489
[JSON-LD]: https://w3c.github.io/json-ld-syntax/
[RFC 2392]: https://tools.ietf.org/html/rfc2392
[RFC 5322]: https://tools.ietf.org/html/rfc5322
