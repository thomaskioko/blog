---
title: "Custom Ktor Plugin - Guarding Authenticated Requests"
date: "2026-03-18"
draft: false
hideToc: true
tags: ["KMP", "ktor", "authentication", "testing", "architecture"]
series: "Tv Maniac Journey"
---

In an [earlier post](/posts/intercepting_network_requests/), I set up `HttpResponseValidator` to intercept error responses and map them into a sealed `ApiResponse` type. That worked well for *handling* errors after they happen. This post goes a step further: preventing unauthorized requests from ever reaching the network using a Ktor Custom Plugin.

The Trakt API exposes both public endpoints (trending shows, popular shows) and authenticated ones (watchlist, calendar, user lists). Previously, the client treated them all the same. Every data source called `httpClient.safeRequest { ... }` regardless of whether the endpoint needed auth or not.

I then added a call in the repository that checks if the user is authenticated and returning early.

```kotlin
@Inject
public class DefaultUserRepository(
    private val traktAuthRepository: TraktAuthRepository,
) : UserRepository {
    override suspend fun fetchUserProfile(username: String, forceRefresh: Boolean) {
        if (!traktAuthRepository.isLoggedIn()) return
        ...
    }
}
```
This approach looks good at first glance. But, it doesn't scale well. If we add a new API call that needs authentication, we need to add the Trakt Api dependency and remember to add this check.

## Scattered Implementation

There was no consistent way to enforce auth requirements across the codebase. Some data sources checked auth state before calling the API. Others didn't. Background sync interactors would fire off network requests for logged-out users, only to get errors. Auth enforcement was scattered across individual call sites with no single place to manage it.

The HTTP layer didn't help either. Ktor's `Auth` plugin would return `null` tokens for unauthenticated users, but the request would still go out, fail with a 401, and surface as an error in the UI. For endpoints like the watchlist or calendar, these requests were guaranteed to fail. The client already had enough information to know that, but nothing was acting on it.


## Building an Auth Guard Plugin

Instead of spreading auth checks across every data source, I moved enforcement to the HTTP layer with a custom Ktor client plugin. The plugin intercepts requests *before* they leave the client, checks a request attribute, and short-circuits if auth is required but the user isn't logged in.

Ktor's [`createClientPlugin`](https://ktor.io/docs/client-custom-plugins.html) API is designed for exactly this kind of cross-cutting concern. You define a configuration class, install the plugin on your `HttpClient`, and hook into the request/response pipeline. The `onRequest` hook runs before every outgoing request, which makes it the right place to enforce auth. Our plugin reads the `RequiresAuth` attribute, checks the user's auth state, and throws if the request needs auth but doesn't have it.

```kotlin
internal val TraktAuthGuard = createClientPlugin("TraktAuthGuard", ::TraktAuthConfig) {
    val isAuthenticated = pluginConfig.isAuthenticated

    onRequest { request, _ ->
        val requiresAuth = request.attributes.getOrNull(RequiresAuth) == true
        if (requiresAuth && !isAuthenticated()) {
            throw AuthenticationException(
                message = "Authentication required for ${request.method.value} ${request.url.buildString()}",
            )
        }
    }
}
```
One thing worth noting: `onRequest` hooks can inspect and modify requests, but they can't return synthetic responses. You *can* use `return@onRequest` to exit the hook early, but that just skips the rest of the hook — the request still goes out to the network. To actually prevent the request from being sent, you need to throw. That's why the plugin uses an exception. But we don't want that exception leaking across layers, so the goal is to catch it as close to the throw site as possible and convert it into something the rest of the app already understands.


## Handling Network Exceptions

The `safeRequest` extension from the [earlier post](/posts/intercepting_network_requests/) wraps all Ktor exceptions into `ApiResponse` variants. It's the natural boundary between what Ktor throws and what the rest of the app works with. So that's where we catch `AuthenticationException` and convert it into `ApiResponse.Unauthenticated`:

```kotlin
suspend inline fun <reified T> HttpClient.safeRequest(
    block: HttpRequestBuilder.() -> Unit,
): ApiResponse<T> =
    try {
        val response = request { block() }
        ApiResponse.Success(response.body())
    } catch (e: AuthenticationException) {
        ApiResponse.Unauthenticated
    } catch (exception: ClientRequestException) {
        ApiResponse.Error.HttpError(...)
    } catch (e: SerializationException) {
        ApiResponse.Error.SerializationError(...)
    } catch (e: Exception) {
        ApiResponse.Error.GenericError(...)
    }
```

`ApiResponse.Unauthenticated` is a dedicated variant on the sealed class, distinct from HTTP errors. A 401 from the server means "your token was rejected." `Unauthenticated` means "we knew you weren't logged in and didn't bother asking." The distinction matters because downstream code can handle them differently — retry with a refreshed token for the former, skip silently for the latter.

The exception exists because of a Ktor limitation in the plugin layer, but it never escapes the network boundary. Everything upstream sees a regular `ApiResponse` and handles it through existing paths. No special cases, no leaking abstractions.


## One Method Call per Data Source

With the exception contained, the next step was making auth requirements declarative at the call site. `authSafeRequest` checks auth state *before* the request reaches the plugin, returning `ApiResponse.Unauthenticated` early if the user isn't logged in. If they are, it sets the `RequiresAuth` attribute and delegates to `safeRequest`:

```kotlin
suspend inline fun <reified T> HttpClient.authSafeRequest(
    block: HttpRequestBuilder.() -> Unit,
): ApiResponse<T> {
    val isAuthenticated = attributes.getOrNull(IsAuthenticated)
        ?: return ApiResponse.Unauthenticated
    if (!isAuthenticated()) return ApiResponse.Unauthenticated
    return safeRequest {
        attributes.put(RequiresAuth, true)
        block()
    }
}
```

This creates a two-layer defense. The `authSafeRequest` pre-check is the first line — it handles the common case where a logged-out user triggers a fetch. The plugin acts as a safety net for the narrow race where a user logs out between the pre-check and the actual request execution, or for any caller that sets `RequiresAuth` without going through `authSafeRequest`.

With this in place, the migration is straightforward. Every auth-required endpoint switched from `safeRequest` to `authSafeRequest`:

```kotlin
// Public endpoint, no auth needed
override suspend fun getUser(userId: String): ApiResponse<TraktUserResponse> =
    httpClient.safeRequest { ... }

// Auth-required endpoint
override suspend fun getUserList(userId: String): ApiResponse<List<TraktPersonalListsResponse>> =
    httpClient.authSafeRequest { ... }
```

The difference is one method call. When you read a data source, the choice between `safeRequest` and `authSafeRequest` tells you immediately whether that endpoint requires authentication. No need to check external docs or trace through auth middleware.


## What Happens to Unauthenticated Users

This is where the layering pays off. When a logged-out user opens a screen that triggers an auth-required fetch:

1. `authSafeRequest` pre-checks auth state and returns `ApiResponse.Unauthenticated` — no HTTP call is made
2. The Store's fetcher maps this to `FetcherResult.Error`
3. The Store falls back to its `SourceOfTruth` (local database)
4. The user sees cached data, or an empty state if there's nothing cached

No request ever leaves the device. No error toast. No special handling in the domain layer. The user simply sees what's available locally, and when they log in, the sync kicks in naturally.


## Testing

Three layers of tests cover different concerns, all using Ktor's `MockEngine`.

**Plugin tests** verify the guard in isolation — the matrix of `RequiresAuth` present/absent crossed with authenticated/not. Four cases, each asserting the plugin either throws `AuthenticationException` or lets the request through.

**Extension tests** verify that `authSafeRequest` returns `ApiResponse.Unauthenticated` early when the user isn't logged in, and that `safeRequest` catches any `AuthenticationException` from the plugin layer and converts it to `Unauthenticated` rather than letting it propagate.

**Data source integration tests** exercise the full stack: data source calling `authSafeRequest`, the plugin intercepting, and the mock server responding. These verify request construction (method, path, query params, body) alongside the auth guard behavior. When auth is missing, the data source returns `ApiResponse.Unauthenticated` — the same shape downstream code handles for all auth failures, so nothing needs to care *why* the request was skipped.

JSON fixtures live in `jvmTest/resources` and are loaded via a `loadJson()` helper, keeping tests readable and fixture data versioned alongside the test code.


## Wrapping Up

We moved from implicit, scattered auth handling to explicit, declarative enforcement at the HTTP layer. Adding a new auth-required endpoint means switching one method call from `safeRequest` to `authSafeRequest`. The plugin handles the rest.

The pattern here is the same one that keeps coming up across TvManiac: push decisions to the lowest layer that has enough information to make them, and keep the layers above unaware of the details. The network layer knows about auth state, so it enforces auth. Everything above just works with `ApiResponse`.

Until we meet again, folks. Happy coding! ✌️


### References

- [TvManiac Repository](https://github.com/c0de-wizard/tv-maniac)
- [Ktor Client Plugins Documentation](https://ktor.io/docs/client-custom-plugins.html)
- [Ktor MockEngine Documentation](https://ktor.io/docs/client-testing.html)
- [Intercepting Ktor Network Responses in KMP](/posts/intercepting_network_requests/)
