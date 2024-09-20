# Securelay

A secure, and ephemeral, small data relay API for serverless apps.

Securelay was originally conceptualized for [EasyForm](https://github.com/SomajitDey/EasyForm), but then it evolved into an awesome microservice in its own right.

# How it works

**Duality:**

Securelay works in two ways.:
1. Aggregator mode: Many can POST (or publish) to a public path for only one to GET (or subscribe) at a private path. This may be useful for aggregating HTML form data from one's users.
2. Key-Value Store mode: Only one can POST (or pub) to a private path for many to GET (or sub) at a public path. See security section below for a significant usecase. In addition to the one-to-many relay, there can also be one-to-one relay if path is suffixed with a uid query parameter. That is to say, when one POSTs to `https://api.securelay.tld/private_path?uid=<uid>`, there can be only one GET consumer at `https://api.securelay.tld/public_path?uid=<uid>`, after which any more GET at that path would result in a 404 error. This is useful for sending a separate response to each POSTer.

**Ephemeral:** POST(s) and GET(s) can be concurrent. If not, POSTed data is stored until next GET. However, if no GET appears within a certain time-window, the POSTed data is deleted. Hence, Securelay is *ephemeral*, if not storageless.

**CORS:** Allowing CORS is a must. Otherwise, browsers would block client side calls to the API.

**Futureproof:** The URL(s) of the API endpoint(s) may be found with a GET at https://securelay.github.io/api-endpoints.json.

# Security
Security is brought about by the use of dual paths, one private and the other public. Note here that other relay services like [piping-server](https://github.com/nwtgck/piping-server), [http-relay](https://httprelay.io) or [pipeto.me](https://pipeto.me) use the same path for both GET and POST.

Public path should be derivable from private path but not the other way round. This is easily achieved by defining public_path as some function of `sha256(private_path)`.

Another part of security is to ensure that no two users can have the same private path, even by accident. This is achieved by accepting only those paths that were uniquely generated and signed by [the Securelay API server](https://api.securelay.tld) itself! Signature may consist of a substring of `hmac(UUID, type, securelay_secret)` where `type=[public | private]` and `UUID` is the randomiser used to generate the remaining part of the private path. This signature is easy to validate and requires storing only the secret. A GET to https://api.securelay.tld/paths returns a new private-public path pair. This is a proposed path pair generation schema:

```
private_path = concatenate(UUID, substring(hmac(UUID, "private", securelay_secret))).
public_path = concatenate(private_path, substring(hmac(private_path, "public", securelay_secret))).
```
Note that given any path it is trivial to determine whether it is public or private while validating its signature.

There is more. One can POST his cryptographic public key at his private_path for his users to consume at the corresponding public_path. His users then POST to the public_path as application/base64 their data encrypted with that public key for him to decrypt upon a subsequent GET at the private_path. This ensures the relay is end-to-end encrypted, not even the Securelay server can read user data.

# Limits
To mitigate abuse, the API accepts only two enctype modes. Securelay validates the data to see it is only of these two types.
1. application/x-www-form-urlencoded. Securelay parses data into JSON to validate and returns as application/JSON on GET.
2. application/base64. Securelay validates by checking success of base64 decoding of data. Returns ditto as application/base64 on GET.

It also accepts POSTs only if they have Content-Length less than a size-limit.

Another limit is imposed on how long POSTed data persists if there is no GET. 

# Possible Implementation

Python using flask /fastapi. Or NodeJS using fastify.

Host on Glitch or [alternatives](https://support.glitch.com/t/temporary-glitch-alternatives/26915) like Vercel. Or, self-host using ngrok exposure. If it can be hosted on ready-to-deploy services like Glitch for free, then anybody can host - resulting in much needed redundancy. Provide as nvm/pypi package or docker for the willing ones to self-host.

NodeJS notes
---
Set expiry by using `const uid = settimeout(delete(filepath), timeWindow)`. If filepath is being unlinked apriori upon GET, use `clearTimeout(uid)`. 

Perhaps a good way to use lock in npm:fs is using callback function with async [fs.mkdir](https://nodejs.org/api/fs.html#fsmkdirpath-options-callback). The callback simply is a closure containing the data to append to the path. Callback also deletes the dir upon completion, thus releasing the lock.

Use fs.watch with callback for executing the callback whenever a file changes. [Here](https://thisdavej.com/how-to-watch-for-file-changes-in-node-js/) is the way. Use md5 hash or debounce (timeout delays) if need be as mentioned in the article.
