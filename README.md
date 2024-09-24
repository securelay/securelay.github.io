# Securelay

A secure, and ephemeral, small data relay API for serverless apps.

Securelay was originally conceptualized for [EasyForm](https://github.com/SomajitDey/EasyForm), but then it evolved into an awesome microservice in its own right.

# How it works

**Ephemeral:** POST(s) and GET(s) can be concurrent. If not, POSTed data persists for a preset maximum period of time, say 24 hrs. Hence, Securelay is *ephemeral*, if not storageless.

**Duality:**

Securelay works in the following ways:
1. Aggregator mode (many to one): Many can POST (or publish) to a public path for only one to GET (or subscribe) at a private path. POSTed data persist until next GET or expiry, whichever is earlier. This may be useful for aggregating HTML form data from one's users.
2. Key-Value Store mode (one to many and one to one):
   - (one to many) Only one can POST (or pub) to a private path for many to GET (or sub) at a public path. POSTed data persists till expiry. See security section below for a significant usecase. Optional: <strong>GET at the private path retains the POSTed data for a further period of say 24 hrs. So one can use any uptime monitor to keep the data alive.</strong>
   - (one-to-one) If path is suffixed with a user-given `key` query parameter. POSTed data persists until next GET or expiry, whichever is earlier. That is to say, when one POSTs to `https://api.securelay.tld/private_path?uid=<uid>`, there can be only one GET consumer at `https://api.securelay.tld/public_path?key=<key>`, after which any more GET at that path would result in a 404 error. This is useful for sending a separate response to each POSTer.

**CORS:** Allowing CORS is a must. Otherwise, browsers would block client side calls to the API. So securelay server replies with the HEADER- `Access-Control-Allow-Origin: *`

**Futureproof:** The URL(s) of the API endpoint(s) may be found with a GET at https://cdn.jsdelivr.net/gh/securelay/api/endpoints.json courtesy of [jsdelivr](https://www.jsdelivr.com/?docs=gh). So it acts as a sort of dynamic DNS. This requires storing the list of URLs in minified JSON format in a endpoints.json file in the https://github.com/securelay/api repository. The format is `[{"<id>":["<url1>","<url2>", ...]}...]` where `<id>` is unique for a database. i.e. if two separate endpoints `url1` and `url2` share the same database, then a POST at `url1` maybe retrieved with a GET at `url2`. so those URLs are interchangeable, may be used for load balancing. Also, if an endpoint changes only its domain name, its `id` should persist! This helps prevent domain lock-in thus keeping domain costs lower for the provider.

# Security
Security is brought about by the use of dual paths, one private and the other public. Note here that other relay services like [piping-server](https://github.com/nwtgck/piping-server), [http-relay](https://httprelay.io) or [pipeto.me](https://pipeto.me) use the same path for both GET and POST.

Public path should be derivable from private path but not the other way round. This is easily achieved by defining public_path as some function of `sha256(private_path)`.

Another part of security is to ensure that no two users can have the same private path, even by accident. This is achieved by accepting only those paths that were uniquely generated and signed by [the Securelay API server](https://api.securelay.tld) itself! Signature may consist of a substring of `hmac(UUID, type, securelay_secret)` where `type=[public | private]` and `UUID` is the randomiser used to generate the remaining part of the private path. This signature is easy to validate and requires storing only the secret. A GET to `https://api.securelay.tld/paths` returns a new private-public path-pair. This is a proposed path pair generation schema:

```
private_path = concatenate(UUID=v4UUID(), substring(hmac(UUID, "private", securelay_secret))).
public_path = concatenate(UUID=sha256(private_path), substring(hmac(UUID, "public", securelay_secret))).
```
Note that given any path, it is trivial to determine whether it is public or private while validating its signature.

One exciting feature is the `?verify=<email>` parameter when requesting the path-pair at `https://api.securelay.tld/paths`. That prompts securelay to send an ephemeral nonce to the provided email. On presenting this nonce on a subsequent path-pair request with the query `?nonce=<nonce>&salt=<custom>` generates the desired path-pair. The UUID used to create the private path in this path-pair is : `hmac(<email>, <custom>)`. Any POST to this private path persists for a much longer period, say 1 month, but the content length is severely limited. So, basically it deterministically maps a *verified* email and a user-chosen salt to a private path. A possible usecase for this is : User settings may be POSTed at the email mapped private path to be retrieved later upon login using email. 

There is more. One can POST his cryptographic public key at his private_path for his users to consume at the corresponding public_path. His users then POST to the public_path as application/base64 their data encrypted with that public key for him to decrypt upon a subsequent GET at the private_path. This ensures the relay is end-to-end encrypted, not even the Securelay server can read user data.

Note that in all the above securelay doesn't require an any API key for protecting data. The private path itself serves as one. Data can be overwritten only by the owner of the private path.

# Limits
To mitigate abuse, the API accepts only two enctype modes. Securelay validates the data to see it is only of these two types.
1. application/x-www-form-urlencoded. Securelay parses data into JSON to validate and returns as application/json on GET.
2. application/base64. Securelay validates by checking success of base64 decoding of data. Returns ditto as application/base64 on GET.

It also accepts POSTs only if they have Content-Length less than a strict size-limit.

Another limit is imposed on how long POSTed data persists.

Requests to all private paths are heavily rate-limited, say, at max 60 requests per minute. After a certain number of 429 responses 403 bans are imposed. [404s are also rate limited](https://github.com/fastify/fastify-rate-limit?tab=readme-ov-file#preventing-guessing-of-urls-through-404s).

# Use cases
- Forms
- Comments
- Chats
- PubSub
- Dynamic Key Value Store
- Dynamic DNS
- Single click URL shortener
- Configuration sharing between microservices

# Possible Implementation

Python using flask /fastapi. Or NodeJS using fastify.

Host on Glitch or [alternatives](https://support.glitch.com/t/temporary-glitch-alternatives/26915) like Vercel or Deno. Or, self-host using ngrok exposure. If it can be hosted on ready-to-deploy services like Glitch for free, then anybody can host - resulting in much needed redundancy. Provide as nvm/pypi package or docker for the willing ones to self-host.

NodeJS notes
---
Set expiry by using `const uid = settimeout(delete(filepath), timeWindow)`. If filepath is being unlinked apriori upon GET, use `clearTimeout(uid)`. 

Perhaps a good way to use lock in npm:fs is using callback function with async [fs.mkdir](https://nodejs.org/api/fs.html#fsmkdirpath-options-callback). The callback simply is a closure containing the data to append to the path. Callback also deletes the dir upon completion, thus releasing the lock.

Use fs.watch with callback for executing the callback whenever a file changes. [Here](https://thisdavej.com/how-to-watch-for-file-changes-in-node-js/) is the way. Use md5 hash or debounce (timeout delays) if need be as mentioned in the article.

If using [fastify](https://fastify.dev/docs/latest/Guides/Getting-Started/), also use these plugins: [@fastify/cors](https://github.com/fastify/fastify-cors) and [@fastify/form-body](https://github.com/fastify/fastify-formbody).

If an in memory scheduler is required, use [toad-scheduler](https://github.com/kibertoad/toad-scheduler) and [@fastify/schedule](https://github.com/fastify/fastify-schedule).
