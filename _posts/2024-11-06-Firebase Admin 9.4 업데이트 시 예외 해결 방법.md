---
layout: post
title: "Firebase Admin(FCM) 9.4+ 업데이트 시 예외 해결 방법"
date: 2024-11-06
categories: [라이브러리]
tags: [라이브러리, FCM, Firebase Cloud Messaging]
description: "Firebase Admin SDK 9.4 업데이트 후 발생하는 Apache HttpClient 관련 예외(ClassNotFoundException) 원인 분석과 Spring Boot 3 이하 환경에서 FirebaseOptions 설정으로 문제를 해결하는 방법을 제공합니다."
---
## 개요

사내에는 어드민과 알림 관련 서비스 프로젝트에서 Firebase Cloud Messaging(이하 FCM)을 사용하고 있었다. 어드민은 Spring Boot 2.7 버전이고, 알림 서비스는 3.2 버전이다.
라이브러리 버전은 `Firebase-admin:9.2.0`을 사용하였는데, 해당 버전에서 스레드풀 설정 이후 `FirebaseMessaging sendEachForMulticastAsync`
메소드를 사용하면 문제가 생겨서(이는 추후 포스팅하겠다.) 라이브러리 버전을 업데이트 해야했다.
따라서 최신 버전인 `Firebase-admin:9.4.1`로 업데이트 했는데, 어드민쪽 프로젝트에서 다음과 같은 예외가 터졌다.

```
...
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'firebaseApp' defined in class path resource ...: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.google.firebase.FirebaseApp]: Factory method 'firebaseApp' threw exception; nested exception is java.lang.NoClassDefFoundError: org/apache/hc/client5/http/config/ConnectionConfig
Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.google.firebase.FirebaseApp]: Factory method 'firebaseApp' threw exception; nested exception is java.lang.NoClassDefFoundError: org/apache/hc/client5/http/config/ConnectionConfig
Caused by: java.lang.NoClassDefFoundError: org/apache/hc/client5/http/config/ConnectionConfig
Caused by: java.lang.ClassNotFoundException: org.apache.hc.client5.http.config.ConnectionConfig
at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:641)
at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:188)
```

## Firebase Message Cloud 9.4 업데이트 시 참조사항
9.4.0 버전부터 SDK에서 사용되는 기본 HttpTransport가 Google API의 기본 HTTP Transport인 `NetHttpTransport`에서 HTTP/2를 지원하는 ApacheHttp2Transport을 사용하는 구현체로 변경되었다고 한다. 
이는 FCM 배치 엔드포인트가 더 이상 사용되지 않음에 따라 증가하는 FCM 요청에 필요한 HTTP/2 지원을 제공하기 위한 노력의 일환이라고 하는데 코드로 확인하자.

`HttpTransport`는 FCM의 모든 푸시 요청에 사용한다.
예를 들어 `FirebaseMessaging sendEachForMulticastAsync`를 호출하면 결국 `FirebaseMessagingClientImpl send`를 호출하는데, 이곳에서도 사용하는 것을 볼 수 있다.

```
final class FirebaseMessagingClientImpl implements FirebaseMessagingClient {
  ...
  
  public String send(Message message, boolean dryRun) throws FirebaseMessagingException {
    return sendSingleRequest(message, dryRun);
  }

  private String sendSingleRequest(
      Message message, boolean dryRun) throws FirebaseMessagingException {
    HttpRequestInfo request =
        HttpRequestInfo.buildJsonPostRequest(
            fcmSendUrl, message.wrapForTransport(dryRun))
            .addAllHeaders(COMMON_HEADERS);
    MessagingServiceResponse parsed = httpClient.sendAndParse(
        request, MessagingServiceResponse.class);
    return parsed.getMessageId();
  }
}
```
send는 sendSingleRequest를 호출하고 여기서 `httpClient.sendAndParse`를 호출한다.
메소드를 따라 내려가면 결국 `FirebaseMessaging.fromApp`를 호출하는데 아래 코드들을 호출한다.
```
  static FirebaseMessagingClientImpl fromApp(FirebaseApp app) {
    ...
    return FirebaseMessagingClientImpl.builder()
        .setProjectId(projectId)
        .setRequestFactory(ApiClientUtils.newAuthorizedRequestFactory(app))
        .setChildRequestFactory(ApiClientUtils.newUnauthorizedRequestFactory(app))
        .setJsonFactory(app.getOptions().getJsonFactory())
        .build();
  }

  public static HttpRequestFactory newAuthorizedRequestFactory(
      FirebaseApp app, @Nullable RetryConfig retryConfig) {
    HttpTransport transport = app.getOptions().getHttpTransport();
    return transport.createRequestFactory(new FirebaseRequestInitializer(app, retryConfig));
  }

  public static HttpRequestFactory newUnauthorizedRequestFactory(FirebaseApp app) {
    HttpTransport transport = app.getOptions().getHttpTransport();
    return transport.createRequestFactory();
  }
```
아래 코드를 확인하자.
```
...
.setRequestFactory(ApiClientUtils.newAuthorizedRequestFactory(app))
.setChildRequestFactory(ApiClientUtils.newUnauthorizedRequestFactory(app))
```
해당 코드에서 ApiCliecntUtils의 스태틱 메소드를 호출한다. 

```
public class ApiClientUtils {
  static final RetryConfig DEFAULT_RETRY_CONFIG = RetryConfig.builder()
      .setMaxRetries(4)
      .setRetryStatusCodes(ImmutableList.of(503))
      .setMaxIntervalMillis(60 * 1000)
      .build();
  ...

  public static HttpRequestFactory newAuthorizedRequestFactory(FirebaseApp app) {
    return newAuthorizedRequestFactory(app, DEFAULT_RETRY_CONFIG);
  }

  public static HttpRequestFactory newAuthorizedRequestFactory(
      FirebaseApp app, @Nullable RetryConfig retryConfig) {
    HttpTransport transport = app.getOptions().getHttpTransport();
    return transport.createRequestFactory(new FirebaseRequestInitializer(app, retryConfig));
  }

public static HttpRequestFactory newUnauthorizedRequestFactory(FirebaseApp app) {
    HttpTransport transport = app.getOptions().getHttpTransport();
    return transport.createRequestFactory();
  }
}
```
`newAuthorizedRequestFactory` 메소드와 `newUnauthorizedRequestFactory` 메소드 모두 `app.getOptions().getHttpTransport();`를 호출한다.

즉 FCM 푸시 요청을 보낼 때 사용하는 메소드들은 모두 Http 요청을 보내는데 요청은 `HttpTransport` 클래스를 사용하는 것이다.
이곳에서 예외가 터졌으니 이제 이제 `HttpTransport`가 초기화되는 곳을 살펴보면 예외를 해결할 수 있다. HttpTransport는 `FirebaseOptions`에서 가져온다. FirebaseOptions가 초기화되는 코드를 살펴보자.
아래는 9.4 이전 버전 코드다.

```
public final class FirebaseOptions {
  ...
  private FirebaseOptions(@NonNull final FirebaseOptions.Builder builder) {
    this.httpTransport = builder.httpTransport != null ? builder.httpTransport
      : ApiClientUtils.getDefaultTransport();
  }
}

public class ApiClientUtils {
  ...
  public static HttpTransport getDefaultTransport() {
    return Utils.getDefaultTransport();
  }
}

public final class Utils {
    public static HttpTransport getDefaultTransport() {
        return Utils.TransportInstanceHolder.INSTANCE;
    }

    private static class TransportInstanceHolder {
        static final HttpTransport INSTANCE = new NetHttpTransport();
    }
}
```
`FirebaseOptions`에 `httpTransport`를 설정하지 않으면 `ApiClientUtils.getDefaultTransport()`로 초기화하는데 이는 `NetHttpTransport`를 반환한다.

이제 문제가 된 업데이트 이후를 살펴보자. 아래는 최신 버전인 `9.4.1`이다.

```
public final class FirebaseOptions {
  ...
  private FirebaseOptions(@NonNull final FirebaseOptions.Builder builder) {
    this.httpTransport = builder.httpTransport != null ? builder.httpTransport
      : ApiClientUtils.getDefaultTransport();
  }
}
public class ApiClientUtils {
  ...
  public static HttpTransport getDefaultTransport() {
        return new ApacheHttp2Transport();
  }
}

```
9.4 버전 이후부터는 `ApiClientUtils.getDefaultTransport()` 호출 시 `ApacheHttp2Transport`를 반환한다.

그럼 왜 어드민쪽 프로젝트에서 예외가 터졌을까? [해당 블로그 포스팅](https://velog.io/@chiyongs/Spring-Boot-3.x-%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%93%9C-%EC%8B%9C-Apache-HttpClient-%EC%9D%98%EC%A1%B4%EC%84%B1-%EB%AC%B8%EC%A0%9C)으로 원인을 확인할 수 있었다.

>이유 : Spring Framework 6.0 버전부터 Apache HttpClient에 대한 지원이 제거되면서 org.apache.httpcomponents.client5:httpclient5 라이브러리로 교체되었습니다.

스프링 프레임워크 6.0 이상부터 httpclient5 의존성이 생긴다. 어드민 프로젝트는 스프링부트 2.7 버전이었기 때문에 해당 라이브러리 의존성이 없어 예외가 터졌던 것이다.

![의존성이 없는 사진](/assets/images/posts/2024-11-08/1.png)

## 해결
해결은 간단하다. `FirebaseOptions` 빌더에 원하는 `HttpTransport`를 추가하면 된다.

```
FirebaseOptions options = FirebaseOptions.builder()
    .setHttpTransport(Utils.getDefaultTransport())
    .build();
    
return FirebaseApp.initializeApp(options);
```
`Utils.getDefaultTransport()`는 라이브러리 버전 9.4 이전에 `transport`를 초기화하던 방식이다. 해당 방식으로 `FirebaseApp`을 생성한다면 예외가 터지지 않는다.

참고
> https://github.com/firebase/firebase-admin-java/wiki/HTTP-Transport
