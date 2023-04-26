## Authors:
- Phillis Tang <phillis@google.com>
- Daniel Murphy <dmurph@google.com>

## Participate
- [Issue tracker](https://github.com/philloooo/pwa-unique-id/issues)
- [Discussion thread](https://github.com/w3c/manifest/issues/586)


<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of Contents

- [Introduction](#introduction)
- [Background](#background)
  - [Relevant concepts](#relevant-concepts)
    - [manifest_url](#manifest_url)
    - [start_url](#start_url)
    - [scope](#scope)
    - [updatable](#updatable)
    - [global_id](#global_id)
    - [migration](#migration)
    - [processed field](#processed-field)
    - [Android implementations](#android-implementations)
    - [Desktop implementations](#desktop-implementations)
- [Problem statement](#problem-statement)
- [Requirements](#requirements)
- [Key Scenarios](#key-scenarios)
    - [Scenario 1](#scenario-1)
    - [Scenario 2](#scenario-2)
    - [Scenario 3](#scenario-3)
    - [Scenario 4](#scenario-4)
    - [Scenario 5](#scenario-5)
      - [Sub-scenario - update the scope](#sub-scenario---update-the-scope)
    - [Scenario 6](#scenario-6)
    - [Status today:](#status-today)
    - [Scenario 7](#scenario-7)
    - [Scenario 8](#scenario-8)
- [Out of scope](#out-of-scope)
- [Options for global_id](#options-for-global_id)
  - [Recommended: 1. start_url_origin + specified id](#recommended-1-start_url_origin--specified-id)
    - [Options for the default global_id](#options-for-the-default-global_id)
      - [processed start_url as default global_id (prefered option)](#processed-start_url-as-default-global_id-prefered-option)
      - [No specified default global_id](#no-specified-default-global_id)
      - [processed manifest_url as default global_id](#processed-manifest_url-as-default-global_id)
      - [start_url_origin + relative manifest_url as default global_id](#start_url_origin--relative-manifest_url-as-default-global_id)
      - [processed scope](#processed-scope)
      - [start_url_origin as default global_id](#start_url_origin-as-default-global_id)
  - [2. global_id = id (default processed manifest_url)](#2-global_id--id-default-processed-manifest_url)
  - [3. global_id = processed manifest_url](#3-global_id--processed-manifest_url)
    - [1.Always use the original_manifest_url as global_id](#1always-use-the-original_manifest_url-as-global_id)
    - [2.Support changing global_id for apps.](#2support-changing-global_id-for-apps)
  - [4. global_id = processed start_url](#4-global_id--processed-start_url)
- [Migration for user agents](#migration-for-user-agents)
  - [App initiated migration](#app-initiated-migration)
- [Conclusion](#conclusion)
- [Other topics](#other-topics)
    - [Should Service Worker have an identifier mechanism too? Is this related to PWA unique ID?](#should-service-worker-have-an-identifier-mechanism-too-is-this-related-to-pwa-unique-id)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

What uniquely identifies a web application is not defined in [app manifest specification](https://www.w3.org/TR/appmanifest/). This document describes and compares all the options for web app unique identifiers, and [proposes](#proposal) adding an `id` member to the manifest. 


## Background

https://web.dev/progressive-web-apps/ 

A web app manifest contains information about how an installed app should behave on a user's device. It’s provided as a `link` element in the document html. The link element can be a relative URL or an absolute URL with cross origin, it can also be a data URL with the full content embedded in the element.

Example manifest:


```
{
  "name": "Super Racer 3000",
  ...
  "scope": "/racer/",
  "start_url": "/racer/start.html",
  ...
}
```


When the user agent loads a web page, it fetches the manifest and parses the content according to the [Web App Manifest specification](https://www.w3.org/TR/appmanifest/#webappmanifest-dictionary). 

For a user agent, 3 things happen in the background:



1. It finds the installed app that matches the **identity **of the current manifest, and[ queues up a task](https://docs.google.com/document/d/1q5kmxNU7i4eem22LouMaIJ6123jOw5_zuxVQW1wfW8Y/edit) to update the installed app’s metadata, eg: scope, start_url, etc.
2. It checks whether the app matches installability criteria, if so and if the app is not already installed, it will allow users to install it..
3. It checks if the current URL falls under any installed apps’ scopes and allow users to launch the site to the installed app’s window.

A web app’s **identity** is also used outside of the user agent’s context:



*   [Web app origin association files](https://github.com/WICG/pwa-url-handler/blob/master/explainer.md#web-app-to-origin-association) used for URL handling.
*   PWA stores and app stores that include PWAs, eg: Google Play Store.

This document is focused on improving the user agents and third parties’ mechanism used to determine each web application's identity.


### Relevant concepts


#### manifest_url

The URL that’s specified by the `link` element for user agents to fetch manifest content.

Example:


```
Document URL: https://example.com
<link rel="manifest" href='manifest.json' />

-> processed manifest_url: https://example.com/manifest.json

Another example:
<link rel="manifest" href='https://www.other-origin.com/my-manifest.json' />

-> processed manifest_url: https://www.other-origin.com/my-manifest.json
```



#### [start_url](https://www.w3.org/TR/appmanifest/#start_url-member)

The URL that the user agent will load when users launch the app.

start_url is **processed **by the user agent to be a full URL as follows.



*   If the URL is specified as an absolute URL, eg: `https//example.com/abc/start.html`, it is required that the **origin is the same** as the document’s origin.
*   Otherwise manifest_url is used as the base\_url.
*   If it’s an empty string, default to the document's URL.

Example:


```
manifest_url: https://example.com/abc/manifest.json
start_url:../start.html 

-> processed start_url: https://example.com/start.html
```



#### [scope](https://www.w3.org/TR/appmanifest/#scope-member)

Scope defines the navigation scope of this web app. Sometimes a website wants to host multiple apps with the same origin or within the same path on an origin. Eg: `https://example.com/chat`, `https://example.com/doc`, `https://example.com/path/app1`, `https://example.com/path/app2`.

scope is **processed **by user agent to be a full URL as:



*   The default is the start_url’s base URL.
*   If scope is “.”, the default will also be used.  
*   If the scope is a relative URL, manifest_url is used as base URL.
*   If the manifest is hosted under another origin, the scope has to be specified as absolute URL with the same origin as the document URL.

Example:


```
start_url: http://example.com/start/abc
scope: ""
-> processed scope: https://example.com/start

manifest_url: https://example.com/manifest/manifest.json
scope: "../"
-> processed scope: https://example.com/
```



#### updatable

Able to update certain parts of the web app’s metadata and still be considered the **same app**. When a field is not **updatable**, it means when it changes, the app is recognized as a different app and can be installed as a second app to the user's device.


#### global_id

Id that guarantees global uniqueness, will be used by external PWA stores. Could be a composite key. This document will be exploring options for the global_id used to identify apps, as well as the default global_id to be used when it’s not specified.


#### migration

Backend of the user agent needs to internally migrate data. See the 
[migration](#migration-for-user-agents) section below for details here. This means adding implementation and operational costs for the specific user agent.

Desktop Chromium also supports syncing apps across users’ devices, migration can be done for devices that update to the latest version of Chrome, but old versions will get synced with two duplicate apps.


#### processed field

A **processed** field, eg: processed start_url is the returned full URL when user agents follow [W3 spec](https://www.w3.org/TR/appmanifest/#start_url-member) to parse the manifest. The processed field is saved in the user agent's db for this web app entity.

Example:


```
Document at https://example.com/home/
with <link rel="manifest" href='manifest.json' />
Manifest:
{
  …
  start_url: "../start",
  scope: ".",
  …
}
->
processed manifest_url: https://example.com/home/manifest.json
processed start_url: https://example.com/start
processed scope: https://example.com/
```



#### Android implementations

When referring to behaviors on Android, this doc is specifically talking about Chromium-based browsers on Android. iOS Safari doesn't support updating installed apps so this document has minimal impact on functionality on iOS Safari.

Firefox Android uses start_url as the app id and also supports updating installed apps, so it behaves similarly to Chromium Desktop on this subject.

KaiOS also supports PWAs.  The currently released devices uses the Open Web Apps (OWA) manifest from Firefox OS, while the new upcoming major release will use PWA manifests. The OWA manifest as well as the future PWA manifest uses manifest_url as the id and start_url is enforced to have the same origin as the manifest_url. Data url is not supported and only store initiated installs are supported right now (No browser initiated installs.). 

#### Desktop implementations

When discussing behaviors on desktop, this doc covers Chromium-based browsers. MacOS Safari and Firefox on desktop don't support installing PWAs.


## Problem statement

The [appmanifest spec](https://www.w3.org/TR/appmanifest/) doesn’t explicitly define what uniquely identifies a PWA. Currently, on the desktop versions of Chromium-based browsers and Firefox on Android, PWAs are uniquely identified by ` start_url ` and Android Chromium-based browsers use `manifest_url` instead. We need a uniformed mechanism to uniquely identify PWAs because:



*   There is no specified way of cross-browser support for apps to be **updatable**.
*   External entities to the browser have no reliable way to reference specific web apps. Eg: third party PWA stores, [web app origin associations](https://github.com/WICG/pwa-url-handler/blob/master/explainer.md#web-app-to-origin-association).


## Requirements



*   Backwards compatibility - existing installed apps on both Android and Desktop should continue to work and migrations should be handled appropriately if the ID mechanism is changed on a specific platform.
*   Allow web apps to be globally uniquely identifiable.
*   Be spoof-proof - We don't want apps stealing another app's identity to gain access to privileged APIs.
*   Allows manifest_url and start_url to be updatable.
*   Allows manifest_url to be hosted on a different origin than start_url (allowed by spec).
*   Allows manifest data URL to be updatable.
*   Any new ID scheme should be compatible with an older manifest without that scheme.
*   The ID should be serialisable so it can be referenced externally. Eg: in [web app origin association file](https://github.com/WICG/pwa-url-handler/blob/master/explainer.md#web-app-origin-association-file).

Extra considerations



*   Currently there is no expectations set for developers about which field ties to web app’s identity. They might have some **implicit** expectations based on their experience interacting with the specific user agent, e.g., Android developers could be updating start_url and knowing it doesn’t change the identity of their app. We should ensure for the new ID mechanism, the **expectation is clearly set** to allow them to perform reliable updates to web apps.
*   Implementation cost for user agents.


## Key Scenarios

The following example will be used in many scenarios:

https://www.example.com/index.html


```
<link rel="manifest" href="https://www.example.com/manifest.webmanifest"/>
```


https://www.example.com/manifest.webmanifest


```
{
  start_url: "index.html",
  scope: "/"
}
```



#### Scenario 1

A web app is installed on a user’s Android device.

On Chromium Android this app has an id minted using platform, browser, normalized_manifest_url. The WebAPK’s Android package name is org.chromium.webapk.<id>.


[Migration](#migration-for-user-agents) will have to be performed to support existing apps for options that do not use manifest_url as the identifier.


#### Scenario 2

A web app is installed on a user’s desktop device.

On Chromium this app has an id [minted ](https://source.chromium.org/chromium/chromium/src/+/master:chrome/browser/web_applications/web_app_install_finalizer.cc;l=130;drc=b669ad85b5fc10b3e7a9e70d5a470659843be39f;bpv=1;bpt=1) using start_url. The app is synced to all user’s desktop devices.


[Migration](#migration-for-user-agents) will have to be performed to support existing apps for options that do not use start_url as the identifier.


#### Scenario 3

A web app is installed both on Android and desktop devices.

A responsive app can use a single manifest for it’s Android and desktop users.  In this case, it’s keyed differently on these platforms according to Scenario 1 and 2.


#### Scenario 4

A web app needs to **update** its manifest_url.

For example, index.html could change to have the following other links:


```
<link rel="manifest" href="https://www.example.com/manifest2.webmanifest"/>
```


or


```
<link rel="manifest" href="https://www.real-cdn.com/example-com-manifest.webmanifest"/>
```


Use cases:

*   Urls are versioned. Example [1](https://github.com/w3c/manifest/issues/586#issuecomment-386820382), [2](https://github.com/w3c/manifest/issues/586#issuecomment-741540834) (e.g. 8.4% of manifests on https://pwa-directory.appspot.com seem versioned)
*   The site re-architected its directory structure, thus updating manifest URL too.
*   [Web apps serve different manifest URLs for different languages.](https://www.w3.org/TR/appmanifest/#internationalization)

Status today

*   Firefox Android - possible
*   Chromium desktop - possible
*   Chromium Android - not possible
*   KaiOS - not possible


#### Scenario 5

A web app needs to **update** its start_url.



*   Web apps can update the query params of the start_url. This is frequently used for attribution.
*   Web apps can move locations, especially for rebrandings / reorgs.

For example.html, the manifest could change to the following:


```
{
  ...
  start_url: "/FormalizedExampleBrand/index.html",
  scope: "/FormalizedExampleBrand/",
  ...
}
```


Status today



*   Firefox Android - not possible
*   Chromium desktop - not possible
*   Chromium Android - possible
*   KaiOS - possible


##### Sub-scenario - update the scope

When a start_url changes because of structural changes, it is often accompanied by a scope change. See the above example how the scope changes.


#### Scenario 6

A web app is installed with a data URL and wants to update the manifest.

Example:


```
<link rel="manifest" href='data:application/manifest+json,{ "name": "app name", "start_url": "home.html" }' />
```



#### Status today:



*   Firefox Android - possible
*   Chromium desktop - possible
*   Chromium Android - not possible
*   KaiOS - data URL is not supported


#### Scenario 7 

A web app has its manifest_url under a different origin from the document. Currently chrome on Android treats it the same as the same origin manifest_url. The full URL with the different origin is used to generate the app id.

Example:

Site’s URL is <code>https://example.com</code> with <code>link</code>


```
<link rel="manifest" href="https://real-cdn.com/manifest.webmanifest">
```



#### Scenario 8 

A web app is installed via [A2HS](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Add_to_home_screen). This app doesn’t need to have a manifest, but after it’s installed, it’s treated the same as an [installable](https://web.dev/install-criteria/) PWA by user agents. User agents will auto populate critical information like `start_url` and `scope` for the app when it’s installed.


## Out of scope



*   For a given id scheme, migrations between unique ids won’t be handled.
*   Migrating between origins.


## Proposal
Define an optional `id` field in the manifest. The id field would be an opaque string, and being parsed as an URL using start_url's origin as the base_url. See [detail about processing algorithm](https://pr-preview.s3.amazonaws.com/philloooo/manifest/pull/988.html#id-member). When `id` is undefined, use `start_url` as the default [global_id](#global_id). Below section covers in detail the pros and cons for the proposed solution (noted as `recommended`) and alternatives.

## Comparing options

### Recommended: 1. start_url_origin + specified id

Developer would specify an `id` field in the manifest. The id field would be an opaque string. [URL parsing](https://url.spec.whatwg.org/#concept-url-parser) `id` with `start_url_origin` as the base_url produces the `global_id`. If id is not specified, a default global_id will be used.

The default value will determine how existing web apps work.

This option covers the widest range of use cases, because a new id field is introduced, it allows other information to be updated. The downside is that it requires the parser to fetch the document to get the updated manifest when manifest_url is not stable. This will require more work for non-user agent entities like PWA stores and kiosks for updating apps.

These are general pros and cons for this option. There are more complete trade off comparisons based on which **default** global_id to use listed in subsections below.

Pros:

*   Allows metadata in manifest to be updatable, like manifest_url and start_url.
*   Have a fallback mechanism for apps without an explicit ID defined.
*   manifest_url\_origin can be changed

Cons:

*   A new field is defined, work needed for app developers to define this if they don’t want to rely on the default identifier.
*   [Migration](#migration-for-user-agents) needs to be done for existing browser implementations that don’t use the default identifier to index PWAs. 
*   The global_id is not a resolvable URL, this means the identifier can not be used solely to make the app discoverable.


#### Options for the default global_id

This subsection explores options for the default global_id to use when `id` is not specified. These are sub-options under the `global_id = start_url_origin + specified id`, so they share the same pros and cons for the overall option, but have more trade offs compared with each other.


##### processed start_url as default global_id (prefered option)

global_id defaults to processed start_url when id is not specified.

This matches the existing desktop implementation.

example.com global_id = `https://www.example.com/index.html`

developer specified id to match = `index.html`

Pros:

*   Align with the existing situation of desktop apps. No migration needs to be done for [S2 - installed desktop apps. ](#scenario-2)
*   [S4](#scenario-4), [S6](#scenario-6), [S7](#scenario-7) are supported without needing to set the `id`.
*   [S5: Apps that need to change start_url](#scenario-5)  will set the `id` field to previous start_url to keep the identity.
*   [S8: web apps that don’t have manifest URLs](#scenario-8) can still be keyed consistently. 

Cons:

*   [S1: installed apps on Android](#scenario-1) need to handle [migration](#migration-for-user-agents).
*   For [S3: Apps that are installed on both Android and desktop](#scenario-3), Android needs to handle migration.
*   Change of implicit expectations for Android web app developers

Other considerations:

*   [start_url spec](https://www.w3.org/TR/appmanifest/#start_url-member) **recommends** user agents to allow users to change start_url. user agents will need to store a separate user preferred start_url in this case. However this is not implemented by any user agent right now.


##### No specified default global_id

The user agents will be free to use their existing id mechanism as the default, and the specification will not require a specific default. 

Apps can “upgrade” to use an explicit id by performing an 
[app initiated migration](#app-initiated-migration).

Pros:

*   No migration needs to be done for [S1: installed apps on Android](#scenario-1), [S2 - installed desktop apps, S7: manifest is hosted under a different origin.](#scenario-2)
*   [S5: Apps that need to change start_url](#scenario-5) on **desktop** can perform explicit [app initiated migration](#app-initiated-migration) by specifying both the `id` and a `legacy_ids` field, or they can just set the `id` to `start_url - origin` to keep the existing identity.
*   [S4: Apps that need to change manifest_url](#scenario-4), [S6: Manifests with data URL](#scenario-6) on **Android** will perform explicit [migration](#migration-for-user-agents) by specifying both the `id` and a `legacy_ids` field. Or if the manifest is under the same origin, they can set the `id` to `manifest_url - origin` to keep the existing identity.
*   For [S3: Apps that are installed on both Android and desktop](#scenario-3), they will include both manifest_url and start_url in legacy_ids to migrate to a consistent id. User agents will look up with the legacy field that matches with their previous implementation to find existing apps to migrate.
*   No change of expectations for all web app developers

Cons:



*   Since there isn't a consistent default to be used, third party PWA stores might only index web apps with an explicit `id` field, or choose a default on their own for apps without `id` specified. This will cause issues for developers who want to adopt explicit IDs, as it’s unclear whether it will be recognized by the PWA stores as the same app.
*   User agents might forever need to support migrating a non-id app to an id-app.
*   Prevent future syncing apps between Android and desktop Chrome for apps that don't have id specified.

The Cons might be acceptable because if we choose to not have specified default value, we will strongly encourage developers to migrate to specify the `id` field. We can have a warning in the lighthouse saying that the app will not be installable after X releases, unless an `id` is added.


##### processed manifest_url as default global_id

global_id defaults to processed manifest_url when id is not specified.

This matches the existing Android implementation.

example.com global_id:


```
https://www.example.com/manifest.webmanifest
```


example.com with cdn-manifest global_id:


```
www.real-cdn.com/example-com-manifest.webmanifest
```


Pros:

*   Allows [S5: apps to change start_url](#scenario-5) without adding id field.
*   [S4: Apps that need to change manifest_url](#scenario-4) will add the id field which equals previous manifest\_ur’s pathl to keep the identity.
*   [S1: installed apps on Android](#scenario-1) keep working without needing to add an id field.

Cons:

*   [S6: Manifests with data URL](#scenario-6) won’t be supported/updatable.
*   Change of implicit expectations for desktop WebApp developers
*   [S7: manifest is hosted under a different origin](#scenario-7): could be a security concern to use a URL from different origin as the global key. 
*   When [S7: manifest is hosted under a different origin](#scenario-7) needs to change manifest_url, they can’t create an id field that matches previous manifest_url to keep the identity.
*   [S2: installed apps on desktop](#scenario-2) need to handle [migration](#migration-for-user-agents).
*   For [S3: Apps that are installed on both Android and desktop](#scenario-3), desktop needs to handle migration.
*   [S8: web apps that don’t have manifest URLs](#scenario-8) won’t be supported, they will have to use a different ID system and need to be migrated when they “upgrade”to have a manifest_url.

For 
[S6: Manifests with data URL](#scenario-6), we either don’t support updating it, or make a special case to use start_url as default only for this. But this will make the spec messy.


##### start_url_origin + relative manifest_url as default global_id

global_id defaults to start_url_origin + relative manifest_url when id is not specified.

Replaces the manifest_url’s origin with the start_url_origin when they are different.

example.com global_id:


```
https://www.example.com/manifest.webmanifest
```


example.com with cdn-manifest global_id:


```
https://www.example.com/example-com-manifest.webmanifest
```


Pros:

*   Allows [apps to change start_url](#scenario-5) without adding id field.
*   [S4: Apps that need to change manifest_url](#scenario-4) will add the id field which equals previous manifest_url to keep the identity.

Cons:

*   [S6: Manifests with data URL](#scenario-6) won’t be supported.
*   Non-intuitive design - might increase confusion and complexity for developers and user agents.
*   [S1: installed apps on Android](#scenario-1), [S2: installed apps on desktop](#scenario-2), [S3: Apps that are installed on both Android and desktop](#scenario-3) needs to handle [migration](#migration-for-user-agents).


##### processed scope

global_id defaults to processed scope URL when id is not specified.

example.com global_id:


```
https://www.example.com/
```


Pros:

*   Conceptually more aligned with the identity of the app.
*   [S4](#scenario-4), [S5](#scenario-5), [S6](#scenario-6), [S7](#scenario-7) are supported without needing to set the `id`.
*   Should be more stable than start_url & manifest_url
*   Apps that need to change scope will set the `id` field to previous scope to keep the identity.

Cons:

*   Change of implicit expectations for Android and desktop app developers.
*   No existing browser implementation uses the scope to key web apps. [S1: installed apps on Android](#scenario-1), [S2: installed apps on desktop](#scenario-2), [S3: Apps that are installed on both Android and desktop](#scenario-3) needs to handle [migration](#migration-for-user-agents).


##### start_url_origin as default global_id

global_id defaults to start_url_origin when id is not specified.

example.com global_id:


```
https://www.example.com
```


Pros:

*   Simplicity. Makes sense to have an empty string as default for the id field.
*   [S4](#scenario-4), [S5](#scenario-5), [S6](#scenario-6), [S7](#scenario-7) are supported without needing to set the `id`.
*   Most stable option - least frequent for apps to change origin.

Cons:

*   For a site that has multiple apps under the same origin, they will need to be migrated by developers to add an id field for each app and tell users to uninstall the old apps.
*   Share all the same cons as using `scope`. 


### 2. global_id = id (default processed manifest_url)

This option is listed here to compare it with the first option. We can have an id field that’s required to use the URL scheme and just use this as the global_id.

The pros for this option is it’s simpler than the first one, you don’t need to get two values and concatenate them. The obvious con is that users can spoof the IDs to be under whatever origin they want. That’s the reason we had the id prefixed with the document's origin for the first option.

Takeover example, where any site could have a manifest that could apply to any app:


```
{
  ...
  id: 'https://www.google.com/manifest.json',
  start_url: "https://malware.com/step_3_profit",
  ...
}
```


Alternatively we can require the ID to have the same origin as the start_url. So ID will be validated as :



*   Has to be a url with https:// scheme
*   Has to have the same origin as start_url.

This makes this approach functions the same as the first option: 
`start_url_origin + ID`. The difference is being fully explicit vs the first option being more terse. Since the origin part is required to be the same, it's redundant information, so the first option is preferred.


### 3. global_id = processed manifest_url

The processed absolute manifest_url used as the global_id. This is how Android operates today.

This means changing manifest_url will change the global_id. We already have lots of apps that do change manifest_url, see 
[Scenario 4](#scenario-4), and it’s **important **to support this use case.

There are two possible ways to mitigate this problem:


#### 1.Always use the original_manifest_url as global_id

Document always sticks to link to the original manifest_url, the original manifest_url redirects to latest manifest_url. User agents will be required to follow 301 redirects to fetch the latest content, but the global_id stays the same. This also means if there are PWA stores that do web scrawls, they will get multiple manifest_urls, and they will need to du-dup them. This basically means users will need to stick with the same manfiest\_url forever. And change of CDN and document structure won’t be allowed as the previous url is dead. So 
[Scenario 4](#scenario-4) is not really supported.


#### 2.Support changing global_id for apps.

Document uses the latest manifest_url, old manifest_url redirects to the latest, so that user agents with an installed app can follow the old manifest_url to find it’s global_id is changed. This means the global_id **won’t be stable**, and it would require user agents to support migrating apps’ ids as a feature.

In reality it could be quite problematic to support this, as the global_id is used as the primary key for apps synced across devices. If an app with manifest\_v1 is installed, and the user browses to an installable site on a tab, the user agent needs to go through all existing installed apps, refresh their manifest, then check if the site is installed or not. This will substantially affect the performance of the user agents to do a O(N) operation.

Also, if manifest\_v1 is installed on an offline device 1 and manifest\_v2 is installed on an offline device 2, and sync is later turned on, it will result in temporary duplicated apps before the browser finishes the manifest update and dedupe it.  In general it’s very error prone to make the primary key changeable.

Overall, if we use manifest_url as the global_id, we need to **not support** the use case of making manifest_url updatable or deal with **non stable primary keys** for managing apps across devices. The upside of this approach is that the manifest_url is a **resolvable URL**. The global_id can be used alone to discover the app, however that’s a “nice to have” feature, but not a core requirement for the global_id. Another benefit is that it allows non user agent entities having a simpler flow to keep the app up to date - they just need to save the manifest_url as the key and periodically fetch it. If there weren't existing apps & infrastructure that relied on changing manifest_url, and this was being designed from the beginning, this option is a lot more attractive.

example.com global_id:


```
https://www.example.com/manifest.webmanifest
```


example.com with cdn-manifest global_id:


```
https://www.real-cdn.com/example-com-manifest.webmanifest
```


Below pros and cons use the first option to support 301 redirects - that is, always use the original manifest_url as global_id. As the second approach is not reliable.

Pros

*   Simplicity. A single existing field can be used instead of introducing a new field.
*   The id is a meaningful URL, external parties that use this id to reference the web app can use it directly to get the metadata of the web app.
*   Allows [S5: apps to change start_url](#scenario-5) without special handling.

Cons	

*   [S2: installed apps on desktop](#scenario-2), [S3: Apps that are installed on both Android and desktop](#scenario-3) needs migration.
*   Change of implicit expectations for desktop WebApp developers
*   **[S6: Apps that specify manifest as data URL](#scenario-6) will not be updatable**
*   PWA stores will recognize different versions of manifest URLs as different web apps. Alternatively they can load the start_url to find the manifest link, and follow redirect to find the latest manifest_url and use that as the authoritative manifest. 
*   [S4: Apps that need to update manifest URLs](#scenario-4) will need to keep supporting the previous manfiest\_url, handle 301 to the latest manifest forever. The document will need to always use the original manifest_url. An extra roundtrip will always be needed for loading the app.
*   If the manifest_url is hosted under a CDN, they will need to stick to that CDN provider.
*   Difficult to reconcile with [S8: apps installed from A2HS](#scenario-8), which does NOT require a manifest. ID creating here is weird, and instead would have to have custom upgrade handling for when a manifest is found on the page.


### 4. global_id = processed start_url

The processed absolute start_url. This is how Desktop web operates today.

If apps want to change the start_url, it will be recognized as a new app. We could require user agents to support 301 redirect, in this case, the **manifest still always needs to use the original start_url **to be recognized as the same app** **, and app developers will just make the original start_url redirect to the latest version of start_url.

example.com global_id:


```
https://www.example.com/index.html
```


Pros



*   Simplicity. A single existing field can be used instead of a new field.
*   [S4](#scenario-4), [S6](#scenario-6), [S7](#scenario-7) are supported

Cons	

*   [S1: installed apps on Android](#scenario-1) need migration.
*   Change of implicit expectations for Android WebApp developers
*   [S5: Apps that need to update start URLs](#scenario-5) will need to keep supporting the previous start_url, handle 301 to the latest start_url. The document will need to always use the original start_url. An extra roundtrip will always be needed for loading the app. This might not make sense if the reason for changing start_url is for attribution.


## Migration for user agents

For solutions that add a new id field, if the default id used **doesn’t match** the current user agent’s id solution, **migration** needs to be done for installed apps.

To avoid breaking developers’ existing expectations, we can perform the migration in two phases. Or if it’s too complicated, we can just do phase 2 directly.

Phase 1

Support the new `id` field but still use the previous id solution as the default. 

Support 
[app initiated migration](#app-initiated-migration). Additionally we can have a warning in the lighthouse to encourage developers to move to the new `id` field, and state that the new default value will be used if id is not specified after X releases.

Phase 2

Migrate to use the new default value automatically by user agents.

**Main & hardest complication for desktop:** old versions of chrome do not 'ignore' synced apps with malformed AppIds. Any migration here will cause those old versions to have broken & probably unstable webapps, which could cause crashes (this needs to be tested to see if it is salvageable).


### App initiated migration

An app initiated migration means the app will specify a legacy field to instruct the user agent to look up installed apps and perform backend migration to the new ID. The main advantage of app-initiated migration is that  developers get to opt into having their app ID changed. . By contrast, if user agents perform app ID migration automatically, the developers may not be aware of the migration that their apps are subjected to. In a worst-case scenario, developers would learn about app ID migration when an unintended consequence causes their apps to break.

On Android:



1. Apps without an ID field continue to use manifest_url to generate ids.
2. New apps with an ID field specified will use the new ID.
3. Apps that already exist that want to use the ID field will specify both an ID field and a **legacy_ids_** field. This instructs user agents to look up installed apps by legacy_ids to do a **backend** **migration** to the new ID scheme.

On desktop:

1. Apps without an ID field continue to use start_url to generate ids.
2. New apps with an ID field specified will use the new ID.
3. Apps that already exist that want to use the ID field will specify both an ID field and **legacy_ids_** field. This instructs user agents to look up installed apps by legacy_ids to do a **backend** **migration** to a new ID scheme.

Alternatively, we can have separate legacy_manifest_url and legacy_start_url field to be explicit. The Cons with this approach is we have one more field and they are platform-specific.

Pros:

*   Be specific so the app developers know exactly which field they are migrating from.

Cons:

*   Platform specific and not generic. This could be a anti-pattern for manifest design.

When legacy_ids is presented, exactly how to migrate will be an implementation detail of the user agent.

For Chromium desktop, these are the possible ways to do the migrate:

Plan A:

When a manifest is loaded, look up existing apps with id = &lt;new id>/&lt;legacy_ids>. 

If found, mark the old app as “deprecated”. Hide it from chrome://apps. create a new app. Remove local shortcuts for the old app.

If not found, save the new app as the new id.

Plan B:

When a manifest is loaded, look up existing apps with id = &lt;new id> or &lt;legacy_ids>. 

If found, apply updates without changing the id.

If not found, save the new app as the new id.

Plan A could cause some friction when swapping the local shortcuts and app instances. Any local changes saved for the old app before the user closes the old app won’t be transferred to the new local app.

Plan B won’t have this problem, but it will leave installed apps with old ids for a longer time.

For **previous versions of Chrome**, if it receives a sync entity from other devices with the new ID, it will be recognized as a separate app. 


## Conclusion

Using existing manifest_url/start_url would be bad because there are multiple use cases that require changing them. There are ways to mitigate changing of manifest_urls if old ones can still be supported, but they are sub-optimal and don’t satisfy all the use cases of manifest_urls updating. Having the global_id to be a resolvable URL is a pros for using manifest_url, it’s a “nice to have” property, but not a required feature for the global_id. 

With the primary goal of ensuring having a stable identifier to support various use cases of updating app's metadata. Using [start_url_origin + id](#recommended-1-start_url_origin--specified-id) is preferred as this is the most stable option.

A fallback default id is supported to allow smooth transition from existing platforms where an **id** field doesn't exist in the manifest. Therefore when considering which default field to choose, being backward compatible is the key consideration.

[start_url](#processed-start_url-as-default-global_id-prefered-option) is preferred as the default **global_id** because:

- It matches the exiting behavior of two implementations (Chromium Desktop & Firefox Mobile), allowing easy transition for the codebase of those browsers & developer expectations.
- Chromium Android (and other manifest_url -based designs) have to migrate anyways to support data urls and cross origin manifest_url.
- Since the **id** field can be recommended & eventually required for installability, this default doesn't matter as much once all manifests specify an **id**.

User agent implementations should strongly encourage apps to specify a stable **id**, and potentially add it to installability criteria after apps adopt to specify the **id** field.

## How do app developers adopt to use the manifest ID

If you are creating an app from scratch (no existing installs to update)
id is just a string, like a UUID, that uniquely identifies the app on the start_url_origin

(at https://www.new-app.com/):

{
  ...
  id: "NewAppId",
  start_url: "/index.html", // <- updatable now
  ...
}
The global id evaluates to https://www.new-app.com/ + NewAppId = https://www.new-app.com/NewAppId. This is not intended to be evaluate-able, but looks like a URL.

If you have an existing app (installs exist where the manifest did NOT have an id set)
id should be the relative path of the start_url so that the new manifest (with id specified) will apply, and thus update, the old manifest. EX:

old (at https://www.example.com/):

{
  ...
  start_url: "/index.html"
  ...
}
The default global id is: https://www.example.com/index.html

new:

{
  ...
  id: "index.html",
  start_url: "/index.html", // <- now this is updatable!
  ...
}
The global id evaluates to https://www.example.com/ + index.html = https://www.example.com/index.html, thus matching the old manifest.

Hopefully that makes sense?

(example of updating the start_url of the app)

{
  ...
  id: "index.html",
  start_url: "/nested/index.html",
  scope: "/nested/",
  ...
}

## Other topics


#### Should Service Worker have an identifier mechanism too? Is this related to PWA unique ID?

This is tracked as a separate [issue](https://github.com/w3c/ServiceWorker/issues/1512). Service workers are not required/needed to have a 1-to-1 mapping with PWA. Thus the identification of a PWA should be independent of its service workers. See more discussion on github [thread](https://github.com/w3c/manifest/issues/586#issuecomment-609513249).

