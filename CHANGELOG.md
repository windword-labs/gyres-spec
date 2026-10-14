# Changelog

Breaking changes to the client interface (HTTP endpoints, request and response
fields, events, auth levels), newest first, with what a frontend must change.
Additions are not listed; they never break a client that ignores unknown
fields.

Every entry raises `api_version` (reported by `GET /v1/version`) by one.
A frontend built for `api_version` N works with a Core that reports N. When
Core reports a higher number, read the entries above N here.

Each entry:

    ## api_version N (Gyres X.Y.Z)

    - What changed (endpoint, field or event).
      Migration: what a client must do.

## api_version 1 (the first Gyres release with `GET /v1/version`)

First published version of this spec. `GET /v1/version` is new; a Core that
answers it with `404` is older than this spec.
