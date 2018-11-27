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
