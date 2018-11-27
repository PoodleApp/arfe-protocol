# ARFE

ARFE is a data protocol on top of email ([RFC 5322][]) to enable exchange of
structured data where message content parts may refer to, modify, or retract
other content parts in the same conversation.
Email conversations are a read-only medium;
so ARFE messages take the form of an append-only log that is produced and
consumed collectively by participants in a conversation.
Interpreting ARFE messages and other content parts in a conversation produces
a view to be presented to the user where view elements do not necessarily
correspond one-to-one with messages in the underlying conversation.
For example an ARFE message may be an "edit" to a previous message in the
conversation.

## Conversation

A conversation is a sequence of email messages that are related by message IDs
in `References` headers.
A conversation may be identified by the `mid:` URI of a "root" message
(see [RFC 2392][]).
The conversation consists of all messages available to a user that include the
ID of the root message in a `References` header,
or that reference a message ID that is part of the conversation.
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

Each email message may have many content parts,
thanks to the MIME data format.
Exactly one content part in each message is the primary content part as
determined by its position in the MIME structure.
(Note that email clients may disagree on which is the primary content part in
a message due to the fact that MIME allows for fallback representations of
a content part,
and clients are permitted to ignore content types that they are not programmed
to display.)

TODO: Link to to rules for determining primary content part,
or write them here.

A "resource" is an ARFE document that is derived from one or more email content
parts.
Each resource is introduced by a content part that represents its initial
revision.
Subsequent ARFE messages may modify a resource.

A conversation is interpreted as a set of resources each identified by URI.
Some of those resources come from primary content parts,
and should be presented to the user directly.
Other resources come from non-primary content parts that may be referenced by
the primary resources in some way.

An ARFE message
(an email message with an ARFE document in the primary content position)
has the effect of changing the interpretation of a conversation by replacing
the content and/or metadata of a resource with new values,
or has the effect of introducing a new resource to the conversation.
A resource that has been modified by an ARFE message retains the same URI.
Thus resources are "living" - they may change over time but have a stable URI.

In the ARFE model an email conversation resembles a Git repository in that an
ARFE update specifies a specific revision of a resource,
and specifies new content and metadata.
In the Git model there is a single versioned resource:
the set of files contained in a repository.
But in the ARFE model a conversation has at least one resource per email
message, and each is modified independently.

Old versions of revision content may be presented to the user via a "view
revision history" feature,
but are otherwise inaccessible via the resource URI.

## References to Living Resources vs Static Content

A resource is a logical construct computed by interpreting a set of ARFE
messages, and other email content parts.
Each of those inputs is a piece of static content with its own URI.
This will usually be a `mid:` URI as defined by [RFC 2392][].

An ARFE-enabled client must be able to interoperate with Classic Email clients,
which means that a primary content part in an email message that does not
contain an ARFE document must be interpreted as a resource introduced into the
conversation.

It must be possible to refer to the "current" state of a resource by URI,
as distinguished from the URIs of the email messages and content parts that
make up the history of the resource.
This is possibly the most important part of the protocol;
but at this time a decision has not been made on how to reference living
resources.

TODO: Refer to prior art for possible solutions. For example, Dat or Camlistore.

Listed here are some possible solutions:

### Option 1: The interpretation of a `mid:` URI changes over time

Every content part in every email message in a conversation introduces
a resource that is identified by the URI of the content part.
The same URI also identifies the initial revision of the resource.
An ARFE document that modifies another resource serves as a revision:
the URI of the ARFE document serves as the URI of the content at the point in
time the update was made.

During interpretation (while computing revisions for each resource) the URI of
a content part resolves to that content part, not to any living resource.

After interpretation is complete any reference to the URI of a content part in
the context of an interpreted conversation resolves to an ARFE document that
supplies the current version of the content and metadata of the corresponding
resource.
In other words, in the interpreted view of the conversation each content part
URI becomes a living resource URI.

The downsides are that there is no way to refer to the original static content
part from the context of an interpreted conversation,
and URIs do not behave according to specified semantics.

### Option 2: A new URI scheme for resources

Every content part in every email message in a conversation introduces
a resource that is identified by a URI derived from the URI of the content part.
This probably requires a new URI scheme.
For example if a content part URI is `mid:66bc009/2e24899` then the URI of the
corresponding resource might be, `arfe:mid:66bc009/2e24899`

The URI of the content part also serves as the URI of the initial revision of
the resource.
The URI of any ARFE document that updates the resource serves as the URI of the
new revision.

The resources from the interpreted conversation are displayed to the user,
as opposed to primary content parts as a Classic Email client would do.
Each attachment part is replaced by its corresponding resource.

The downside is that any reference to the content part itself will not resolve
to the living document.
For example a message might include inline images;
the images could be modified using ARFE,
but the HTML part that references those images is likely to use `mid:` or `cid:`
URIs -
especially if the message was produced by a Classic Email client.
This would also require the Activity over Email spec to be written to take into
account the special handling of `arfe:` URIs.

### Option 3: Interpret `mid:` URIs that do not specify a content part as branch pointers

A `mid:` URI includes a message ID part,
and optionally a content ID part to refer to a specific content part within
a message.
Technically according to [RFC 2392][] a URI such as `mid:messageid` refers to
the message itself (i.e. an [RFC 5322][] IMF document).
It could be abused to refer to the primary content part of a message.
This use could be signalled with a trailing slash: `mid:messageid/`.

In this approach every *primary* content part in every message in
a conversation introduces a resource identified by the URI of the containing
message with a trailing slash.
The full URI of the content part serves as the URI of the initial revision of
the resource.
In the context of the interpreted conversation any use of a message URI with
a trailing slash resolves to an ARFE document that provides the current version
of the resource that was introduced by the original primary content part of
that message.

As with Option 2 this approach means that uses of content part URIs point to
static content,
not to living resources.
As it stands this option does not provide a means to edit or remove attachments
from an existing message.
Attachments cannot be modified in this approach because only primary content
parts can be living resources.
This also introduces a use of URIs that does not match any specified semantics.

## ARFE Message Format

In this document "ARFE message" refers to a primary content part in an email
message that contains an ARFE document, and has the `application/ld+json`
content type.
ARFE is based on [JSON-LD][] which allows ARFE and Activity Streams to coexist
in one document.
An "ARFE document" is an Activity Streams document that may also include
ARFE-specific properties.

TODO: Should we use the more specific `application/activity+json` MIME type?
The issue is that ARFE is not actually part of Activity Streams.

TODO: How to accommodate multiple ARFE operations or activities in one email
message?
Ideally each ARFE operation / activity should have have a distinct URI.

ARFE-specific properties include a property to indicate a previous revision of
a resource that the new document replaces,
and properties that declare access control policies.

### Updating a Resource

An ARFE message that does not specify a revision that it replaces creates a new
resource.
This is a "create" operation.
This implies that any valid Activity Streams document in a primary content part
creates a new resource.

A primary content part that is not an ARFE message is implicitly a "create"
operation where the content of the new resource is a default, minimal activity
whose object is the content part itself.
This is also a "create" operation.

An "update" operation is one where the ARFE message points to a specific
revision of a resource in the same conversation that it replaces.
The replaced revision is given by the URI using the [RFC 2392][] `mid:` scheme.
The URI must specify message ID *and* content part so that update events can be
interpreted in the correct order.

The new content and metadata of the resource is given by the ARFE message.
The new message replaces the old -
any properties that were specified in the old revision that are not present in
the new message are not included in the new version of the document.

An ARFE "delete" operation is an update that changes the resource activity type
to "Noop".

TODO: Activity Streams does not currently have such an activity type, so we
will have to define it.

### Access Control

ARFE is intended to enable collaborative editing.
For example a user can share a document,
and other participants in the conversation can edit the document.
But users would not expect that someone else would be allowed to edit each
other's comments.
Access control rules limit update operations and other actions to correspond to
users' expectations and preferences.

By default a user can edit the content and metadata of their own messages.
They can reword their own comments, edit their own shared documents, delete
activities, and so on.
By default users are allowed to reply to or forward / reshare messages as they
please.
An ARFE message that creates or updates a resource may specify that other users
beside the original author are allowed to modify given properties of the
resource.
They may also restrict actions such as replies and resharing.
Possible policies for who may make modifications include:

- anyone who receives the content
- anyone with an email address in a given domain
- a list of people identified by email address

Possible policies for actions that may be performed include:

- modify resource content
- modify content type
- modify access control policies
- delete the resource
- public reply
- forward / reshare

TODO: Policies declare specific sets Activity Stream or ARFE properties that
a given set of people are authorized to change.

An ARFE update operation may specify a revised access control policy,
assuming the existing access control policy allows the updater to make access
control changes.

Obviously an access control policy will not prevent other users from sending
ARFE update messages.
A compliant client will ignore ARFE messages that violate an access control
policy,
with the exception that the client may present the message as a request to make
the given changes to the users who are authorized to do so.
If those users choose to accept the changes they may do so by constructing
a new ARFE message.
The new message should include an Activity Streams `attributedTo` property to
credit the author of the changes.

Likewise an access control policy cannot prevent users from replying or
forwarding.
A compliant client should indicate to the user that the owners of the
resource have indicated via a policy that they do not want content to be
reshared,
or that they do not want replies.

Note that the access control policy of a resource can change over time.
The decision on whether an update operation is allowed must be based on the
policy in effect at the time when the operation was made.
Access control policy changes, and other updates are resource revisions.
Every revision points to a specific revision that it replaces.
This means that there is a canonical ordering to updates.
The access control policy in effect when an update operation is made is exactly
the policy given in the revision that the update replaces.

## Compatibility with Classic Email

As described in the Activity over Email spec,
any message that uses Activity Streams for its primary content should include
a fallback representation.
That applies equally to ARFE documents.
ARFE adds the complication of representing update events.

If resource content differs from the previous revision then changes should be
presented to Classic Email users somehow.

If content represents a shared document,
or in any case where it is important that users see the same content,
then the new content should be presented to Classic Email users in its entirety.
Content may be presented in a fallback representation, or as an attachment.
Fallback content may also include a note that content has been updated,
or a summary of changes.
If content is presented as an attachment then an additional note in fallback
content is strongly encouraged.
If in doubt, present updated content in its entirety.

Otherwise if the authoring client is able to produce a user-friendly summary of
changes then it may present that summary as fallback content.
The summary may be auto-generated, or provided by the author.
For example this change to the content of a comment:

    "Then we will go for desert!" → "Then we will go for dessert!"

Could be represented in fallback content with a message like,
"s/desert/dessert" or "When I said 'desert' of course I meant 'dessert'".

Some update operations might change resource metadata,
such as access control policies,
but leave content unchanged.
In these cases an ARFE-enabled client will not show a new element in the
conversation view;
but a Classic Email client will,
and users will be confused if the new message is does not explain itself
somehow.
It is suggested that the fallback content for such a message include
a user-friendly description of the change,
and a link to an ARFE-enabled client or to an article that describes what ARFE
is.
For example,

> Alice has granted you permission to edit the Q4 Planning Doc.
> You can use Poodle <https://github.com/PoodleApp/poodle> to make edits.

## Handling Update Conflicts

A conflict occurs in cases where two or more authorized ARFE update messages
replace the same revision of a resource.
It is critical that conversation participants have a consistent view of the
state of each resource.
It is also desirable that conflicts be resolved with minimal user intervention.

A later iteration of this spec might define options for automatically merging
conflicting versions of a resource in limited cases.
Such an algorithm must be fully-specified to ensure consistency.

A user may explicitly resolve a conflict by sharing an ARFE update that points
to all conflicting revisions as its previous versions.
(A.k.a. Git rules)

A conflict results in essentially more than one branch of a resource.
It is possible that users may have sent further updates to one or more of those
branches.
If one branch has ARFE updates from more distinct participants than any of the
others then it is considered to be the current version of the resource.
(A.k.a. Blockchain rules)

If one version of the resource is at least five minutes newer than any other
version then it is considered to be the current version of the resource.
The time of the message is determined by time received as reported by the users
email service.
In the case of the sender that will be the time the message was submitted for
delivery.
(A.k.a. first come, first served)

Case study: Alice boards an airplane, and updates Q4 Planning Doc with the
new projections.
She is offline, so she cannot share the changes for a few hours until the plane
lands.
Two hours into Alice's flight Bob updates Q4 Planning Doc to add emoji.
Bob is online, so his changes are shared immediately.
Several coworkers have time to see Bob's changes before Alice's plane lands.
When the plane does land, Alice's computer sends the update.
Everyone's email servers are in agreement that Alice's update was received
hours after Bob's;
so Bob's update is considered to be the current version of the document.

When a client detects that changes made by its user have resulted in a conflict
should notify the user to give them an opportunity to reconcile the conflicting
versions.
If the conflict can be resolved automatically then the author of the version
that was accepted does not need to be notified.
But any user whose changes were not accepted should be notified so that their
changes are not lost without warning.

## References

- [Activity Streams 2.0][]
- [JSON-LD][]
- [RFC 2392][]: `mid:` and `cid:` URN schemes for email messages and content parts
- [RFC 5322][]: Internet Message Format, A.K.A. Email

[Activity Streams 2.0]: https://www.w3.org/TR/activitystreams-core/
[JSON-LD]: https://w3c.github.io/json-ld-syntax/
[RFC 2392]: https://tools.ietf.org/html/rfc2392
[RFC 5322]: https://tools.ietf.org/html/rfc5322
