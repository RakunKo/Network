### ABSTRACT

- data center switch의 이용가능한 제한된 buffer 공간에 대한 TCP의 요구는 latency를 초래한다.
- 이를 해결하기 위한 DCTCP는 ECN을 사용
    - end host에게 다중 bit feedback을 제공
    - shallow buffered switch를 사용
    - 90% 적은 buffer를 사용하고도 TCP의 같거나 더 좋은 throughput을 제공
    - 짧은 흐름에 대해 높은 burst tolerance와 low latency를 제공

### 1. INTRODUCTION

- data center의 디자인은 낮은 cost와 상품 요소를 이용하여 높은 가용성, 높은 연산능력과 저장소 infra를 구축한다.
- low-cost switch → the top of the rack(ToR)
    - low latency for short flows
        - 직접적으로 반환되는 결과에 대한 품질에 영향을 미친다.
    - high burst tolerance
    - high utilization for long flows
        - 지속적으로 내부의 데이터 구조를 업데이트 해야한다 → 결과의 품질 영향을 미친다
- data center의 traffic의 99.91%는 TCP traffic이다.
    - query traffic (2KB to 20KB)
        - **incast impairment** 문제가 발생한다.
    - delay sensitive short message (100KB to 1MB)
    - throughput sensitive long flow (1MB to 100MB)
        - long flow가 switch의 이용 가능한 buffer를 소비하고 있다면, Query 와 delay-sensitive short message는 긴 latency를 경험한다.

<img width="442" alt="스크린샷 2024-01-26 오후 3 27 04" src="https://github.com/RakunKo/Network/assets/145656942/a50f35bc-eec5-41ce-ad77-33c484bccfc9">

- DCTCP는 ECN과 결합한 protocol
- source는 표시된 packet의 비율을 추정하고, 해당 추정치를 congestion 정도에 대한 신호로 사용한다,
    - DCTCP가 높은 throughput 상태에서 매우 낮은 buffer occupancies를 차지하게 작동한다.
- world wide area network와는 다르게 매우빠른 RTT와 높은 대역폭, 매우 낮은 latency를 가진다.
    - 외부 인터넷과의 연결은 load balancer와 application proxy에 의해 관리된다.

- Delay-based protocols는 congestion과 늘어나는 queueing delay의 신호로 증가하는 RTT 측정값을 사용한다.
    - 정확한 RTT 측정에 심하게 의존한다.
- Active Queue Management(AQM)은 congested switch로 부터 생성된 명백한 feedback을 사용한다.
    - DCTCP에선 AQM 방법을 선택

### **2. COMMUNICATIONS IN DATA CENTERS**

**2.1 Partition/Aggregate**

<img width="422" alt="스크린샷 2024-01-26 오후 3 42 38" src="https://github.com/RakunKo/Network/assets/145656942/7253b019-1146-416d-b313-6ade5d995503">

- 상위 계층으로 부터의 Request는 조각으로 쪼개져 하위 계층의 worker로 전달된다.
- 결과를 생산하기 위해 worker의 reponses들이 집계된다.
- application의 backend 부분에는 230~300ms가 할당되고, 이를 **SLA**라고 부른다.

- 한 계층의 지연은 다른 계층의 지연을 유발하고, response를 준비하기 위해 aggregator가 여러개의 request를 worker에게 전달하는 과정을 **반복적**으로 요구할 수도 있다.
- Partition/Aggregate에 지연 instances가 추가되어 전체 **SLA**를 위협할 수도 있다.
    - 지연이 SLA target과 가깝다면, 개발자는 최선의 결과를 위해 이용가능한 시간 예산 모두를 사용해야 한다.
    - 전체 SLA 침해를 막기위해, worker node는 **tight한 deadline**을 할당 받는다 (10~100ms)
        - deadline을 놓치면 그 response 없이 계산이 진행되므로 결과의 품질이 떨어진다.
        - 추가로 worker의 latency가 증가한다.
        - 99.9번째 percentile는 1000번의 response중 적어도 1개의 낮은 품질과 긴 지연을 의미
            - latency는 99.9 percnetiles에서 추적되고 dealine은 높은 percentiles과 연관이 있다
- **결국 SLA를 위배하지 않으면서 deadline을 놓치지 않을 적절한 tight한 deadline 설정이 중요하다.**

**2.2 Workload Characterization**

- data center : 웹 검색 및 기타 서비스를 지원하는 세 개의 프로덕션 클러스터
- server : 6000개의 server와 150개의 racks
- 트래픽 유형
    - soft 실시간 쿼리 트래픽
    - 긴급한 짧은 message 트래픽
    - 지속적인 background 트래픽

**Query Traffic**

- cluster 내의 Query traffic은 Partition/Aggregate pattern을 따른다.
    - 매우 짧은 latency-critical flow
- High-Level aggregator(HLA)는 query를 Mid-Level aggregators(MLAs)로 나눈다.
    - MLA는 동일한 rack에 있는 43개의 server에 대해 각 query를 분할한다.
    - server는 MLAs와 workers를 동시에 수행

<img width="210" alt="스크린샷 2024-01-26 오후 5 15 39" src="https://github.com/RakunKo/Network/assets/145656942/c2a0fc6d-8630-4c36-8571-4bcab2d37851">

**Background Traffic**

<img width="455" alt="스크린샷 2024-01-26 오후 5 18 26" src="https://github.com/RakunKo/Network/assets/145656942/b790c3c4-6d65-4bee-9a5b-26894676614f">

- 크고 작은 flows들로 구성되어 있다
- 크기가 큰 flow가 많진 않지만 차지하는 분포를 확인하면 대부분이 큰 사이즈의 flow임을 알 수 있다.
- Figure 3(b)는 새로운 background flows의 도착 사이 시간을 나타낸다.
    - background flow 사이의 도착 시간은 application을 지원하는 다양한 서비스의 중첩과 다양성을 반영한다.
        
        1) interarrival time의 변화가 매우 심하고, 꼬리가 매우 무겁다.
        
        2) 내장된 스파이크가 발생 (**최대 50번째 백분위수까지 y축을 껴안는 CDF를 설명하는 0ms 도착 간 발생)**
        
        3) 상대적으로 많은 수의 나가는 flow가 주기적으로 나타나며, 이는 worker가 업데이트된 파일을 찾기 위해 정기적으로 여러 peers를 polling하는 결과
        

**Flow Concurrency and Size**

<img width="430" alt="스크린샷 2024-01-26 오후 5 26 13" src="https://github.com/RakunKo/Network/assets/145656942/da6aef5f-e88f-44e1-9117-ac25e67a05c2">

- 최근에 쳠아한 MLA나 worker node의 개수의 CDF
- 99.9 percentile는 1600이상의 concurrent Connections
- large flow만 고려한다면, 통계적 다중화 정도가 매우 낮다.
    - median 값은 1이며, 75th percentile는 2이다.
    - large flow는 여러 RTT동안 지속될 정도로 충분히 크며, 대기열 축적을 유발하여 상당한 buffer space를 소비할 수 있다.

**throughput-sensitive large flows, delay sensitive short flows, bursty query traffic은 data center network에 공존한다.**

**2.3 Understanding Performance Impairments**

***2.3.1 Switches***

- 클러스터의 switch는 모든 switch의 port에서 이용 가능한 논리적 packet buffer를 사용하여 통계적 다중화 이득을 목표로하는 ***shared memory switch***다
    - interface에 도착하는 packet은 모든 interface에 공유된 high speed multi-ported memory에 저장된다.
    - 공유 공간의 memory는 MMU에 의해 packet이 할당된다.
        - MMU는 한 interface의 최대 memory 양을 동적으로 조절하여 불공정을 방지하면서 각 interface가 필요한 만큼의 memory를 할당하려 시도한다.
        - 나가는 interface에 queued 되어 있고, 이미 최대의 memory 할당이며, 공유 공간이 고갈되었다면 packet은 drop 된다.
- 큰 multi-ported memories는 너무 비싸기 때문에, 대부분의 싼 switch는 ***shallow buffer이다.***
    - packet buffer가 가장 적은 리소스라 **3가지의 문제**가 일어난다.

***2.3.2 Incast***

<img width="444" alt="스크린샷 2024-01-27 오후 6 17 25" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/9a9f1ac3-c724-4c68-87ee-ad4f4263c8ce">

- Figure 6(a)처럼 많은 flow가 같은 switch의 interface에 단기간에 모이면, packet이 switch의 memory나 최대 허용된 buffer 모두를 고갈 시킬 수 있다. → packet loss의 결과
- 데이터에 대한 요청이 worker의 반응에 동기화되고 aggregator와 연결된 switch port의 queue에 incast를 생성하기 때문에 Partition/Aggregate 디자인 패턴에서 자연스럽게 일어난다.
- **incast 문제는 사용자의 경험과 성능 둘다 감소시킨다.**
- incast를 발생시키는 response는 aggregator의 deadline 대부분을 놓치고, 최종 결과에서 누락된다.

<img width="443" alt="스크린샷 2024-01-27 오후 6 25 03" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/6a076267-f2ee-42eb-8215-810a0db9eba4">

- 모든 response가 switch에 들어갈 확률을 높이기 위해 response의 크기를 고의로 2KB로 제한했다.
- 임의의 시간만큼 고의로 응답을 지연하여 응답을 비동기화 하기 위해 application-level jittering을 추가
    - 중간 response 시간을 늘리는 대신 (지연 추가 때문에), 높은 percentiles의 response 시간을 줄인다.
    - 높은 percentiles의 비율을 높인다.

<img width="463" alt="스크린샷 2024-01-27 오후 6 32 20" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/ff14fd6d-24a9-415f-beba-ec699e57f1cf">

### 요약

incast는 한 곳에 많은 flow가 갑자기 모였을 때 발생하며, Partition/Aggregate에서 흔하게 발생한다

이를 해결하기 위해 response 크기를 고의로 2KB로 제한하고, application-level jittering을 추가하여 충간 response 시간을 늘리고, 높은 percentiles의 response 시간을 줄인다.

***2.3.3 Queue buildup***

- 수명이 길고 탐욕스러운 TCP flow는 packet이 drop 될때 까지 병목 큐의 길이를 늘리는 원인
- long, short flow가 같은 큐에 들어오게 되면 2개의 문제가 일어난다.
    - short flow의 packet loss가 incast 문제를 일으킨다. (대역폭 낭비, packet header의 overhead, queue delay)
    - queue buildup 문제 : packet이 loss되지 않더라도, 큰 flow 때문에 뒤쳐진 short flow의 latency가 증가하는 현상 → 매우 자주 발생
        - background traffic(large flow)의 영향 받은 queue의 공간 차지가 주된 문제
- long flow가 latency에 영향을 주는지 확인하기 위해 worker와 aggregator의 RRT를 측정한다
    - worker가 response를 보내고 aggregator로 부터 **RTT+Queue**로 명칭된 TCP ACK을 받는 시간 간격

<img width="441" alt="스크린샷 2024-01-27 오후 6 52 15" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/93fc4cc5-8b03-4d35-bbce-a9270d30d3a6">

- 90%의 response packet은 1ms 미만의 RTT, 나머지 10%는 1~14ms의 RTT
    - 실질적으로 queuing delay를 경험하게 된다.
    - 요청에 응답하기 위해 여러 번의 반복이 필요할 수 있다 → 지연의 영향 확대
- **latency는 queuing 때문에 일어나므로, 큐 크기를 줄이는게 유일한 해결방법이다(질문)**

***2.3.4 Buffer pressure***

- Figure 6(c)처럼 short flow를 가진 하나의 port가 다른 port에 의해 영향 받는 것은 매우 흔하다.
- short flow의 loss는 다른 port의 long flow traversing의 영향을 받는다.
- long flow는 큐에 대기열을 생성하여 Partition/Aggregate traffic으로 부터 traffic burst 흡수시키는 이용 가능한 buffer size를 줄인다. → buffer pressure
- packet loss와 timeout을 일으킨다.

### 3. THE DCTCP ALGORITHM

- DCTCP의 목적은 높은 burst tolerance, low latency, high throughput을 shallow buffered switch로 성취하는 것
    - throughput의 손실없이 작은 queue occupancies로 작동하게 설계
- DCTCP는 buffer 점유율이 고정된 작은 threshold를 초과하는 즉시 Congestion Experienced(CE)를 설정하는 switch에 간단한 표시 방식을 사용
    - DCTCP의 source는 표시된 packet의 비율에 따라 window를 줄인다.
    - 비율이 클수록 감소도 커진다.
- 단일 bit seqence 표시에 있는 정보들로 다중 bit feedback을 파생
- DCTCP에서는 네트워크가 단일 bit feedback만을 제공해야 하므로 최신 TCP stack, switch에서 이미 사용 할 수 있는 ECN 기계 대부분을 재사용할 수 있다.

- 기존의 TCP는 ECN 알림을 받으면 window size를 2배로 줄이는데, 이는 매우 빠른 data center에선 buffer underlows와 throughput의 손실을 이끈다 → DCTCP의 필요성

**3.1 Algorithm**

1. Simple Marking at the switch
    - only a single parameter, the marking threshold, $K$
    - 만악 도착한 packet의 큐 점유율이 $K$보다 크다면 CE codepoint을 표시한다.
        - 빠르게 큐의 overshoot을 알릴 수 있다.
        
2. ECN-Echo at the receiver
    - DCTCP receiver는 표시된 packet의 정확한 순서를 sender에게 정확하게 전달하려한다.
        - ECN-Echo flag를 packet이 CE codepoint로 표시되었을 때 세팅하여 모든 packet에 ACK을 보내기

<img width="369" alt="스크린샷 2024-01-27 오후 8 09 11" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/8c33df86-0061-4090-97ee-fe967bde4eec">

- DCTCP의 receiver는 ECN-Echo bit를 설정하기 위해 2가지 상태 머신을 사용한다
    - 상태는 마지막으로 받은 packet에 CE codepoint로 표시 되어있는지 아닌지
    - sender는 각 ACK이 커버하는 packet의 수를 알고 있기 때문에 정확하게 receiver가 본 표시의 runs를 재구성할 수 있다.

1. Controller at the Sender
    - sender는 mark된 packet의 비율 측정을 유지한다 → $\alpha$
        
        $\alpha=(1-g) * \alpha + g*F$
        
    
    → queue의 크기가 K보다 클 확률을 추정
    
    - F는 마지막 데이터의 window 안의 mark된 packet의 비율
    - 0<g<1 → $\alpha$ 추정시 과거 대비 새로운 표본에 부여된 가중치
    - $\alpha$ 값이 1에 가까울 수록 congestion의 정도가 높은 것

- DCTCP는 $\alpha$를 사용하여 $cwnd$를 조절한다
    
    $cwnd = cwnd *(1-\alpha/2)$
    
- $\alpha$값이 0에 가까우면 매우 작은 감소가 일어난다.
    - $K$값을 queue가 초과하자 마자 DCTCP sender는 매우 작게 window를 감소시킨다.
- 이는 높은 처리량을 유지하면서 DCTCP가 적은 queue length를 유지할 수 있는 이유이다

**3.2 Benefits**