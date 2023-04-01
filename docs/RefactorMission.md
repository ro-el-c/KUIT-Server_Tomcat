# Tomcat 구현 - 리팩토링
### Tomcat 구현 2단계 -> 객체지향적 고민을 통한 리팩토링

***
## 요구사항
> #### 요구사항 1: Enum 활용하기

HttpHeader, HttpMethod, URL, 상태 코드, User의 queryKey 등의 상수 -> Enum 클래스 사용

<br><br>

> #### 요구사항 2: HttpRequest 메시지 분리

HttpRequest메시지와 HttpResponse메시지를 현재 RequestHandle에서 분석하고 있는데, 이는 단일 책임 원칙에서 어긋난다.

BufferedReader 를 통해 HttpRequest 메시지를 분석하는 HttpRequest 클래스를 만들어보자.

HttpRequest 메시지는 무조건 분석되어 초기화 되어야하기 때문에 때문에 최초 생성 시부터 데이터를 넘겨야 한다.

정적 팩토리 메서드 패턴을 사용해보자.

`e.g., Httprequest httpRequest = Httprequest.from(BufferedReader);`

<br><br>

> #### 요구사항 3: HttpResponse 분리

HttpResponse도 HttpRequest와 비슷하게 OutputStream에 쓰는 Response 메시지를 쓰는 책임을 분리할 수 있을 것 같다.

outputStream을 인자로 받아 원하는 html 파일을 보여주는 forward(path) 함수와 redirect 시켜주는 redirect(path) 함수를 구현해보자.

<br><br>

> #### 요구사항 4: url 마다의 작업 분리

RequestHandler는 각 url마다 분기처리도 해주어야하고, 분기마다의 작업 처리도 해주어야한다. 

즉, 책임의 분리가 필요해보인다.

각 작업의 처리를 각 작업마다 Controller 클래스를 만들어 분리시켜보자.

<br><br>

> #### 요구사항 5: 분기문 처리

모든 컨트롤러가 Controller를 implements 하고 있으므로,
Map 자료구조에 Controller들을 url과 매핑 시켜 담을 수 있다.

이를 통해 위 분기문들을 제거해보자.

e.g.,
```java
    //url, controller가 key value 형태로 저장
    Map <String,Controller> controllers = new HashMap<>();
    controllers.get(url)
```

이러한 RequestHandler가 이 기능을 맡는 것도 책임이 과중한 것 같다.

RequestMapper클래스를 구현해 요청을 controller와 매핑시키는 책임을 맡게끔 해보자.

<br>