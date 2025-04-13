# ServiceDto (Command, Request, Response) 설계 정리

## 1. 배경

일반적으로 **Layered Architecture**에서 `Controller`와 `Service` 레이어 간에 데이터를 주고받는 방식은 다음과 같이 구분할 수 있습니다.

1. **Controller → Service (요청)**
2. **Service → Controller (응답)**

초기에 많이 사용하는 방법은 아래와 같습니다.

- **Service에서 도메인 객체를 바로 반환**하여 Controller에서 변환 후 응답
- **Service에서 `ResponseDto`를 직접 생성** 하여 Controller에 반환

여기서 “**어떤 계층에서 어떤 Dto를 만들고, 어떤 로직을 담당해야 하는가**” 라는 고민이 생깁니다. 특히, **유지보수성**과 **계층 간 결합도** 관점에서 설계를 어떻게 해야 할지에 대한 고민으로 이어집니다.

---

## 2. Service → Controller 응답

### 2.1 두 가지 방식

#### (1) Service에서 바로 `ResponseDto` 생성 후 반환

- 가장 흔히 볼 수 있는 방식
- Service 계층이 응답에 필요한 항목(포맷)을 모두 구성하고 Controller에 전달
- Controller에서는 Service가 준 Dto를 그대로 리턴만 하면 되므로 간편

#### (2) Service에서 **도메인 객체**를 반환하고, Controller에서 `ResponseDto`로 변환

- Service 계층이 **Web/화면 요구사항(View)에 종속되지 않도록** 설계
- **View 요구사항이 바뀔 때**(응답 스펙 변경) Service 로직이 변경될 필요가 크게 줄어듦
- 단점: Controller에 도메인 객체가 노출되며, **Controller에서 도메인 로직이 실행**될 위험이 생길 수 있음

### 2.2 (2)번 방식을 사용하는 이유

1. **역할 분리**
    - Service는 비즈니스 로직 담당,
    - Controller는 요청/응답 포맷 담당
    - “응답 포맷을 만드는 일”을 Controller에서 수행하여 **관심사를 분리**함

2. **View 요구사항 변경으로부터의 격리**
    - 화면(View) 요구사항이 자주 변경되는 경우 `ResponseDto` 또한 변경될 가능성이 큼
    - Service가 `ResponseDto`를 리턴하도록 되어 있으면 Service도 변경해야 하는 불필요한 의존이 생김
    - 도메인 자체를 반환하면 Service 수정은 최소화됨

### 2.3 유지보수 & 위험성

- **유지보수**
    - Service가 도메인을 반환 → Controller에서 응답 포맷을 만듦
    - View(응답 포맷) 변화 시 Service 수정 부담 감소
- **위험성**
    - Controller에서 도메인 객체를 직접 다룰 수 있기 때문에, **도메인 비즈니스 로직이 Controller에 들어갈 수도 있음**
    - 이 위험을 최소화하려면, **코드리뷰나 컨벤션, 협업 프로세스**를 통해 Controller에서 비즈니스 로직이 들어가지 않도록 관리해야 함

---

## 3. Controller → Service 요청

### 3.1 일반적인 방식: `RequestDto` → Service

일반적으로 Controller에서 요청을 받을 때,  
`RequestDto`를 그대로 Service로 넘기는 방식을 자주 사용합니다.
예시:

```java
@PostMapping
public ResponseEntity<StationSaveResponse> createStation(@RequestBody StationRequest stationRequest) {
    StationSaveResponse response = stationSaveService.saveStation(stationRequest);
    return ResponseEntity.ok(response);
}
```

## 3.2 Service Dto (Command) 사용하는 방식

Controller에서 받은 `RequestDto`를 **Service 계층에서 사용하는 전용 Dto**(예: `StationCommand`)로 변환 후,  
Service로 넘기는 방식입니다.

이렇게 하면 **Controller에서 원하는 포맷**(입력 데이터 형태)과 **Service에서 실제로 사용하는 비즈니스 포맷**을 명확히 분리할 수 있습니다.

- **네이밍**: 보통 `Command`, `ServiceRequest`, `ServiceDto` 등으로 부르는 경우가 많습니다.

**예시 코드**:

```java
@PostMapping
public ResponseEntity<StationSaveResponse> createStation(@RequestBody StationRequest stationRequest) {
    // (1) StationRequest -> StationCommand 변환
    StationCommand stationCommand = stationRequest.toCommand();

    // (2) 비즈니스 로직 수행
    StationSaveResponse stationSaveResponse = stationSaveService.saveStation(stationCommand);

    // (3) Controller는 결과를 반환
    return ResponseEntity.ok(stationSaveResponse);
}
```

## 3.3 왜 번거롭게 분리할까?

- **요청 포맷과 비즈니스 로직이 필요한 포맷이 달라질 가능성**
    - 외부 API 연동, 추가적인 전처리/검증, 비즈니스 요구사항 변경 등으로  
      실제 DB나 내부 로직에서 필요한 파라미터가 달라질 수 있음
    - 이럴 때, Controller의 `RequestDto`를 그대로 사용하면 결합도가 높아질 수 있음

- **유지보수성**
    - Controller에서 받는 요청 형식이 달라지면, Service 내부로도 영향이 전파되지 않도록 분리

---

## 3.4 리팩토링 시점

- 처음부터 Service Dto를 무조건 분리하기보다,  
  **Controller와 Service가 원하는 포맷이 달라지는 상황**이 분명해졌을 때 **리팩토링**을 고려하는 것이 좋을 수 있음
- 실제로 대규모 프로젝트에서는, **외부 API**나 특정 비즈니스 요구사항을 새로 추가할 때  
  Service Dto 방식으로 **점진적으로 변경**하기도 함

---

## 4. 결론

### Service → Controller 응답

1. **Service에서 `ResponseDto`를 반환**
    - 간단하지만 **Service가 View 요구사항에 종속**
2. **Service에서 도메인 객체를 반환**하고, Controller에서 `ResponseDto` 생성
    - **유지보수성**이 높아지지만, Controller에서 도메인 로직이 섞일 위험을 관리해야 함

### Controller → Service 요청

1. **Controller에서 받은 `RequestDto`를 그대로 Service에 전달**
2. **Controller에서 Service 전용 Dto(예: Command)** 로 변환 후 전달
    - **유지보수성** 관점에서 유리
    - Controller와 Service 간 결합도를 줄일 수 있음

결국, **프로젝트 규모**, **팀 컨벤션**, **요구사항 변동성**, **협업 방식** 등에 따라 선택이 달라질 수 있습니다.  
처음부터 모든 것을 분리하기보다는, **프로젝트 상황**과 **변경 가능성**을 고려해 적절한 시점에 리팩토링하는 것이 일반적입니다.

---

### 참고 링크

- [우아한형제들 기술 블로그: Controller와 Service의 강한 결합](https://techblog.woowahan.com/2711/)
