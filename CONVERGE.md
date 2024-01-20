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
