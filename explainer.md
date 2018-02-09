# Querying Encryption Scheme Support Through EME

## Motivation

Some platforms or key systems only support AES-128 in CTR mode, while others
only support CBCS mode.  Still others are able to support both.

These two encryption schemes are incompatible, so applications must be able to
make intelligent choices about what content to serve to which user agents.

https://github.com/w3c/encrypted-media/issues/400


## Overview

[`navigator.requestMediaKeySystemAccess()`][] takes a sequence of
[`MediaKeySystemConfiguration`][] objects to specify configurations the
application could use.  This contains sequences of
[`MediaKeySystemMediaCapability`][] objects with details about the audio and
video capabilities which are required.

We should extend [`MediaKeySystemMediaCapability`][] to allow the application to
specify which encryption schemes it could use.

This is based on the informal proposal I made in [EME issue 400][].

[`navigator.requestMediaKeySystemAccess()`]: https://www.w3.org/TR/encrypted-media/#navigator-extension:-requestmediakeysystemaccess()
[`MediaKeySystemConfiguration`]: https://www.w3.org/TR/encrypted-media/#dom-mediakeysystemconfiguration
[`MediaKeySystemMediaCapability`]: https://www.w3.org/TR/encrypted-media/#dom-mediakeysystemmediacapability
[EME issue 400]: https://github.com/w3c/encrypted-media/issues/400#issuecomment-332674527


## Web IDL

```js
enum EncryptionScheme { "cenc", "cbc1", "cens", "cbcs" };
```

The EncryptionScheme enumeration is defined as follows:
 - *cenc*: The 'cenc' mode, defined in [ISO 23001-7:2016][], section 4.2a.
           AES-CTR mode full sample and video NAL subsample encryption.
 - *cbc1*: The 'cbc1' mode, defined in [ISO 23001-7:2016][], section 4.2b.
           AES-CBC mode full sample and video NAL subsample encryption.
 - *cens*: The 'cens' mode, defined in [ISO 23001-7:2016][], section 4.2c.
           AES-CTR mode partial video NAL pattern encryption.
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

NOTE: Implementations **must** reject configurations specifying unsupported
encryption schemes.

NOTE: Applications **should** specify encryption scheme(s) they require, since
different encryption schemes are generally incompatible with one another.


## Examples

```js
function tryTwoEncryptionSchemes(keySystem) {
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
```


## Backward Compatibility

Applications which don't use these new fields will not be affected.  If they
make assumptions about the capabilities of certain platforms or key systems,
they can continue to use those assumptions, with no additional risk.

User agents which don't recognize these new fields will ignore them.
Applications which are aware of the new fields may still need to hard-code
assumptions about these older user agents.

Applications could detect whether or not a user agent recognizes the new fields
by checking [`MediaKeySystemAccess.getConfiguration()`][].  If the configuration
returned by `getConfiguration()` does not contain the new fields, they were
ignored.

With this technique, a polyfill could take over the role of hard-coding
assumptions about what encryption schemes older user agents support.

[`MediaKeySystemAccess.getConfiguration()`]: https://www.w3.org/TR/encrypted-media/#dom-mediakeysystemaccess-getconfiguration
