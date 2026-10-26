# Aignostics

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **Not from the company, and here with a question?** You are welcome here — we would rather be the
> front line and point you the right way than have a good report go nowhere. What this repository
> can answer is narrow, though, so it is worth knowing who you are actually looking for:
>
> - **A question about how the API works, an account, billing, or a bug in the service** — that is
>   the company's own support, not us. We profile this API; we do not operate it and cannot see
>   your account.
> - **A bug in an open-source project we only catalog** — file it on that project's own repository.
>   This has happened with a real and correct bug report that reached us instead of the people who
>   could fix it, which helped nobody.
> - **Anything about this listing itself** — the description, the tags, the rating, a missing or
>   wrong artifact — is ours. Open an issue here.
> - **Not sure, or something general about API Evangelist or APIs.io** — open an issue on the
>   [APIs.io Inbox](https://github.com/api-search/inbox) and we will route it.
>
> **This repository contains no software, and we will never ask you to download anything.** There is
> no build, release, installer, or binary here — only text and machine-readable API descriptions, so
> there is nothing here that can be "corrupt" or need "repairing". Any issue, comment, or email
> claiming otherwise and offering a download link is not from us and is hostile. Do not follow the
> link; it is a lure. Report it to GitHub and, if you like, tell us at
> [info@apievangelist.com](mailto:info@apievangelist.com) so we can take it down.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Aignostics GmbH (Berlin, Germany) builds AI foundation models and analysis applications for
computational pathology. The commercial surface is the **Aignostics Platform** — a cloud service that
runs pathology applications over whole slide images on dedicated GPU infrastructure, with Atlas
H&E-TME tumor-microenvironment profiling as the flagship.

## API surface

| | |
|---|---|
| API | Aignostics Platform API |
| Base URL | `https://platform.aignostics.com/api/v1` |
| Contract | OpenAPI 3.1.0 — 26 operations, 45 schemas, `info.version` 1.8.0 |
| Contract URL | <https://platform.aignostics.com/api/v1/openapi.json> (public, unauthenticated) |
| Auth | OAuth 2.0 bearer — authorization-code and device-code (RFC 8628) against an Auth0 tenant |
| Access | Organization subscription. No anonymous access, no API keys, no self-serve signup. |

The contract was captured from the live host, not from the copy vendored in the SDK repository —
that copy was still at `1.4.0` with 14 operations and pointed its OAuth URLs at a **staging** Auth0
tenant on the day the live document served `1.8.0` with 26 operations against production.

## Links

- Documentation — <https://aignostics.readthedocs.io/en/latest/>
- Get started with the API — <https://aignostics.readthedocs.io/en/latest/get_started_api.html>
- Status — <https://status.aignostics.com/> (Better Stack; JSON, RSS and Atom feeds)
- GitHub — <https://github.com/aignostics>
- Python SDK — <https://pypi.org/project/aignostics/> · TypeScript SDK — <https://www.npmjs.com/package/@aignostics/sdk>
- Company — <https://www.aignostics.com/>

## Notable findings

- **No idempotency.** The docs state plainly that `POST /v1/runs` is not idempotent — calling it
  twice analyzes the slides twice, and bills twice. No `Idempotency-Key` exists anywhere in the spec.
- **Strong reversibility.** Every costly or destructive write has a named reversal with a *stated*
  window: cancel any time before `TERMINATED` (pending items "will not add to the cost"), revoke a
  grant or share token at any time. Artifact deletion is the one-way exception, and artifacts are
  auto-deleted 30 days after a run finishes regardless.
- **402, not 429.** The only exhaustion signal is a commercial quota (slides per run, monthly slides)
  whose values are unpublished. No rate-limit headers, no `Retry-After`.
- **Domain standards in the contract.** `application/dicom` is an accepted input media type, and
  `ApplicationReadResponse.regulatory_classes` enumerates `RUO`, `IVDR`, `FDA` per application.
- **MCP ships but is dark.** `aignostics mcp run` starts a real stdio MCP server inside the Python
  SDK, but no FastMCP tools are registered in-tree, and the customer guide is withdrawn (404).
- **`GET /v1/me` returns credentials** — the organization's GCS HMAC secret access key, a Logfire
  token and a Sentry DSN. Do not log or cache that response.
- **No pricing, no terms, no `security.txt`.** A vulnerability disclosure process does exist, in the
  SDK's security policy.

Harvest source: <https://equityzen.com/company/aignostics>
