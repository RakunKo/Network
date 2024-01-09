# CUBIC: A New TCP-Friendly High-Speed TCP Variant

### 1. Abstract

CUBIC : a congestion control protocol for TCP (리눅스에서 사용중인 알고리즘)

- 기존 선형적으로(+1씩) 증가하는 함수에서 CUBIC function으로 수정
    
    ⇒ 고속, 장거리 네트워크에 대한 확장성의 향상 (기존 TCP는 점점 성능이 나빠졌음)
    
    ⇒ cwnd의 size를 RTT에 독립적으로 작용시켜 다른 RTT를 가진 흐름에서도 균등한 대역폭 할당을 달성
    
    ⇒ cwnd의 size를 saturation porin에서 멀면 aggressive, 가깝다면 slowly
    

### 2. Introduction

기존 TCP방식은 매우 빠르고 장거리 네트워크 paths에 취약하다(성능이 좋지 않다)

⇒ window size를 RTT당 한번 올리기 때문

Ex) bandwidth = 10Gbps, RTT = 100ms, packets = 1250bytes = 1250*8bits

- bandwidth = 10*10^9 bps, BDP = 10^9=10^9/1250*8 =100000packets
- linear하게 올린다면 50000(BDP의 절반)까지 가는데 50000RTT⇒5000sec(1.4hr)

BDP = Bandwidth * Delay 

⇒ 대폭폭을 충분히 사용한 상태에서 in flight(sender가 ACK을 받지 못함) 상태인 packet의 총 수 = cwnd size

BIC-TCP의 다음 version인 CUBIC

- CUBIC function으로 오목하고 볼록한 모양을 재배치하여 window adjustment algorithm을 간단화
- window의 증가는 두 개의 연속적인 congestion event 사이의 real time에만 의존
    - congestion epoch : the time when TCP undergoes fast recovery.
    - RTT에 독립적이다
    
    ⇒ bottleneck에서 경쟁하여 RTT에 독립적으로 거의 동일한 window size를 가질 수 있다.
    
    ⇒ RTT가 짧을 때는 window 증가 속도가 고정되어 TCP 표준보다 증가 속도가 느릴 수 있다.
    
- 주요 upgrade
    - floating point operation의 계산 비용을 줄이기 위해 Newton-Raphson method 사용
    - removal of window clamping(BIC-TCP에서 도입, target mid-point가 매우 크다면 선형적으로 증가하게 된다)

### 3. CUBIC Congestion Control

**3.1 BIC-TCP**
=

- unique window growth function
    - packet loss가 일어나면, BIC-TCP는 a multiplicative factor β에 의해 window를 줄인다.
    - 축소 직전 window size = Wmax
    - 축소 직후 window size = Wmin
    - 그후 Wmax, Wmin의 중간 지점으로 점프(additive increase)
        - mid-point가 fixed-constant Smax보다 크다면 Smax만큼 증가시킨다
        - Wmin은 그 Point에서 packet loss가 일어나지 않았다면 그 window로 설정한다.
        - 이 과정은 window growth가 Smax보다 작을 때 까지 계속한다
    - binary search 수행 (Wmax까지 진행)
- window size가 이전의 Wmax에 도달! ⇒ max probing
    - 전에 사용한 방법을 대칭적으로 적용한다
    - 새로운 Wmax를 찾지 못한다면 더 멀리 떨어져 있다고 가정
        
        binary search⇒addtive increase로 전환
        

**3.2 CUBIC window growth function**

BIC-TCP 단점 

- short RTT, low speed networks에서 growth function의 증가폭이 너무 급격하다
- several different phases(binary search, additive increase, max probing) ⇒ 구현의 어려움

CUBIC

- cubic function 사용
- window growth를 위해 the concave and convex profiles(오목, 볼록)을 둘 다 사용한다.

- loss event가 일어난 곳에서 Wmax를 등록하고 multiplicative decrease를 window decrease constant β에 의거하여 사용한다.
- fast recovery를 통해 congestion avoidance에 진입하게 되면 concave profile이 적용된 cubic function이 사용된다. Wmax가 될 때까지 지속한다.
- Wmax에 도달하면 cubic function은 convex profile로 전환된다.
    - concave, convex profile 형태는 protocol과 network의 안정성을 향상(높은 네트워크 효율성을 유지하면서)
    - Wmax 근처에선 느린 속도로 증가하기 때문에..가능하다.
    - convex growth function은 window increment가 크기 때문에 packet loss가 크게 발생할 수도 있다.
    

window growth function of CUBIC : $W(t) = C(t-K)^3 + Wmax$

C : CUBIC parameter

t : the elapsed time from the last window reduction (마지막 윈도우 축소에서 경과된 시간)

K is the time period that the above function takes to increase W to Wmax when there is no further loss event,  $K = \sqrt[3]{\frac{W_{\text{max}}B}{C}}$

- congestion avoidance 중 ACK을 받는다면, CUBIC은 Eq를 이용해 다음 RTT 기간동안의 window growth rate 를 계산한다.

$W(T+RTT)$ congestion window 후보 target으로 설정

- cwnd size에 따른 3가지 모드
    - TCP mode : loss event가 발생하고 t time이 지나 도달할 window size보다 cwnd가 작은 경우
    - concave region : cwnd가 Wmax보다 작은 경우
    - convex region : cwnd가 Wmax보다 큰 경우
    

**3.3 TCP-friendly regions**

- congestion avoidance 상태에서 ACK을 받는 다면, 프로토콜이 TCP region 인지 아닌지 확인해야 한다.
    - average window size of AIMD = $\frac{1}{RTT}\sqrt{\frac{\alpha}{2}\frac{2-\beta}{\beta}\frac{1}{p}}$
    - window size of TCP in terms of the elapsed time t
        
         = $Wtcp(t)=Wmax(1-\beta)+3*\frac{\beta}{2-\beta}*\frac{t}{RTT}$
        
        cwnd가 Wtcp(t)보다 작다면, 프로토콜은 TCP mode 인 것이다. cwnd를 Wtcp(t)로 설정
        
        ⇒ cubic_tcp_friendliness()
        

**3.4 Concave region**

- congestion avoidance 상태에서 ACK을 받았고, TCP mode가 아니고 cwnd가 Wmax보다 작다면 concave region이다.
- 이 상태에선 cwnd가 $\frac{W(t+RTT)-cwnd}{cwnd}$ 씩 증가한다.

**3.5 Convex region**

- 만약 window size of CUBIC이 Wmax보다 크다면 concave region을 지나 convex region
- cwnd가 Wmax보다 크다는 것은 더 많이 사용 가능한 대역폭을 의미한다. (비동기적이라 대역폭의 변동이 항상 존재한다)
- maximum probing phase (새로운 Wmax를 찾는 작업)
- 이 상태에선 cwnd가 $\frac{W(t+RTT)-cwnd}{cwnd}$ 씩 증가한다.

**3.6 Multiplicative decrease**

- packet loss가 일어나면 CUBIC은 cwnd를 factor B를 이용하여 감소시킨다.
- B를 0.5 이하로 설정하면 수렴 속도가 느리다.

**3.7 Fast Convergence**

- 기존 flow에 의해 대역폭의 release를 증가시키기 위해서 사용
- loss event가 발생하면, cwnd를 줄이기전에, protocol은 Wmax를 기억한다. (Wlast_max)
- 현재의 Wmax값이 Wlast_max값보다 작다면 대역폭의 변화로 saturation point가 감소하고 있음을 알 수 있다.
- Wmax를 더 줄임으로써 flow가 더 많은 대역폭을 release할 수 있게 한다.
- 감소된 Wmax는 더 빨리 안정 상태⇒ 새로운 flow의 cwnd를 증가시키는 시간이 길어진다
- 그 시간동안 새로운 flow가 window size를 잡을 수 있다.

### 4. CUBIC in Linux Kernel

 **4.1 Evolution of CUBIC in Linux**

- performance와 implementation efficiency improvement에 초점
    - cubic root calculation (Newton-Rhaphson method)
- algorithmic changes to enhance its scalability, fairness and convergence speed

**4.2 Pluggable Congestion Module**

- Stephen Hemminger가 제안한 new architecture “pluggable congestion module” in Linux 2.6.13
    - dynamically loadable and allows switching between different congestion control algorithm modules without recompilation
    - new congestion control algorithm ⇒ define cong_avoid and ssthresh (나머진 옵션)

### 5. Discussion

- the number of packets between two successive loss event is always $\frac{1}{p}$
    
    ⇒ concave window profile
    
    ⇒Average window size of CUBIC  $E\{W_{cubic}\} = \sqrt[4]{\frac{C(4-\beta)}{4\beta}(\frac{RTT}{p})^3}$
    
- To ensure fairness, C = 0.4, $\beta$ = 0.2
    - to be large enough to encompass most of the environments where Standard TCP performs well
    - while preserving the scalability of the window growth function
    
    ⇒ $E\{W_{cubic}\} = 1.17\sqrt[4]{(\frac{RTT}{p})^3}$ used to argue the fairness of CUBIC
    

**5.1 Fairness to standard TCP**

- Standard TCP performs well in the following two types of networks
    - networks with a small bandwidth-delay product(BDP)
    - networks with a short RTT, but not necessarily a small BDP
    
    ⇒ CUBIC은 두 경우에서 매우 비슷하게 행동하도록 설계되어 있다.
  
    
    - RTT = 10ms, p = $10^{-6}$
    - average window size = 1200 packets
    - packet size = 1500byte
        - average rate of 1.44Gbps
        - CUBIC achieves exactly the same rate as Standard TCP
    

**5.2 CUBIC in action**

- bottleneck capacity = 400Mbps, RTT = 240ms
    
    ⇒ two flows converges to a fair share within 200 second
    

- over short-RTT network path (8ms)
    
    ⇒ TCP-SACK use the full bandwidth of the path
    
    ⇒ CUBIC operates in the TCP-friendly mode
    
    ⇒ shares the bandwidth fair with the other TCP-SACK flow by maintaining the congestion window of CUBIC similar with that of TCP-SACK
    
- under the long-RTT(82ms)
    
    ⇒ Standard TCP has the under-utilization problem
    
    ⇒ CUBIC use cubic function to be scalable for this environment.
    

- bandwidth = 400Mbps
- RTT = 40ms
- 100% BDP of flow
    - not cause much disturbance
    
    ⇒ total network utilization = 95%
    
    ⇒ CUBIC takes 72%
    
    ⇒ TCP-SACK takes 23%
    

### 6. Experimental Evaluation

**6.1 Experimental setup** 

- each end points consists of a set of Dell Linux servers dedicated to high-speed TCP variant flows and background traffic.
- Background traffic is generated by us- ing a modification of a web-traffic generator, called Surge and Iperf
- the maximum bandwidth of the bottleneck router is set to 400Mbps. The bottleneck buffer size is set to 100% BDP

**6.2 Intra-Protocol Fairness**

- RTT vary between 16ms and 324ms

**6.3 Inter-RTT Fairness**


- one flow has a fixed RTT of 162ms and the other flow varies its RTT from 16ms to 162ms

**6.4 Impact on standard TCP traffic**

- Being fair to Standard TCP is critical to the safety of the protocol.


- how much these high-speed protocols steal the bandwidth from competing TCP-SACK flows.
    - (a) TCP-SACK은 slow window growth function 때문에 100%효율을 내지 못한다.
    - (a) 다른 방식들은 growth function의 차이로 fully utilize
    - (a) CUBIC은 다른 알고리즘보다 TCP-SACK에게 더 많은 공간을 제공한다.
    - (b)TCP-SACK은 fully utilize
    - (b) CUBIC은 다른 프로토콜에 비해 friendly(자원을 공정하게 나눈다)
    - (c)RTT가 적당하면 모두 reasonable friendliness

### 7. Conclusion

- fast and long distance networks에서 좋은 성능
- BIC-TCP window control을 간단화하고 TCP-friendliness와 RTT-fairness를 향상시켰다.
- window growth가 RRT와 독립적으로 수행
