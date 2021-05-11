---
layout: post
title:  "Advanced Frida Witchcraft: Turning an Android Application into a Voodoo Doll"
date:   2021-05-11 18:30:00 -0500
categories: frida android
---

When faced with obfuscated native cryptography libraries, why spend time
reverse engineering if you can just turn the library into an oracle to do your
work for you? This post dives into the process of hooking a native Android
library using Frida and turning it into an API by stuffing it with a web application
like a Thanksgiving turkey.

## Turning an Obfuscated Library into a Cryptographic Oracle

A recent engagement presented me with an Android application using
[Whitebox Cryptography](https://crypto.stackexchange.com/a/392) through a native
library embedded into the application. Whitebox Cryptography is essentially an
obfuscation technique for cryptographic keys and algorithms. In this case, the
application used a JNI wrapper like the following:

~~~java
public class CryptoLib {

  public class CryptoResult {
    public boolean success;
    public int errCode;
    public byte[] result;
    public byte[] tag;
  }

  public native CryptoResult encryptGCM(
    byte[] plaintext,
    byte[] iv,
    byte[] associated_data
  );

  public native CryptoResult decryptGCM(
    byte[] ciphertext,
    byte[] iv,
    byte[] associated_data,
    byte[] tag
  );

  static {
    System.loadLibrary("whitebox-crypto.so");
  }
}
~~~

The point of the encryption was to communicate securely with a BLE device, which
was the real target of the test. In order to attack the device, I needed to be
able to encrypt and decrypt arbitrary data. There are a number of possibilities:

* Attempt to reverse engineer the native library to find the keys
* Write my own Android application to call the library
* Use QEMU[^qemu] to run the code directly, and hook into it with native code or Java
* Attempt to pull the keys from the library via a side-channel attack like DFA[^dfa]

[^qemu]: <https://wiki.qemu.org/Documentation/Platforms/ARM>
[^dfa]: <https://blog.quarkslab.com/differential-fault-analysis-on-white-box-aes-implementations.html>

All of these are reasonable solutions to the problem. However, given that the
point of the whitebox solution was to prevent these types of attacks, it's likely
that there are countermeasures in place:

* The code is obfuscated to make reverse engineering painful
* The library is likely checking its environment to make sure it isn't being run
  by another application or in an emulator

Furthermore, I had already begun instrumenting the application heavily using
[Frida](https://www.frida.re/), so it made the most sense to continue using
that. Wrapping the class using a Frida script is pretty straightforward:

~~~js
setTimeout(Java.perform(function() {
  try {
    const CryptoLib = Java.use("com.myclient.crypto.CryptoLib")
    const cryptolib = CryptoLib.$new()

    const payload = Java.array('byte', [192, 174, 233, 7, 93, 64, 98, 244, ...])
    const iv = Java.array('byte', [76, 30, 206, 36, 137, 116, 45, 53, 117, 15, 189, 227])
    const ad = Java.array('byte', [38, 234, 129, 140, 203, 46, 212, 16])
    const tag = Java.array('byte', [29, 127, 142, 110, 214, 59, 170, 120])

    const maybeCryptoResult = cryptolib.decryptGCM(payload, iv, ad, tag)

    // I always force-cast Frida Java objects to their expected class
    // This makes type errors occur immediately rather than causing issues later
    const CryptoResult = Java.use("com.myclient.crypto.CryptoResult")
    const decryptResult = Java.cast(maybeCryptoResult, CryptoResult)

    // JSON.stringify is usually the easiest way to make Java objects printable
    console.log("result: " + JSON.stringify(decryptResult.result))
  }
  catch(err) {
    console.log(String(err.stack))
  }
}
~~~

For simple tests, this was perfect. I could use it immediately to encrypt and
decrypt some test strings and check that my code worked. But having to rewrite
the script and reload it every time I wanted to test a new attack was just not
efficient. I began to think of what a more useful interface to this code would
look like. I wanted to encrypt and decrypt arbitrary data, without reloading
any Frida code. Furthermore, it needed to be accessible programmatically.

## Frankenstein's Node.js Monster

Basically, I wanted to turn this into an API. But the data felt like it was
jailed inside in the application where the code was running. Is it possible
to create an external interface to the application's data? Frida offers some
support for this via its `send` and `recv` [messaging API](https://www.frida.re/docs/messages/).
But if you use those, you're stuck using one of Frida's language bindings to
access the application, and I'm not particularly enthusiastic about Frida's
stability in general.

It was when I started looking at [`frida-compile`](https://github.com/frida/frida-compile)
that inspiration struck. The best documentation for this package comes in the
form of a [Frida release note](https://www.frida.re/news/2016/10/25/frida-8-1-released/)
from 2016. The post shows a Frida script which is compiled with off-the-shelf
node modules to inject an express.js[^express] web application into a Frida target. By
creating an HTTP API, I could interact directly with the cryptography library
or use any language I wanted to write tools around the library. Here's what that
looked like:

[^express]: <https://expressjs.com/>

~~~js
const express = require('express');
const app = express();

// see https://gist.github.com/tprynn/711c2caa1687fe66b4c496324089f826
const util = require('./util.js')

// assumes query params plaintext, iv, ad hex-encoded strings
app.get('/encrypt', function(req, res) {
  const plaintext = util.hexStringToIntArray(req.query['plaintext'])
  const iv = util.hexStringToIntArray(req.query['iv'])
  const ad = util.hexStringToIntArray(req.query['ad'])

  Java.perform(function() {
    try {
      const CryptoLib = Java.use("com.myclient.crypto.CryptoLib")
      const cryptolib = CryptoLib.$new()

      const payload_obj = Java.array('byte', plaintext)
      const iv_obj = Java.array('byte', iv)
      const ad_obj = Java.array('byte', array)

      const maybeCryptoResult = cryptolib.encryptGCM(payload_obj, iv_obj, ad_obj)
      const CryptoResult = Java.use("com.myclient.crypto.CryptoResult")
      const encryptResult = Java.cast(maybeCryptoResult, CryptoResult)

      const ciphertext = util.byteArrayToHexString(encryptResult.result)
      const tag = util.byteArrayToHexString(encryptResult.tag)

      res.status(200).json({ciphertext: ciphertext, tag: tag})
      res.end()
    }
    catch(err) {
      console.log(String(err.stack))
      res.status(500).json({error: String(err.stack)})
      res.end()
    }
  })
})

app.listen(4000)
~~~

After creating a `package.json` file and installing the dependencies[^npm], I
got my API up and running:

[^npm]: <https://docs.npmjs.com/files/package.json>

~~~bash
# Compile my script, api.js, into compiled.js
frida-compile -o compiled.js api.js

# Inject compiled.js into the running application
frida -U com.myclient.theapp -l compiled.js

# Forward the TCP port on the phone to localhost
adb forward tcp:4000 tcp:4000

# Encrypt!
curl -v 'http://localhost:4000/encrypt?plaintext=68656c6c6f20776f726c64&iv=80e82358f17342a21822d850&ad=1822d850'
~~~

## Conclusion

This technique offers immense flexibility in instrumenting and controlling mobile
applications. Further, by turning mobile methods into web APIs, existing tools
can be used to attack and automate the application.

Consider an application with an exported BroadcastReciever or ContentProvider
that's potentially vulnerable to SQL injection[^sqli]. Typically, Android IPC
mechanisms are difficult to test in an automated way without writing a custom tool.
But a thin Frida wrapper to turn the handling function into a web API could enable
it to be tested using off-the-shelf tools.

[^sqli]: <https://solidgeargroup.com/sql-injection-in-content-providers-of-android-and-how-to-be-protected>
