---
title: Interfaces for Humans on an API-First Platform
description: APIs are for people too, so let's stop treating developers like machines.
thumbnail: photo-1536104968055-4d61aa56f46a.jpg
date: 2020-04-20 19:35:45
updated: 2020-11-24 21:18:51
tags:
    - Product Management
    - C Sharp
---

<!-- markdownlint-disable no-inline-html -->
![Image](photo-1536104968055-4d61aa56f46a.jpg)<span class="caption">Image: [True Agency](https://unsplash.com/@trueagency)</span>
<!-- markdownlint-enable no-inline-html -->

_Update 2020-11-21_: I particularly enjoyed [this post](https://www.arp242.net/api-ux.html) by [Martin Tournoij](https://www.arp242.net). I think he captures the type of empathy for developers that I struggled to express to the revenue-driven side of our business before I invested in my product management chops. Poorly designed and documented APIs contribute to (though are not entirely the cause of) poorly executed integrations - which is surely felt when they add risk to strategic partnerships. Back to regularly scheduled programming...

During my time at [MindTouch](https://mindtouch.com), the early design decision to build an [API-first platform](https://swagger.io/resources/articles/adopting-an-api-first-approach) has yielded some _amazing_ technical gains that I benefited from as a software architect, but also some real pains that I experienced years later as a technical product manager positioning the same API for partner developers and system integrators.

[Steve Bjorg](https://twitter.com/bjorg), Founder/CTO of MindTouch, explained the decisions around API-first in 2007...

{% youtube XxSjNkkqyoo %}

In 2019, one of MindTouch's principal engineers, [Juan Torres](https://twitter.com/onema), elaborated on the downstream benefits of MindTouch's event-driven service model, which benefits greatly from Steve's original API-first vision...

{% youtube mxKhbU_ToMs %}

Steve's original concept, combined with MindTouch's strong open-source friendly outlook at the time, were the core reasons I joined the organization. However, as the years progressed and MindTouch began to reposition itself for more turn-key use cases, I sensed a lack of ownership over the developer experience, as IT system admin and integrator personas and needs lacked strong understanding or empathy from our organization. As a result, our engineers added new functionality to the shared API simply to facilitate the creation of new features in the product experience, with neither them nor product managers considering how these APIs could be leveraged directly to create value.

When I took on the challenge of positioning the API to solve partner and B2B customer integration problems, I found that years of neglect had led to poorly maintained documentation and inconsistency across API input parameters and response data structure patterns. API endpoints that were clearly only created for internal DevOps needs were publicly exposed. Access to these endpoints was fortunately controlled, but still resulted in a confusing experience for partner developers.

Unfortunately, due to lock-in with [our proprietary RESTful API framework](https://github.com/MindTouch/DReAM), we lacked some of the "auto-documenting" capabilities of tools like [Swagger](https://swagger.io). After building similar capabilities into our framework, it was time to enforce some sort of standard for API visibility and documentation. I wrote and shared the following document with our engineering team to reintroduce this discipline. I'm sharing this document publicly to demonstrate a level of control a product manager may need to place on the interface, to provide a consistent _documentation_ experience. After API parameters and descriptions are documented consistently and intuitively, the natural next step is to apply standards (possibly in the form of a design spec) on the input and output data structures themselves.

A UI/UX designer considers intuitiveness and consistency with their approach with graphical user interfaces, why not take the same approach with a developer's interface, an API?

---

## API Feature Development

As a [product capability](https://success.mindtouch.com/Integrations/API), the public API requires consistency in its presentation to customers and partner developers. Following these conventions and best practices ensures that the public API documentation and experience will remain consistently high quality.

### API Visibility

All new API features (endpoints) are initially created in one of two allowed states: _hidden_ or _internal_. Neither hidden nor internal features appear in API documentation. Depending on how permissions are checked, hidden features can be accessible by any user type or role, whereas internal features can only be access by customer site administrators or internal MindTouch staff.

With few exceptions, new features _require_ token validation and include an attribute to ensure that _all_ sites enforce this behavior on these features. This includes older sites that do not yet have token validation enforcement enabled as the default behavior for _all_ API features.

Some internal features are intended to only be used by Engineering, DevOps, and Support whilst on the MindTouch employee network or VPN. These features are marked as _trusted internal features_ with an attribute.

```csharp
// Hidden features have a public visibility keyword, so that they can
// be accessed by non-administrators in user experience, however the
// feature remains hidden in API documentation until it is reviewed by
// the API product manager.
[DreamFeature("GET:widgets", "Retrieve list of widgets", Hidden = true)]

// Here is the one use case for disabling authorization or API token
// enforcement: downloading log reports, images, file attachments, etc.
// Browsers download these resources as media files and will not send
// authorization if the user is anonymous.
[DreamFeature("GET:files/{filename}", "Download file")]
[DreamFeature("GET:users.csv", "Download list of users")]
[TokenNotRequiredFeature]
public DreamMessage GetFile(IAttachmentsBL attachmentsBL) { }

// Internal features have a internal visibility keyword and can only be
// accessed by internal MindTouch staff or customer site administrators.
// Internal features are intended to never be visible in API documentation.
[DreamFeature("PUT:site", "Destroy a site")]

// Add this attribute if the internal feature is intended by used internally
// by DevOps
[TrustedInternalFeature]
internal DreamMessage PutSite(INukeBL nukeBL) { }
```

### API Feature Descriptions

API descriptions use common semantics and phrases to articulate what they are used for. While these are the general rule of thumb, exceptions can be made for XML-RPC style features (move, copy, etc), that do not follow RESTful conventions. As necessary with any product messaging, the API product manager will assist you in providing the correct phrasing and tone.

```csharp
// GET features "retrieve" resources.
// A feature that retrieves more than one resource, retrieves a "list".
[DreamFeature("GET:widgets", "Retrieve list of widgets")]
[DreamFeature("GET:widget/{id}", "Retrieve widget")]

// GET features "download" resources if they have a
// "Content-Disposition" header
[DreamFeature("GET:files/{filename}", "Download file")]
[DreamFeature("GET:users.csv", "Download list of users")]

// PUT features "update" a resource.
// The resource is never prefixed with an article ("a", "an", "the").
[DreamFeature("PUT:widgets/{id}", "Update widget")]

// POST features "create" or "create or update" a resource
// (depending on the behavior of the method).
// The resource is never prefixed with an article ("a", "an", "the").
[DreamFeature("POST:widgets", "Create widget")]
[DreamFeature("POST:widgets", "Create or update widget")]

// DELETE features "remove" a resource.
// The resource is never prefixed with an article ("a", "an", "the").
[DreamFeature("DELETE:widgets/{id}", "Remove widget")]
```

### API Feature Parameters

API feature parameters also require consistency in phrasing and tone. Several parameters such as `{pageid}`, `{fileid}`, `{userid}`, and `{groupid}` are reused through different features, and are resolved from the request in a common way (ex: `pageid` can be an integer, "home", a `=` prefixed title, or a `:` prefixed GUID).

```csharp
[DreamFeature(
    "GET:pages/{pageid}/files/{filename}",
    "Retrieve page file attachment"
)]

// there are constant descriptions for common parameters
[DreamFeatureParam("{pageid}", "string", PARAM_PAGEID_DESCRIPTION)]
[DreamFeatureParam("{filename}", "string", PARAM_FILENAME_DESCRIPTION)]

// parameter descriptions never end with a "." character
[DreamFeatureParam("description", "string?", "File attachment description")]

// default values on optional values are expressed with (default: {value})
[DreamFeatureParam(
    "limit", "int?", "File attachment download size limit (default: 2048)"
)]
```

### Obsolete API Features

Obsoleting a feature means either a different feature should be used, or the feature is sunsetted entirely. Marking a feature obsolete has the same effect as hiding it, so the Hidden property is redundant.

```csharp
// directing users to a new feature is expressed by "Use {feature}" in
// the Obsolete property.
[DreamFeature(
    "GET:site/widgets",
    "Retrieve list of widgets",
    Obsolete = "Use GET:widgets"
)]

// the method for a sunsetted feature is also marked with an
// ObsoleteAttribute, and an API product manager approved sunset
// message is set in the Obsolete property.
[Obsolete]
[DreamFeature(
    "GET:widgets",
    "Retrieve list of widgets",
    Obsolete = "Widgets have been sunsetted are no longer available"
)]
```
