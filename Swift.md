# Swift : Delay is Simple an d Effective for Congestion control in the Datacenter

### ABSTRACT

- Swift는 end-to-end delay를 AIMD control로 극한의 congestion 속에서도 속도를 유지시켜 준다.
- Swift는 긴 RPCs에 대해 높은 throughput 제공하면서 지속적인 낮은 low tail completion times를 짧은 RPCs에 제공한다.
- DCTCP보다 10배 낮은 loss rate를 달성하며, incast 문제로 발생하는 O(10k)를 처리할 수 있다.

### 1. INTRODUCTION

- With disaggregation, 차세대 저장소 잠재력을 두드리기 위해 낮은 latency messaging이 필요하다.
- Tight tail latency 또한 중요한데, 많은 host들 사이에서 partition-aggregate 통신 패턴을 사용하기 떄문이다.

- DCTCP는 큰 스케일의 상황에서 tail latency를 milliseconds로 경험하기 때문에 적합하지 않다.
    - Swift는 **TIMELY**에 기반을 둔 congestion control이다.
- Swift는 매우 낮은 latency messaging 성능과 주요 요구를 충족시켜준다.
    - 기술적 트랜드로 인해 datacenter가 빠르게 변화하는 동안 protocols을 배포하고 유지 관리가능
    - 공유된 fabric 안에서 새로운 traffic의 고립
    - host의 CPU와 NIC resources의 효과적인 사용
    - incast를 포함한 넓은 traffic 패턴을 다룰 수 있음

- Swift는 end-to-end 간 지연에 맞게 속도를 독립적으로 조정하는 host 기반을 기반으로 구축
- 정확하게 NIC timestamps로 delay를 측정하고 목표에 대해 신중하게 추론할 때 높은 수준의 성능 향상을 이룰 수 있다.
- 데이터 센터에선 경쟁하는 flow와 다른 path간에 delay 목표를 조절하는 것이 쉽다.

- Swift는 지연의 단순성과 효율성을 활용한다.
    - DCTCP, PFC, DCQCN 등 프로토콜은 **explicit feedback**을 사용하여 네트워크 대기열을 짧게 유지하고 RPC 완료시간을 낮게 유지한다.
    - IOPS-intensive workload나 큰 incast에는 도움이 되지 않는다.
    - 또한 다중 tenant 환경에 적합하지 않다.

- Swift는 bursty/incast 통신 패턴을 포함한 다양한 workloads에서 클러스터의 낮은 end-host, switch queuing delay를 유지할 수 있다.
    - 높은 활용성과  낮은 delay, 0에 가까운 loss
    - 낮은 RPC 완료 시간을 제공한다.

- delay를 congestion 신호로 사용함으로써 작동적 문제를 도와주는 간단함과 함께 좋은 성능을 보여주는 것을 효과적으로 증명
- Swift는 TIMELY로 간단화 되어 디자인 되었다.
- 큰 범위의 incast를 포함한 넓은 범위의 traffic 패턴을 지원해야한다.

### 2 MOTIVATION

**Storage workloads**

- storage는 데이터센터 네트워크의 지배적인 workload
- 기존에는 중요하지 않았지만, latency는 클러스터 기반  storage 시스템이 발전되고 나서 중요해졌다
    - disk traffic은 10ms의 access latency의 지배를 받았지만 현재는 10ms보다 더 적은 access latency를 요구한다.
    - storage access가 많은 장치에 영향을 미치고 단일 storage 작업의 전체 대기 시간은 가장 긴 네트워크 작업의 대기 시간에 따라 결정되므로 엄격한 네트워크 tail latency가 필요하다.
- Swift에선, throughput의 저하 없이 **tail latency**를 지속적으로 단축하는 것이 요구된다.

**Host Networking Stacks**

- RDMA, NVMe 같은 새로운 stack은 낮은 latency 저장소 작업을 위해 설계되었다.
- OS의 overhead를 피하기 위해, OS 우회 stack인 Snap이나 NIC offloaded에서 구현된다.
    - Swift또한 Snap에서 작동한다.
- 낮은 end-to-end 대기열을 유지하기 위해 host congestion을 해결하는 것이 중요
- Swift에서 delay는 fabric과 host congestion을 완화하기 위해 분해하는 것이 요구

**Datacenter Switches**

- DCTCP처럼 switch가 marking한 packet과 ECN을 사용하면 적절한 threshold 설정과 line speed와 buffer size 변화와 같은 구성을 큰 범위에서 유지하는 것은 어려움이 있다.
- line speed가 변하고 threshold을 변경할 때 매우 주의해야한다.
- 또한 여러 switch는 다른 방법의 buffer 관리를 한다.
- Swift는 switch의 신호와 통합하는 것보다 **낮은 대기 시간을 호스트에서 지연 목표를 발전**시키는 것이 더 쉽다는 것을 알았다.

RPC : Remote Procedure Call → 원격 프로시저의 호출(프로시저 또는 서비스를 호출하는 매커니즘)

### 3 SWIFT DESIGN & IMPLEMENTATION

- DCTCP와 같은 end-host-based scheme들은 큰 incast와 host의 congestion을 다룰 능력이 없다
    
    (1) 낮고 tightly-bound한 네트워크 latency, 0에 가까운 loss, 높은 throughput(큰 datacenter에서)
    
    (2) fabric과 NIC, host 모두 congestion을 관리하는 종단 congestion control을 제공(software stack)
    
    (3) CPU 효율성이 뛰어난 OS 우회 통신을 손상시키지 않도록 CPU 효율성이 높아야 함
    
- Latency, Loss, Throughput, Endpoint congestion, CPU efficiency, ***target delay***
- Swift는 end-to-end RTT를 NIC-NIC 및 endpoint delay 구성 요소로 분해하여 fabric과 host/NIC의 congestion에 별도로 대응한다.

<img width="453" alt="스크린샷 2024-02-02 오후 3 44 51" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/46a7dd68-8a96-49dd-942b-1496f39d2e36">

- Pony Express로 구현, 정확한 RTT측정을 위해 NIC 사용
- command, completion queue API를 제공
    - app은 commands를 “Ops”라고 알려진 Pony Express에 제출하고, completions을 받는다.
        - Ops는 네트워크 flows를 mapping하고, Swift는 각 flow의 전송 rate를 관리한다.
        - Algorithm 1

**3.1 Using Delay to Signal Congestion**

- TIMELY는 RTT가 최신 하드웨어로 정확하게 측정되어질 수 있다는 것에 주목
    - congestion의 존재뿐만 아니라 정도 또한 인코딩 된다.
- Swift는 end-to-end RTT를 분리된 fabric을 host 문제로 부터 분해한다.
    - NIC의 timestamps와 polling-based 전송(Pony Express)의 결합을 통해 delay 측정이 더 정확하게 한다.

**Component Delays of RTT**

<img width="535" alt="스크린샷 2024-02-02 오후 6 06 42" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/111a1673-05fe-4f83-878a-0e0ba8a8a908">

- *Local NIC Tx Delay* : packet이 wire로 방출되기 전에 NIC Tx queue에서 소요되는 시간
    - host가 NIC이 보낼 준비가 되면 NIC으로 packet을 보내기 때문에 이 delay는 무시할만 하다.
    - host delay는 더 상위계층에서 나타난다.
- *Forward Fabric Delay* : serialization, propagation, queuing delay의 총합 (NIC serialization delay도 포함한다)
    - 네트워크 congestion의 주요 indicator
- *Remote NIC Rx Delay* : packet이 remote stack에 의해 뽑히기 전까지 remote NIC queue에서 소요된 시간
    - 만약 host가 bottleneck이라면 이 delay는 엄청나게 증가한다.
    - memory preesure와 CPU scheduling 때문에 packet processing capacity가 떨어지면 delay가 급격하게 증가한다.
    - host의 congestion을 알 수 있는 주요 indicator
- *Remote Processing Delay* : stack이 데이터 packet을 처리하고 명시적인 ACK 지연을 포함하여 ACK packet을 생성하는 시간
- *Remote NIC Tx Delay* :  ACK packet이 NIC Tx queue에 머무른 시간
- *Reverse Fabric Delay* : reverse path에서 ACK packet에 걸리는 시간
    - Forward와 Reverse는 대칭적이지 않다.
    - reverse path 또한 실전에선 Swift에 의해 traffic이 control 된다.
- *Local NIC RX Delay* : ACK packet이 데이터 packet의 전달을 표시하기 위해 stack에 처리되기 전에 소요된 시간

**Measuring Delays**

<img width="369" alt="스크린샷 2024-02-02 오후 6 23 55" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/ede069c9-ac75-49bb-b275-e60990733d4c">

- Swifts는 다중 NIC과 host timestamps를 delay의 구성요소로 분리하기 위해 사용한다.
- $t_1$은 stack에 의해 기록된 data packet이 전송된 시각
- $t_2$는 packet이 원격 NIC에 도착한 시각
- $t_3$는 packet이 stack에 의해 처리되는 시간 → $t_3-t_2$은 NIC queue에 머문 시간
    - NIC과 host의 시간을 동기화해야 한다!
- $t_4$는 stack에 의해 나갈 준비가 된 ACK의 시간 → $t_4-t_3$ 은 processing time

- remote-queuing을 얻기 위해 NIC Rx delay와 processing time를 더하고 이 delay를 ACK packet의 header를 통해 sender에게 반영한다.
    - NIC Rx timestamp는 local로 추가되고 wire로 보내지지 않는다.
- $t_5, t_6$은 ACK에 대한 동일한 receive timestamps이다.
- end-to-end delay : $t_6-t_1$

**3.2 Simple Target Delay Window Control**

- Swift 알고리즘의 핵심은 target delay를 측정된 delay가 초과했는지 않했는지를 기반으로 한 AIMD controller

- AIMD controller는 ACK packet을 수신하면 트리거된다.
    - 최소, low-pass filtered delay와 반대되는 ***instantaneous delay***를 사용하여 congestion에 빠르게 대응
    - Swift는 ACKs을 delay하지 않는다
    - delay를 congestion의 신호로 사용한다

<img width="436" alt="스크린샷 2024-02-05 오후 2 32 20" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/1b62dd14-4689-44d2-9a36-5a83878be79f">

- 만약 delay < target_delay 이라면 cwnd의 크기를 늘려준다. (Additive Increase)
- 만약 delay가 target_delay보다 큰 상황이고, cwnd의 크기를 줄일 수 있다면 크기를 줄인다.
    - multiplicative decrease는 RTT당 한번으로 제한되기 때문에 같은 congestion에 대해 여러번 반응하지 않는다.

**3.3 Fabric vs. Endpoint Congestion**

- host의 congestion은 fabric congestion 못지 않게 중요하고 다른 congestion response가 요구
- RTT를 fabric delay(links, switches)와 end-host delay(NIC, host 네트워킹 stack)로 나눈다
- Swift는 remote-queuing(echoed in the ACK)과 Local NIC Rx Delay($t_6-t_5$)
- 그 후 RTT-endpoint-delay를 계산해 fabric-delay를 계산한다.

- $fcwnd$ : fabric congestion을 추적
- $ecwnd$ : endpoint(host) congestion을 추적

- 효과적인 congestion window는 min($fcwnd, ecwnd$)
- Snap의 문맥에서 host congestion의 delay를 측정하는 것이 window를 알리는 것보다 좋다
    - delay는 host의 모든 병목과 묶여있다
        - CPU, memory PCI bandwidth, caching effects, thread scheduling 등등
    - window advertise는 memory 할당만을 capture
        - flow-control을 위해 사용된다.
        - host가 병목일때 flow 끼리의 fairness에 목적을 두지 않는다.

이러한 이유로 Swift에선 congestion의 신호로 delay를 사용하게 된다.

**3.4 Large-Scale Incast**

- 대규모 incast를 해결하기 위해 Swift는 congestion window를 최소 *0.001 packet* 아래로 떨굴 수 있게 한다.

<img width="308" alt="스크린샷 2024-02-05 오후 2 57 50" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/3b4f946f-ebee-490b-841b-8e314e2f38ac">

- 소수형태의 congestion window를 구현하기 위해  sender가 cwnd에 사용하는 packet을 네트워크로 전송하는데 사용하는 inter-packet delay로 번역한다.

<img width="195" alt="스크린샷 2024-02-05 오후 2 59 51" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/52698817-3f3b-4dff-aab3-8e4ee241b7e4">

- 0.5의 cwnd는 2RTT후에 packet을 전송한다.

- 일정량 이상의 높은 flow를 pacing하는 것은 ACK-clocked window보다 더 좋은 성능을 내지 못한다.
- CPU 효율성 측면에서도 좋지 않다.
- TIMELY는 CPU에서 효과적인 사용이 가능한 64KB의 chunk를 사용하여 문제를 해결한다.

Swift에선 일반적으로 ACK-clocked congestion window를 사용하여 좋은 성능을 내고, cwnd가 1 이하로 떨어지면 pacing으로 전환하여 대규모 incast문제를 해결한다.

**3.5 Scaling the Fabric Target Delay**

<img width="444" alt="스크린샷 2024-02-05 오후 3 14 57" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/2fcfcb4c-7af4-42a8-b3a9-cc8742cfe016">

- target delay는 고정된 fabric delay와 동적 fabric delay가 캡슐화되어 있다.

**Topology-based Scaling**

- 데이터 센터 내에서 propagation, serialization delay를 포괄할 만한 큰 단일 target delay를 사용하면 throughput에서 좋은 성능을 내지만, 짧은 경로를 사용하는 traffic을 위해 더 큰 대기열을 구축하는 비용 발생
- 데이터 센터 내에선 topology가 알려져 있고 거리가 제한적이다.
    - 고정된 base delay와 고정된 per-hop delay를 더한 것을 사용하여 target delay으로 변환
    - 알려진 TTL에서 IP TTL을 뺴서 forward hop count를 얻고 이를 header에 추가

Flow-based Scaling

<img width="445" alt="스크린샷 2024-02-05 오후 3 28 29" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/31e8de37-3701-40af-bf7f-456480384c03">

- 다른 flow들이 한번에 queue로 들어온다면…? → Figure 4
- Swift에선 흐 간의 임의 충돌 가능성으로 인해 자연스럽게 발생할 수 있는 대기열의 양을 모델링하고 과잉 반응을 방지하기 위해 대기열 headroom에 대기열의 양을 요소로 넣는다.

- sender는 병목의 flow 개수를 알지 못하기 때문에, target을 조절하기 위한 다른 수단이 필요
- target을 $1/\sqrt{cwnd}$ 로 조절한다.
    - cwnd가 작아질 수록 target delay가 커진다.
- flow가 적을때 대기열을 낮출 뿐만 아니라 공정성을 향상시킨다.
    - cwnd가 1보다 작을때 특히 유용하다.

**Overall Scaling**

<img width="333" alt="스크린샷 2024-02-05 오후 3 42 07" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/5af7d2d5-d14a-4946-b91d-98b6c43725a1">

- target delay : $base\_target +\#hops*ħ +max(0, min(\frac{\alpha}{\sqrt{fcwnd}}+\beta,fs\_range))$
- $\alpha = \frac{fs\_range}{\frac{1}{\sqrt{fs\_min\_cwnd}}-\frac{1}{\sqrt{fs\_max\_cwnd}}}$, $\beta = -\frac{\alpha}{\sqrt{fs\_max\_cwnd}}$

<img width="437" alt="스크린샷 2024-02-05 오후 3 51 02" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/b08c0979-2555-4981-b030-1313db37761e">

- base target : 적은 개수의 flow로 하나의 hop 네트워크에서 100% utilization를 제공하기 위해 필요한 최소 target delay
- $fs\_range$ 는 점진적으로 $[fs\_min\_cwnd, fs\_max\_cwnd]$ 사이에서 감소하는 베이스 위의 추가 target
- $ħ$ 
				
			
		는 per-hop scaling factor
    - 이 2개의 파라미터가 scaling 곡선 및 이 곡선이 적용되는 cwnd범위의 기울기를 결정한다

- 더 카파른 곡선은 flow가 느려지면 속도가 증가할 가능성이 높아지므로 fairness와 convergence를 향상시킨다.
    - 오히려 네트워크의 대기열을 증가시킬 수도 있다. →질문
- cwnd가 증가하면 queuing delay가 증가하고, 이는 target delay의 감소로 이어진다.

결과적으로 고정된 target delay보다 더 빠르게 queuing 및 target delay를 동일하게 하여 더 빠른 수렴과 더 나은 공정성을 제공한다.

**3.6 Loss Recovery and ACKs**

**Loss Recovery**

- Swift는 packet loss를 낮게 유지하기 때문에 good gail latency를 loss recovery에 최소한의 노력을 기울여야한다.
- selective acknowledgments(SACK) : fast recovery
    - bitmap의 구멍으로 loss를 감지하면, 재전송되며, cwnd가 감소한다.
- retransmission timer : receiver부터 온 ack의 부재에도 data 전달을 보장한다
    - retransmission timeout(RTO)
    - 잠재적으로 심각한 congestion에 신속하게 대응하기 위해, cwnd는 RTO상에서 최대 곱셈 factor만큼 감소된다.

**ACKs**

- Swift는 congestion에 신속하게 대응하기 위해 ACKs를 지연하지 않는다.
- 양방향 traffic에 경우 ACK이 delay되지 않게 주의해야한다.
- 대신 paced flow를 위윟해 aat와 ACk packet을 분해한다.
    - 다가오는 data packet은 remote end를 unblock하기 위해 바로 보내질 순수 ACK을 생성한다
    - 그러나 reverse data packet은 모든 pacing-delay를 고려한다.

**3.7 Coexistence via QoS**

- Swift traffic은 다양한 형태의 congestion control이 switch buffers에 대한 경쟁 없이 공존할 수 있어야한다.
- 그렇지 않으면 latency가 커지고, throughput이 감소한다.
    
    →Qos 사용
    
    - 스위치에는 사용량에 따라 port간 buffer space를 공유할 수 있는 port당 최대 10개의 QoS 대기열이 있다.
    - QoS의 queue를 반전시키고, weighted-fair-queueing을 통해 link 용량을 공유하게 한다.
- 우선순위가 높은 traffic에 대해 더 큰 스ㅔ줄러 가중치를 사용함으로써 서로 다른 traffic class를 가진 tenant를 다룰 수 있다.

### 4 TAKEAWAYS FROM PRODUCTION

**4.1 Measurement Methodology**

- Switch statistics : link 활용 정도와 loss rate를 알려준다
    - switch로 부터 모아진 output-packets과 output-discards counters를 per-port를 사용하여 30초 간격으로 계산
- Host round-trip-times : NIC 하드웨어 timestamp가 있는 NIC-to-NIC은 fabric RTT data를 준다
    - host와 fabric 부분이 포함된 end-to-end RTT는 Swift에서 측정된다.
- Application metrics : Swift가 application에 주는 영향을 강조한다.

- Swift와 비교하여 DCTCP는 최근 ECN mark rate를 기반으로 하여 곱셈 decrease로 조절
- Swift와 DCTCP는 좋은 비교군

**4.2 Performance At Scale**

**Takeaway: Swift achieves low loss even at line-rate**

- Swift로 이동하면서 가장 큰 이점은 packet loss의 감소 → loss packets/sent packets

<img width="443" alt="스크린샷 2024-02-06 오후 2 14 43" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/67722ec3-5bf8-4ba5-b78c-090a73de3333">

- Swift는 ToR to host에서 port utilization이 90%를 넘겨도 낮은 loss rate를 유지하는 것을 확인
- 반면에 DCTCP 기반 GCN은 Swift와 비교하면 너무 큰 loss rate

<img width="438" alt="스크린샷 2024-02-06 오후 2 17 32" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/61eb4fd6-8620-40c0-8d94-a424074c3edd">

- Swift는 Fabric link 측면에서도 높은 port utilization을 유지하며 낮은 Loss rate를 보여준다.

<img width="429" alt="스크린샷 2024-02-06 오후 2 19 50" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/c7031e2a-a243-4bc3-b587-4b0ee27b296e">

- Swift는 대기열 utilization을 높게 유지하면서도 낮은 loss rate를 보여준다.

<img width="444" alt="스크린샷 2024-02-06 오후 2 21 30" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/27c92f18-d13c-46c3-8087-e958b247839f">

- Swift의 낮은 Loss rates는 target delay로 감지된 congestion에 대한 즉각적인 반응의 결과, 대규모 incast로 확장도 가능하다.
- End-to-end 재전송률도 매우 낮기 떄문에 loss recovery에 크게 투자하지 않는 목표에 달성

<img width="433" alt="스크린샷 2024-02-06 오후 2 24 29" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/fa3a9541-bc47-49d6-9894-dab58119ccdc">

- Swift의 성능 향상은 link speed와 연관이 깊다.
- Swift는 GCN보다 port speed가 달라지더라도 GCN보다 큰 loss rate를 나타내지 않음

**Takeaway: Swift achieves low latency near the target**

<img width="426" alt="스크린샷 2024-02-06 오후 2 27 31" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/a53978ad-ac7d-47ef-9d06-5faf4671f60b">

- Swift가 GCN에 비해 높은 RTT를 가질 확률이 현저히 적고, 가장 높은 RTT 또한 적다.

<img width="438" alt="스크린샷 2024-02-06 오후 2 29 08" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/60e8db5e-8e63-4b95-bc38-c7d3d844622b">

- Throughput과 비교하여 RTT를 측정한 결과 Swift가 GCN보다 낮은 RTT를 보여준다
- Swift는 평균 RTT가 base target delay와 가깝다는 것을 알 수 있다.
- 일부 클러스터에선 평균 RTT가 base target보다 위에 있을 수 있는데, topology와 flow 조절로 대상의 상한 아래에 존재하게 된다.

**Thus, Swift achieves near line-rate throughput while maintaining its promise of low loss and low latency.**

- Swift는 flow의 개수가 네트워크 bandwidth-delay를 초과하는 극한의 congestion 상황에서도 빠르게 cwnd를 줄인다.
- Swift Fabric congestion과 더불어 end host의 congestion을 완화한다
- Swift는 GCN 임계값으로 달성하기 어려운 목표인 중간 link rate 및 buffer size에 관계없이 예측 가능하게 end-to-end delay를 제한

**4.3 Use of Shared Infrastructure**

<img width="455" alt="스크린샷 2024-02-06 오후 2 41 22" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/3ac85850-a2d2-425a-890c-38f5fc52bf83">

- 엄격한 스케줄링 우선순위의 이점이 있는 GCN과 낮은 weight으로 작동하는 Swift 비교
- Swift가 ToR-to-Hos link utilization이 높아져도 GCN보다 낮은 loss rate를 보여준다

**4.4 Fabric and Host Congestion**

<img width="451" alt="스크린샷 2024-02-06 오후 2 45 49" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/7f584909-7dde-4207-837c-8bbb902c621f">

- End-to-End delay는 $t_6-t_1$로 계산된다
- host delay는 thorughput-cluster의 경우 작고 빠듯하지만 IOPS-cluster의 fabric 지연만큼 기여할 수 있다.
- host와 fabric을 RTT에서 분리하는 것은 디버깅에서도 귀중하다

**4.5 Application Performance**

- Swift는 IOPS-intensive, Throughput-intensive application, 짧은 Ops를 통해 지연 시간에 민감한 workload를 지원하고 이를 함께 잘 실행할 수 있도록 하는 것이 중요하다.

**In-memory BigQuery Shuffle**

- Swift는 Snap 위에 구축된 BigQuery Shuffle을 위한 분리된 in-memory file system을 지원

<img width="224" alt="스크린샷 2024-02-06 오후 2 54 29" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/58d9b18f-7f51-45bc-9471-e29dc464f179">

- Op completion time이 Swift의 target delay와 가까운 것을 확인할 수 있다.
- host와 fabric contestion의 분리된 처리는 application이 모든 클러스터 안의 접근 latency SLOs를 충족시킬 수 있게 한다.

**Storage**

- 일부 Storage traffic은 Swift를 통해 제공된다.

<img width="224" alt="스크린샷 2024-02-06 오후 2 58 46" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/91ab6518-c845-44bb-bd07-b800d1ca0203">

- 99.9번째 애플리케이션 지연 시간을 4배 낮춘다.
- Swift는 100% 성공률로 ~7% 더 높은 IOPS를 달성하는 반면, GCN의 손실은 마감 시간 초과로 인해 OPS의 1.7%를 실패

**4.6 Production Experience**

- Swift의 손실이 거의 없는 line-rate로 운영되는 능력은 다른 것들이 이러한 규모의 성능에 익숙하지 않기 때문에 혼란을 야기한다.
- Swift의 매우 낮은 지연 시간은 불필요하게 처리량을 감소시켜 application 성능을 저하시킬 수도 있다는 의견도 있다.
    - 다른 출시된 버전으로 target delay 증가, pacing 및 cwn<1 mode 비활성화, window floor 와 timeout 단축을 포함하여 대기열 증가에 따른 비용으로 처리량을 우선시
    - 처리량은 증가하지 않았고, 추가적으로 RTT와 loss가 증가했다.
    - Swift는 delay와 loss를 낮게 유지하기 위해 traffic을 throttling하지 않는다는 것을 확인
- DCTCP에서 ECN을 tuning하는 것보다 Swift의 target delay를 변경하는 것이 더 쉽다.

### 5 EXPERIMENTAL RESULTS

**5.1 Effect of Target Delay**

<img width="434" alt="스크린샷 2024-02-06 오후 4 00 46" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/fab95cff-7649-4d77-8d7c-9608a47cc353">

- 100-flow incast 상황에서 평균 RTT는 target delay와 매우 가까운 것을 확인 가능
- 초반 target delay가 매우 낮을때는 throughput을 throttling 한다
- 이후 처리 속도를 줄이면서도 delay를 줄일 수 있도록 한다.

**5.2 Throughput/Latency Curves**

<img width="440" alt="스크린샷 2024-02-06 오후 4 06 06" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/9b74a65f-86cd-46c9-9ab3-955305748925">

- line reaterk 80을 넘을때까지 RRT는 거의 증가하지 않고 처리량만 증가하게 된다.

**5.3 Large-scale Incast**

- Swift는 cwnd <1 이면 대큐모 incast를 다루는 pacing을 지원한다.

<img width="454" alt="스크린샷 2024-02-06 오후 4 10 17" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/1969c107-f5b4-4b28-9298-f9a30e5cb3f1">

- Swift는 대규모 incast에서 line-rate throughput을 낮은 latenct와 거의 0에 가까운 loss를 보여준다.
    - cwnd<1이 없는 Swift는 낮은 성능을 보여준다.

**5.4 Endpoint Congestion**

<img width="441" alt="스크린샷 2024-02-06 오후 4 12 37" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/024482f1-bda0-41e0-a724-a9e2e5eff51e">

- endpoint-congestion 상황에선 ewnd의 size를 줄이고 fabric-congestion 상황에선 fcwnd의 size를 줄이게 된다

<img width="443" alt="스크린샷 2024-02-06 오후 4 15 29" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/c2b886f3-da59-42c7-99c3-c2ab6d4f9077">

- end-to-end RTT를 fabric과 endpoint로 나누는 것은 Swift가 network와 host의 congestion에 대해 다른 반응을 만든다.
- end-to-end delay에 대해 단일 target만 가지고 있는 Swift는 처리량이 낮거나 RTT가 높은 문제를 가지지만, fabric과 endpoint로 나누는 Swift는 처리량과 RTT 측면에서 우수하다

**5.5 Flow Fairness**

- flow와 topology의 조절은 Swift가 flow에게 대역폭의 공평한 할당을 달성하는데 도움을 준다.

**Flows with same path lengths**

<img width="227" alt="스크린샷 2024-02-06 오후 4 20 26" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/1e63a90a-afbc-4004-a65f-65a85a2b1ec9">

- flow가 추가되도 공평하고 빠듯하게 flow를 할당한다.

<img width="435" alt="스크린샷 2024-02-06 오후 4 21 48" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/3bd194e5-a4e6-4179-80b6-3d743a434523">

- flow-based scaling을 한 Swift는 안한 것 보다 공평한 throughput이 나왔고, CDF 측면으로 봐도 FBS를 사용하는 것이 더 공평한 분포를 보여준다.

**Flows with different path lengths (RTT Fairness)**

<img width="242" alt="스크린샷 2024-02-06 오후 4 23 51" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/e503e381-b6e7-49a7-bda9-611027360049">

- Swift scales the target delay for a flow based on network path length.
- 짧은 path에 latency를 줄여주며, 공평성을 제공한다.