# Standardizing API Error Handling and Slack Incident Reporting

**Hana Kim · Front-End Team Lead, BCSD Lab · KOIN · 2024**  
**Stack:** React, TypeScript, Axios, Slack

## At a glance

KOIN is a campus community platform developed by BCSD Lab. I worked on making API failures easier for developers to handle consistently and unexpected client failures easier to report.

The approach was to normalize application errors at the API boundary, distinguish them from other failures with a TypeScript discriminator, and provide a shared path for user feedback and Slack reporting. I documented the convention for the team and built **bbi-bot** for production-error alerts.

[Original Korean team documentation](https://github.com/BCSDLab/KOIN_WEB_RECODE/wiki/Koin-Error-Convention-%282024%29) · [Application repository](https://github.com/BCSDLab/KOIN_WEB_RECODE) · [BCSD Lab](https://bcsdlab.com/)

## My contribution

As a front-end team lead, my work included standardizing typed API error handling, authoring the team's error-handling and reporting convention, and building bbi-bot to connect production errors with Slack alerts.

The document describes the shared approach for both **KOIN_WEB_RECODE** and **KOIN_OWNER_WEB**. The former normalizes errors in its API client; the latter applies normalization through Axios response interceptors. The code below explains that design rather than reproducing the complete production implementation.

## The problem

An API failure was not always an application error with a usable response body. KOIN's application errors contained a numeric `code` and a `message`, while Axios also exposed transport-related information and errors without a response.

Without a common convention, feature code had to inspect Axios-specific properties, check whether response data existed, and decide how to display or report the failure. Repeating those decisions across features made behavior harder to keep consistent.

The design needed to answer two different questions:

- **Is this a recognized API failure that the feature can handle?** Show relevant feedback to the user.
- **Is this an unexpected failure?** Report it through a shared utility and give the user a fallback message.

## Design: normalize once, handle intentionally

```text
API request fails
    |
    v
API client / Axios response interceptor
    |
    +-- Recognized KOIN error payload
    |       -> { type: "koin-error", status, code, message }
    |       -> Type guard and feature-specific user feedback
    |
    +-- Other failure
            -> { type: "axios-error", ... }
            -> Shared reporting utility -> Slack
            -> Fallback user feedback
```

### A shared error contract

The normalization layer assigned a `type` discriminator to the error. Feature code could then check the application-level error contract instead of repeatedly navigating Axios's nested response structure.

In the original convention, the KOIN error carried the HTTP status alongside the server's numeric code and message. The alternative branch retained the Axios error information and tagged it as `axios-error`.

### Integration at the API boundary

KOIN_WEB_RECODE handled conversion inside its API client. KOIN_OWNER_WEB used response interceptors for its clients, including the multipart client.

Both integration points served the same purpose: normalize the error before it reaches feature-specific handling. This put the common conversion logic at a boundary shared by requests.

### Explicit handling in features

The convention included an `isKoinError` type guard so feature code could narrow an `unknown` error. The login example used HTTP status codes to choose user-facing messages for recognized API failures.

For the other branch, `sendClientError` forwarded error information and the current page URL to the Slack reporting endpoint. That gave developers a common reporting entry point instead of requiring every feature to build its own Slack request.

## Illustrative TypeScript

**These examples are explanatory adaptations, not verbatim production code.** They make runtime checks explicit and preserve an unexpected failure as a `cause`. The original code uses different helper signatures and retains Axios fields in its fallback object. The endpoint below is a placeholder; this is not a drop-in production client.

### 1. Convert an unknown failure into a discriminated union

```ts
type KoinError = {
  type: "koin-error";
  status: number;
  code: number;
  message: string;
};

type UnexpectedError = {
  type: "axios-error";
  cause: unknown;
};

type ClientError = KoinError | UnexpectedError;

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null;
}

function normalizeError(error: unknown): ClientError {
  if (isRecord(error) && isRecord(error.response)) {
    const { status, data } = error.response;

    if (
      typeof status === "number" &&
      isRecord(data) &&
      typeof data.code === "number" &&
      typeof data.message === "string"
    ) {
      return {
        type: "koin-error",
        status,
        code: data.code,
        message: data.message,
      };
    }
  }

  return { type: "axios-error", cause: error };
}
```

### 2. Normalize errors in the response interceptor

```ts
client.interceptors.response.use(
  (response) => response,
  (error: unknown) => Promise.reject(normalizeError(error)),
);
```

This example assumes an existing Axios client. It illustrates the interceptor-based integration described for KOIN_OWNER_WEB.

### 3. Separate expected feedback from unexpected-error reporting

```ts
function handleFailure(error: ClientError) {
  if (error.type === "koin-error") {
    // A feature can branch on status/code and choose appropriate wording.
    showToast("error", userMessageFor(error));
    return;
  }

  // The reporter serializes the error and sends approved context
  // to a server-side reporting endpoint, which forwards it to Slack.
  void reportClientError(error.cause, window.location.href).catch(() => {
    // A reporting failure must not prevent fallback user feedback.
  });

  showToast("error", "Something went wrong. Please try again.");
}
```

`showToast`, `userMessageFor`, and `reportClientError` represent application helpers, not complete implementations. The original document calls the reporting helper `sendClientError` and shows the notification request, with serialization details omitted.

## Outcomes and evidence

The original team document records a concrete convention, client integration examples, a type guard, feature usage, and a Slack reporting utility. It is useful evidence of the design and the guidance made available to developers.

The intended benefits were more consistent handling, less repeated interpretation of Axios errors, and a shared way to report unexpected failures. **The source does not provide before-and-after measurements for error counts, development time, or incident response time.** I therefore do not attach a percentage improvement or MTTR claim to this case study.

## Trade-offs and limitations

- **HTTP status versus business meaning.** The convention's example branches on HTTP status because the numeric application codes were not sufficiently clear for the intended handling. A documented business-error taxonomy would make more precise handling possible.
- **A type guard needs a runtime contract.** A discriminator is convenient for narrowing, but it is not a substitute for validating data at the boundary. The illustrative code above adds explicit checks; that should not be read as a claim about checks in the historical implementation.
- **Slack reporting is only one part of incident response.** The document does not establish deduplication, rate limiting, alert routing, or incident ownership. Those remain important follow-up questions as reporting volume grows.
- **Reporting context needs care.** Page URLs and serialized errors can contain sensitive data. Redaction, payload allowlists, and keeping Slack credentials server-side would be requirements for a production-ready reporting design; the source alone does not establish those protections.

## What this work demonstrates

The engineering decision was to establish a contract between infrastructure and feature code: normalize failures centrally, make branching explicit, and give the team a reusable reporting path. The convention also made the design teachable through written examples rather than leaving error handling as an implicit practice.

## Source and scope

This English case study is based on [Koin Error Convention (2024)](https://github.com/BCSDLab/KOIN_WEB_RECODE/wiki/Koin-Error-Convention-%282024%29), the original Korean team documentation. It is a retrospective explanation, not a claim that the repository's current implementation is unchanged. The original remains the reference for the historical code; illustrative adaptations and future considerations are labeled above.
