# ARFE

A data protocol on top of email ([RFC 5322][]) to enable exchange of structured
data where message content parts may refer to, modify, or retract other content
parts in the same conversation.
Email conversations are a read-only medium;
so ARFE messages take the form of an append-only log that is produced and
consumed collectively by participants in a conversation.
Interpreting ARFE messages and other content parts in a conversation produces
a view to be presented to the user where view elements do not necessarily
correspond one-to-one with messages in the underlying conversation.
For example an ARFE message may be an "edit" to a previous message in the
conversation.

In broad terms ARFE introduces two new concepts to email conversations:
[Activity Streams 2.0][] content parts which add a new layer of metadata to
exchanges,
and ARFE messages modify the user's view of conversation content.

## Conversation

A conversation is a sequence of email messages that are related by message IDs
in `References` headers.
A conversation may be identified by the `mid:` URI of a "root" message
(see [RFC 2392][]).
The conversation consists of all messages available to a user that include the
ID of the root message in a `References` header,
or that are reference a message ID that is part of the conversation.
It should be noted that a conversation is a directed acyclic graph (DAG);
as messages are added to a conversation the conversation may merge with another
conversation,
or may branch.
(More on branching later.)
Another important point is that users may conclude different sets of messages
for a conversation with a given URI if they have distinct sets of messages
available to them.
One user might have only a partial version of a conversation;
or two users might each have access to messages that the other does not have
access to.

## Interpretation of a Conversation

The premise of ARFE is that email messages do not need to map one-to-one to
elements of the view of a conversation that is presented to a user.
Content parts are presented to the user;
but ARFE messages in the same conversation affect how content is displayed.
For example an ARFE "edit" message may replace data of a content part.
The original content is hidden from the user
(except via a "view revision history" feature that the user's client may
implement).
Replacement content specified by the ARFE message is displayed in the place of
the original content.

## Content Parts

Historically the primary content part in an email message is usually plain text
or HTML.
A message thread consists of these content parts presented to the user in order
received.
ARFE adds [Activity Streams 2.0][] as primary content parts.
Activity Streams parts provide many options for presenting content via
declarative activity types such as announcements, photo albums, calendar
events, invitations, likes, etc.

TODO: How does Activity Streams compare to formats such as iCal, or Google's travel itinerary thing?

Activity Streams includes its own specification for metadata that is also
specified by email such as authorship, and the time when the activity was
created.
Email metadata is expected to be present in most cases,
but messages will not always contain Activity Streams content parts.
In addition email metadata such as sender can be verified via [DMARC][].
So metadata should be taken from email message headers,
not from activity content parts.

## ARFE Messages

In this document "ARFE message" refers to a primary content part in an email
message that has the ARFE format and MIME type.
An ARFE message is a [JSON-LD][] document,
like an [Activity Streams 2.0][] document.

TODO: We need a MIME type for ARFE messages.

TODO: If ARFE messages are represented with JSON-LD they could in principal be
combined into the same object with activities.
But that would lead to ambiguity over what the MIME type of the content part
should be.
(The MIME type for Activity Streams is `application/activity+json`).
One option is to position ARFE as an extension of Activity Streams.
Another option is to use a generic MIME type such as `application/ld+json` for
parts that could contain Activity Streams, or ARFE content, or both.

TODO: How to accommodate multiple ARFE messages in one email message? Ideally
each ARFE operation should have have a distinct URI.

An ARFE message specifies a CUD operation (i.e. create, update, delete),
and may also specify access control policies.

### CUD Operations

The interpretation of a conversation is as a set of data parts identified by
URI. Some of those parts are primary, and should be presented to the user.
(Primary parts are those that appear in a certain position in the MIME
structure of a message, or that are created by an ARFE create operation.)
Others parts are resources that may be referenced by the primary parts in some
way.

ARFE operations are special in that they may remove and replace primary parts
from the interpretation of a conversation.
Removed parts are replaced with new content (in the case of an update
operation) or with a tombstone (in the case of a delete operation).

ARFE operations resemble git commits in that a log of operations is assembled
into a view of the current state of a resource.
But with git there is a single resource: the set of files that make up a repo.
With ARFE every primary part is a distinct resource that is modified
independently.
Primary parts correspond to trees in the git model,
and ARFE messages correspond to commits.
The git model also has branches which provide a way to refer to the "current"
state of a resource.
In the ARFE model the "branches" are email message URIs.
The URI of a content part refers to a fixed piece of content;
but the interpretation of the URI of a message (with no content part specified)
refers to the content that may change over time as more ARFE messages appear in
a conversation.
Thus `mid:` URIs with no content part are the "branches" of the ARFE model.

TODO: The view of email message URIs as branch pointers could use some careful
thinking.
Ultimately we need some way to distinguish between a reference to the "current"
version of a primary content part, and a reference to a fixed resource.
A primary part can be referenced by either `mid:messageId/contentPart` or by
`mid:messageId`, which provides a convenient way to make that distinction.
On the other hand such a use of the `mid:` scheme would technically violate
[RFC 2392][].
And it might be nice to divorce content parts from email messages (except for
purposes of computing metadata).

A primary content part that is not an ARFE message is implicitly a "create"
operation where the data in the part is the new content.
Any ARFE create operation produces a new primary part.

Update and delete operations specify the URI of a target primary part in the
same conversation using the [RFC 2392][] `mid:` scheme.
The URI must specify message ID *and* content part so that update events can be
interpreted in the correct order.
Create and update operations specify new content via a URI which should
point to a content part in the same email message.
A new content URI does not necessarily point to a primary content part.
It may point to a fallback part, or any other content part.

### Access Control

ARFE is intended to enable collaborative editing.
For example a user can share a document,
and other participants in the conversation can edit the document.
But users would not expect that someone else would be allowed to edit each
other's comments.
Access control rules limit update operations to correspond to users'
expectations.

By default a user can edit the content of their own messages.
They can reword their own comments, edit their own shared documents, delete
activities, and so on.
An ARFE message that creates a resource may specify that other users beside the
original author are allowed to modify the resource.
Possible policies are:

- anyone who receives the content may edit it
- anyone with an address in a given domain may edit
- a given set of people may edit, specified by email address

There are similar policy operations for metadata changes -
in particular who is allowed to make changes to the access control policies of
a resource, and who is allowed to delete a resource.

An ARFE update operation may specify a revised access control policy,
assuming the existing access control policy allows the updater to make access
control changes.

Obviously an access control policy will not prevent other users from sending
ARFE update messages.
A compliant client will ignore ARFE messages that violate an access control
policy,
which the exception that the client may present the message as a request to
make the given changes to the users who are authorized to do so.
If those users choose to accept the changes they may do so by constructing
a new ARFE message.

Note that every access control change points to a specific content part
representing the previous version of a resource,
and every update operation also points to a specific content part.
This makes it possible to determine whether a given update operation occurred
in a window where the author of the update was authorized to make the specified
changes.

### Revision Metadata

The interpretation of a conversation includes logs of resources that have been
modified including modification times, and the person who made modifications.

TODO: In the case where an user accepts a change request from another user it
would be useful to be able to record metadata in the update that gives credit
to the author of the changes.

### Limitations

The use of email message URIs as "branch" pointers means that a message may
only contain one activity.
Otherwise it would be difficult to reference the latest version of a specific
activity.
For example if a user re-shares a document they should link to the message ID
of the original email message with document,
not to any specific content part in that message.
Otherwise they would be sharing a specific version of the document at a given
point in time as opposed to the "current", living document.


## Compatibility with Classic Email

## Retrieving Conversation History, and Quoting

## ARFE, PGP, and S/MIME

## Order of Presentation

## Handling Update Conflicts


## References

[Activity Streams 2.0]: https://www.w3.org/TR/activitystreams-core/

[DMARC][]: Domain-based Message Authentication, Reporting and Conformance
[DMARC]: https://tools.ietf.org/html/rfc7489

[JSON-LD]: https://w3c.github.io/json-ld-syntax/

[RFC 2392][]: `mid:` and `cid:` URN schemes for email messages and content parts
[RFC 2392]: https://tools.ietf.org/html/rfc2392

[RFC 5322][]: Internet Message Format, A.K.A. Email
[RFC 5322]: https://tools.ietf.org/html/rfc5322
