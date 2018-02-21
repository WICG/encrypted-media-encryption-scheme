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
enum EncryptionScheme { "cenc", "cbcs" };
```

The EncryptionScheme enumeration is defined as follows:
 - *cenc*: The 'cenc' mode, defined in [ISO 23001-7:2016][], section 4.2a.
           AES-CTR mode full sample and video NAL subsample encryption.
 - *cbcs*: The 'cbcs' mode, defined in [ISO 23001-7:2016][], section 4.2d.
           AES-CBC mode partial video NAL pattern encryption.

[ISO 23001-7:2016]: https://www.iso.org/standard/68042.html


```js
dictionary MediaKeySystemMediaCapability {
    DOMString contentType = "";
    DOMString robustness = "";
    EncryptionScheme? encryptionScheme = null;
};
```

`encryptionScheme` of type `EncryptionScheme`, defaulting to `null`

The encryption scheme used by the content.  A missing or `null` value indicates
that any encryption scheme is acceptable.

Implementations **must** reject configurations specifying unsupported encryption
schemes.


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

    { // A configuration which uses the "cbcs" encryption scheme
      videoCapabilities: [{
        contentType: 'video/mp4; codecs="avc1.640028"',
        encryptionScheme: 'cbcs',
      }],
      audioCapabilities: [{
        contentType: 'audio/mp4; codecs="mp4a.40.2"',
        encryptionScheme: 'cbcs',
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
      { // A capability object which uses the "cbcs" encryption scheme
        contentType: 'video/mp4; codecs="avc1.640028"',
        encryptionScheme: 'cbcs',
      },
    ],
    audioCapabilities: [
      { // A capability object which uses the "cenc" encryption scheme
        contentType: 'audio/mp4; codecs="mp4a.40.2"',
        encryptionScheme: 'cenc',
      },
      { // A capability object which uses the "cbcs" encryption scheme
        contentType: 'audio/mp4; codecs="mp4a.40.2"',
        encryptionScheme: 'cbcs',
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

If the application does not specify an encryption scheme,
[`MediaKeySystemAccess.getConfiguration()`][] **must** fill in a supported value
in the `encryptionScheme` fields of `MediaKeySystemMediaCapability` objects.

Applications could therefore detect whether or not a user agent recognizes the
new field by checking [`MediaKeySystemAccess.getConfiguration()`][].  If the
configuration returned by `getConfiguration()` does not contain
`encryptionScheme`, the field was ignored by the user agent.

With this technique, a polyfill could take over the role of hard-coding
assumptions about what encryption schemes older user agents support.

[`MediaKeySystemAccess.getConfiguration()`]: https://www.w3.org/TR/encrypted-media/#dom-mediakeysystemaccess-getconfiguration


## Open issues

1. How do we best define what these schemes mean for WebM content?
