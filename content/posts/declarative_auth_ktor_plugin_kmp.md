------
title: "Building a Custom Ktor Plugin to Guard Authenticated Routes"
date: "2026-03-18"
draft: false
hideToc: true
tags: ["KMP", "ktor", "authentication", "testing", "architecture"]
series: "Tv Maniac Journey"
---

In an [earlier post](/posts/intercepting_network_requests/), I set up `HttpResponseValidator` to intercept error responses and map them into a sealed `ApiResponse` type. That worked well for *handling* errors after they happen. This post goes a step further: preventing unauthorized requests from ever reaching the network using a Ktor Custom Plugin.

This handles errors after they occur. I am now preventing unauthorized requests from reaching the network using a Ktor Custom Plugin.

The Trakt API includes both public and authenticated endpoints. Previously, data sources called `httpClient.safeRequest { ... }` without differentiating between them. I initially added manual login checks to repositories:

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

This approach does not scale. Adding new authenticated endpoints requires manually adding dependencies and checks.

## Scattered enforcement limitations

The codebase lacked a consistent method for enforcing authentication. Some data sources checked state before API calls while others did not. Background sync interactors occasionally attempted network requests for logged out users, leading to errors.

Ktor's `Auth` plugin would return null tokens for unauthenticated users, but requests still reached the network and failed with 401 errors. For endpoints requiring authentication, these failures were predictable.

## Auth guard plugin implementation

I moved enforcement to the HTTP layer using a custom Ktor client plugin. The plugin intercepts requests, checks an attribute, and short circuits if the user is unauthenticated.

Using `createClientPlugin` allows defining a configuration and hooking into the request pipeline. The `onRequest` hook runs before outgoing requests to enforce requirements.

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

Ktor's `onRequest` hooks cannot return synthetic responses. Preventing a request requires throwing an exception. I catch this exception at the network boundary to convert it into a standard response type.

## Exception handling at the network boundary

The `safeRequest` extension wraps Ktor exceptions into `ApiResponse` variants. I catch `AuthenticationException` and convert it to `ApiResponse.Unauthenticated`:

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
    } catch (e: Exception) {
        ApiResponse.Error.GenericError(...)
    }
```

`ApiResponse.Unauthenticated` indicates the request was skipped because the user was logged out. This allows downstream code to differentiate between server rejections (401) and client side guard triggers.

## Declarative auth requirements

I created `authSafeRequest` to check auth state before a request reaches the plugin. It returns `ApiResponse.Unauthenticated` early if the user is logged out or sets the `RequiresAuth` attribute and delegates to `safeRequest`:

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

This provides a two layer defense. The pre-check handles common cases, while the plugin serves as a safety net for race conditions or direct `RequiresAuth` usage. Data sources now explicitly define requirements:

```kotlin
// Public endpoint
override suspend fun getUser(userId: String): ApiResponse<TraktUserResponse> =
    httpClient.safeRequest { ... }

// Auth-required endpoint
override suspend fun getUserList(userId: String): ApiResponse<List<TraktPersonalListsResponse>> =
    httpClient.authSafeRequest { ... }
```

## Unauthenticated user experience

When a logged out user triggers an authenticated fetch, `authSafeRequest` returns `ApiResponse.Unauthenticated` without making an HTTP call. The Store fetcher maps this to an error, and the UI falls back to local database content. No request leaves the device, and the user sees cached data without error messages.

## Verification strategy

I use Ktor's `MockEngine` for three levels of testing.

**Plugin tests** verify the guard matrix including authenticated and unauthenticated states. Tests confirm the plugin throws `AuthenticationException` or permits the request based on the `RequiresAuth` attribute.

**Extension tests** ensure `authSafeRequest` returns `ApiResponse.Unauthenticated` early for logged out users. They also verify `safeRequest` converts `AuthenticationException` to the correct variant.

**Data source integration tests** verify the full stack from call site to mock server response. These confirm correct request construction and auth guard behavior.

## Final considerations

I transitioned from scattered auth checks to declarative enforcement at the HTTP layer. This follows the pattern of pushing decisions to the lowest capable layer while maintaining a consistent `ApiResponse` surface for upstream consumers.

Until we meet again, folks. Happy coding! ✌️
