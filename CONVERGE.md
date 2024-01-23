# CONVERGE: QoE-driven Multipath Video Conferencing over WebRTC

### ABSTRACT

화상회의가 일상의 필수가 되었지만, 화상회의를 지원하는 protocols는 차세대 네트워크 혁신에도 불구하고 속도를 따라 잡지 못하고 있다.

- 또한 여러 곳에서 video가 대중적이게 되면서 QoE에 대한 요구가 증가하고 있다.
- 이에 대한 해결책으로 Multipath protocols이 나타난다.

이 논문에선 multipath를 지원하는 WebRTC 확장에 대한 것을 보여준다.

- 오히려 multipath를 지원하는 WebRTC의 간단한 확장이 Legacy WebRTC 및 다중 경로 화상회의 플랫폼인 Propose Converge보다 성능이 떨어짐을 보여준다.
- CONVERGE는 3가지의 구성 요소로 QoE를 향상시킨다.
    - a video-aware scheduler
        - packet을 스케줄링하기 위해 real-time video structure를 사용한다.
    - video QoE feedback
        - 각 path의 packets의 개수를 scheduler가 조절하게 돕는다. (from receiver)
    - video-aware and path-specific packet protection
        - FEC와 QoE사이의 절충안을 고려하며 기존의 WebRTC의 FEC mechanism을 향상시킨다.
- CONVERGE를 사용하여 media 처리량이 1.2X 향상되고, 양단 latency가 20% 줄었고, image quality가 55% 증가하였다.

### 1. INTRODUCTION

bandwidth-intensive applications는 최근 무선 네트워크에선 문제이며, 낮은 품질의 화상 회의를 제공할 수도 있다.

- 더 많은 용량을 가진 차세대 네트워크이 제공되도, 화상회의는 높은 latency와 frame drop을 포함한 아직 낮은 성능을 보여준다.

<img width="702" alt="스크린샷 2024-01-19 오후 4 48 06" src="https://github.com/RakunKo/Network/assets/145656942/d68fc0c3-83ed-41e2-bbb6-edf1a33b7f38">

- variations in frames per second(FPS) and per-frame end-to-to latency(E2E)는 call안에서 방해의 원인이 되고, 낮은 QoE를 제공하는 원인이 된다.
- network가 최소 요구 bandwidth를 제공하지 못할때, 다른 network가 보상할 수도 있다.
- dual-SIM, WiFi with cellular, aggregating these connection이 잠재적 해결책이 될 수도 있다.

**Converge**

- multipath video conferencing을 해결하고 사용자 QoE를 향상시킨다.
- a video-aware scheduler, video QoE feedback, video-aware and path-specific packet protection을 이러한 향상을 위해 사용한다.
- MPTCP, MPRTP, MPQUIC말고 화상회의의 특정 요구 사항을 해결시켜준다.

**Video-aware scheduler**

- 기존의 패킷 스케줄러는 복잡함에 대한 지식 부족으로 decoding sequences가 중단되고, 빈번한 멈춤과 frame drop을 화상회의 동안에 사용자가 부정적인 영향을 받는다.
- 인코더의 real-time video information을 활용하여 패킷을 스케줄링하여 문제를 해결한다.
    - scheduler는 receiver로 부터 QoE를 최적화하는 path에 packet을 할당한다.
    - receiver로 부터 받은 QoE feedback을 기반으로 하여 각 경로로 보내지는 packet의 개수를 지속적으로 조절한다.
        - 패킷 할당 및 속도 제어는 네트워크 조건뿐만 아니라 수신기 애플리케이션의 비디오 구성을 기반으로 한다.
        

**Video QoE feedback**

- sender에게 결정적인 video QoE feedback을 제공하기 위해 receiver의 buffering과 video frame construction process를 통해 배운다.
- receiver는 경로 비대칭성의 원인이 되는 video QoE안에서 모든 악화를 찾기 위해 video frame construction를 모니터링 한다.
    - 경로 비대칭은 불가피하기 때문에, receiver에서 video QoE가 언제 영향받기 시작하는지 아는 것이 필수적
- receiver로 부터 sender가 video QoE에 대해 multipath의 영향을 이해하는걸 가능하게 한다.

**Video-aware and path-specific packet protection**

- 더 나은 QoE를 달성하기 위해 packet loss를 보호하는 것은 중요하다.
- frame을 decode하는데 FEC에 단독으로 의존하게 되면 E2E latency를 증가시킨다 → QoE에 부정적인 영향을 준다.
    - 모바일 네트워크에서 보내지는 FEC의 비율은 25%인데 그중 10~25%만이 실질적으로 이용된다.
- receiver의 실시간 video QoE feedback에 기반한 **Video-aware and path-specific mechanisms**으로 해결한다.
    - 이 방식은 WebRCT의 정적 table 기반 FEC 선택보다 뛰어난 성능을 보여준다.

Converge를 Real-time Transport Protocol(RTP)m Real-time Transport Control Protocol(RTCP)와 같은 실시간 스트리밍 프로토콜과 넓은 범위의 디바이스에 대해 경쟁력을 보장하기 위해 **user-space**에 개발함

- 개발 비용을 줄이기 위해 WebRTC 상단에 탑재
- multipath와 여러개의 카메라를 지원한다
- 두 end point중 하나가 다중경로를 지원하지 않는 경우 표준 WebRTC 프로토콜로 원활하게 돌아가므로 기존의 많은 WebRTC 기반 화상 회의 애플리케이션과 역호환이 보장된다.

### 2. BACKGROUND

**2.1  Overview of WebRTC**

- WebRTC has three components : sender, network controller, receiver

<img width="624" alt="스크린샷 2024-01-20 오후 2 24 12" src="https://github.com/RakunKo/Network/assets/145656942/8781b7a7-082c-4f01-8f98-b9da476ed772">

- sender : takes the inferred rate from the network controller and encodes video frames captured by the camera at that rate
    - 인코딩된 비디오 frame은 RTP로 패킷화되어 전송되어진다.
- receiver : receives the RTP packets and utilizes two buffers to generate video frames.
    - RTCP 패킷을 사용하여 sender에게 loss와 delay reports를 전송한다.
- network controller : analyzes these reports and adjust the encoding rate for the sender

**Network controller**

- WebRTP는 Google Congestion Control(GCC)를 사용한다.
    - 이 알고리즘은 이용 가능한 네트워크 경로를 동적으로 추정한다.
- sender의 GCC는 loss와 delay reports를 사용하여 두개의 rate를 계산한다.
    - 두가지 rate중 낮은 rate를 사용해서 인코딩하여 congestion을 최소화한다.
    - GCC는 rate를 receiver로 부터 feedback이 올때 계산하게 된다 → feedback 주기는 bandwidth와 관련이 있다.

**Encoding and decoding pipeline**

- encoder : the raw image frames→video frames (network controll에서 정해진 rate로 인코딩)
    - Keyframes(I-frame) : 완전한 frame의 정보를 담고 있다.
    - delta frame : key frames의 변화를 포착한 정보를 담고 있다.
    - 두 frame은 RTP packet으로 packet화되어 network를 통해 전송된다.
        - RTP packet은 media data와 control information(decoding info)으로 나뉜다.
        - control packet이 없다면 deconding되지 못한다!
- sender에선 RTP packet을 Loss로 부터 보호하기 위해 FEC packet을 생성한다.
    - WebRTC에선 XOR-based FEC를 사용한다.

**Receive buffers in WebRTC**

- The packet buffer : 크기에 제한이 있으며, frame을 만들기 위해 특정 frame에 속하는 모든 RTP packet을 모은다.
    - 만약 frame과 관련된 모든 packet이 도착하지 않거나 너무 늦게 도착하면..→ buffer는 packet을 버린다.
    - frame을 구성하기 위해 모든 packet을 모으는데 필요한 시간을 gathering delay라고 한다.
        - FEC packet이 처리되는 것에도 delay가 증가될 수도 있다.
- frame이 완성되면 frame buffer로 push 된다.
    - frame buffer 또한 크기에 제한이 있으며, 다가올 frame에 충분한 공간이 없다면 old frame을 없앨 수 있다.
    - missing or purged frame이라면 packet을 drop할 수도 있다.
    - frame buffer에선 frame들을 모아 decoder로 전송하게 된다.
        - frame buffer에서 frame이 도착하는 사이의 시간을 interframe delay라고 한다.

**Video QoE and parameters**

- 화상회의의 QoE는 frame rate, freeze duration, E2E latency, media throughput, image quality에 의해 결정된다.
    - gathering delay와 interframe delay는 비디오 QoE에 큰 영향을 미친다.

**2.2  Existing Multipath Protocols**

- MPTCP : packet을 이용 가능한 경로에 분배한다. 최소 RTT을 가장 선호한다.
- MPQUIC : user-space multipath extension of QUIC protocol (화상회의보다 latency에 덜 민감한 비디오에서 사용)
- MPRTP : UDP-based multipath extension to RTP. (latency에 민감한 실시간 media 통신에 사용)
    - feedback을 제공하지 않으며, packet을 분배하기 위해 이용 가능한 모든 경로를 사용한다.

- multipath transport protocol에선 Head-Of-Line blocking(HOL) issues가 나타난다.
- schedulers : Musher, minRTT, MPRTP
- protocol : MPTCP, MPQUIC, MPRTP

**2.3  Multipath Is Not Enough**

<img width="624" alt="스크린샷 2024-01-20 오후 2 24 12" src="https://github.com/RakunKo/Network/assets/145656942/8781b7a7-082c-4f01-8f98-b9da476ed772">

- WebRTP는 QoE의 만족을 더한 방해없는 화상회의를 제공하는 것에 실패했다. (첫번째 그림)
- camera streams의 개수가 늘어날 수록 QoE가 악화되었다. (2번째 그림)
- 화상회의를 위해 디자인되지 않은 scheduler는 낮은 성능을 보였다.
- converge는 반면에 좋은 성능을 확인할 수 있다. (높은 FPS, 낮은 freeze duration, E2E latency)

<img width="656" alt="스크린샷 2024-01-20 오후 3 31 37" src="https://github.com/RakunKo/Network/assets/145656942/96b9832a-4e89-4294-ab06-2a0c7b765477">

- 좋은 비디오 QoE의 FPS는 24이다. → WebRTC는 도달하지 못했다.
- 다른 multipath은 오히려 WebRTC보다 더 높은 freeze duration을 보여준다. (camera stream의 개수가 증가하면 더 증가하게 된다.)
    - 더 많은 frame drop과 keyframe request는 낮은 FPS와 더 많은 video freezes를 설명
- packet이 도착하는 순서가 어긋하면  receiver에서 decoding을 방해하는데, dropping을 이끈다.
- receiver는 QoE와 관련없는 delay와 loss report만을 각 경로에게 보낸다
- CONVERGE는 video-aware scheduler를 feedback 매커니즘과 사용한다 → 효과적으로 frame drop을 피하고, 더 좋은 FPS를 최소한의 video freeze로 제공한다.
    - E2E latency 또한 경로 손실을 기반으로 FEC 속도를 결정하는 경로별 접근 방식을 채택하여 효과적으로 완화 가능
- FEC overhead는 굉장한 processing delay와 E2E latency의 증가로 이어진다. (bandwidth를 최대로 활용하지 못하면서…)

### 3. DESIGN OF CONVERGE
**3.1  Video-aware Packet Scheduling**

- multipath를 사용할때 video 구성이 방해받는 다면 QoE는 빠르게 악화될 수 있다.
- 이를 방지하기 위해 비디오 frame안에 패킷의 우선순위와 의존성을 기반으로 3개 level의 control을 scheduler안에 디자인 했다.

1. Frame-level control
    - **“Keyframe”**은 frame의 완전한 정보를 운반하고 독립적으로 해독되어질 수 있다.
    - “delta frame”은 receiver에서 해독되어질 수 있는 연관있는 **keyframes** 없이는 쓸모가 없어진다.
        
        → multipath scheduler가 어떻게 다른 frames들로 부터 packet의 우선순위를 정할지 가이드라인을 제공한다.
        

<img width="623" alt="스크린샷 2024-01-20 오후 4 58 03" src="https://github.com/RakunKo/Network/assets/145656942/bda8e1cd-52de-4082-8255-9fdbebf9b561">

- keyframe인 $F_{i-1}$의 packet중 3이 loss가 되고, fast path에 속한 delta frame인 $F_i$와 $F_{i+1}$이 도착하게 된다면… → 문제가 발생한다.
    - $F_i$와 $F_{i+1}$이 packet buffer에 도착하면 packet들을 모아 frame buffer로 push 된다.
    - 하지만 $F_{i-1}$에선 packet이 다 도착하지 못해 frame buffer로 push가 불가능하다.
    - $F_i$와 $F_{i+1}$은  $F_{i-1}$에 의존적이므로  $F_i$와  $F_{i+1}$은 frame buffer에서 버려진다.
    - $F_{i-1}$은 다시 sender에게 요청되어진다.
    - Keyframe을 완전히 받을 때 까지, video는 freeze 상태로 들어가고 QoE에 부정적인 영향을 미친다.

- $F_{i-1}$이 keyframe인것을 알고 있고, delta frames을 해독하기 위해 필요하단 걸 알고 있다면 freezing of the video를 막을 수 있다.
    - 이 케이스에선 packet 3,4는 fast path로,  $F_i$와 $F_{i+1}$은 slow path으로 scheduling
    - keyframe missing을 줄여 video freezing을 예방하고 QoE를 향상

1. Packet-level control
    - video frame이 RTP packet으로 패킷화되어질 때, 성공적인 해독을 위한 필수적인 정보는 **두개의 다른 RTP packet으로 분배**되어야 한다.
    - keyframe과 delta frame에는 PPS packet이 필요하고, delta frame group에는 SPS packet이 필요하다. → 디코딩에 사용되므로 성공적인 전송이 보장되어야 한다.

<img width="619" alt="스크린샷 2024-01-20 오후 5 17 18" src="https://github.com/RakunKo/Network/assets/145656942/8af2eedd-43a8-4140-b7ca-8a5b383af70c">

- packet 3은 PPS packet이고 packet 4는 SPS packet이다
- 결과는 video freezing, 사용자 QoE에 부정적인 영향을 미쳤다.
- 만약 scheduler가 keyframe의 중요성을 알고 덜 중요한 다른 packet에게 slow path를 할당한다면 QoE drop을 피할 수 있다.

1. Reliability-level control
    - 아무리 packet scheduler가 최적의 결정을 내리고 video-aware하고 자세한 feedback을 receiver로부터 받아도,  **packet loss는 여전히 일어날 수 있다.**
        - FEC나 NACK같은 보호 매커니즘을 적용하여 packet loss를 완화할 수 있다.
    - FEC packet은 sender에서 lost packet을 복구하기 위해 사전에 만들어 진다.
        - media packet과 같이 보내지고 해독되기 때문에, receiver에서 보내고 과정 delay를 소개하기 위한 추가적인 네트워크 대역폭이 필요하다.
    - NACK은 receiver가 sender에게 특정 packet의 재전송을 요청한다.
    
    <img width="631" alt="스크린샷 2024-01-20 오후 5 27 05" src="https://github.com/RakunKo/Network/assets/145656942/31e3177a-2756-44be-a52f-c3c329120223">
    
    - $F_i$, $F_{i+1}$, $F_{i+2}$는 2개의 추가적인 보호 packet이 존재한다. → 33%의 frame의 품질 저하 발생
        - media, FEC packet은 둘다 네트워크 경로의 bandwidth constraint를 받기 때문에
    - NACK에 의해 재전송된 packetdms fast path에 할당되게 된다. → 성공적인 재건설 가능

**3.2  Multipath-aware QoE Feedback**

**Drops in the receive buffers due to multipath asymmetry**

- receiver는 크기가 제한된 packet buffer와 frame buffer 2개를 이용한다
    - multipath를 사용하면, path asymmetry가 buffer를 drop시키고, 이는 frame gathering delay와 interframe delay를 증가시킨다.
    - path asymmetry로 인해 늦게 도착하는 packet이 존재하고, 늦게 도착한 packet은 새로 들어올 packet을 위해 drop된다.
    - 결국 frame이 gathering 되지 못하고 그 전에 drop되는 현상이 벌어지고 frame buffer에 push 되지 못한다.

**Feedback to avoid the QoE drop over multipath**

- buffer의 drop을 줄이기 위해선 path asymmetry의 정도를 줄이는 것이 필수적
- sender에게 packet이 drop되는 경로에 대한 정보를 포함한 QoE feedback을 제공한다.
- sender가 feedback을 받으면 drop을 유발하는 path에 scheduling하는 것을 줄인다.

**3.3 Protection for QoE over Multipath**

- protection of packet은  video frame의 대역폭을 악화시키고 추가적인 decoding delay를 발생시킨다.
- 반면에 적은 protection은 packet과 frame의 missing때문에  QoE를 굉장히 감소시킨다.

**Path-specific protection**

- FEC가 application-level protection인 것과 반대로 FEC가 각 경로에 대해 특정하게 생성된다.
- P1 0%, P2 10%의 loss rate, media packet 60, P1에 30 P2에 30packet씩 할당
- **application-level로 보면 6 FEC packet이 필요하다(질문)**  반면 path-specific은 3 FEC packet이 필요
- QoE feedback은 추가적으로 FEC packet의 비율을 줄이는데 도움을 준다.
    - 20개의 늦게 도착하는 packet이 있다고 feedback이 오면, 20 packet을 P1에 10 packet을 P2에 할당하면 FEC가 P2에 1개만 필요해진다.

**Protection based on the video structure**

- 기존에 존재하던 WebRTC FEC 매커니즘은 다른 packet에 똑같은 FEC rate를 적용했다.
    - 이런 접근은 multipath에서 비효율적이다.
- P1이 5%, P2가 10%의 loss rate, P2는 굉장히 큰 대역폭
    - sender’s scheduler는 우선순위가 높은 packet을 보호하기 위해 높은 비율의 FEC를 path P1에 할당할 수 있다.
    - 만약 feedback이  P2가 QoE에 확실한 영향을 주지 않는다고 하면, scheduler는 keyframes을 높은 FEC rate(비록 큰 사이즈여도)로 P2를 통해 전달할 수 있다.
    - 만약 P1의 loss rate가 무시할만 하다면, P1에 keyframe을 P2에 delta frame을 전송할 수도 잇다. (QoE의 하락 없이)
- keyframe이 낮은 loss rate로 전달되는 것은 방해받지 않는 video 재생을 가능하게 한다.

### 4. CONVERGE AS SYSTEM

<img width="520" alt="스크린샷 2024-01-22 오후 3 17 40" src="https://github.com/RakunKo/Network/assets/145656942/2e211763-60b9-476f-a5fa-83e5ef6eeae2">

- **encoder** : congestion control module에서 파생된 rate를 사용하여 원래의 frames을 encode한다.
    - camera stream → RTP packets of different priorities
- **video-aware scheduler** : 우선순위가 높은 packet을 주로 fast path로 보내고, 남은 packet을 이용 가능한 모든 경로로 보낸다.
- **video QoE feedback**(receiver) : 비대칭이 QoE에 악영향을 끼치는지 인식하고 sender에게 QoE의 감소가 관촬되면 feedback을 보낸다. 이를 scheduler가 사용한다.
- **FEC controller** : 품질, 지연, QoE에 대한 영향을 고려하여 loss를 막기위해 FEC packet을 생성한다.

**4.1 Video-aware RTP Scheduler**

- congestion control module은 encoding rate를 제공하고, scheduler가 fast path와 sending rate를 추론하는 것을 돕는다.
- 각 RTP packet의 우선순위를 확인하고 우선순위 packet에 fast path를 선택한다.
    - path에 대한 QoE feedback을 통해 각 경로의 sending rate를 조절한다.

**Congestion control**

- 각 경로의 loss와 delay같은 통계는 encoding rate를 파생하기 위해 사용된다.
- CONVERGE의 congestion control module은 GCC를 확장한 모든 이용가능한 경로로 확장한 것이고, encoding rate를 encoder에서 결정한다.
    - RTCP feedback이 사용된 모든 경로의 receiver로 부터 있으면 sending/encoding rate를 업데이트 한다.
    - sending rate $S_i$ for path $i$는 feedback에 기반하여 계산되어 진다.
        - The sum of the rates for each path (the aggregate rate) = $\sum_{{i=1}}^{n}S_i$
        - encoder는 aggregate rate과 maximum rate set(N개의 RTP packet을 생성하기 위한…)중 최소값을 사용한다.
        - $n$은 사용가능한 경로의 활성화된 경로의 개수를 나타낸다.
    - $P_{max}$ : $S_i$에 기반하여 각 경로에 보낼 수 있는 최대 packet 개수 (packet수를 조정하기 위한 upper threshold 역할을 한다.)

**Priority packets**

- encoder만이 packet의 type과 우선순위를 알고 있고, QoE 악화를 방지하기 위해 scheduler에 이 정보를 알린다.

<img width="431" alt="스크린샷 2024-01-22 오후 3 31 02" src="https://github.com/RakunKo/Network/assets/145656942/e1ea1473-e37d-47ec-accf-96f2613471b7">

- Retransmitted packet (NACK 요청에 대한 response) : packet의 recovery, 연속성, frame의 recovery를 돕는 가장 높은 우선순위이다.

<img width="515" alt="스크린샷 2024-01-22 오후 3 34 12" src="https://github.com/RakunKo/Network/assets/145656942/8d757a6f-8a4a-4121-b88e-c06ed2996216">

- 가장 짧은 transmission completion time을 가진 경로를 선택한다.
- $cpt_i(N) = (N*k/rate_i)+rtt_i/2$ : N개의 packet을 i번째 link를 통해 전송할때 요구되는 시간
- 만약 fast path의 $P_{max}$를 넘어서는 우선순위 packet은 우선순위에 따라 다른 경로로 분배된다.
- 한가지 예외는 FEC packet이 수용될 수 없다면, FEC packet이 만들어진 경로에 보내진다.

**Media packets**

- 우선순위가 없는 media packet은 $S_i$에 비례하여 이용 가능한 경로에 분배된다.
- $P_i = (S_i/\sum\limits_{j=1}^{n}S_j) *N$ : path rate $P_i$ (0이 되면 비활성화)

<img width="447" alt="스크린샷 2024-01-22 오후 3 48 39" src="https://github.com/RakunKo/Network/assets/145656942/bf6fbb8b-e609-4583-87db-b005cb99a0f1">

- $P1$은 $rate_1 = 15Mbps$, $P2$는 $rate_2 = 5Mbps$, 40 media packet (scheduler에 의해 전송되야 함)
- feedback이 $P2$에게서 왔고, $\alpha$가 5다.
- $P1$에는 15/20*40 = 30 packet이, $P2$에는 5/20*40 = 10 packet이 할당된다.
- scheduler가 $\alpha$를 $P2$에게 알리면 $**P1$에는 35packet $P2$ 5packet**이 QoE 악화를 피하면서 보내진다.

**4.2 QoE-based Feedback**

- frame construction process를 mutilpath asymmetry로 인한 QoE 악화를 탐지하기 위해 추적한다. schedyler에게 signal을 보내 사고를 방지한다.
- sender의 scheduler는 각 경로의 packet을 조절하고 필요하다면 경로를 활성화하거나 비활성화

**Video QoE parameters**

- Frame Construction Delay(FCD), InterFrame Delay (IFD)
    - multipath asymeetry의 영향을 측정하고 QoE를 나타내기 위해
- FCD : the delay in gathering all the packets of a frame.
    - packet이 늦거나 loss되면 FCD는 증가한다. (모든 packet이 도착해야 decoding 가능하기 때문)
    - FCD가 증가하면 frame buffer안의 frame이 감소하고 IFD의 증가로 이어진다.
- IFD : the time interval between the arrival of two consecutive frames in the frame buffer
    - IFD의 증가는 낮은 frame rate에 의한 QoE 악화를 나타낸다.
- video QoE feedback을 구축하기 위해 FCD, IFD 2개의 매개변수를 사용한다.

**Video QoE feedback**

- slow path로 보낸 모든 packet이 frame construct 전에 도착하면 더 많은 packet을 slow path를 통해 보낼 수 있다는 걸 의미
- 반대로 후에 도착하면 FCD가 증가하게 되고, slow path로 보내는 packet 양을 줄여야한다.
    - 새로운 frame이 frame buffer에 들어오면 IFD를 계산하고, 각 경로의 늦은 packet의 개수를 기록한다.
- 예상되는 $IFD_{exp}$를 얻기 위해 sender가 설정한 frame rate를 반전시킨다.
    - sender’s frame rate는 RTCP(SDES) message를 사용하여 보고된다.
- feedback은 $IFD > IFD_{exp}$일 때 보내진다 (QoE가 악화될 때)
    - the number of early/late packets : $\alpha$값으로 나타난다. Equation 2
    - FCD : sender가 경로를 관리하는 것을 돕는다.
    - path $id$ (영향의 원인이 되는 경로를 나타낸다)

<img width="1075" alt="스크린샷 2024-01-22 오후 5 14 36" src="https://github.com/RakunKo/Network/assets/145656942/19fbcdaa-6bf1-48fc-b62f-f3c7d4dd74d4">

- packet 18,24가 늦게 도착하여 $FCD_3$과 $FCD_4$가 증가하게 되고 이는 $IFD$의 증가
    - $IFD > IFD_{exp}$ → QoE drop, feedback이 path2를 통해 보내진다.
    - $id$ = 2, $\alpha$ = -1(1개씩 늦게 도착), $FCD_4$
    - packet ratio가 4:2→5:1로 변경
    - packet 36이 path 1로 경로 변경
- $\alpha$ 의 양수 값은 packet 개수의 증가를 나타낸다.

**Sender’s reaction to feedback**

- sender는 QoE feedback을 받으면, Equation 2에 따라 packet의 개수를 조절한다.
    - packet의 개수가 0이 되면 그 경로는 비활성화 된다.
    - $[rtt_{fast}-rtt_i]/2 <= FCD$ 라면 다시 경로를 활성화 한다.

**4.3 Reliability via FEC Controller**

- CONVERGE는 QoE feedback을 통합하여 각 path으로 향하는 packet의 FEC packet 수를 결정함으로 FEC overhead를 줄인다.
    - FEC rate조절이 필요하면 NACK을 사용하기도 한다.
- 높은 FEC rate를 사용하여 packet을 보호하기 보단, 경로의 packet 개수를 줄이는 방식을 사용한다.
- packet의 중요성과 각 경로의 loss rate를 고려하는 video-aware scheduler를 사용
    - 이는 더 좋은 cost와 reliability를 달성 가능하다.
- the number of FEC packets $FEC_i$ = $l_i*P_i*\beta$
    - $\beta$는 NACK에서 동적으로 조절된다.
    - FEC packet이 lost packet을 recover하기 충분하다면, NACK request를 받지 않는다
    - NACK request를 받는다는 것은, FEC packet의 개수를 늘릴 필요가 있다.
        - $\beta = [1+(NACK_i/(P_i-FEC_i)]$

### 5. IMPLEMENTATION

**Connections management**

- multipath connections과 encryption을 지원하기 위해 WebRTC의 3가지 수정이 필요하다
    - Session Description Protocol(SDP)
        - 각 peer에게 다중경로 capabilities를 알려주게 수정
    - Interactive connectivity Establishment(ICE)
        - 다중 경로를 통한 가능한 네트워크 연결을 포함하게 확장
    - RTP, RTCP, SRTP
        - RTP, RTCP를 암호화가 있든 없든 다중경로 사용을 보장하게 확장
        - RTP, SRTP WebRTC keys를 사용하여 다중경로를 사용할 할 수 있게 보장
- CONVERGE의 다중경로 관리와 WebRTC의 기존 연결 완화(migration)의 충돌을 막기위해 **wrapper**을 추가하여 연결 상태를 모니터링하고, WebRTC 연결 관리 시스템에 동기화한다.

**Transport layer**

- 각 경로에 대해 특정 delay와 loss 통계를 제공하도록 GCC 알고리즘을 강화했다.

**Media layer**

- transport layer로부터 강화된 통계를 통합하여 확장한다.
- FEC packet generator와 Media Optimization (MO) module이 video-aware, path-specific loss recovery 매커니즘을 포함하게 수정했다.

**Receiver feedback**

- receiver가 feedback을 위한 path ID 포함하게 수정한다.
- RTCP message
    - sender : 예측된 frame rate
    - receiver : QoE based feedback to the sender
        
        

**Multipath WebRTC variants**

- multipath WebRTC variants
    - M-TPUT : throughtput-based scheduler from Musher into WebRTC
    - SRTT : use minRTT scheduler (MPTCP, MPQUIC에서 기본 스케줄러)
    - M-RTP : packet을 loss-based estimated sending rate로 보낸다.

### 6. EVALUATION

- Signal-to-Noies Ratio(PSNR), video stall duration, Quantization parameter(QP), E2E latency
    - 높은 PSNR과 낮은 QP는 더 좋은 image 품질을 나타낸다

**6.1 Converge in the Wild**

<img width="1045" alt="스크린샷 2024-01-23 오후 3 16 54" src="https://github.com/RakunKo/Network/assets/145656942/2a77e30e-bd84-46a9-8c98-d15ff64fc9bd">

- Converge가 다른 것들에 비해 높은 video throughput과 19~24FPS, 낮은 E2E latency를 보여줌
- WebRTC를 이용한 것은 낮은 throughput과 비디오 정지를 경험하게 된다.

<img width="477" alt="스크린샷 2024-01-23 오후 3 21 22" src="https://github.com/RakunKo/Network/assets/145656942/d83f3582-8c48-4c9e-9ce6-d2c6f8dbe905">

- converge는 높은 throughput, FPS, 낮은 stalls duation, QP를 보여준다.
- WebRTC보다 41%, 31%의 FPS 향상을 보여준다.
- WebRTC보다 33%, 52%의 video stall을 줄여준다.

<img width="496" alt="스크린샷 2024-01-23 오후 3 24 46" src="https://github.com/RakunKo/Network/assets/145656942/0d248cf1-f215-47ca-832f-dd23920c035d">

- converge는 여러 카메라 stream에 대해 더 좋은 QoE를 제공한다.
    - FEC overhead를 줄이고, FEC 활용을 향상시켜서 달성된다.
        - FEC overhaed 68%, 51% 향상
        - IFD와 FCD를 FEC overhead를 줄여 최소화할 수 있다.

**6.2 Converge in Controlled Environment**

**The benefit of QoE feedback**

<img width="1056" alt="스크린샷 2024-01-23 오후 3 29 37" src="https://github.com/RakunKo/Network/assets/145656942/387f0827-c4c1-4281-bee5-ede909968ac5">

- path1이 지속적으로 25Mbps를 유지해도 path2의 영향으로 전체적인 received rate는 최대 encoding rate보다 떨어지게 된다.
- IFD와 FCD또한 크게 증가했다.
- receiver는 빠르게 sender에게 feedback을 제공하고, sender는 path2의 packet수를 조절한다.
    - 중복 패킷은 화상회의에 사용되지 않더라도 네트워크 손실 및 지연을 측정하기 위해 path2에 전송
- 결과적으로 FCD가 감소한다.

**QoE trade-off analysis of FEC**

<img width="508" alt="스크린샷 2024-01-23 오후 3 40 08" src="https://github.com/RakunKo/Network/assets/145656942/425672fe-22ea-466c-a817-7241d256b818">

- table-based FEC에선 100 media packet마다 loss 비율이 1%라도 40개의 FEC packet을 보내야한다… (40%의 추가 패킷 → but 그중 20%만 실질적으로 사용)
- path-specific FEC에선 5%정도의 overhead, 5%의 FEC packet이 거의 다 활용
- converge는 bandwidth 및 처리 요구 사항을 줄이면서 frame rate 및 대기 시간 목표를 충족할 수 있도록 적절한 양의 FEC를 제공

<img width="524" alt="스크린샷 2024-01-23 오후 3 46 24" src="https://github.com/RakunKo/Network/assets/145656942/62391ad6-2dbc-4ce5-b218-f0fd7b6497cf">

- path-specific을 사용하는 converge가 E2E latency도 낮으면서 throughput도 높다.

<img width="504" alt="스크린샷 2024-01-23 오후 3 48 00" src="https://github.com/RakunKo/Network/assets/145656942/6df06f50-093e-4331-b1f5-f73e37fdc549">

- Converge를 사용함으로써 loss rate가 10%일땐, 97% frame drop 개선, 69% freeze duration 감소, 66%의 keyframe request 감소가 일어났다.

**Comparison with existing solutions**

<img width="1049" alt="스크린샷 2024-01-23 오후 3 50 31" src="https://github.com/RakunKo/Network/assets/145656942/adbf943b-45a4-4387-bd5f-58da0e96214c">

- converge는 높은 throughput을 기반으로 높은 FPS와 video stalls를 줄이고, QP로 인한 video quality를 더 좋게 해준다.
- 또한 FEC overhead는 가장 적은데 효율성은 converge가 가장 높다.
    - 적은 FEC data를 보내면서 모두 활용한다는 뜻
- E2E latency는 화상회의에서 가장 중요한 요인
    - converge는 E2E에서도 가장 낮은 평균을 보여준다.

<img width="461" alt="스크린샷 2024-01-23 오후 3 54 56" src="https://github.com/RakunKo/Network/assets/145656942/a4bad292-1166-4b75-abcb-be6d910c46f7">

- Qp와 PSNR또한 converge에서 가장 낮고 높게 나타났다 → 좋은 image 품질
