# ARFE

ARFE is a data protocol on top of email ([RFC 5322][]) to enable exchange of
structured data where message content parts may refer to, modify, or retract
other MIME parts in the same conversation.
Email conversations are a read-only medium;
so ARFE messages take the form of an append-only log that is produced and
consumed collectively by participants in a conversation.
Interpreting ARFE messages and other content parts in a conversation produces
a view to be presented to the user where view elements do not necessarily
correspond one-to-one with messages in the underlying conversation.
For example an ARFE message may be an "edit" to a previous message in the
conversation.

## Key Points

The ability to declare changes to existing resources enables a new set of
interaction options,
such as collaborative editing.

There is no single canonical view of an email conversation.
Users will have different sets of messages available.
Sometimes users will have a partial history of a conversation.
In some cases conversations will diverge,
and no single user will have access to the full set of messages.
A user who has an incomplete set of messages should be able to compute
a coherent view of at least a portion of the conversation.
This informs certain protocol decisions such as the requirement that a resource
revision includes the entire content of the resource as opposed to a change set.

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
On or more parts in a message are primary content as determined by their
positions in the MIME structure.
(Note that email clients may disagree on which are the primary content parts in
a message due to the fact that MIME allows for fallback representations of
a content part,
and clients are permitted to ignore content types that they are not programmed
to display.)

TODO: Link to to rules for determining primary content parts,
or write them here.

A "resource" is an ARFE document that is derived from one or more email content
parts.
Each resource is introduced by a MIME part that represents its initial
revision.
Subsequent ARFE messages may modify a resource.

A conversation is interpreted as a set of resources each identified by URI.
Some of those resources come from primary content parts,
and should be presented to the user directly.
Other resources come from non-primary MIME parts that may be referenced by
the primary resources in some way.

An ARFE message
(an ARFE document in a primary content position of an email message)
has the effect of changing the interpretation of a conversation by replacing
the content and/or metadata of a resource with new values.
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
revision history" feature.

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
as distinguished from the URIs of the email messages and MIME parts that
make up the history of the resource.
A few possibilities for achieving this are listed here.
The version of this document that you are reading assumes Option 2.

TODO: Refer to prior art for possible solutions. For example, Dat or Camlistore.

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

Every MIME part in every email message in a conversation introduces
a resource that is identified by a URI derived from the URI of the MIME part.
This probably requires a new URI scheme.
For example if a MIME part URI is `mid:66bc009/2e24899` then the URI of the
corresponding resource might be, `arfe:66bc009/2e24899`

The URI of the MIME part also serves as the URI of the initial revision of
the resource.

The resources from the interpreted conversation are displayed to the user,
as opposed to primary content parts as a Classic Email client would do.
Each attachment part is replaced by its corresponding resource.

The downside is that any reference to the MIME part itself will not resolve
to the living document.
For example a message might include inline images;
the images could be modified using ARFE,
but the HTML part that references those images is likely to use `mid:` or `cid:`
URIs -
especially if the message was produced by a Classic Email client.
But this is not a deal breaker:

The Activity over Email spec should take into account the special handling of
the new scheme,
and use the new scheme in URIs where the intention is to refer to a living
resource.
For example a `Like` activity targeting a `mid:` URI would have the effect of
liking a specific version of a resource -
the like count would not carry over if the resource is updated.
Resharing by `mid:` URI would share a snapshot of a resource -
the view of the resource in the reshare conversation would not update when the
resource updates in the original conversation.
Activity over Email is a companion to ARFE,
so interdependency should be OK.

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

## The `arfe:` URI scheme

An `arfe:` URI refers to a living resource.
It has the form `arfe:messageId/contentId`.
The procedure for computing the current state of a resource is based on finding
ARFE documents in an email conversation;
so the `arfe:` scheme is tied to [RFC 5322][] messages.
An `arfe:` URI must refer to a MIME part in an IMF message.

### Constructing an `arfe:` URI

The `Content-Id` header of MIME parts is optional.
In case a MIME part does not have an ID specified then its content ID is
inferred using a SHA-256 hash of the part including its headers according to
this format:

    sha256:contentparthash

TODO: Are all message recipients guaranteed to have a byte-by-byte identical
view of the MIME part's headers?
IMAP processing might mess this up.
If that is the case then content ID calculation should be changed to use a hash
of the content of the part excluding headers.

### Resolving an `arfe:` URI

An `arfe:` URI is resolved to a `mid:` URI paired with ARFE metadata as follows:
The `messageId` portion of an `arfe:` URI identifies a message which serves as
the root of a conversation.
(It is likely that this root is a reply to a larger parent conversation.)
The `contentId` portion identifies a MIME part in the same message that is
the initial revision of the resource.
The URI is resolved by processing all ARFE messages in the conversation that
refer to the given message ID and content ID either as a previous revision or
as an updated reversion,
or that refer to a later revision of the same resource.
After conflict pruning, revisions specified by those ARFE messages form a DAG
with a single leaf node containing a `mid:` URI and metadata.

TODO: In order to ensure that the conversation described above includes all
relevant ARFE messages it is necessary to specify that any email message
carrying an ARFE message include the ID of the email message containing the
previous revision in a `References:` header.

It may be the case that there are no ARFE messages in the conversation that
refer to the given MIME part.
In that case the `arfe:` URI resolves to the `mid:` URI of the part,
and an empty set of ARFE metadata.

### Overlapping Resources

Note that according to the resolution procedure an `arfe:` URI may be
constructed with the message ID and content ID of a non-initial revision of
another resource.
The current states of both resources resolve to the same `mid:` URI and
metadata,
but one resource has a shorter revision history than the other.

## ARFE Document Format

In this document "ARFE message" refers to a primary content part in an email
message that contains an ARFE document, and has the appropriate content type.
An ARFE document is a [JSON-LD][] document that includes a list of ARFE update
operations.

TODO: What is that content type?

### Updating a Resource

An ARFE document is a table that describes updates to one or more resources.
A document maps the URI of the previous revision of each resource to an updated
URI paired with updated ARFE metadata.
Each row in the table is an "update operation".

TODO: Should this be changed so that an ARFE document declares an update to
a single resource?
In that case multiple updates would be given by multiple ARFE documents in an
email message.
That would have the advantage of providing a distinct URI for each update.

An ARFE document may describe metadata to be associated with a new resource
with no previous revision in which case the previous revision entry for the
update will be `None`.

TODO: Define the special value `None`.

Revision URIs must use the `mid:` or `cid:` schemes (see [RFC 2392][]),
and must specify a MIME part.
The URI of the new revision of a resource must point to a part in the same email
message with the ARFE document.
The 

There is no option for metadata-only updates.
Every update must point to a new MIME part that reproduces the entire
content of the resource.
This is because the URI of the MIME part acts as the revision ID;
if the same content URI were used in multiple updates then ordering of content
vs metadata changes would be ambiguous.

TODO: There might be some cleaner way to handle metadata.
It could be specified in MIME part headers for example.
That would at least avoid the need for a separate ARFE message to attach
metadata to a new resource.

Each resource must not be updated more than once in a single email message.

Any email message that includes an ARFE message must include a message ID in
its `References:` header corresponding to every previous revision entry in the
ARFE message.

ARFE metadata consists of access control policies.
Every update must list resource metadata in its entirety.
Metadata specified in an update is *not* merged with metadata of the previous
revision.

### Deleting a Resource

A resource can be "deleted" by setting the URI of the new revision to the
special value `None`.

A client implementation should take into consideration how presentation of
a conversation is affected when a resource is deleted.
If the resource is an Activity Streams `Activity` or `Object` then it may be
preferable to replace the resource with an `Activity` whose object is
a `Tombstone` value,
or to replace the resource with a `Tombstone` object.

### Access Control

ARFE is intended to enable collaborative editing.
For example a user can share a document,
and other participants in the conversation can edit the document.
But users would not expect that someone else would be allowed to edit each
other's comments.
Access control rules limit update operations and other actions to correspond to
users' expectations and preferences.

By default a user can edit the content and metadata of their own resources
(i.e. resources originating in email messages sent by or signed by the user).
Users can reword their own comments, edit their own shared documents, delete
activities, and so on.
By default users are allowed to reply to or forward / reshare messages as they
please.
An ARFE message that provides metadata for a resource may specify that other
users beside the original author are allowed to modify given properties of the
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

TODO: Policies declare specific sets of Activity Stream or ARFE properties that
a given set of people are authorized to change,
and possibly specific sets of allowed values for those properties.

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

An ARFE message is a table that may declare updates for multiple resources.
If a client determines that an update violates access control policies then the
client must ignore that update.
Any other updates in the same ARFE message that do not violate access control
policies are applied.

Likewise an access control policy cannot prevent users from replying or
forwarding.
A compliant client should indicate to the user that the owners of the
resource have indicated via a policy that they do not want content to be
reshared,
or that they do not want replies.
The recommendation is to prompt the user for confirmation in such cases,
and to allow the action if the user confirms.

Note that the access control policy of a resource can change over time.
The decision on whether an update operation is allowed must be based on the
policy in effect at the time when the operation was made.
Access control policy changes are updates just like content updates.
Every update points to a specific revision that it replaces.
This means that there is a canonical ordering that encompasses both access
control policy changes and content changes.
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
Content may be presented in a fallback part, or as an attachment.
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

## TODO

### Message level policies

How to handle message level policies? E.g., forwarding, replies

Option 1: Put access control policies in MIME and IMF headers instead of in
ARFE documents.
Message-level policies are listed in email message headers.
Policies cascade to apply to all nested resources unless overridden by a policy
on that resource.
The problem is that message-level policies could not be edited later unless
ARFE is expanded to allow updating at the message level.

Option 2: When forwarding, consider the most restrictive forwarding policy of
any resource that would be included in the forwarded message.



## References

- [Activity Streams 2.0][]
- [JSON-LD][]
- [RFC 2392][]: `mid:` and `cid:` URN schemes for email messages and MIME parts
- [RFC 5322][]: Internet Message Format, A.K.A. Email

[Activity Streams 2.0]: https://www.w3.org/TR/activitystreams-core/
[JSON-LD]: https://w3c.github.io/json-ld-syntax/
[RFC 2392]: https://tools.ietf.org/html/rfc2392
[RFC 5322]: https://tools.ietf.org/html/rfc5322
