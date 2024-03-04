MP-DASH: Adaptive Video Streaming Over
Preference-Aware Multipat

### **ABSTRACT**

- leveraging multipath (WiFi, cellular)는 QoE(Quality of experience)를 극적으로 향상시켜줌
- MPTCP는 하나의 path를 다른 것보다 우선시 하게 하는 자원이 부족하다.
    - 원하지 않는 네트워크의 과도한 사용의 원인이 될 수 있다.
- MP-DASH : a multipath framework for video streaming
    - chunk 전송을 예약하여 user preferences를 만족시킨다.
    - 작은 수정으로도 기존의 video rate adaptation algorithm과 함께 작동할 수 있다.
    - QoE의 저하가 거의 없이 cellular usage 99%, radio energy consumption 85% 감소

### **1. INTRODUCTION**

- Mobile user’s QoE for video streaming은 까다로운 네트워크 환경(불안정한 WiFi, mobility)에서 만족스럽지 못하다.
    - open WiFi는 1080p video를 재생할만한 안정적인 throughput을 제공하지 못했다.
    - LTE는 충분한 throughput이지만 사용자들은 이용량을 제한
    
- Multipath TCP (MPTCP) : mulitpath solution allowing app to transparently use multiple path
    - video streaming에 대한 QoE를 극적으로 향상시킨다.
        - 추가적인 네트워크 용량 제공
        - 강력한 소통 (robust communications)
    - 그러나 MPTCP가 원하는 네트워크 인터페이스 설정을 지원하는 것은 어렵다
        - 집에선 LTE보다 WiFi를 선호
        - 이는 원하지 않는 네트워크 사용이 발생할 것이다(substantial over-utilization of the metered cellular link)
        - 이는 multipath로 인한 사용자의 이득을 줄인다.
    
- MP-DASH : a multipath framework for video streaming
    - MPTCP가 사용자가 지정한 인터페이스 설정에 따라 adaptive video streaming을 지원하게 향상
    - DASH (Dynamic Adaptive Streaming over HTTP)
        - 코덱에 구애받지 않으며 모든 코덱을 사용할 수 있다.
        - the extreme success of HTTP that is easy to deploy, middlebox-friendly, supported by infra(CDN)
    - MP-DASH scheduler
        - 사용자의 인터페이스 preference, video chunks’ size, delivery deadline을 input으로 한다
        - delay tolerant nature를 이용해 multipath를 통한 최선의 video chunks fetching 전략을 결정한다
        - needs to be online, lightweight, robust, generalizable to different user preference
    - the video adapter
        - 기존의 DASH Algorithm을 multipath-friendly하게 만든 lightweight한 것
        - 기존 DAH algorithm과 MP-DASH scheduler 사이에 위치하며, 둘의 상호작용을 다룬다.
    - scheduler와 adapter는 multipath를 통한 video delivery의 효율성 향상을 도모한다.

### 2**. MOTIVATIONS**

**2.1 Multipath TCP (MPTCP)**

- mulitpath solution allowing app to transparently use multiple path (Linux kernel에 구현)
    - wireless interfaces를 어떻게 활용할지 preference를 구체화하는 능력이 없다.
- MPTCP의 default scheduler는 low latency path를 선호
    - smallest RTT estimation을 선택한다
- decoupled congestion control → each path runs congestion control independently
    - to offer fairness at shared bottlenecks
    

**2.2 Measurement Study in the Wild**

1. WiFi only is never able to support the highest bitrate of a 1080p (64%)
2. WiFi는 가끔 좋은 질의 재생이 가능하지만, 항상은 아니다 (15%)
3. WiFi는 거의 항상 가장 높은 bitrate를 안정적으로 지원할 수 있다 (21%)
    
    → WiFi 스스로는 최상의 비디오 스트리밍 경험을 제공할 수 없다
    
- 보통 bandwidth throttling, limited backhaul connectivity에 의해 WiFi성능이 약해진다.
- MPTCP는 모든 위치에서 가장 높은 playback bitrate를 유지할 수 있다.
    - mobile video streaming의 QoE향상에 유용할 것이다
    - 하지만 3번 같이 불필요한 cellular data 사용의 원인이 될 수 있다. (WiFi만으로 높은 rate를 유지할 수 있을 경우)

**2.3 Controlled Experiments**

<img width="443" alt="스크린샷 2024-01-12 오후 1 18 10" src="https://github.com/RakunKo/Network/assets/145656942/ac15929f-60de-41e2-8450-c428d55bde5b">

- MPTCP에선 user preferences를 알지 못하기 때문에 LTE용량이 거의 다 활용된다.
- WiFi는 3.8Mbps이므로 4Mbps 비디오를 시청하기 위해 0.2Mbps를 cellular로 할당하고 싶어한다
- delay-tolerant
    - player의 buffer가 가득차면, chunks 다운로드를 마친 후 네트워크는 다음 chunk를 가져오기 전에 idle 상태가 된다 (플레이어가 버퍼를 소비할 때까지 기다림)
    - playback deadline이 충족되는 한 chunks가 늦게 도착하더라도 QoE는 영향을 받지 않는다.

### **3. THE MP-DASH FRAMEWORK**
****

**3.1 MP-DASH System Design**

- to accommodate the interface preferences specified by users
- schedule the video delivery based on the preferences
- video player와 MP-DASH 간에 비디오 관련 정보를 교환할 수 있는 메커니즘이 필요
- 전체적인 시스템은 scalable하고 lightweight해야함

→이를 만족시키기 위한 MP-DASH scheduler와 MP-DASH video adapter

<img width="680" alt="스크린샷 2024-01-12 오후 1 32 28" src="https://github.com/RakunKo/Network/assets/145656942/b0cf51a5-8f1f-4311-99c2-699a43971bb1">

- MP-DASH scheduler(MP-TCP을 강화한다)
    - aware of the preference of multiple path
    - aware of the deadline of each video chunk
        
        ⇒ user-specified preference를 만족시키면서 deadline을 충족시키는 목표를 가지고 전략적으로 network path를 관리한다.
        
- MP-DASH video adapter
    - a lightweight add-on that makes existing DASH algorithms multipath-friendly.
    - 주요 임무는 MP-DASH scheduler에게 통합된 인터페이스를 통해 chunks’ size와 deadline을 알리는 것
    - DASH algorithm과 추가적인 상호작용이 할 수 있다.

**3.2 Specific Design Decisions**

**The Interface of the MP-DASH Scheduler.**

- MP_DASH_ENABLE : data size S 와 deadline D를 kernel로 보내기 위한 socket option
    - MP_DASH_ENABLE을 수신하면 다음 S byte의 데이터에 대해 MP-DASH가 활성화
        - S bytes가 성공적으로 전송되면 비활성화
        - deadline을 지나면 비활성화
        - 또 다른 socket option인 MP_DASH_DISABLE으로 명시적인 비활성화
- 인터페이스의 두번째 부분을 사용하면 DASH adapter가 속도 적응에 필요한 정보를 얻을 수 있다.
    - MPTCP는 player를 인식하지 못하여 모든 경로의 네트워크 상태를 볼 수 없기 때문에 이 인터페이스가 필요하다
    

**Function Split between Client and Server.**

- MP-DASH scehduler는 video streaming server에서 작동해야 하지만, server는 DASH logic을 실행하지 않는다.
- MP-DASH adapter는 client 측에 있어야 한다.
    - decision function on the client
        - video player의 정보를 기반으로 path를 관리하는 방법을 결정하고, enforcement function에게 a reserved bit in the MPTCP option을 이용하여 알린다.
    - enforcement function on the server
- 이러한 디자인은 server측을 비저장 상태(stateless)로 만들어 MP-DASH가 확장 가능하게 만들어 준다.

**Interface for the User.** 

- MP-DASH는 사용자에게 interface preference 같은 multipath policies를 지정할 수 있게 한다.
- preferring WiFi over cellular
- preferring cellular over WiFi
- 선호하는 인터페이스를 MPTCP의 기본 인터페이스로 설정하여 정책을 시행한다.

### 4**. DEADLINE-AWARE MP-DASH SCHEDULER**

- Deadline-aware MP-DASH scheduler
    - the interface preference and the delay-tolerant nature of video traffic into consideration
    - playback deadline of a chunk를 충족하면서도 전체 cost를 최소화하는 것이 목표

**General Formulation**

- the number of network paths N ≥ 1, chunk size S, download deadline D
- duration of each time slot is d
- the available bandwidth of interface i at time slot j (1≤i≤N, 0≤j<D) ⇒ b(i,j)
- the unit-data cost of using interface i at time j⇒ c(i,j)
- a binary decision variable x(i,j)=1 , interface i is used at time j, otherwise x(i,j)=0

<img width="411" alt="스크린샷 2024-01-12 오후 4 12 50" src="https://github.com/RakunKo/Network/assets/145656942/52f3d363-7996-4bac-a90c-711217983d41">

<img width="308" alt="스크린샷 2024-01-12 오후 4 13 21" src="https://github.com/RakunKo/Network/assets/145656942/901de048-bfe0-4512-a228-24e36fe04a0e">

- 이 문제는 0/1-knapsack problem과 같다. (weight = b(i,j)d, value = c(i,j)b(i,j)d)
- dynamic programing으로 해결 가능하며 시간 복잡도는 O(N*D*S)

**Practical Online Algorithm**

- 실전에선, MP-DASH는 throughput estimations이 계속해서 업데이트 되는 온라인 환경에서 작동한다
- 모바일 기기로 생각하면 N=2(WiFi, cellular)로 간단해진다.
- 보통 WiFi를 cellular보다 더 선호하기 때문에 c(wifi,j) < c(cell,j)

- MP-DASH는 기존 MPTCP scheduler를 활용하여 multipath에 packet을 배포하고, cellular 하위 흐름을 제어하는 방법에 대한 지능을 추가한다.

<img width="453" alt="스크린샷 2024-01-12 오후 4 18 49" src="https://github.com/RakunKo/Network/assets/145656942/99b44b9f-2cb8-4067-99cc-78477f6f1ff6">

- input으로는 video chunk size S와 the length of download time window D
    1.  초기에는 WiFi subflow를 최대 용량으로 구동하고, cellular subflow를 끈다.
    2. data transfer 진행 상황을 모니터링하고, cellular subflow를 필요하다면 키게 된다.
    3. while loop는 MPTCP를 통해 chunk’s data를 보내는 것이다. (line 11)
- Rwifi in line 15 is the current estimation of WiFi throughput.
    1. 각각의 packet을 보낸 후에 MP-DASH는 WiFi 혼자 남은 데이터를 보낼 수 있는지 확인한다.
        1. 가능하다면 cellular subflow를 disables하게
    2. (19~21) WiFi 혼자 deadline 전에 남은 데이터를 완전히 보낼 수 없는 상황인지 확인하고 cellular subflow를 다 시 활성화 하는지 확인해야한다.
- $\alpha$ 값이 작을 수록 deadline를 놓칠 가능성이 낮아진다.
    - 하지만 작을 수록 cellular links를 통해 더 많은 데이터를 보내게 된다.
- cellular로 보내기로 결정했으면, 최대한 빠르게 최대한의 bandwidth로 활용해야한다.

**Optimality**

- 우리가 항상 정확하게 deadline까지의 bandwidth를 추정할 수 있다면 Algo1은 최적의 결과가 도출된다
- 16~19 line에서 늦게 cellular를 키는 것을 방지한다 → 최적의 결과

- online MP-DASH scheduling algorithm은 multiple interfaces with varying costs의 일반화
- 코스트 별로 정렬하여 scheduling을 할 수도 있다.

schedular : multipath를 이용한 DASH video의 셀룰러 사용량과 에너지 효율성 향상시킨다.

- DASH rate adapatation algorithms과 MP-DASH scheduler의 상호작용은 복잡하다.
    - MP-DASH의 스케줄링 결정은 DASH 속도 선택에 영향을 미치고, 이는 다시 chunk size 및 deadline 설정을 통해 MP-DASH 스케줄링에 영향
- 너무 많은 DASH rate control algorithm이 존재한다.
    - 카테고리 별로 DASH 알고리즘을 수정해야 한다.
    

### **5. MP-DASH VIDEO ADAPTER**

**5.1 The Basic Approach**

DASH의 rate adaptation algorithms

- 비디오는 똑같은 play time의 chunk로 쪼개진다.(1에서 15초 사이), 각 chunk는 개별 bitrate level로 인코딩된다.
    - player는 chunk의 경계에서 서로 다른 bitrate stream 간 전환이 가능하다.
    - QoE를 최적화 할 수 있는 chunks’ bitrate를 선택하는 알고리즘의 요구

- throughput-based
    - 향후 network capacity를 나타내는 estimated throughput을 기반으로 chunk의 encoding rate를 조절한다.
- buffer-based(BBA)
    - video player의 buffer 점유율 정도에 기반하여 bitrate를 선택한다.

- video player는 MP-DASH에게 chunk size와 deadline을 제공해야 한다.
    - HTTP responses의 헤더 영역의 Content-Length에 chunk size가 있다.
    - DASH의 manifest file에는 chunk size 정보가 없다.
- how to set the deadline D
    - 예상하지 못한 네트워크 품질 변화에 대한 완충 역할을 하는 playback buffer를 가지는 목적
    - 대신, buffer의 occupancy를 감소하지 않고 유지하는 것으로 deadline을 설정
    - duration-based
        - D is a video chunk’s playout duration
        - aims to maintain the buffer level in the short term
    - rate-based
        - D is the chunk size divided by the nominal(average) video encoding bitrate
        - to maintain the buffer level in the long run
            
            →since it leverages the average bitrate of the entire video
            
- extend the deadline when the buffer level is high (셀룰러 사용을 줄이기 위한 기회)
    - 임계값이 Φ이고 현재 buffer level이 b>Φ라면 deadline을 b-Φ만큼 확장한다.
- low buffer level(임계값Φ보다 낮은 경우) 에선 MP-DASH schedular를 비활성화 한다.

**5.2 Handling Different Categories of DASH Rate Adaptation Algorithms**

5.2.1 Throughput-based DASH Rate Adaptation

- 미래의 throughput을 추정하기 위해 과거 chunks’ throughputs을 사용한다.
- 알고리즘 별 로직을 이용하여 다음 chunk의 품질 수준을 mapping → MP-DASH에겐 안보임
    - 그러나 player는 multipath에 대해 알지 못하고 MPTCP의 처리량을 과소평가 할지도 모른다 (cellular를 비활성화 할시)
    - 이를 해결하기 위해 app에게 인터페이스를 통해 MPTCP throughput estimation을 전달해야한다.
    - 따라서 player는 전체 이용가능한 네트워크 리소스에 대한 일관적인 시야를 얻게 된다.
- deadline extension을 위해 Φ를 전체 buffer 용량의 80%로 설정했다
- Ω 의 최솟값을 전체 buffer 용량의 40%로 설정했다. (T time window에서 T’시간동안 chunk 보냄)
    - Ω = T −T′ (Ω가 음수인 것은 0으로 생각한다.)

*5.2.2 Buffer-Based DASH Rate Adaptation*

- 작은 것부터 시작해서 buffer occupancy증가하면 다음으로 큰 것으로 encoding bitrate의 교체…
    - 다시 buffer occupancy가 감소하여 원래 작은 것으로 복귀
    - video bitrate가 진동하는 것을 알 수 있음

<img width="447" alt="스크린샷 2024-01-12 오후 5 46 44" src="https://github.com/RakunKo/Network/assets/145656942/68799b32-cd0f-4539-b72b-aa632cc4633f">

- BBA의 진동은 QoE를 감소시키고 MP-DASH의 자원 절약 용량을 제한한다.
    
    → BBA를 MPTCP의 실제 throughput보다 크기 않은 bitrate를 선택하도록 바꾼다.
    
    → BBA-C(cellular-friendly version of BBA)
    
    - player가 가장 높은 bitrate를 유지할 수 있을 때 MP-DASH scheduler가 사용 가능하다.

*5.2.3 Hybrid DASH Rate Adaptation*

MPC : predictive control algorithm that leverage both throughput estimation and buffer occupancy.

- ***pre-generated table*** to select the optimal bitrate based on the buffer level, previous bitrate, throughput estimation.
- MPC는 chunk deadline을 chunk size를 최소 throughput으로 나눠서 설정한다.

### **6. IMPLEMENTATION**
****

- MP-DASH scheduler를 Linux의 kernel에 구현 → 복잡한 app 선별 scheduler에 적합
    - MPTCP를 이용하여 통신 overhead를 줄일 수 있다.
- client는 예약된 MPTCP DSS(Data Sequence Signal) option bit를 통해 cellular subflow를 활성화 해야하는지에 대한 결정을 서버에게 알린다
    - user-specified interface preference는 MPTCP interface를 설정하여 실현된다.
    - subflow의 throughput를 추정하기 위해 Holt-Winters(HW)를 사용한다.
        - He et al이 제안한 매개변수를 사용하여 MP-DASH 구현
- schedular를 구현하기 위해 두번째 subflow를 효과적으로 관리해야 한다.
    - 각 packet에 대해 사용 가능한 subflow를 선택하는 기존 MPTCP schedular 활용
        - cellular가 사용불가 하다면 skip한다 (handshake message 교환의 overhead가 없다)
        - MPTCP scheduler처럼 다양한 네트워크 환경을 다룰 수 있다.
- MP-DASH Video Adapter
    - based on the open-source GPAC video player
        - throughput-based rate adaptation algorithm
        - 마지막 chunk의 다운로드 시간을 측정하여 throughput을 추정하고, estimated throughput보다 작은 가장 큰 encoding bitrate를 선택한다.
    - FESTIVE
        - 더 강력하고, 공정하며, 안정적인 대표적인 throughput-based DASH algorithm
    - BBA
        - origin buffer-based adaptation algorithm
    - BBA-C
        - less aggressive bandwidth usage
- **Multipath Video Analysis Tool**
    - utilization, rebuffering, video quality switch, energy consumption같은 multipath를 통한 video streaming을 분석하는 것을 용이하게 한다.

### **7. EVALUATION**
****

**7.1 Methodology**

- set up an MPTCP testbed using a client laptop and a server machine (Linux Kernel)
- MP-DASH에 의해 강화면 MPTCP로 작동한다.
- client는 LTE, WiFi interface를 가지고 있다.
    - WiFi : 5GHz frequency band, RTT < 1ms, bandwidth > 50Mbps (client↔server)
    - WiFi latency to be 50ms
    - LTE : 50~60ms RTT
- MP-DASH scheduler는 cellular data usage를 줄이는 것이 목표 → 에너지 소비 감소

**7.2 Performance of MP-DASH Scheduler**

*7.2.1 Experiments over Real WiFi and LTE*

- WiFi : 3.8 Mbps, cellular : 3.0 Mbps
    - ~10.5 sec alone WiFi, ~6sec use MPTCP
- evaluate the impact of the MP-DASH scheduler on both the default and round-robin MPTCP schedulers

<img width="446" alt="스크린샷 2024-01-13 오후 1 12 43" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/e143347c-7a81-4754-8903-778067608aa9">

- MP-DASH는 cellular data와 radio energy usage를 확실히 줄였다 (수정되지 않은 MPTCP보다)
    - deadline이 길어질 수록 절약되는 것은 늘어난다
    - 68%의 cellular data and 44%의 radio energy의 감소
- with high bandwidth prediction errors, the deadline might be missed (실전에선 잘 일어나지 않음)
    - Algorithm 1의 $\alpha$값의 조정으로 더 줄일 수 있다.
    - $\alpha$값이 0.8일때도 28%, 15%의 감소를 보인다.

*7.2.2 Trace-Driven Simulation*

- 처리량이 완벽하게 알려진 최적의 경우와 비교하여 처리량 예측 오류가 있는 현실적인 네트워크 조건에서 성능을 이해하기 위해 MP-DASH 스케줄러의 추적 중심 시뮬레이션을 수행
- WiFi : 3.8 Mbps, cellular : 3.0 Mbps
- 순간 처리량의 표준 편차는 σ=10% and 30%
- Algorithm 1과 Holt-Vinter algorithm 사용 (50ms RTT)

<img width="454" alt="스크린샷 2024-01-13 오후 1 25 01" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/674ebd5a-21fd-4add-bf79-300185789612">

<img width="443" alt="스크린샷 2024-01-13 오후 1 28 23" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/d7057e3a-3a78-449c-a835-5b0310aa45ee">

- bandwidth는 계속 변동하는 경향이 있다
- online algorithm이 더 좋은 성능을 가진다.

**7.3 Evaluation of the MP-DASH Frame-work**

- use chunk duration of 4 sec
- The length of each video is 10 minutes
    - 150chunk로 구성된다.
- <img width="441" alt="스크린샷 2024-01-13 오후 1 37 26" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/c6182f17-5330-42ae-82e7-0626203fa67c">

*7.3.1 Inefficiency of Throughput Throttling*

<img width="459" alt="스크린샷 2024-01-13 오후 1 41 14" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/72b349e0-6d38-4ad8-8519-ba9fe3c1411d">

- MPTCP의 throughput은 TCP의 congestion control로 인해 가장 높은 bitrate level로 인코딩 되지 못한다.
- throttling the cellular path는 cellular traffic을 줄이지만, 높은 radio energy consumption이 요구된다.
    - MP-DASH는 둘 다 이룰 수 있다.
    - <img width="446" alt="스크린샷 2024-01-13 오후 1 46 29" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/811e8014-b114-4e26-880f-b8e1bbc309ea">
    - 3번째 그림은 default MPTCP로 cellular를 많이 사용하고 있음을 알 수 있고, Figure 1과 비슷하다
    - 2번째 그림은 MP-DASH로 WiFi 혼자 deadline을 충족시키지 못할 것 같은 경우 cellular가 사용된다.

*7.3.2 Controlled Experiments*

- using higher LTE throughput will lead to more savings of cellular data brought by MP-DASH.

**FESTIVE AND GPAC**

<img width="319" alt="스크린샷 2024-01-13 오후 2 00 50" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/5500c0b7-81c3-4d7b-b6ca-e04f86ea97c7">

- The cellular data usage is shown in bars and the energy consumption is marked as round dots.
- MP-DASH는 cellular data and radio energy를 확실히 줄인다.
- WiFi 3.8→3.0, cellular 사용량이 증가하게 된다→saving 감소
- worst network에선 saving이 증가한다.
    - chunk quality가 level5에서 4로 떨어지고 WiFi throughput이 level 4의 bitrate과 비슷하기 때문에 효율 증가
    - GPAC도 throughput-based라 비슷한 결과를 도출한다.
    
    <img width="457" alt="스크린샷 2024-01-13 오후 2 10 02" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/817ba4b1-c033-4d35-84fd-975eecfc64f7">
    
- for chunks with higher-than- average encoding bitrate, the duration-based setting requires more cellular data than the rate-based one
- MP-DASH eliminates most of the idle gaps

**BBA**

<img width="305" alt="스크린샷 2024-01-13 오후 2 48 24" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/9646ae82-9d50-4e3b-a9a5-4aafbfc5b55d">

- BBA의 aggressive한 특성으로 인해 saving이 적다.
- W2.2/L1.2 상황에서 cellular data saving을 제공하지 않는다.
- 전체 네트워크 용량이 가장 높은 encoding bitrate를 지원할 수 없을때 더 공격적이게 된다.

**BBA-C**

<img width="317" alt="스크린샷 2024-01-13 오후 3 00 52" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/a0ee5a48-7a84-4bd3-af3b-e3cd1ee7b7da">

- cellular-friendly version of BBA
    - bitrate 진동을 방지하기 위해 선택된 bitrate가 실제 네트워크 용량보다 높지 않게 제한
- 처음 2개의 결과는 BBA와 같은 결과
    - 전체 대역폭이 가장 높은 비트 전송률을 지원할 수 있다.
- W2.2/L1.2에서 BBA-C는 효과적으로 bitrate oscillation을 막는다. (level 4로 고정)
    - cellular data를 줄일 수 있는 기회를 준다.
- BBA-C with MP-DASH reduces cellular data usage and radio energy consumption by 69% and 50%
- mobile video에서 resource usage, playback bitrate, playback smoothness 제공

*7.3.3 MP-DASH in Real-World Settings*

- FESTIVE, BBA(DASH algorithm)
    - vanilla MPTCP and MP-DASH with rate-based, duration-based
- BBA와 BBA-C는 같은 결과를 도출했다.
    - MPTCP의 용량이 가장 큰 video bitrate보다 크기 때문에… 비트 전송률을 제한할 필요 없음
- 일반적으로 FESTIVE가 BBA 보다 더 saving이 크다
    - radio energy consumption saving에서도 비슷하다.
- 82.65%에서 playback bitrate reduction이 없다
- the average reduction is only 2.5% → MP-DASH가 효과적으로 cellular data와 Radio energy를 줄여준다.