# Rules for this repo

This repository is public. It holds the GitHub Actions steps that customers of the umatix platform use on the
runners umatix provides.

- Nothing internal goes in here: no host names of internal services beyond what a customer needs, no credentials,
  no names of customers, no notes about the platform's internals.
- Changes go by pull request with a review, never directly to `main`.
- Release tags are immutable. A change gets a new tag; a tag is never moved.
- A step must work with what the runner provides (see the README) and must need no secret from the calling repo.
- Test a change with the cases under `testdata/`.
