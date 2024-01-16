---
title: "Intercepting Ktor Network Responses in Kotlin Multiplatform"
date: "2023-07-25"
draft: false
hideToc: false
tags: ["KMP", "ktor"]
series: ["Tv Maniac Journey"]
---

![Photo by Omar Flores on Unsplash](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*2d0w3iXK1ATLsCtQXFSOOQ.jpeg)

### **Introduction:**

Handling various HTTP response codes is crucial when building mobile applications that communicate with APIs. This article will explore how to use [Ktor’s `HttpResponseValidator`](https://ktor.io/docs/response-validation.html) to intercept and handle responses in mobile applications.

We’ll use Trakt as a use case since it’s what I am for my project, but the concept should work for any API. One of the [status codes](https://trakt.docs.apiary.io/#introduction/status-codes) from Trakt is `403` (Forbidden), which indicates that the server understands the request but refuses to authorize it. We’ll see how we can intercept such responses, format the message and present this to the user.

Below is what we’ll be able to achieve at the end of this. 😎

![](https://cdn-images-1.medium.com/max/1600/1*kuq97P-RTzBUJS17yrRuXQ.png)

### TL;DR 

_If you just want to look at the code, click the link below._

[**Ktor Error Handling by c0de-wizard · Pull Request #95 · c0de-wizard/tv-maniac**  
_Description Implement HttpResponseValidator and intercept network responses. Previously we just showed an empty screen…_github.com](https://github.com/c0de-wizard/tv-maniac/pull/95 "https://github.com/c0de-wizard/tv-maniac/pull/95")[](https://github.com/c0de-wizard/tv-maniac/pull/95)

### Understanding HttpResponseValidator

[Ktor](https://ktor.io/) is a robust Kotlin-based framework for building server-side and client-side applications. It provides a flexible and extensible architecture to efficiently handle HTTP requests and responses. One of the key components in Ktor is the `HttpResponseValidator,` which allows developers to define a custom response.

``` kotlin

when (statusCode) { 
   in 300..399 -> throw RedirectResponseException(response) 
   in 400..499 -> throw ClientRequestException(response)  
   in 500..599 -> throw ServerResponseException(response) 
 }
 ```

### Intercepting Responses

To intercept responses, we need to add `HttpResponseValidator` inside the HttpClient’s body Ktor’s to check the response’s status code and take appropriate action. Before doing that, we need to enable default validation by setting the `expectSuccess` property to `true.` This terminates `HttpClient.receivePipeline` if the status code is unsuccessful.

``` kotlin
HttpClient(httpClientEngine) {  
 expectSuccess = true  
 ...  
  
 HttpResponseValidator {    
     validateResponse { response ->    
     
         if (!response.status.isSuccess()) {    
             val httpFailureReason = when (response.status) {    
                 HttpStatusCode.Unauthorized -> "Unauthorized request"    
                 HttpStatusCode.Forbidden -> "${response.status.value} Missing API key"                HttpStatusCode.NotFound -> "Invalid Request"    
                 HttpStatusCode.UpgradeRequired -> "Upgrade to VIP"    
                 HttpStatusCode.RequestTimeout -> "Network Timeout"    
                 in HttpStatusCode.InternalServerError..HttpStatusCode.GatewayTimeout ->    
                     "${response.status.value} Server Error"    
                 else -> "Network error!"    
             }    
     
             throw HttpExceptions(    
                     response = response,    
                     cachedResponseText = response.bodyAsText(),    
                     httpFailureReason = httpFailureReason,    
             )    
         }    
     }    
 }  
}
```

Here’s our custom exception class. 

``` kotlin
class HttpExceptions(    
    response: HttpResponse,    
    failureReason: String?,    
    cachedResponseText: String,    
) : ResponseException(response, cachedResponseText) {    
    override val message: String = "Status: ${response.status}." + " Failure: $failureReason"    
}
```

### API Response Wrapper

With the HttpResponseValidator in place, we can create an extension function to wrap API responses.

``` kotlin
suspend inline fun <reified T, reified E> HttpClient.safeRequest(    
    block: HttpRequestBuilder.() -> Unit,    
): ApiResponse<T, E> =    
    try {    
        val response = request { block() }    
        ApiResponse.Success(response.body())    
    } catch (exception: ClientRequestException) {    
        ApiResponse.Error.HttpError(    
            code = exception.response.status.value,    
            errorBody = exception.response.body(),    
            errorMessage = "Status Code: ${exception.response.status.value} - API Key Missing",    
        )    
    } catch (exception: HttpExceptions) {    
        ApiResponse.Error.HttpError(    
            code = exception.response.status.value,    
            errorBody = exception.response.body(),    
            errorMessage = exception.message,    
        )    
    } catch (e: SerializationException) {    
        ApiResponse.Error.SerializationError(e.message)    
    } catch (e: Exception) {    
        ApiResponse.Error.GenericError(e.message)    
    }
```

You can also use [kotlin-result](https://github.com/michaelbull/kotlin-result) or [Arrow’s Either](https://apidocs.arrow-kt.io/arrow-core/arrow.core/-either/index.html) depending on your needs. In this case, we will create our class. `ApiResponse`

``` kotlin
sealed class ApiResponse<out T, out E> {    
    /**    
     * Represents successful network responses (2xx).     
     * */    
   
    data class Success<T>(val body: T) : ApiResponse<T, Nothing>()    
    
    sealed class Error<E> : ApiResponse<Nothing, E>() {    
        /**    
         * Represents server errors.           
         * @param code HTTP Status code    
         * @param errorBody Response body    
         * @param errorMessage Custom error message    
         */          
        data class HttpError<E>(    
            val code: Int,    
            val errorBody: String?,    
            val errorMessage: String?,    
        ) : Error<E>()    
    
        /**    
         * Represent SerializationExceptions.           
         * @param message Detail exception message    
         * @param errorMessage Formatted error message    
         */          
        data class SerializationError(    
            val message: String?,    
            val errorMessage: String?,    
        ) : Error<Nothing>()    
    
        /**    
         * Represent other exceptions.           
         * @param message Detail exception message    
         * @param errorMessage Formatted error message    
         */          
        data class GenericError(    
            val message: String?,    
            val errorMessage: String?,    
        ) : Error<Nothing>()    
    }    
}
```

### API Requests

We can now create a function with a return type `ApiResonse<T>` and use the extension function we created to make the API call.

``` kotlin
override suspend fun getPopularShows(
page: Long
): ApiResponse<List<TraktShowResponse>, ErrorResponse> =    
    httpClient.safeRequest {    
        url {    
            method = HttpMethod.Get    
            path("shows/popular")     
        }    
    }
```

You can now invoke the Api call and handle the response based on your implementation. In my case, I am using [Store5](https://mobilenativefoundation.github.io/Store/). (Blog coming soon). We make the API call inside the `fetcher.` If successful, cache it else, throw an exception.

``` kotlin
{  
...  
fetcher = Fetcher.of {   
    val apiResponse = remoteDataSource.getPopularShows(page = 1)  
     when (apiResponse) {    
      is ApiResponse.Success -> {    
         // Map and cache data to local-source.  
      }    
      is ApiResponse.Error.HttpError -> throw Throwable("${response.errorMessage}")    
      is ApiResponse.Error.GenericError -> throw Throwable("${response.errorMessage}")    
      is ApiResponse.Error.SerializationError -> throw Throwable("${response.errorMessage}")  
 }  
}
```

By doing this, Store will wrap the exception in the `StoreReadResponse.Error` type, ensuring the flow does not break the stream and will still receive updates when data changes. We can now propagate the result and handle it in the presentation layer. In this case, when we get an exception, we show a SnackBar.

And that’s it! 🎊

I hope this post helped. If you liked it, give it some claps! 👏


### Reference material

- [Create a Fancy Toast Component Using SwiftUI](https://betterprogramming.pub/swiftui-create-a-fancy-toast-component-in-10-minutes-e6bae6021984)
- [Response validation](https://ktor.io/docs/response-validation.html)