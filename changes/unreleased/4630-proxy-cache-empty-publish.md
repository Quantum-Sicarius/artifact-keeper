---
section: Fixed
issues: [#4630]
---
- **A proxy-cache entry whose storage publish landed a zero-byte object no longer returns HTTP 500 forever** (#4630). The streaming cache writer validated the bytes it staged and then published them onto the live key with a `copy()` whose result was never measured, so an S3-compatible endpoint that reported success while storing nothing produced a metadata sidecar carrying the artifact's real size and SHA-256 over an empty body. Streaming reads then sent that size as `Content-Length` ahead of no bytes, and the connection was aborted mid-response on every request: the existing self-heal fires only when the object is absent, and this one was present.

  The writer now measures the published object, and unless the live key holds the bytes that were staged it writes no sidecar and drops the published object, so the entry misses cleanly on the next request. A truncated publish is refused the same way. Reads refuse a sidecar that pins the ETag of an empty object while claiming a non-zero size, in the streaming path, the presigned-redirect freshness probe and the stale-if-error fallback, so entries already poisoned in an existing deployment become a cache miss and heal on the next request instead of waiting for an operator to delete the objects or for a lifecycle policy to reclaim them by age.
