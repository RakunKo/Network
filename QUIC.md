# **The QUIC Transport Protocol:
Design and Internet-Scale Deployment**

### 1. **ABSTRACT**

- to improve  transport performance for HTTPS
- to enable rapid deployment and continued evolution of transport mechanisms
    
    ⇒encrypted, multiplexed, low-latency transport protocol 
    
    ⇒현재 구글에서 사용중이다
    
    <img width="402" alt="스크린샷 2024-01-09 오후 8 05 34" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/e62aae8b-ee7a-4692-a767-0470c15ca1fa">
    

### 2. Introduction

- QUIC replaces most of the traditional HTTPS stack : HTTP/2, TLS, TCP
- UDP를 기반으로 한 user-space transport
    
    ⇒ user-space transport는 applications의 일부로 개발이 용이
    
    ⇒UDP를 사용함으로서 QUIC packets이 middlebox를 통과할 수 있다.
    
    - middlebox를 통해 authenticated and encrypted, preventing modification and limiting ossification of the protocol
- cryptographic handshake
    - minimizes handshake latency for most connections by using known server credentials on repeat connections
    - by removing redundant handshake-overhead at multiple layers in the network stack
- eliminates head-of-line blocking delays
    - using a lightweight data-structuring abstraction, stream, which are multiplexed

<img width="436" alt="스크린샷 2024-01-09 오후 8 06 26" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/74a3b4a1-4c67-4cba-9a97-88acaae3f906">

### 3. **MOTIVATION: WHY QUIC?**

- reducing web latency에 대한 전례없는 요구
    - web latency ⇒ 사용자의 경험 개선에 대한 장애물
    - tail latency ⇒ web platform 확장에 대한 장애물
- rapidly shifting from insecure to secure traffic, which adds delays.

**Protocol Entrenchment:**

- middleboxes have accidentally become key control points in the internet’s architecture
    - firewall, Network Address Translators(NATs)…
    
    ⇒ middlebox에서 inspect and modify하게 되었다.
    

**Implementation Entrenchment:**

- there is a need to be able to deploy changes to clients rapidly
    - continues to evolve
    - attacks on various parts of the infra
    - remain a threat
- TCP는 **OS**에 탑재되어 있다
    - 그 말은 TCP의 수정은 OS의 upgrade가 요구되어지게 된다.
    

**Handshake Delay:**

- the **cost**s of layering have become increasingly visible with increasing latency demands on the HTTPS stack.
- TCP connection은 보통 적어도 한번의 round-trip delay가 발생
- TLS는 2번의 round-trip delay가 발생한다.
    
    ⇒unnecessary handshake round trip은 가장 큰 영향을 주게 된다.
    

**Head-of-line Blocking Delay:**

- to reduce latency and overhead costs of using multiple TCP connections
    - HTTP/1.1 : limiting the number of connections
    - HTTP/2 : multiplexes multiple object and use a single TCP connection to any server
    
    ⇒ 그러나 손실된 TCP segment에 대한 재전송과 통신 프레임을 제어하는 것을 방지
    
- QUIC은 UDP를 사용하여 재전송을 하지 않고, 네트워크 제공자에 대한 의존성을 줄인다.(배포권이 있다)
    - 전에는 배포를 하기 전에 web server, client, transport stack, middlebox 수정이 필요했다.

### **3. QUIC DESIGN AND IMPLEMENTATION**

- several goals
    - deployability, security, and reduction in handshake and head-of-line blocking delays.
    - combine its cryptographic and transport handshakes to minimize setup RTTs.
    - mulitplexes multiple requests/responses over a single connection by providing each with **its own stream.**
        
        ⇒어느 response도 다른 response를 block 할 수 없다.
        
        ⇒middleboxes에 의한 변조를 막기 위해 encrypts and authenticates
        
        - improve loss recovery by using **unique packet numbers** to avoid retransmission ambiguity and by using **explicit signaling in ACKs** for accurate RTT measurements.
            
            ⇒5-tuple(IP/Port) 대신 connection ID를 통한 ip address migrate
            
            ⇒slow receiver’s buffer의 데이터 양을 제한하는 flow control 제공
            
            ⇒ensure that a single stream does not consume all the receiver’s buffer by using per-stream flow control limits.
            

**3.1 Connection Establishment** 

QUIC relies on a combined cryptographic and transport hand-shake for setting up a secure transport connection

- handshake 성공시
    
    → client caches information about the ***origin***
    
- 후속 connection to the same origin
    
    → client can establish an encrypted connection with no additional round trips
    
    → data can be sent immediately following the client handshake packet
    
- a dedicated reliable stream
    
    → summarizes the mechanics of QUIC’s cryptographic handshake
    

**Initial handshake:**

- 클라이언트와 서버 모두 정보가 없다
1. client sends an ***inchoate client hello***(CHLO) message
2. server sends a reject(REJ) message
    1. **a server config** that includes the server’s long-term Diffie-Hellman public value
    2. **certificate** chain authenticating the server
    3. a **signature** of the server config (using the private key)
    4. ***a source-address token (an authenticated-encryption block)***
3. client sends a source-address token back to the server in later handshakes
    
    →demonstrating ownership of its IP address
    
4. client sends a complete CHLO, containg the client’s ephemeral Diffe-Hellman public value.

**Final (and repeat) handshake:**

All keys for a connection are established using Diffie-Hellman.

→after sending a complete CHLO, the client is in possession of initial keys for the connection

결국 이 시점부터 client는 서버에게 자유롭게 데이터를 보낼 수 있다.

0 RTT를 달성하기 위해 client는 서버의 응답을 기다리기 전에 initial key로 암호화된 데이터 전송을 시작해야함

- handshake is successful, server returns a server hello(SHLO) message.
    
    → encrypted using the initial keys
    
    → contains the server’s ephemeral Diffie-Hellman public value
    
    → server switches to sending packets encrypted with the forward-secure keys.
    
- SHLO를 client가 받으면
    
    → the client switches to sending packets encrypted with the forward-secure keys.
    

<img width="438" alt="스크린샷 2024-01-09 오후 8 06 53" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/8a581997-e18d-403d-9003-988f60ae6fef">

QUIC’s cryptography provides two levels of secrecy:

- initial client data is encrypted using **initial keys**
    
    → provide protection  analogous to TLS session resumption with session ticket
    
- subsequent client data and all server data are encrypted using forward-secure keys.
- The client caches the server config and source-address token
    
    → 이후 전송에서 서버의 응답을 기다리지 않고 전송이 가능하다.
    
- source address token, server config 가 만료되거나 인증이 바뀐다면…?
    - server는 REJ message를 client에게 전송한다.

**Version Negotiation:**

QUIC clients and servers perform version negotiation during connection establishment to avoid unnecessary delays.

1. client proposes a version to use for the connection in the first packet of the connection and encode the rest of the handshake using the proposed version
2. server does not speak the client-chosen version
    
    → forces version negotiation by sending back a version negotiation packet to the client carrying all of the server’s supported versions, causing a round trip of delay before connection establishment.
    
3. 만약 speak 한다면 round-trip latency를 제거할 수 있다.

downgrade attacks를 막기위해, version은 최종 키를 생성하는 동안 client와 server의 key-derivation function에 입력된다.

**3.2 Stream multiplexing**

Applications commonly multiplex units of data within TCP’s single- bytestream abstraction.

- QUIC supports multiple streams within a connection
    
    → avoid head-of-line blocking due to TCP’s sequential delivery
    
    → lost UDP packet only impacts those streams whose data was carried in that packet
    
- a lightweight abstraction stream
    
    → provide a reliable bidirectional bytestream
    
    →identified by stream IDs (avoid collisions, odd IDs→client, even IDs → server)
    
- stream 생성은 첫번째 바이트를 보낼때 이루어 진다.
    
    → stream closing은 스트림 frame에 “FIN”을 설정한다.
    
- client와 server 둘중 어디든 더 이상 연결이 필요가 없다면..?
    
    → QUIC **연결을 끊지 않고** 취소할 수 있다.
    
    <img width="446" alt="스크린샷 2024-01-09 오후 8 07 22" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/501229ad-1d3a-4549-8112-fdd797890a6f">
    
- A QUIC packet is composed of a common header followed by one or more *frames*
- QUIC stream multiplexing 은 encapsulating stream data in one or more stream frames
    
    → single QUIC packet can carry stream frames form multiple stream
    

**3.3 Authentication and Encryption**

- Flag, Connection Id, Version Number, Diversification Nonce, Packet Number
    
    → QUIC packet header outside : packet routing, decrypting에 사용
    
    - Flag : encode the presence of the Connection ID field and length of the Packet Number field, and must be visible to read subsequent field.
    - Connection ID : routing, identification purpose (load balancer에 사용)
    - Version number, diversification nonce : present in early packets
        - server generates the diversification nonce and sends it to the client in the SHLO packet to add entropy into key generation
        - server and client use the packet number as a per-packet nonce, which is necessary to authenticate and decrypt packets.
    - Packet Number
        - placed outside of encryption cover to support decryption of packets
- Version Negotiation packet 같은 암호화되지 않은 패킷
    - included in the derivation of the final connection keys
- Reset packets are sent by a server that does not have state for the connection, which may happen due to a routing change or due to a server restart.

결국 서버는 connection’s keys를 가지지 않고, reset packets은 비암호화, 비인증된 상태로 보내진다.

**3.4 Loss Recovery**

TCP seq num은 reliability 와 represent the order를 가능하게 한다.

→ retransmission ambiguity 문제가 발생한다. (동일한 seq num의 패킷을 보내기 때문에)

→ 일반 전송인지 재전송인지 결정할 수 없다.

→ 재전송된 패킷의 loss는 보통 값비싼 timeout으로만 재전송을 감지하게 된다.

QUIC에서는 **new packet number**와 재전송 데이터를 담고 있다.

→ retransmission ambiguity 해결 (원래 전송인지 재전송인지 구별할 매커니즘 필요하지 않음)

→ stream offsets in stream frames : used for delivery ordering

→ represents and explicit time-ordering (simpler and more accurate loss detection)

QUIC acknowledgments explicitly encode the delay between the receipt of a packet and its acknowledgment being sent.

→ packet number의 monotonically-increasing

→ 정확한 RTT estimation을 가능하게 한다. (loss detection에 도움)

→ delay-sensing congestion controllers에게도 도움을 준다.

→ 최대 256 ACK block을 지원하여 재정렬 및 손실에 대한 복원력이 더 뛰어나다.

더 간단하고 효과적인 매커니즘을 가지고 있다.

**3.5 Flow Control**

QUIC : connection-level flow control, stream-level flow control

→ limit the aggregate buffer that a sender can consume at the receiver across all stream

→ limit the buffer that a sender can consume on any given stream

credit-based flow-control :

→ QUIC receiver advertises the absolute byte offset within each stream up to which the receiver is willing to receive data

→ 특정 스트림으로 보내고, 받고, 전달되면 점진적으로 window update frames를 보내 더 많은 데이터를 스트림을 통해 보내도록 허락한다.

**3.6 Congestion Control**

특정 congestion control algorithm에 의존하지 않으며, pluggable interface를 가지고 있는다.

→ CUBIC 사용

**3.7 NAT Rebinding and Connection Migration**

QUIC connection은 64-bit Connection ID로 구별된다.

→ client의 IP, port의 변경에도 연결이 유지되도록 한다.

→ NAT timeout, rebinding, client changing network connectivity to a new IP address.

→ NAT rebinding 문제를 쉽게 제거한다.

→ client-initiated connection migration is a work in progress with limited deployment at this point.

**3.8 QUIC Discovery for HTTPS**

client does not know a priori whether a given server speaks QUIC.

1. client makes an HTTP request, it sends the request over TLS/TCP
2. server advertis QUIC support by including and “Alt-Svc” header in their HTTP responses.
3. this header tells a client that connections to the origin may be attempted using QUIC
4. client can now attempt to use QUIC
- 후속 HTTP request는 TLS/TCP와 QUIC과 경쟁하지만 QUIC을 더 선호하게 된다. (딜레이가 짧다)
- 먼저 연결에 성공한 것이 요청에 사용된다.

### 5**. INTERNET-SCALE DEPLOYMENT**

**5.1 The Road to Deployment**

- QUIC support was added to Chrome in June 2013.
- 2014, turned it on via Chrome’s experimentation framework for a tiny fraction of users
- 2017, turned it on for almost all users of Chrome and the Android Youtube app

**Unencrypted data in 0-RTT requests:**

→ could result in 0 RTT requests being sent unencrypted in an exceedingly rare corner case.

→ 잠시 중단 후에 다시 QUIC traffic이 재건되었다.

**Increasing QUIC on mobile:**

- Youtube에서 September 2016부터 QUIC을 사용하기 시작했다.

**5.2 Monitoring Metrics: Search Latency**

search latency : the delay between when a user enters a search term into the client and when all the search-result content is generated and delivered to the client

<img width="444" alt="스크린샷 2024-01-09 오후 4 54 43" src="https://github.com/RakunKo/Network/assets/145656942/d7db13d1-8ec1-4ada-857a-e7aebd09c1c5">

- changes in infrastructure and to a client configuration bug (1)
- the client configuration bug had been fixed, and search latency improved (2)
- the deployment of UDP-proxying at their ***REL***s (3)
    - restricted edge locations(RELs) and UDP *proxying*
    - search latency increased form about 4% to over 7%

### **6. QUIC PERFORMANCE**

<img width="446" alt="스크린샷 2024-01-09 오후 5 13 14" src="https://github.com/RakunKo/Network/assets/145656942/e142df90-6865-49da-9ecb-9414a2684a9a">

**6.2 Transport and Application Metrics**

*Handshake latency :* the amount of time taken to establish a secure transport connection

→TLS/TCP 에서는 TCP와 TLS handshake의 시간이 모두 포함된다.

<img width="647" alt="스크린샷 2024-01-09 오후 5 31 09" src="https://github.com/RakunKo/Network/assets/145656942/81675e8e-2850-4dec-a52a-2eaf07bf577d">

- QUIC 0-RTT handshake : latency is measured as 0ms
    
    → fixed latency cost of 0-RTT handshake로 인해 RTT에 대해 매우 둔감하다.
    
    → 증가는 not success fully connect in 0-RTT로 인한 증가
    
    → higher resilience to loss in general and lower latency for short connection(적은 RTT)
    
- TCP/TLS 에서는 RTT가 증가할 수록 handshake latency가 선형적으로 증가하게 된다.

1. networking remains just one constituent of end-to-end app measures.
    1. handshake latency는 search latency와 video latency에 20% 미만을 차지한다.
2. the sensitivity of application metrics to networking changes depends on the maturity of the application.
    1. highly optimized and mature Web applications → improving end-to-end metrics in them is difficult

**6.3 Search Latency**

search latency : the delay between when a user enters a search term and when all the search-result content is generated and delivered to the client by Google Search

<img width="446" alt="스크린샷 2024-01-09 오후 5 57 17" src="https://github.com/RakunKo/Network/assets/145656942/953040a5-20d3-4137-915a-c6a3ed01ff2e">

- 대부분의 handshake improvements는 0-RTT handshake에서 비롯된다.
    
    → 데스크톱의 QUIC 연결 중 약 88%는 0-RTT handshake (TLS/TCP 보다 2-RTT를 절약)
    
    → 나머지 연결들도 1-RTT의 이점을 얻게 된다.
    
- QUIC’s improved loss recovery may also contribute to Search Latency improvement at higher RTT.
- moblie은 desktop보다 QUIC connection 0-RTT handshake가 68% 달성되어 이점이 더 낮다.

successful 0-RTT handshake 

- a valid server config and a valid source address token in a client’s handshake message

1. mobile users switch networks, their IP address changes, which invalidates the source-address token cached at the client.
2. different server configurations and kets are served and used across different data centers.
    
    → client may hit a different data center where the servers have a different server config than that cached at the client
    

⇒ contributes to about half of the reduction in successful 0-RTT handshakes

**6.4 Video Latency**

video latency : the time between when a user hits “play” on a video to when the video starts playing

→to ensure smooth playbacks, video players typically buffer a couple seconds of video

- 평균 85%의 QUIC connections for video playback on desktop receive the benefit of a 0-RTT handshake, 나머지는 1-RTT의 benefit
- QUIC loss recovery improvements may help Video Latency as client RTT increases.
- 모바일 기기의 이점이 데스크탑 보다 작다.
    
    → 평균적으로 약 65%의 0-RTT handshake가 일어난다.
    
    → 앱은 사용자가 비디오를 탐색하고 검색하는 동안 백그라운드에서 비디어 서버에 대한 연결을 설정하여 handshake 비용을 숨긴다. (오히려 benefit을 줄인다)

**6.5 Video Rebuffer Rate**

To ensure smooth playback over variable network connections, video players typically maintain a small playback buffer of video data.

- 만약 꽉찬다면..? → video는 player가 data를 rebuffer할 수 있을 때까지 기다린다.
- video rebuffer rate, rebuffer rate → 데이터를 rebuffer하기 위해 재생중에 중지되는 시간의 비율
    
    →(rebuffer Time)/(Rebuffer Time + video play Time)
    
- handshake latency와 무관하다.
- loss-recovery latency에 영향을 받는다 → missing data can stall video playback.
- connection’s overall throughput에 영향도 받는다.

**Loss-Recovery Latency:**

video player에선 2개의 TCP connection을 사용한다.

→ 2개의 TCP connection을 사용하면 loss detection이 느려진다 (다른 connection은 loss를 알 수 없다)

QUIC의 loss-recovery improvements는 rebuffer rate가 더 천천히 증가하게 한다

**Connection Throughput:**

Connection throughput is dictated by a connection’s *congestion window*

 → estimated by the sender’s congestion controller, and by its receive window(computed by the receiver’s flow controller)

- The default initial connection-level flow control limit advertised by a QUIC client is 15MB
    
    →which is large enough to avoid any bottlenecks due to flow control
    
- rebuffer rate can be decreased by reducing video quality
    - QUIC palybacks show improved video quality as well as a decrease in rebuffers
- rebuffer가 일어나지 않으면 QUIC과 TCP는 같다.
- QUIC’s benefits are higher whenever congestion, loss, and RTTs are higher.

**6.6 Performance By Region**

differences in access-network quality and distance form Google servers result in RTT and retransmission rate variations for different geographical regions.

- 낮은 평균 RTT와 network loss를 가진 한국은 별 차이가 없다.
- india 같이 높은 평균 RTT와 network loss를 가진 국가에서는 큰 이익을 얻을 수 있다.
    
    → higher average RTT and higher network loss에서 얻을 수 있는 이익이 더 크다.
    

**6.7 Server CPU Utilization**

QUIC’s server CPU-utilization was about 3.5 times higher than TLS/TCP

- cryptography
    - to reduce, employed a hand-optimized version of the ChaCha20 cipher favored by mobile client
- sending and receiving of UDP packets
    - to reduce, used **asynchronous** packet reception form the kernel via a memory-mapped app ring buffer (Linux’s packet_rx_ring)
- maintaining internal QUIC state
    - to reduce, rewrote critical paths and data-structures to be more cache-efficient
- 이러한 최적화로 CPU 웹 트레픽을 제공하는 코스트를 감소시켰다. (TLS/TCP에 약 2배)

**6.8 Performance Limitations**

QUIC’s performance can be limited in certain cases.

- **Pre-warmed connections:**
    - when app hide handshake latency by performing handshakes proactively, these app receive no measurable benefit from QUIC’s 0-RTT handshake.
- **High bandwidth, low-delay, low-loss networks:**
    - plentiful bandwidth, low delay, low loss rate → little gain and occasionally negative performance impact.
    - 오히려 TCP보다 성능이 나빠질 수도 있다.
- **Mobile devices:**
    - 모바일 사용자가 얻는 이점은 데스크탑 사용자보다 낮다.
    - CPU의 성능차이, 콘텐츠 제한시 전송 최적화의 영향이 낮아짐

### **7. EXPERIMENTS AND EXPERIENCES**

**7.1 Packet Size Considerations**

- <img width="664" alt="스크린샷 2024-01-10 오후 1 29 27" src="https://github.com/RakunKo/Network/assets/145656942/a1dd7bf5-b534-4988-aa11-bad061395c36">
- 1450byte UDP payload size부터 급격하게 unreachability가 증가한다.
    - UDP 와 IP header를 더하면 1500byte인 MTU를 넘어서기 때문
- chose **1350bytes** as the default payload size for QUIC

**7.2 UDP Blockage and Throttling**

- successfully used for 95.3% of video clients attempting to use QUIC.
- 4.4% of clients are unable to use QUIC
    - meaning that QUIC or UDP is blocked or the path’s MTU is too small.
    - in corporate networks, behind enterprise firewalls
- 0.3% of users are in networks that seem to rate limit QUIC and/or UDP traffic

**7.3 Forward Error Correction(FEC)**

- uses redundancy in the sent data stream to allow a receiver to recover lost packets without an explicit retransmission
    - XOR-based FEC : low computational overhead, simple to implement, avoids latency associated with schemes that require multiple packets to arrive before and can be processed
- while retransmission rates decreased measurably, FEC had statistically insignificant impact on Search Latency and increased both Video Latency and Video Rebuffer Rate for video playbacks.
    - <img width="676" alt="스크린샷 2024-01-10 오후 1 49 31" src="https://github.com/RakunKo/Network/assets/145656942/d58745b2-4863-4e72-b3aa-09418d48261c">
    - sending additional FEC packets simply adds to the bandwidth pressure.
    - he number of packets lost during RTT-long loss epochs in QUIC to see if and how FEC might help
        - 이점이 30% 미만으로 제한된다.
- 결국 QUIC에서 XOR-based FEC 퇴출

**7.4 User-space Development**

- not as memory-constrained as the kernel
- not limited by the kernel API
- can freely interact with other systems in the server infra
    - extensive logging and integration with a rich server logging infrastructure
    - logging을 통해 cubic의 정지 버그를 발견
        - 버그를 수정해 30% 재전송률 감소, CPU utilization 17% 향상, TCP 재전송률 20% 감소
- Due to these safeguards and monitoring capabilities, able to iterate rapidly on deployment of QUIC modifications.
- <img width="659" alt="스크린샷 2024-01-10 오후 1 57 46" src="https://github.com/RakunKo/Network/assets/145656942/35efc94b-895d-4e7c-84bb-e9e96035d4c8">
- 실제로 점점 빠르게 QUIC의 수정에 대한 배포가 이루어지고 있다.

**7.5 Experiences with Middleboxes**

QUIC encrypts most of its packet header

→ a few fields are left unencrypted, to allow a receiver to look up local connection state and decrypt incoming packet

- 때로는 1-bit의 변화만으로 다양한 Middlebox에서 문제가 발생할 수 있다.
- encryption은 middlebox에서 사용되면 안되는 비트가 사용되지 않도록 보장하는 유일한 수단.
