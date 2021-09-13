# How to experiment with manifest id in Chrome
A prototype of the manifest unique id is present in Chrome 94 for **desktop** OSes. **This is intended for experimentation purposes and is subject to change as a specification takes shape and develops.**

We will do our best to keep these instructions up to date as protocol and API changes occur.

As of approximately August 18, 2021, it is available on Chrome Canary channel. It will subsequently roll into downstream channels.

# Enabling manifest id member in Chrome
1. Download a version of Chrome with manifest id implemented (>=M95). ([Link for Canary channel](https://www.google.com/chrome/))
2. Enable feature flag chrome://flags/#enable-desktop-pwas-manifest-id
3. Navigate to a Relying Party page that specifies the `id` field.

## Relying party implemetation
Specify `id` in the manifest. If this application is already under use, specify `id` to match the start_url to keep the existing app identifier to be backward compatible.
Example manifest before migrating to use `id`:
```
{
  ...
  start_url: "start?a=b",
  ...
}
```
migrate to specify `id`:
```
{
  ...
  start_url: "start?a=b",
  id: "start?a=b",
  ...
}
```

To verify the computed app id after app installation, open devtool, go to "Application", you should see "App Id" displayed in Identity section.
