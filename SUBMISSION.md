# Report

Name: Danil Sobar  Group: PO 25-Z  Date: 22.10.26

## What is done

- [ ] M0: venv created, dependencies installed, fastapi dev main.py starts
- [ ] M1: http://localhost:8000/messages returns []
- [ ] M2: chat-api/ collection created inside the repository, first request saved as a file
- [ ] M3: seven requests saved, codes match the table
- [ ] M4: local environment, {{baseUrl}}, and {{messageId}} from the post-response script
- [x] M5: PATCH written
- [ ] M6: all four PATCH checks pass

## What is in the collection

| Request file | Method and path | Status code |
|---|---|---|
| list-messages | GET {{baseUrl}}/messages | 200 |
| create-message | POST {{baseUrl}}/messages | 201 |
| get-message | GET {{baseUrl}}/messages/{{messageId}} | 200 |
| put-message | PUT {{baseUrl}}/messages/{{messageId}} | 200 |
| delete-message | DELETE {{baseUrl}}/messages/{{messageId}} | 204 |
| get-missing | GET {{baseUrl}}/messages/999999 | 404 |
| create-invalid | POST {{baseUrl}}/messages | 422 |
| patch-message | PATCH {{baseUrl}}/messages/{{messageId}} | 200 (404 for id 999999) |

## PATCH in your own words

What makes my PATCH different from my PUT. The actual lines:

PUT takes `MessageIn`, where both fields are required, and overwrites both:

    message.author = payload.author
    message.text = payload.text

PATCH takes `MessagePatch`, where both fields are optional, and writes only what the client sent:

    changes = payload.model_dump(exclude_unset=True)
    for field, value in changes.items():
        setattr(message, field, value)

`exclude_unset=True` leaves out every field the client did not send, so those fields keep their old values.

What happens on a PATCH with an empty body, `{}`, and why that is correct:

`changes` is an empty dict, so the loop does nothing. The server returns 200 with the message exactly as it was. That is correct because PATCH means "change these fields", and with no fields listed there is nothing to change. The request is valid, not an error.

## Homework

The github-api/ collection, in this same repository.

### Requests

| Address | Method | Required parameters | Status code |
|---|---|---|---|
| {{baseUrl}}/users/octocat | GET | username (in path) | 200 |
| {{baseUrl}}/users/octocat + If-None-Match | GET | username, header If-None-Match with the ETag in quotes | 304 |
| {{baseUrl}}/users/octocat/repos?per_page=2 | GET | username (in path); per_page is optional | 200 |
| {{baseUrl}}/users/definitely-not-real-xyz | GET | username (in path) | 404 |
| {{baseUrl}}/users/octocat | DELETE | username (in path) | ? (write what you saw) |
| {{baseUrl}}/users/octocat | POST | username (in path) | ? (404 or 405) |
| {{baseUrl}}/users/octocat + Accept: application/xml | GET | username, header Accept | ? (200 JSON, or an error) |
| https://httpbin.org/headers | GET | none | 200 |

### Three ways their API differs from mine

1. **Errors.** GitHub's 404 body is `{"message": "Not Found", "documentation_url": "..."}`: a human message plus a link to the docs for that error. My 404 is `{"detail": "Message not found"}`, which FastAPI builds from `HTTPException`. Mine tells the client what went wrong, but not where to read more. I did it this way because it is the FastAPI default and I only have one resource.

2. **Caching.** GitHub sends `etag`, `last-modified` and `cache-control: public, max-age=60`, so a client can send `If-None-Match` and get a 304 with no body. My API sends none of these, so every GET returns a full 200. I did not add them because my data is small, changes all the time, and lives in memory.

3. **Rate limits.** GitHub counts requests and reports `x-ratelimit-limit`, `x-ratelimit-remaining` and `x-ratelimit-reset` in every response, and refuses when the quota is spent. My server has no limit, so one client can send as many requests as it likes. I did not add one because it runs only on localhost for the lab, but a public API needs it.

### Questions from HOMEWORK.md

What ETag saves the client from doing, and what it saves the server:

The client does not download and parse the body again when nothing has changed: it sends `If-None-Match`, gets a 304 with no body, and reuses its copy. The server does not have to send the body over the network, which saves bandwidth. On GitHub a 304 is also not charged to the rate limit.

Why pagination is in a header rather than the body, and what breaks if a client ignores it:

With the `Link` header the body stays a plain array of repositories, and the navigation (`next`, `last`) is kept separate from the data. If a client ignores `Link`, it only ever gets the first page (30 items by default) and does not notice that the list is incomplete, so it silently works with partial data.

Which headers Bruno added that you never wrote, and which ones curl added:

Bruno: <copy from httpbin /headers, e.g. User-Agent: bruno-runtime/..., Accept: application/json, text/plain, */*, Accept-Encoding: ...>
curl: <copy from the curl output, e.g. User-Agent: curl/..., Accept: */*>
Both also send Host. httpbin itself adds X-Amzn-Trace-Id.

404 or 405 on DELETE /users/octocat, and your argument for it:

<write what GitHub returned>. My argument for 404: the response should not reveal more than needed. 405 would say "this resource exists, but you cannot delete it", while 404 says "nothing you are allowed to delete here". Argument for 405: the path exists for GET, so the correct code for a wrong method is 405 with an `Allow` header. (If your observed code was 405, swap the two arguments.)
