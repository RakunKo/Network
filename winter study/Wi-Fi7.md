access to wireless connectivity is regarded as a must in our globalised society

- WiFi carries most of the global data traffic in an ever-expanding variety of application
- More and more households will accommodate smart-home appliances besides 8K displays and virtual reality gadgets, turning into dense environments with many concurrently connecting devices.

Wi-Fi 7 generation based on IEEE 802.11be Extrmely High Throughput(EHT)

multi-access point(AP) coordinated beamforming(CBF)

- improved peak throughput with boosted network efficiency, lower latency, and higher reliability

WiFi는 2.4와 5 GHZ 2개의 대역폭을 사용한다. → WiFi 6에서 새로운 채널이 생겨난다

- 5.925–7.125 GHz라는 새로운 대역폭이 생긴다.
- nealy quadruples the amount of available bandwidth
- 사용가능한 채널 수가 많고, 전파 범위가 짧다.
- 이를 사용할 수 있는 장치에는 E가 붙는다.

WiFi 7은 6보다 약 4배 빠른 AP당 최소 30Gbps를 지원, 2.4,5,6 대역에서 기존 장치와 호환 및 공존 보장

Release 1 

- Multi-Link Operation : 효율적인 작업을 위해 2.4,5,6 대역폭 모두를 사용한다.
    - multi-link device(MLD)는 위의 Logical link control(LLC) layer에 연결된 여러개의 AP 또는 station과 단일 MAC service access point(SAP)으로 정의된다
        - multi link discovery and setup
        - traffic link mapping : 모든 setup link에 TID가 맵핑된다.
        - channel access and power saving : •  **Each AP/STA of
        a MLD performs independent channel access over its
        links and maintains its own power state.
            - 효율적인 STA 전력관리를 위해 AP는 활성화된 링크를 활용하여 다른 링크를 통한 전송을 위해 버퍼링된 데이터 표시를 전달할 ㅜㅅ 있다.
    
- Low-Complexity AP coordination
    - multi-AP coordination이 지원된다.
    - Coordinated Spatial Reuse(CSR) : low complexity, transmission opportunity(TXOP)를 획득한 공유 AP는 하나 이상의 다른 공유 AP가 적절한 전력 제어 및 링크 적응을 통해 동시 전송을 수행하도록 발 수 있다.
        - 더 많은 공간 재사용(동일한 무선 주파수를 동시에 사용) 기회를 창출하고 충돌 횟수를 줄인다.
- Direct Enhancements :
    - Support of 320MHz transmission
    - Use of higher modulation orders : 이것은 더 많은 비트를 동일한 주파수 대역폭으로 전송할 수 있다는 것을 의미합니다.
    - Allocation of multiple resource unit : 직교 주파수 분할 다중 접속 → 주파수 대역을 여러개의 작은 서브채널로 분할하여 여러 장치가 동시에 데이터를 전송하고 수신할 수 있는 기술

Release 2

- MIMO Enhancements
    - 지원되는 단일 사용자, 다중 사용자 MIMO 공간 스트림의 최대 수를 16개로 2배 늘려 결과적으로 용량도 증가
    - 여러 사용자가 동시에 동일한 주파수 채널을 사용하여 데이터를 전송할 수 있게 하여 네트워크 용량과 효율성을 증가시킨다.
    - 공간적으로 다중화된 STA의 최대 수와 STA당 공간 스트미의 최대 수를 가각 8개와 4개로 제한
        - MIMO 전처리기의 복잡성과 CSI(채널 상태 정보) 획득 overhead를 줄인다.
            - 공간 스트림의 갯수를 제한아여 AP와 장치의 계산 부담이 줄어들얻디자인이 간단해지고 전력소비가 줄어든다.
            - CSI는 MIMO전송에 필수적, 정확한 CSI를 획득하려면 수신 장치의 피드백이 필요하여 이는 신호 및 대역폭 소비 측면에서 오버헤드를 준다. 공간 스트림의 개수를 제한하여 CSI의 양을 줄여 overhead를 줄인다.
            - 
            
            # **Reduce overhead by limiting the number of spatial streams to reduce the amount of CSI**
            
- *Hybrid Automatic Repeat Request (HARQ)*
    - 잘못된 정보를 버리지 않고 이를 재전송된 단위와 부드럽게 결합하여 올바른 디코딩 가능성을 높이려고 시도한다.
        - 전송된 데이터 패킷의 수신 확인을 기다리지 않고 바로 에러 감지를 시도하며, 에러가 감지되면 일단 FEC로 복구를 시도하며, 복구가 불가능할때 재전송을 요청한다.
        - 무선통신에서 널리 사용되며 신호 대 잡음 비율이 낮은 환경에서도 높은 신뢰성 제공
- *Low-Latency Operations*
    - SFD는 최악의 대기 시간을 줄이고 안정선을 향상하는데 전념하는 프로토콜 개선 사항도 수집
- *Advanced AP Coordination*
    - *Coordinated OFDMA : TXOP를 획득한 몌sms 20MHz 채널의 배수로 주파수 자원을 인접 AP와 공유할 수 있다.*
    - *Joint single- and multi-user transmissions:연결된 STA에 데이터를 집합적으로 전송하려면 AP의 동기화 오류나 타이밍 오프셋을 제한해야한다.*
        - *적절한 backhauling이 가능하다면 손상에 대한 합리적인 가치를 고려할 떄 Joint transmissions은 이익을 제공한다.*
        - *협력한 AP에 연결된 STA와 연결되지 않은 STA 모두 CSI가 필요하므로 joint multi AP sounding scheme를 정의한다.*
        - *이러한 방식으로 AP는 사운드 프레임을 동시에 전송하고 주소가 지정된 STA는 모든 AP와 관련된 CSI 피드백을 전달한다.*
    - *Coordinated beamforming:*
        - 공간적 다중화 능력을 활용하면서 이웃하지 않은 STA와의 전파 null을 배치한다.
        - CBF는 각 STA가 단일 AP와 데이터를 송수신하기 때문에 공동 데이터 처리를 필요하지 않아, backhauling 요구를 상당히 줄일 수 있ㄷ다.
        - CBF는 복잡성을 줄이면서 상당한 처리량과 지연 시간 향상을 제공

### IV. AUGMENTING SPATIAL REUSE VIA MULTI-AP COORDINATED BEAMFORMING

***A. Parameterised Spatial Reuse in 802.11ax***

- 1. To guarantee that transmissions taking advantage of the spatial
reuse opportunity do not impact the uplink data reception of
AP1, the trigger frame includes a *PSR field*.
    - AP1이 업링크 수시능ㄹ 손상시키지 않고 수신할 수 있는 최대 간섭 수준
    - 간섭 계산을 용이하게 하기 위한 AP1의 전송 전력
- trigger frame(TF)를 받으, OBSS 장치는 PSR field에서 주어진 정보를 기반으로 전송된 전력 수준을 측정하고, 매체에 접근할 수 있는지 여부와 전송 전력을 결정한다.
- STA23은 AP1에 근접에 있기 때문에 AP1이 설정한 간섭 조건을 충족할 수 있어 채널 access를 위해 경재할 수 없다.
- PSR의 장점 : 네트워크 처리량 증가,**STA가 경합에 소비하는 시간이 줄어들기 때문에 STA 파일 처리량이 늘어남, 시간에 민감한 짧은 파일 트래픽을 가진 STA는 광대역 STA가 긴 전송을 종료할 때까지 기다릴 필요가 없으므로 대기 시간이 줄어듭니다**
- 단점 : 간섭을 제한하기 위해 전송 전력을 낮춰 처리량이 감소된다. 공간 재사용 기회에도 access 못하는 문제

***B. Coordinated Beamforming in 802.11be***

- *Multi-AP Coordination :***두 개 이상의 협력 AP가 두 가지 목적으로 제어 프레임을 교환합니다.**
- 여러개의 AP가 서로 협력하여 최적의 데이터 전송 경로를 찾는다.
- AP는 서로 채널 상태 정보(CSI)를 공유하여 각 사용자 기기(STA)의 채널 상태를 파악합니다. 이 정보를 바탕으로 AP는 각 STA에게 가장 적합한 데이터 전송 경로를 선택합니다.
    - *Coordination set establishment and maintenance:*
        - AP는 OBSS STA와 통신해야 해야한다( 특정 공간에 위치에 null을 배치하는데 필요한 CSI를 획득한다.)
        - **BSS 간 조정 세트는 협업 AP 간에 정의되며, 여기에는 CBF 전송에 참여하는 모든 AP 및 STA의 ID가 포함되어야 합니다.**
    - *Dynamic coordination of the subsequent spatial reuse opportunity:*
        - **AP1이 TXOP를 획득하면 들어오는 업링크 트리거 전송을 광고해야 하며 조정 세트의 장치와 함께 후속 CSI 획득 및 데이터 통신 단계에 어떤 STA가 포함될 것인지 결정해야 합니다.**
        - **AP2는 AP1이 보낸 동적 조정 프레임에 응답하여 어느 STA가 안전한 공간 재사용 기회를 부여받음으로써 가장 많은 이점을 얻을 수 있는지를 나타냅니다.**
- *CSI Acquisition:* **AP1과 AP2 모두 관련 내부 BSS 및 OBSS 장치에서만 CSI를 획득합니다**
    - 이 단계에서는 STA가 AP로부터 CSI를 받습니다. STA는 AP에 CSI 요청 메시지를 보내고 AP는 CSI 응답 메시지를 보내 응답합니다. CSI는 채널의 감쇠, 간섭 등의 정보를 포함하며, 이 정보는 데이터 전송의 효율성을 높이는 데 사용됩니다.
- *Data communication:* **STA22 및 STA23의 공간 재사용 전송이 불리한 조건에서 적시에 성공할 가능성을 훨씬 더 높여줍니다**
    - 그들의 최대 전송 전력을 사용하여 공간 재사용 전송 기회를 얻는다
    - **AP2는 STA21, STA22 및 STA23으로부터 업링크 전송을 수신하는 동안 STA11 및 STA12에 의해 생성된 들어오는 간섭을 억제할 수 있습니다.**
