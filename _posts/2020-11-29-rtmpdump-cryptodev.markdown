---
layout: post
title:  "RTMPdump support RTMPS with cryptodev hardware engine"
date:   2020-11-29 11:50:54 +0800
categories: Linux crypto
tags:
    - Work
    - Linux
    - Crypto
---
Due to [Facebook policy][fb-policy], RTMP will be deprecated from the Live API, so we need to use RTMPS (RTMP over a TLS/SSL connection).

<script src="https://gist.github.com/JasperHsieh0415/0d0ebbb292582b4df4deade57c0aa204.js"></script>

Three important changes:

- -enable-openssl -enable-nonfree: Enable OpenSSL support
- -enable-librtmp: Enable RTMPdump support
- **DO NOT** -disable-protocols: FFmpeg will only support native protocols (RTMPS not included, of course..).

We can check supported protocols with

## ffmpeg -protocols

As for now, we could start streaming with RTMPS by FFmpeg + librtmp + OpenSSL, but only with software encryption, making the embedded linux system extremely busy.
To enable OpenSSL hardware encryption, we need to use cryptodev and enable OpenSSL support, check OpenSSL Linux kernel Cryptodev engine.
We may need to modify RTMPdump also, first, lets see **RTMP_TLS_Init()** in rtmp.c

![img](/img/in-post/cryptodev.png)

SSL_CTX_new(): creates a new SSL_CTX object as framework to establish TLS/SSL enabled connections.

SSLv23_method(): general-purpose version-flexible SSL/TLS methods, should use TLS_method() or TLS_client_method() instead.

We have to load cryptodev engine before SSL_CTX_new, add start_cryptodev() function:

{% highlight c %}
int start_cryptodev()
{
	RTMP_Log(RTMP_LOGWARNING, "start_cryptodev");
	int ret = 0;
	ENGINE *e;
	const char *engine_id = "cryptodev";
	ENGINE_load_builtin_engines();
	ENGINE_register_all_complete();
	e = ENGINE_by_id(engine_id);
	if(!e) {
		/* the engine isn't available */
		RTMP_Log(RTMP_LOGWARNING, "cryptodev init fail");
		ret = -1;
		return ret;
	}
	if(!ENGINE_init(e)) {
		RTMP_Log(RTMP_LOGWARNING, "cryptodev init fail 2");
		ENGINE_free(e);
		ret = -1;
		return ret;
	}
	if (!ENGINE_set_default(e, ENGINE_METHOD_ALL)) {
		RTMP_Log(RTMP_LOGWARNING, "can't use that engine");
		ENGINE_free(e);
		ret = -1;
		return ret;
	}
	ENGINE_set_default_ciphers(e);
	ENGINE_set_default_digests(e);
	OpenSSL_add_all_algorithms();
	SSL_load_error_strings();
	//ENGINE_register_complete(e);

	return ret;
}
{% endhighlight %}

If insmod cryptodev and install self-crypto driver swimmingly, I think it should work. **BUT**, unfortunately, it didn’t.

That’s check Facebook supported versions of TLS, as well as supported ciphers. Here is the [script][crypto-script]

![img](/img/in-post/fb_crypto.png)

Let’s modify RTMPdump to set the list of available ciphers.

{% highlight c %}
int SSL_CTX_set_cipher_list(SSL_CTX *ctx, const char *str);
{% endhighlight %}

Since our crypto driver does not support GCM cipher, first we ruled all CGM-type cipher out, the SSL client (i.e., librtmp) will notify server (i.e., Facebook) our available ciphers, my guess is that **Facebook default uses CGM-type ciphers**. Once Facebook knows that the client didn’t support CGM, it uses the alternative ciphers, then SSL handshake error (Our own crypto driver problem).

---------------------------------------

07/26 update

Stable Openssl version: **1.1.1b**
Modify the Makefile of openssl 1.1.1b to enable cryptodev engine

``` Makefile
./Configure linux-generic32 shared -DL_ENDIAN -I$(PREFIX_DIR)/include/ — prefix=$(PREFIX_DIR) — openssldir=$(PREFIX_DIR) enable-devcryptoeng
```

[fb-policy]: https://developers.facebook.com/docs/graph-api/changelog/non-versioned-changes/apr-24-2018
[crypto-script]:https://www.ise.io/using-openssl-determine-ciphers-enabled-server/

