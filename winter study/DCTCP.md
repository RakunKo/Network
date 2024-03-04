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
**Queue buildup**

- DCTCP sender는 queue의 길이 $K$를 초과하자마자 반응을 시작한다
    - congestion port의 queuing delay를 줄인다 (long flow의 영향을 줄인다)
- micro-burst(timeout을 이끌 수 있는 비용이 많이 드는 packet loss를 완화)를 *headroom*으로 더 많은 buffer space가 이용가능하다.

**Buffer pressure**

- congested port의 queue length가 매우 크게 증가하지 않는다 → buffer pressure 해결
- shared memory switch에서, 소수의 congested port가 다른 port를 통과하는 flow에 악영향을 주는 buffer resources를 고갈시키지 않을 것이다.

**Incast**

- DCTCP는 순간적인 대기열 길이를 기반으로 미리 marking을 시작하기 떄문에, source는 처음 1~2개의 RTT동안 후속 burst의 크기를 조절할 수 있을 만큼 충분한 mark를 받는다
- 이를 통해 timeout 결과로 이끄는 buffer overflow를 막는다.

**3.3 Analysis**

- N window size가 동기화 되어 있기 때문에 동일한 톱니 모양을 따르며, 시간 t에서의 대기열 크기는
    
    $Q(t) = NW(t) - C *RTT$
    
- W(t)는 단일 source의 window size

<img width="658" alt="스크린샷 2024-01-28 오후 2 41 52" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/bd4b9158-f6db-486b-a278-dcc1b3e0de80">

- 가장 중요한 진동의 진폭(amplitude of queue oscillations)은 congestion 표시에 대해 온화한 비례 반응으로 인해 DCTCP가 꾸준한 대기열을 얼마나 잘 유지할 수 있는지를 보여준다.
- 동기화된 sender의 경우 source가 ECN mark를 수신하고 이에 따라 window size를 줄이기 전에 톱니의 각 기간에서 정확히 하나의 RTT에 대해 대기열 크기가 mark threshold K를 초과한다.
- $\alpha$ 값을 단순히 해당 기간에 마지막 RTT동안 전송된 packet수를 톱니의 전체 기간 동안 전송된 총 packet 수 $T_c$로 나누어 계산이 가능하다.
    
    $S(W_1,W_2) = (W_2^2 - W_1^2)/2$
    
    $W = (C*RTT+K)/N$
    
- sender가 이러한 mark에 반응하는데 시간이 걸리고 window 크기는 하나의 packet만큼 증가하여 W+1에 도달한다.
    
    $\alpha = S(W^*,W^*+1)/S((W^*+1)(1-\alpha/2),W^*+1)$
    
    $\alpha^2(1-\alpha/4) = (2W^*+1)/(W^* +1)^2$ 는 W>1일때 유효하다.
    
    - C, RTT,N, K로 알파값을 구할 때 쓰인다.
    
    <img width="661" alt="스크린샷 2024-01-28 오후 3 04 32" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/f69cf8b9-63f3-4ab2-8cd3-42ce89ea4f3c">
    

<img width="652" alt="스크린샷 2024-01-28 오후 3 07 17" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/e69264db-9eee-4c61-a744-f99db2d8cf52">

- N이 10미만일때 정확하게 예측한다.
- N이 40일때는 비동기화된 flow가 작은 queue variation으로 이끈다.

- DCTCP에서의 amplitude of queue size(A)는 수식(8)에 의해 $O(\sqrt{C*RTT})$로 나오는데, 이는 TCP의 진폭인 $O(C*RTT)$보다 매우 작은 것을 알 수 있다.
- 진폭이 적기때문에 매우 작은 marking threshold K를 처리률의 손실 없이 설정할 수 있다.
    - 진폭이 전은 것은 큰 변동이나 혼잡이 없는 경우
    - 그러므로 미세한 변화에만 반응하게 설정해도 된다.
    - 작은 K는 네트워크 혼잡을 빠르게 감지하며, 성능 조절에 용이하다.

**3.4 Guidelines for choosing parameters**

- C는 packet/sec, RTT는 sec, K는 packet

**Marking Threshold**

$Q_{min} = Q_{max} - A = K+N-1/2\sqrt{2N(C*RTT+K}$

K의 하한을 찾기 위해 N에 대해 (12)를 최소화하고 이 최소값이 0보다 크도록 K를 선택한다

$K > (C*RTT)/7$

**Estimation Gain**

추정 이득 g는 지수 이동 평균(1)이 적어도 하나의 혼잡 이벤트를 spans할 수 있도록 충분히 작아야 함.

$(1-g)^{T_c} > 1/2$

최악의 경우 N=1을 사용하여 (9)를 연결하면

$= g < 1.386/\sqrt{2(C*RTT+K)}$

### 4. RESULT

**4.1 DCTCP Performance**

- **Throughput and queue length**
  
![스크린샷 2024-01-30 오후 2 31 52](https://github.com/RakunKo/Network/assets/145656942/cd2b9f32-17a0-4c65-b24b-35ef054bca46)

    - K = 20으로 설정하여도 DCTCP는 TCP와 같이 link utilization을 거의 100% 달성
    - Figure 13에서 확인할 수 있듯이 DCTCP는 TCP보다 queue length가 매우 짧다.
    - DCTCP는 낮고 안정적인 대기열 길이를 유지한다.
    
![스크린샷 2024-01-30 오후 2 32 38](https://github.com/RakunKo/Network/assets/145656942/013d0fdb-0754-49ee-a603-39fcb5bd2860)

- DCTCP의 성능은 K값에 대해서 둔감하다 (K가 5라는 낮은 값에서도…)
- K값이 추천된 값 65를 넘기는 순간부터 DCTCP는 TCP와 같은 처리량을 가지고, K에 대해 둔감해진다

- **RED**
  
![스크린샷 2024-01-30 오후 2 33 08](https://github.com/RakunKo/Network/assets/145656942/96846ca6-5646-40ac-bf62-14fa1477eb58)

    - K값을 기준으로 mark한 DCTCP와 RED 방식으로 mark한 TCP를 비교
    - Figure 15에선 DCTCP의 대기열 길이가 확실히 낮은 것을 확인할 수 있다.
        - RED (Adative Queue Management)는 대기열 길이에서 큰 진폭의 원인이 된다.
        - 이러한 대기열 축적은 microburst를 흡수할 수 있는 공간이 적다는 것을 의미
    - DCTCP는 매우 작은 대기열 길이에서도 최대 throughput을 달성할 수 있다.

- **Fairness and convergence**
  
![스크린샷 2024-01-30 오후 2 33 27](https://github.com/RakunKo/Network/assets/145656942/aab54fee-da3e-4c57-9653-78363562bcce)

    - DCTCP는 빠르게 throughput이 수렴하는 것을 Figure 16(a)에서 확인할 수 있다.
    - 반면 TCP는 평균은 공평하지만 변화가 많은 것을 확인할 수 있다.
    - flow의 개수가 많아져도 DCTCP는 빠르게 공유의 공평성과 빠른 수렴을 보여준다.

- **Multi-hop networks**
  
![스크린샷 2024-01-30 오후 2 33 43](https://github.com/RakunKo/Network/assets/145656942/bb097938-da62-4c45-a5d1-97212d1f633b)

    - Figure 17에서 Triumph1→Scorpion, Triumph 2→R1이 bottleneck link
    - S1은 R1에게 보낼때 2개의 병목 link 모두를 만난다.
    - DCTCP는 S1이 46Mbps, S3가 54Mbps, S2는 475Mbps의 처리량을 보인다.
    - TCP는 두개의 병목 링크의 대기열 길이 변화가 timeout의 원인이 된다.
    - DCTCP는 다중 bottlenecks와 변화하는 RTT를 다룰 수 있음을 보여준다.

**4.2 Impairment microbenchmarks**

***4.2.1 Incast***

- Basic Incast
  
![스크린샷 2024-01-30 오후 2 34 07](https://github.com/RakunKo/Network/assets/145656942/69052bbf-8e23-44fa-8314-ad21b7a9c5f4)

    - Figure 18 (a)에선 DCTCP가 server의 수가 35를 넘기 전까진 query delay가 더 적다.
        - 35를 넘어가도 TCP와 같은 수준의 성능으로 수렴
    - Figure 18(b)에선 적어도 한번 이상 timeout이 발생한 query의 비율을 보여준다.
        - DCTCP가 확연히 적은 비율을 유지한다.
        - 서버의 수가 늘어나도 TCP와 같은 수준으로 수렴
    - DCTCP sender가 ECN mark를 받고 rate를 늦추며 sender 수가 충분히 많아져 각 전송이 2개의 packet이 정적 buffer의 크기를 초과하는 경우에만 timeout이 발생한다.

- **Importance of dynamic buffering**
  
![스크린샷 2024-01-30 오후 2 34 21](https://github.com/RakunKo/Network/assets/145656942/9962f149-7800-4f59-975b-a765cce7367e)

    - query delay가 Figure 19(a)에서 DCTCP는 10ms수준을 계속 유지 → incast timeout X
    - 반면 TCP는 incast timeout을 겪고 있지만 dynamic buffering은 receiver port에 최대 700KB의 buffer를 할당하여 영향을 완화한다.

- **All-to-all incast**

![스크린샷 2024-01-30 오후 2 35 34](https://github.com/RakunKo/Network/assets/145656942/8fc8efe5-2961-476c-b796-d39294131027)

    - 동시에 여러 incast가 발생하면 어떻게 될까?
    - Figure 20에선 DCTCP의 높은 Query Latency의 비율이 TCP에 비해 적은 것을 확인 가능하다
        - DCTCP는 dynamic buffering이 memory에 대한 모든 요청을 처리할 수 있을 만큼 buffer space에 대한 요구를 낮게 유지하고 timeout을 겪지 않는다.

DCTCP는 sender가 너무 많아 첫번째 RTT에서 보낸 traffic이 buffer를 초과할 때 까지 incast문제를 timeout 없이 다룰 수 있다.

***4.2.2 Queue buildup***

![스크린샷 2024-01-30 오후 2 35 43](https://github.com/RakunKo/Network/assets/145656942/aa69cae2-dee1-40c5-8d22-cdce7d64cc8a)

- Figure 21에선 median delay가 1ms 미만으로 19ms의 TCP보다 매우 짧다.
- 전송되는 data의 양이 작기 때문에 완료 시간은 switch의 queue length에 좌우되는 RTT에 의해 영향을 받는다.
- **DCTCP는 queue lengths를 줄임으로서 작은 flow의 latency를 향상시킨다.**

***4.2.3 Buffer pressure***

![스크린샷 2024-01-30 오후 2 35 59](https://github.com/RakunKo/Network/assets/145656942/a4649f5b-1da3-45a2-b807-7387011d489f)

- Table 2에서는 DCTCP가 background traffic이 있을때나 없을때나 더 빠른 query 완료 시간을 보임
    - TCP를 사용하면 7%가 timeout을 겪고, DCTCP에서는 0.08%만이 timeout을 겪는다
- flow를 연결하는 buffer preesure를 줄여 흐름 간의 성능 격리 향상시킨다.

**4.3 Benchmark Traffic**

- query, short-message, background 3가지 traffic을 모두 발생시킨다
    - query traffic은 각 서버가 도착 시간 간 분포에서 가져온 다음, 다른 모든 서버에 query를 보내고 각 서버가 2KB 응답을 다시 보내도록 함으로써 Partition/Aggregate 구조에 따라 생성된다.
    - short-message와 background는 도착 시간과 흐름 크기 분포에서 독립적으로 추출하여 rack 간 흐름과 rack 내부 흐름의 비율이 클러스터에서 측정된 것과 같도록 endpoint를 선택한다.

![스크린샷 2024-01-30 오후 2 36 14](https://github.com/RakunKo/Network/assets/145656942/d95d3ab7-5bc4-4efd-9ad6-a4bb4e32987a)


- Figure 22에선 short message (100KB~1MB)에서 DCTCP의 완료시간이 더 빠른 것을 알 수 있다.
- background traffic은 어느 protocol에서도 timeout이 일어나지 않았다.
    - short message의 낮은 latency는 DCTCP의 queue buildup 개선 때문이다.
- Figure 23에선 Query Completion Time 측면에서 DCTCP가 TCP보다 빠른 것을 확인할 수 있다.
    - DCTCP에선 timeout을 겪지 않았다.
    
![스크린샷 2024-01-30 오후 2 36 31](https://github.com/RakunKo/Network/assets/145656942/d7340abd-cd6a-4946-8c03-c3d7c3aaa973)



**Scaled traffic**

![스크린샷 2024-01-30 오후 2 36 46](https://github.com/RakunKo/Network/assets/145656942/88514f0c-0496-40df-aeb1-bbcc9f9aee90)


- Figure 24에선 10배의 background, query traffic 상황에서 Completion Time을 나타낸다.
- DCTCP는 TCP, DeepBuf, RED보다 월등히 좋은 성능을 나타낸다.
    - DeepBuf는 short-message에서 80ms라는 큰 패널티를 받는다. (incast만 해결)
    - RED는 queue length의 잦은 변화로 인해 query traffic 성능이 저하된다.
        - 평균적인 queue length를 기준으로 mark하기 때문에 query incast로 인한 burst에 천천히 반응하게 된다.
