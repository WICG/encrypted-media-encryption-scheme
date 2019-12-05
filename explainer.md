# Querying Encryption Scheme Support Through EME

## Motivation

Some platforms or key systems only support AES-128 in CTR mode, while others
only support CBCS mode.  Still others are able to support both.

These two encryption schemes are incompatible, so applications must be able to
make intelligent choices about what content to serve to which user agents.


## Overview

[`navigator.requestMediaKeySystemAccess()`][] takes a sequence of
[`MediaKeySystemConfiguration`][] objects to specify configurations the
application could use.  This contains sequences of
[`MediaKeySystemMediaCapability`][] objects with details about the audio and
video capabilities which are required.

We should extend [`MediaKeySystemMediaCapability`][] to allow the application to
specify which encryption schemes it could use.

[`navigator.requestMediaKeySystemAccess()`]: https://www.w3.org/TR/encrypted-media/#navigator-extension:-requestmediakeysystemaccess()
[`MediaKeySystemConfiguration`]: https://www.w3.org/TR/encrypted-media/#dom-mediakeysystemconfiguration
[`MediaKeySystemMediaCapability`]: https://www.w3.org/TR/encrypted-media/#dom-mediakeysystemmediacapability


## Web IDL

```js
partial dictionary MediaKeySystemMediaCapability {
    DOMString? encryptionScheme = null;
};
```

`encryptionScheme` of type `DOMString`, defaulting to `null`

The encryption scheme used by the content.  A missing or `null` value indicates
that any encryption scheme is acceptable.

NOTE: The empty string is distinct from `null` or missing, and so would be
treated as an unrecognized encryption scheme.

Expected non-`null` values for encryptionScheme are:
 - *cenc*: The 'cenc' mode, defined in [ISO 23001-7:2016][], section 4.2a.
           AES-CTR mode full sample and video NAL subsample encryption.
 - *cbcs*: The 'cbcs' mode, defined in [ISO 23001-7:2016][], section 4.2d.
           AES-CBC mode partial video NAL pattern encryption.  For video, the
           spec allows various encryption patterns.
 - *cbcs-1-9*: The same as 'cbcs' mode, but with a specific encrypt:skip pattern
               of 1:9 for video, as recommended in [ISO 23001-7:2016][], section
               10.4.2.

NOTE: The document [WebM Encryption][] defines WebM encryption to be equivalent
to and compatible with the 'cenc' encryption mode defined in
[ISO 23001-7:2016][].

[ISO 23001-7:2016]: https://www.iso.org/standard/68042.html
[WebM Encryption]: https://www.webmproject.org/docs/webm-encryption/

In the [Get Supported Capabilities for Audio/Video Type][] algorithm,
implementations **must** skip capabilities specifying unsupported or
unrecognized encryption schemes.

If `encryptionScheme` from the application is `null` or missing, the
`encryptionScheme` fields in the returned configuration **must** be `null`.

[Clear Key][] implementations **must** support the "cenc" scheme at a minimum,
to ensure interoperability for users of this common key system.

[Get Supported Capabilities for Audio/Video Type]: https://www.w3.org/TR/encrypted-media/#get-supported-capabilities-for-audio-video-type
[`MediaKeySystemAccess.getConfiguration()`]: https://www.w3.org/TR/encrypted-media/#dom-mediakeysystemaccess-getconfiguration
[Clear Key]: https://www.w3.org/TR/encrypted-media/#clear-key


## Application notes

Applications should specify encryption scheme(s) they require, since different
encryption schemes are generally incompatible with one another.  Asking for
"any" encryption scheme is unrealistic.  Defining `null` as "any scheme" is
convenient for backward compatibility, though.  Applications which ignore this
feature by leaving `encryptionScheme` null get the same user agent behavior they
did before this feature existed: they have to assume what encryption schemes the
user agent supports.

Applications may specify multiple encryption schemes in separate configurations
or in multiple capabilities of the same configuration.

The user agent only selects one configuration.  So if different encryption
schemes are specified in separate configurations, the application will be given
back a configuration containing only one encryption scheme.

If different encryption schemes appear in the same configuration, the user
agent's accumulated configuration will contain the supported subset of the
capabilities specified by the application.  The configuration returned from
`getConfiguration()` may therefore contain more than one encryption scheme.


## Examples

```js
function tryTwoEncryptionSchemesInSeparateConfigurations(keySystem) {
  // Query two configurations with different encryption schemes.
  // Only one will be chosen by the user agent.

  return navigator.requestMediaKeySystemAccess(keySystem, [
    { // A configuration which uses the "cenc" encryption scheme
      videoCapabilities: [{
        contentType: 'video/mp4; codecs="avc1.640028"',
        encryptionScheme: 'cenc',
      }],
      audioCapabilities: [{
        contentType: 'audio/mp4; codecs="mp4a.40.2"',
        encryptionScheme: 'cenc',
      }],
      initDataTypes: ['keyids'],
    },

    { // A configuration which uses the "cbcs-1-9" encryption scheme
      videoCapabilities: [{
        contentType: 'video/mp4; codecs="avc1.640028"',
        encryptionScheme: 'cbcs-1-9',
      }],
      audioCapabilities: [{
        contentType: 'audio/mp4; codecs="mp4a.40.2"',
        encryptionScheme: 'cbcs-1-9',
      }],
      initDataTypes: ['keyids'],
    },
  ]);
}

function tryTwoEncryptionSchemesInOneConfiguration(keySystem) {
  // Query one configuration with two different encryption schemes.
  // The user agent will eliminate any capabilities object it cannot support,
  // so the accumulated configuration may contain one encryption scheme or both.

  return navigator.requestMediaKeySystemAccess(keySystem, [{
    videoCapabilities: [
      { // A capability object which uses the "cenc" encryption scheme
        contentType: 'video/mp4; codecs="avc1.640028"',
        encryptionScheme: 'cenc',
      },
      { // A capability object which uses the "cbcs-1-9" encryption
        // scheme
        contentType: 'video/mp4; codecs="avc1.640028"',
        encryptionScheme: 'cbcs-1-9',
      },
    ],
    audioCapabilities: [
      { // A capability object which uses the "cenc" encryption scheme
        contentType: 'audio/mp4; codecs="mp4a.40.2"',
        encryptionScheme: 'cenc',
      },
      { // A capability object which uses the "cbcs-1-9" encryption
        // scheme
        contentType: 'audio/mp4; codecs="mp4a.40.2"',
        encryptionScheme: 'cbcs-1-9',
      },
    ],
    initDataTypes: ['keyids'],
  }]);
}
```


## Backward Compatibility

Applications which don't use the new field will not be affected.  If they make
assumptions about the capabilities of certain platforms or key systems, they can
continue to use those assumptions, with no additional risk.

User agents which don't recognize the new field will ignore it.  Applications
which are aware of the new fields may still need to hard-code assumptions about
the encryption schemes supported by these older user agents.

Application could detect whether or not a user agent recognizes the new field by
checking [`MediaKeySystemAccess.getConfiguration()`][].  If the configuration
returned by `getConfiguration()` does not contain `encryptionScheme`, the field
was ignored by the user agent.

With this technique, a polyfill could take over the role of hard-coding
assumptions about what encryption schemes older user agents support.

Any user agent which deprecates support for an existing encryption scheme could
introduce backward compatibility issues for older applications which do not use
this new feature.  Appropriate communication with developers will be necessary
to ensure that applications are updated in a timely manner ahead of any such
deprecation.  The need for good communication around deprecations is not unique
to this proposal, but we felt it was worth exploring this potential
compatibility issue explicitly.

[`MediaKeySystemAccess.getConfiguration()`]: https://www.w3.org/TR/encrypted-media/#dom-mediakeysystemaccess-getconfiguration


## Alternatives Considered

### Content Type parameters

We considered using contentType parameters to specify encryption scheme, but
this presented more compatibility issues for developers.  For example, the
existing spec states in the [Get Supported Capabilities for Audio/Video Type][]
algorithm:

> Let parameters be the RFC 6381 [RFC6381] parameters, if any, specified by
> content type. If the user agent does not recognize one or more parameters,
> continue to the next iteration.

This means applications using older browsers would be required to omit the
(hypothetical) encryption scheme parameter to avoid having that capability
skipped.

This solution would also offer no way for polyfills to detect user agent support
for the new feature.  Applications and polyfills alike would need to hard-code a
list of user agents known to support the new feature, as well as a list of the
encryption schemes supported by user agents which don't support the feature.

This seems like too much burden for application developers, and a more difficult
transition path.

[Get Supported Capabilities for Audio/Video Type]: https://www.w3.org/TR/encrypted-media/#get-supported-capabilities-for-audio-video-type
