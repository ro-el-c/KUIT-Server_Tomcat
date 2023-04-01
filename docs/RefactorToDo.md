# Refactor
- 변하지 않는 값 -> Enum 사용
  - 상수
  - final 로 선언한 path 문자열
  - http 헤더, 메서드
- RequestHandler 에서 맡은 일 분리
  - request 메시지 분석 클래스 분리
  - responseHeader 클래스 분리
  - url 별 작업 분리