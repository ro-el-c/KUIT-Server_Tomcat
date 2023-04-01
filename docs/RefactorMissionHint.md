# Tomcat 구현 - 리팩토링
### Tomcat 구현 2단계 -> 객체지향적 고민을 통한 리팩토링

***
## 요구사항
## 요구사항 1: Enum 활용하기

```
HttpHeader, HttpMethod, URL, 상태 코드, User의 queryKey 등의 상수 -> Enum 클래스 사용
```

<br><br>

## 요구사항 2: HttpRequest 메시지 분리

```
HttpRequest메시지와 HttpResponse메시지를 현재 RequestHandle에서 분석하고 있는데, 이는 단일 책임 원칙에서 어긋난다.

BufferedReader 를 통해 HttpRequest 메시지를 분석하는 HttpRequest 클래스를 만들어보자.

HttpRequest 메시지는 무조건 분석되어 초기화 되어야하기 때문에 때문에 최초 생성 시부터 데이터를 넘겨야 한다.

정적 팩토리 메서드 패턴을 사용해보자.
`e.g., Httprequest httpRequest = Httprequest.from(BufferedReader);`
```

### 힌트 및 상세정보
- HttpRequest메시지 구조를 다시 분석해보자.
  - HttpStartLine (Method, path, version)
  - HttpHeader
  - body
- HttpRequest 메시지도 자연스레 3개의 인스턴스 변수를 가지면 되겠다고 생각들지 않는가?
- 하나씩 차례대로 구현해보자.
- 위 클래스들도 꼭 테스트 하면서 진행해보자.
  - HttpRequest에는 InputStream을 넣어주어야 테스트할 수 있는데 어떻게 할 수 있을까?
  - test에 resource 폴더를 생성하고 그곳에 테스트할 HttpRequest 메시지 파일들을 넣어두자.
  - e.g.
    ```
    POST /user/create HTTP/1.1
    Host: localhost:8080
    Connection: keep-alive
    Content-Length: 40
    Accept: */*
    
    userId=jw&password=password&name=jungwoo
    ```
  - 그리고 위 파일을 파일 인풋스트림으로 읽어와 BufferedReader를 만들면, HttpRequest.from(BufferedReader)로 httpRequest 객체를 만들 수 있고 이제 위 텍스트 파일을 httpRequest가 제대로 분석하는지 확인하면 된다.
  - e.g.,
    ```java
    private BufferedReader bufferedReaderFromFile(String path) throws IOException {
            return new BufferedReader(new InputStreamReader(Files.newInputStream(Paths.get(path))));
    }
    
    HttpRequest httpRequest = HttpRequest.from(bufferedReaderFromFile(testDirectory + getPath));
    assertEquals("/user/create", httpRequest.getUrl());
    //...
    ```

<br><br>

## 요구사항 3: HttpResponse 분리

```
HttpResponse도 HttpRequest와 비슷하게 OutputStream에 쓰는 Response 메시지를 쓰는 책임을 분리할 수 있을 것 같다.

outputStream을 인자로 받아 원하는 html 파일을 보여주는 forward(path) 함수와 redirect 시켜주는 redirect(path) 함수를 구현해보자.
```

### 힌트 및 상세정보
- HttpResponse 구조를 다시 분석해보자.
  - StartLine (Version, Status Code, Message)
  - Headers
  - body
- Headers는 Request에 사용한 것을 재활용할 수 있다.
- HttpRequest와 같은 방식으로 outputStream에 내용을 써주기만 하면 된다.
- 마찬가지로 테스트를 하며 HttpResponse가 실제로 잘 내용을 쓰는지 확인하면서 진행하자
    ```java
    private OutputStream outputStreamToFile(String path) throws IOException {
            return Files.newOutputStream(Paths.get(path));
    }
    
    HttpResponse httpResponse = new HttpResponse(outputStreamToFile(testDirectory+forwardPath));
    
    httpResponse.forward("/index.html");
    ```
  - 파일 내용을 하나하나 확인하는 과정이 귀찮긴 하지만, 웹을 실행시켜보면서 확인하는 것보다는 이렇게 확인하는 것이 편할 것이다.


<br><br>

## 요구사항 4: url 마다의 작업 분리

```
RequestHandler는 각 url마다 분기처리도 해주어야하고, 분기마다의 작업 처리도 해주어야한다.

즉, 책임의 분리가 필요해보인다.

각 작업의 처리를 각 작업마다 Controller 클래스를 만들어 분리시켜보자.
```

### 힌트 및 상세정보
- 각 컨트롤러들은 모두 들어온 url을 execute하는 같은 역할을 할것으로 보인다.
- 즉, Controller 인터페이스를 만들어서 모든 컨트롤러들이 implements하게 만들어보자.
- 각각의 Controller에 response와 request를 넘겨주어 알아서 처리하게끔 해보자.
- 위 모두를 구현한 코드다.
    ```java
    package webserver;
    
    import controller.*;
    // import 생략
    
    public class RequestHandler implements Runnable {
        Socket connection;
        private static final Logger log = Logger.getLogger(RequestHandler.class.getName());
    
        private final Repository repository;
        private Controller controller = new ForwardController();
    
        public RequestHandler(Socket connection) {
            this.connection = connection;
            repository = MemoryUserRepository.getInstance();
        }
    
        @Override
        public void run() {
            log.log(Level.INFO, "New Client Connect! Connected IP : " + connection.getInetAddress() + ", Port : " + connection.getPort());
            try (InputStream in = connection.getInputStream(); OutputStream out = connection.getOutputStream()) {
                BufferedReader br = new BufferedReader(new InputStreamReader(in));
                DataOutputStream dos = new DataOutputStream(out);
    
                // Header 분석
                HttpRequest httpRequest = HttpRequest.from(br);
                HttpResponse httpResponse = new HttpResponse(dos);
    
                // 요구 사항 1번
                if (httpRequest.getMethod().isEqual("GET") && httpRequest.getUrl().endsWith(".html")) {
                    controller = new ForwardController();
                }
    
                if (httpRequest.getUrl().equals("/")) {
                    controller = new HomeController();
                }
    
                // 요구 사항 2,3,4번
                if (httpRequest.getUrl().equals("/user/signup")) {
                    controller = new SignUpController();
                }
    
                // 요구 사항 5번
                if (httpRequest.getUrl().equals("/user/login")) {
                    controller = new LoginController();
                }
    
                // 요구 사항 6번
                if (httpRequest.getUrl().equals("/user/userList")) {
                    controller = new ListController();
                }
                controller.execute(httpRequest, httpResponse);
    
            } catch (Exception e) {
                log.log(Level.SEVERE, e.getMessage());
                System.out.println(Arrays.toString(e.getStackTrace()));
            }
        }
    }
    ```

<br><br>

## 요구사항 5: 분기문 처리

```
모든 컨트롤러가 Controller를 implements 하고 있으므로,
Map 자료구조에 Controller들을 url과 매핑 시켜 담을 수 있다.

이를 통해 위 분기문들을 제거해보자.

e.g.,
```
```java
    //url, controller가 key value 형태로 저장
    Map <String,Controller> controllers = new HashMap<>();
    controllers.get(url)
```
```
이러한 RequestHandler가 이 기능을 맡는 것도 책임이 과중한 것 같다.

RequestMapper클래스를 구현해 요청을 controller와 매핑시키는 책임을 맡게끔 해보자.

```

위와 같은 요구사항을 모두 만족 시켰다면 다음과 같은 깔끔한 RequestHandler를 얻을 수 있다.

```java
package webserver;

import http.request.HttpRequest;
import http.response.HttpResponse;

import java.io.*;
import java.net.Socket;
import java.util.Arrays;
import java.util.logging.Level;
import java.util.logging.Logger;

public class RequestHandler implements Runnable {
    Socket connection;
    private static final Logger log = Logger.getLogger(RequestHandler.class.getName());

    public RequestHandler(Socket connection) {
        this.connection = connection;
    }

    @Override
    public void run() {
        log.log(Level.INFO, "New Client Connect! Connected IP : " + connection.getInetAddress() + ", Port : " + connection.getPort());
        try (InputStream in = connection.getInputStream(); OutputStream out = connection.getOutputStream()) {
            BufferedReader br = new BufferedReader(new InputStreamReader(in));
            DataOutputStream dos = new DataOutputStream(out);

            HttpRequest httpRequest = HttpRequest.from(br);
            HttpResponse httpResponse = new HttpResponse(dos);

            RequestMapper requestMapper = new RequestMapper(httpRequest,httpResponse);
            requestMapper.proceed();

        } catch (Exception e) {
            log.log(Level.SEVERE, e.getMessage());
            System.out.println(Arrays.toString(e.getStackTrace()));
        }
    }
}
```

<br>