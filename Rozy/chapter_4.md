# chapter 4 처리율 제한 장치의 설계

클라이언트 or 서비스가 보내는 트래픽의 처리율 제어하기 위한 장치
- DoS 공격에 의한 자원 고갈을 방지 (추가 요청에 대해서는 처리를 중단함으로써 공격 방지)
- 추가 요청 처리 없이 횟수 제한할 수 있어 비용 절감
- 서버의 과부하 방지

### 1단계: 문제 이해 및 설계 범위 확정
- 어떤 종류, 클라이언트를 위한 장치 or 서버측 제한 장치, 시스템 규모, 분산 환경 여부, 독립된 서비스 여부, 사용자에게 공지 여부
- 면접관에게 질문하여 요구사항을 정확하게 확인한다

### 2단계: 개략적 설계안 제시 및 동의 구하기
- 기본적인 클라이언트-서버 통신 모델 사용
- 클라이언트 요청은 쉽게 위변조가 가능하여 이왕이면 서버 측에 설계
- API 게이트웨이라 불리는 컴포넌트에 구현되는 경우도 있음 (처리율 제한을 지원하는 미들웨어)
- 정답은 없고 설계 환경, 목표 등에 따라 달라짐

- 요청의 접수 정도를 추적할 수 있는 카운터를 추적 대상별로 두고 사용자별로 추적? IP 주소별로? API 엔드포인트? 서비스 단위? -> 무엇으로 추적대상을 잡을것인지 정해야 함
- 카운터는 어디에 보관할 것인가? (레디스)

*토큰 버킷 알고리즘*
- 지정된 용량을 갖는 컨테이너 토큰 버킷
- 토큰이 꽉 찬 버킷에는 더이상 토큰 추가 불가
- 요청이 처리될 때마다 하나의 토큰 사용
- 토큰이 없는 경우 해당 요청은 버려짐
- 인자: 버킷 크기, 토큰 공급률
- 구현이 쉽다 / 메모리 사용 효율적 / 짧은 시간에 집중되는 트래픽 처리 가능

*누출 버킷 알고리즘*
- 요청 처리율 고정
- 빈자리가 있는 경우 큐에 요청 추가
- 큐가 가득 차 있는 경우 새 요청 버림
- 지정된 시간마다 큐에서 요청 꺼내어 처리
- 인자: 버킷 크기, 처리율
- 큐의 크기가 제한되어 있어 효율적인 메모리 사용량 / 고정된 처리율 -> 안정적 출력

*고정 윈도 카운터 알고리즘*
- 타임라인을 고정된 간격의 윈도를 나누고, 각 윈도마다 카운터 붙임
- 요청이 접수될 때마다 카운터 값 1씩 증가
- 카운터 값이 임계치에 도달하면 새로운 요청은 새 윈도가 열릴 때까지 버려짐
- 그러나, 윈도 경계 부근에 트래픽이 집중될 경우, 그 사이에 허용 한도보다 많이 처리하게 됨
- 윈도가 닫히는 시점에 카운터를 초과하는 방식은 특정한 트래픽 패턴을 처리하기에 적합

*이동 윈도 로깅 알고리즘*
- 고정 윈도 카운터 알고리즘 문제 해결
- 새 요청이 오면 만료된 타임스탬프는 제거
- 새 요청의 타임스탬프를 로그에 추가
- 로그의 크기가 허용치보다 같거나 작으면 요청을 시스템에 전달 / 않은 경우 처리 거부
- 요청의 타임스탬프를 추적하는 방법
- 그러나, 거부된 요청의 타임스탬프도 보관하기 때문에 다량의 메모리 사용

### 3단계: 상세 설계
- '처리율 제한 규칙은 어떻게 만들어지고 어떻게 저장?'
  - config 설정 파일 형태로 디스크에 저장

- '처리가 제한된 요청들은 어떻게 처리?'
  - 한도 제한에 걸리면 API는 HTTP 429 응답을 클라이언트에 보냄
  - 큐에 보관했다가 나중에 처리할 수도 있음
  - 남은 처리 가능 요청 수, 클라이언트가 전송 가능한 요청 수, 한도 제한에 걸리지 않기 위해 몇초 후 보내야 하는지 - 헤더로 함께 반환

- 동기화 이슈
  - 수백만 사용자를 지원하려면 처리율 제한 장치 서버도 여러대
  - 동기화 필요
  - 같은 클라이언트로 부터의 요청은 항상 같은 처리율 제한 장치로 보내는 방법
  - 레디스와 같은 중앙 집중형 데이터 저장소를 쓰는 방법

- 성능 최적화
  - 에지서버: 세계 곳곳에 에지서버를 두어 사용자의 트래픽을 가장 가까운 에지 서버로 전달하여 지연시간을 줄임
  - 최종 일관성 모델 사용: 제한 장치 간에 동기화 할때 사용

- 모니터링
  - 효과적으로 동작하고 있는지 보기 위해
  - 제한 알고리즘, 처리율 제한 규칙 확인

#### 4단계: 마무리
- 경성(hard) or 연성(soft) 처리율 제한
  - 경성 처리율 제한: 요청의 개수는 임계치를 절대 넘을 수 없다
  - 연성 처리율 제한: 요청 개수는 잠시 동안은 임계치를 넘어설 수 있다
- 다양한 계층에서의 처리율 제한
  - 애플리케이션 계층 뿐만 아니라, IP주소 등 7계층
- 클라이언트 측의 최선
  - 캐시를 사용하여 API 호출 줄이기
  - 짧은 시간동안 너무 많은 메세지 보내지 않도록
  - 예외 처리하는 코드 도입
  - 재시도 로직 구현할 때는 충분한 백오프 시간 두기

# 논의주제
시스템 설계 또한 기획, 전략 모든 팀과 함께 논의하면서 이루어져야 겠구나 느꼈던 점이, 서비스가 어떤 서비스, 어떤 유저를 대상으로 하느냐에 따라 어떤 요청은 경성 처리할지, 어떤 요청은 연성 처리할지도 정해질 수 있겠구나 생각이 들었다.

처음 단순하게 설계 후, 트래픽을 보면서 처리방법에 대해 정하고 업그레이드할 수 도 있지만, 다양한 관점에서 논의 후 애초 설계단계에 처리시스템을 설계해도 좋을것 같다