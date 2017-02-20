# Lessons From Gmail API - a Batch Request Design Doc 

## Abstract
Batch multiple requests into a single one for better performance.

## Background
As an example, consider a web-based mail inbox user interface, showing a list of unread mail's title, sender, timestamp, and so forth. The data is fetched from an HTTP API service.

Supposing a list style API for obtaining the unread mail collection, `/mails/unread` responses with a JSON,

```json
{
  "mail": [
    "id1",
    "id2",
    "id3",
    "id4",
    "an on and on..."
  ]
}
```

What you got is merely the mail's `id` field, in order to fulfill your job, you have to request another API, e.g., `/mails/{id}` for each mail id. The API response may look like this,

```json
{
  "id": "idx",
  "title": "Awesome title",
  "sender": "sender@somewhere.com",
  "date": "2016-01-02 15:04:05 +0800",
  "...": "..."
}
```

The problem is: **If a user has many unread mails(hundreds is fairly reasonable), how would you do?**

An intuitive solution is sending multiple API requests concurrently. However, it brings many drawbacks.

### Time Consuming
Concurrency does not make that much sense in this situation. Imagine hundreds of threads, nearly impossible.

- Only a number of threads could be executed at a time
- TCP connection establishment/retransmission if any

### Resource Wasted
It's obvious the number of requests is **proportional** to the system resources. The more you request, the more resources consumed, including but not limited to,

- CPU
- Memory
- Threads
- TCP connection
- Network bandwidth
- API quota limit(if any)

### Programming Error-Prone
Even worse, it's hard to make it right, where requires more carefulness,

- Threads scheduling and synchronization
- Error handling for each failing request
- Merge each response may required other operations, e.g., sorting


All of these yields a poor user experience.

## Proposal
We can solve this problem by batching multiple requests into a single one or few.

### HTTP Media Types
The HTTP/1.1 specification, [RFC 2616][], introduces a *Media Type*, `application/http`, to cope with such situation,

> The application/http type can be used to enclose a pipeline of one or more HTTP request or response messages (not intermixed).

With this standard *Content-Type*, **how to build the requests and response entity(content body)**? 

### Multipart Media Types
MIME *multipart* emails messages contain multiple messages stuck together and sent as a single, complex message. HTTP also supports multipart bodies, typically used in from submission or range requests. The syntax is defined in section 5.1.1 of [RFC 2046][]. A typical entity body looks like this,

```http
Content-Type: multipart/form-data; boundary=abcde

--abcde
Content-Disposition: form-data; name="name"
hello
--abcde
Content-Disposition: form-data; name="password"
123456
--abcde
Content-Disposition: form-data; name="file"; filename="file.txt"
Content-Type: text/plain; charset=UTF-8
...contents of file.txt...
--abcde--
```

Each component is separated by a *boundary*, plus its own headers and body if any.


Combining the two media types, we could encode the batching messages.

### Messages Encoding and Decoding
The request and response message may looks like below, respectively,

```http
POST /batch HTTP/1.1
Content-Length: 12345
Content-Type: multipart/mixed; boundary=abcde

--abcde
Content-Type: application/http

GET /items/id1
Accept: application/json; charset=UTF-8

--abcde

GET /items/id2
Accept: application/json; charset=UTF-8

--abcde

...other requests...

--abcde--
```

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Content-Type: multipart/mixed; boundary=abcde

--abcde
Content-Type: application/http

HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Content-Length: 12345

{
  "id": 1,
  "title": "awesome title1",
  "content": "awesome content2",
  "others": "..."
}

--abcde
Content-Type: application/http

HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Content-Length: 12345

{
  "id": 2,
  "title": "awesome title2",
  "content": "awesome content2",
  "others": "..."
}

--abcde
Content-Type: application/http
Content-Length: 12345

...other responses...

--abcde--
```

The reverse part, decoding, is pretty straightforward.

## Rationale
Simple design, better performance.

- Drastically reduced latency
- Less resources usage
- Easy programming
- Unrelated requests could be batched, too

These yield a better user experience.

### Alternative: Custom Encoding/Decoding
This is same thing as the preceding, essentially. For example, we could encode multiple requests into a JSON format. However, it's not HTTP-idiomatic, which means it may not utilize existing HTTP related library/code.

### Alternative: HTTP/2 Multiplexing
HTTP/2 introduces a new feature, *Multiplexing*, multiple requests over a single TCP connection. Since it's relatively new, many environments may not fully support it. Another drawback, it still wastes resources, whatsoever you have to pay CPU, memory and so forth for each request.

## Implementation
There are some points should be taken into account,

- Understanding HTTP message format
- Reusing existing library/code for decoding/encoding if possible
- Limit requests count in each batching
- provide some mechanism letting clients strip unwanted fields
- **Caching** results

For server side, Google has provide its [Batch API][]; As of client side example, you could find my Swift [Google Batch Helper][] cocoapods library for iOS and macOS.

## Open Issues
- If one or more request(s) in batching is time consuming, it may impact others
- Server may need limit batch request if necessary

## Acknowledgement
- This doc is inspired by Google Batch API, when I built my Gmail based app, [Pixiu][]
- Design doc template is from [Go Project Design Documents][]
- Title image comes from [Unsplash][]
- Write with Emacs, for the first time

## EOF
```yaml
date: 2017-02-18T21:10:12+08:00
summary: Batch multiple requests into a single one for better performance. Simple design, better performance.
weather: sunny
license: cc-40-by
location: 22,144
background: art.jpg
tags:
  - Design
```

[RFC 2616]: https://www.rfc-editor.org/rfc/rfc2616.txt
[RFC 2046]: https://www.rfc-editor.org/rfc/rfc2046.txt
[Batch API]: https://developers.google.com/admin-sdk/directory/v1/guides/batch
[Google Batch Helper]: https://github.com/longkai/GoogleBatchHelper
[Pixiu]: https://geo.itunes.apple.com/app/id1195433805
[Go Project Design Documents]: https://github.com/golang/proposal
[Unsplash]: https://unsplash.com/
