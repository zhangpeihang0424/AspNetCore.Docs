---
title: SameSite
author: rick-anderson
description: Learn how to react to SameSite changes in ASP.NET Core
ms.author: riande
ms.custom: mvc
ms.date: 11/11/2019
uid: security/samesite
---
# React to SameSite changes in ASP.NET Core

By [Rick Anderson](https://twitter.com/RickAndMSFT)
<!-- >
Work this in 
Each ASP.NET Core component that emits cookies need to decide if `SameSite` is appropriate. 

https://docs.microsoft.com/en-us/dotnet/api/system.identitymodel.services.wsfederationauthenticationmodule?view=netframework-4.8

-->

The [SameSite 2016 draft](https://tools.ietf.org/html/draft-west-first-party-cookies-07) states:

  This document updates [RFC6265](https://tools.ietf.org/html/rfc6265) by defining a  a `SameSite` attribute which allows servers to assert that a cookie ought not to be sent
   along with cross-site requests. This assertion allows user agents to mitigate the risk of cross-origin information leakage, and provides some protection against cross-site request forgery attacks.

Firefox and Chrome based browsers are making breaking changes to their implementations of [SameSite](https://tools.ietf.org/html/draft-west-first-party-cookies-07) for cookies. The SameSite changes impact remote authentication scenarios like [OpenID Connect](https://openid.net/connect/) and [WS-Federation](https://auth0.com/docs/protocols/ws-fed). With this change, OpenID Connect and WS-Federation must opt out by sending `SameSite=None`. However, setting `SameSite=None` doesn't work on iOS 12 and some older versions of other browsers. To support iOS 12 and older browsers, ASP.NET Core app's must detect these browsers and omit `SameSite`.

## SameSite draft standard

The [SameSite 2016 draft](https://tools.ietf.org/html/draft-west-first-party-cookies-07#section-4.1) extension to HTTP cookies:

* Was intended to mitigate cross site request forgery (CSRF).
* Was designed as a feature servers would opt into by adding the new "SameSite" attribute and attribute values to cookies.
* Is supported in ASP.NET 2.0 and later.

## New (2019) SameSite draft standard

The new [SameSite 2019 draft](https://tools.ietf.org/html/draft-west-cookie-incrementalism-00):

* Is not backwards compatible.
* Cookies should be treated as `SameSite=Lax` by default.
* Cookies that explicitly assert `SameSite=None` in order to enable cross-site delivery should be marked as `Secure`. `None` is a new entry to opt out.

`Lax` is OK for most application cookies but breaks cross site scenarios like OpenIdConnect and Ws-Federation login. Most [OAuth](https://oauth.net/) logins are not affected due to differences in how the request flows. The new `None` parameter causes compatibility problems with clients that implemented the prior draft standard (for example, iOS 12). Chrome browsers (Chrome 80) are expected to go live in February 2020 with the 2019 SameSite draft standard.

::: moniker range="= aspnetcore-3.1"
ASP.NET Core 3.1 provides the following SameSite support:

* Redefines the behavior of `SameSiteMode.None` to emit `SameSite=None`
* Adds a new value `SameSiteMode.Unspecified` to omit the `SameSite` attribute.
* All cookies APIs default to `Unspecified`. Some components that use cookies set values more specific to their scenarios, for example
  * The `OpenIdConnect` correlation.
  * `nonce` cookies.

::: moniker-end

::: moniker range=">= aspnetcore-3.0"

Each ASP.NET Core component that emits cookies need to decide if `SameSite` is appropriate. ASP.NET Core 3.0 and later aligns SameSite defaults with the new draft standard. The following API's have changed the default from `SameSiteMode.Lax ` to `SameSiteMode.None`:

* <xref:Microsoft.AspNetCore.Http.CookieOptions> used with [HttpContext.Response.Cookies.Append](xref:Microsoft.AspNetCore.Http.IResponseCookies.Append*)
* <xref:Microsoft.AspNetCore.Http.CookieBuilder>  used as a factory for `CookieOptions`
* [CookiePolicyOptions.MinimumSameSitePolicy](xref:Microsoft.AspNetCore.Builder.CookiePolicyOptions.MinimumSameSitePolicy)

ALL ASP.NET Core components that emit cookies:

* Override the preceding defaults with settings appropriate for their scenarios.
* The overridden preceding default values have not changed.

| Component | Default |
| ------------- | ------------- |
| <xref:System.Web.HttpContext.Session>  | `Lax` |
| <xref:Microsoft.AspNetCore.Mvc.ViewFeatures.CookieTempDataProvider>   | `Lax` |
| `Antiforgery` | `Strict` |
| `CookieAuthentication` | `Lax` |
| `TwitterAuthentication` state cookie | `Lax`  |
| `RemoteAuthentication` correlation cookie (OAuth): | `None` |
| [OpenIdConnectOptions.NonceCookie](xref:Microsoft.AspNetCore.Authentication.OpenIdConnect.OpenIdConnectOptions.NonceCookie)| `None` |
::: moniker-end

## Test apps for SameSite problems

Apps that interact with remote sites such as through 3rd party login need to:

* Test the interaction on multiple browsers.
* Apply the [CookiePolicy browser detection and mitigation]() discussed in this document. See testing and browser detection in this document.

## Testing

Test web apps using a client version that can opt-in to the new SameSite behavior. Chrome, Firefox, and Chromium Edge all have new opt-in feature flags that can be used for testing. After your app applies the SameSite patches, test it with older client versions, especially Safari. For more information, see [Supporting older browsers](#sob) in this document.

### Test with Chrome

Chrome 78+ gives misleading results because it has a temporary mitigation in place. The Chrome 78+ temporary mitigation allows cookies less than two minutes old. Chrome 76 or 77 with the appropriate test flags enabled provides more accurate results. To test the new SameSite behavior toggle `chrome://flags/#same-site-by-default-cookies` to enabled. Older versions of Chrome (75 and below) are reported to fail with the new `None` setting. See [Supporting older browsers](#sob) in this document.

Google does not make older chrome versions available. Follow the instructions at [Download Chromium](https://www.chromium.org/getting-involved/download-chromium) to test older versions of Chrome. Do **not** download Chrome from from links provided by searching for older versions of chrome.

[Chromium 76 Win64](https://commondatastorage.googleapis.com/chromium-browser-snapshots/index.html?prefix=Win_x64/664998/)
[Chromium 74 Win64](https://commondatastorage.googleapis.com/chromium-browser-snapshots/index.html?prefix=Win_x64/638880/)

### Test with Safari

Safari 12 strictly implemented the prior draft and fails when the new `None` value is in a cookie. `None` is avoided via the browser detection code shown below. Test Safari 12, Safari 13, and WebKit based OS style logins using MSAL, ADAL or whatever library you are using. Note that the problem is dependent on the underlying OS version. OSX Mojave (10.14) and iOS 12 are known to have compatibility problems with the new SameSite behavior. Upgrading the OS to OSX Catalina (10.15) or iOS 13 fixes the problem. Safari does not currently have an opt-in flag for testing the new spec behavior.

### Test with Firefox

Firefox support for the new standard can be tested on version 68+ by opting in on the `about:config` page with the feature flag `network.cookie.sameSite.laxByDefault`. There haven't been reports of compatibility issues with older versions of Firefox.

### Test with Edge

Edge supports the old SameSite standard. Edge version 44 doesn't have any compatibility problems with the new standard.

### Test with Edge (Chromium)

SameSite flags are set on the `edge://flags/#same-site-by-default-cookies` page. No compatibility issues were discovered with Edge Chromium.
<!--  No compatibility issues were observed when testing with Edge Chromium 78. Current version is 78 and soon that will change  -->

### Test with Electron

Versions of electron will include older versions of Chromium. For example the version of Electron used by Teams is Chromium 66 which exhibits the older behavior. You must perform your own compatibility testing with the version of Electron your product uses. See [Supporting older browsers](#sob) in this document..

<a name="sob"></a>

## Supporting older browsers

The 2016 SameSite standard mandated that unknown values must be treated as `SameSite=Strict` values. Older browsers which support the 2016 SameSite standard may break when they see a SameSite property with a value of `None`. Web apps must implement browser detection if they intend to support older browsers. ASP.NET Core doesn't implement browser detection because User-Agents values are highly volatile and change frequently. An extension point in <xref:Microsoft.AspNetCore.CookiePolicy> allows plugging in User-Agent specific logic.

In `Startup` add code similar to the following:

::: moniker range="= aspnetcore-3.1"

[!code-csharp[](samesite\sample\Startup31.cs?name=snippet)]

::: moniker-end

::: moniker range="< aspnetcore-3.1"

[!code-csharp[](samesite\sample\Startup.cs?name=snippet)]

::: moniker-end

In the preceding sample, `MyUserAgentDetectionLib.DisallowsSameSiteNone` is a user supplied library that detects if the user agent doesn't support SameSite `None`:

[!code-csharp[](samesite\sample\Startup31.cs?name=snippet2)]
