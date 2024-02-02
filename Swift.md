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