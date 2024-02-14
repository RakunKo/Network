## **See Through Walls with Wi-Fi!**

### 1. INTRODUCTION

- 불투명한 장애물을 꿰뚫어 보는 것은 레이더나 sonar imaging과 비슷하다
- RF 신호의 일부는 물체나 사람에게 반사되어 다시 돌아온다 → 닫힌 방 내부에 신호를 각인하여
- 반사를 포착하여 벽뒤의 물체를 인식할 수 있게된다.
    - 벽을 2번 통과한 신호의 세기는 3~5배 정도 약해져 인식하기 너무 어렵다
    - 추가로 벽의 반사가 다른 반사의 세기가 다른 반사에 비해 너무 강하다
        
        → *Flash Effect* : 이 신호는 ADC를 압도하여 벽 뒤에 있는 물체의 반사에 의한 신호를 감지하지 못한다
        
- 낮은 대역폭, 파워, 간소화된 기술로 벽을 넘어 볼 수 있게 하는 것이 목표
    - Wi-Vi : 20Mhz-wide WiFi 채널로 제한하며, Flash Effect를 해결하기 위해 초광대역 솔루션을 피한다
    - 3개의 작은 MIMO radio 안테나를 사용한다.
        - 2개의 전송 안테나와 1개의 수신 안테나

- MIMO는 다중 안테나 시스템에서 특정 수신 안테나의 신호의 *nulling*(합이 0)이 되도록 전송을 인코딩 가능
- *nulling*을 통해 벽을 포함한 정적인 물체의 반사를 제거한다
    - 2개의 전송 안테나와 수신 안테나 간의 채널을 측정하고 수신 안테나의 신호를 null하기 위해 측정값을 사용한다.
    - 무선 신호는 중간에서 선형적으로 합쳐지므로(반사 신호 변화가 없다), 움직이는 물체의 반사만이 stage2에서 측정된다.

- Inverse Synthetic Aperture Radar(ISAR) : 표적의 움직임을 사용하여 안테나 배열을 가상화한다.

<img width="446" alt="스크린샷 2024-02-09 오후 10 23 46" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/b1da6c2e-da5f-436b-a4d3-665a17fb5d06">

- 표적이 움직이기 때문에 시간에 따른 연속적인 측정은 역으로 안테나 배열을 가상화한다.
    - 마치 움직이는 사람이 Wi-Vi를 이미징하는 것과 같다.
- 연속적인 측정 과정을 통해 Wi-Vi는 표적의 공간적 방향을 인식 가능하다.

- Wi-Vi의 움직임을 추적하는 능력을 사용하여 제스처 기반 통신 채널을 가능하게 한다.
    - 표적은 Wi-Vi 수신기가 없더라도 제스처로 통신할 수 있다.

### 4. ELIMINATING THE FLASH

<img width="447" alt="스크린샷 2024-02-09 오후 10 39 39" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/54afe9bf-95d1-4c5b-896a-7934a59579cd">

- Table 1에선 건축 자재에 따른 단방향 RF 감소 정도를 보여준다.
- 벽을 통과하는 시스템은 장애물을 두번 통과해야 하므로 감쇠또한 2배로 늘어난다.

- 실제 반사된 신호는 반사 계수와 단면에 따라 달라지기 때문에 상당히 약해진다.
    - 벽은 매우 높은 반사 계수를 가지고 있다.
- 전송 안테나의 신호를 수신 안테나에서 바로 받는 것을 제거해야한다.
    - nulling을 통해 벽 반사와 직접적인 신호를 제거하여 표적 신호에 대한 민감도를 상승시킨다

### **4.1 Nulling to Remove the Flash**

<img width="439" alt="스크린샷 2024-02-09 오후 10 50 44" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/df075c44-6e7e-4111-969b-cdb4f200bf47">

- **Initial Nulling** : Wi-Vi는 표준 MIMO nulling을 수행한다.
    - 첫번째 전송 안테나에서 preamble $x$만을 전송하고 수신 안테나에서 $y = h_1x$를 수신하게 된다.
        - $h_1$은 첫번쨰 전송 안테나와 수신 안테나 사이의 채널
    - 수신자는 이 신호를 채널 
    		
    	
    	
    		
    			
    				
    					$h_1$ 
    				
    			
    		의 제거를 계산하기 위해 사용
    - 두번째 전송 안테나에서 똑같이 $x$를 전송하고 채널 $h_2$를 측정한다.
    - $p = -\hat{h_1}/\hat{h_2}$를 통해 ratio를 계산한다.
    - 첫번째 전송 안테나는 $x$를 두번째 전송 안테나는 $px$를 전송한다.
    - 결과적으로 수신기에서 인식된 채널은 $h_{res} = h_1+h_2(-\frac{\hat{h_1}}{\hat{h_2}})$ 로 근사 0을 얻을 수 있다.
    
    → 모든 정적 물체의 신호와 전송 안테나로부터 온 직접적인 신호를 제거
    

 

- **Power Boosting** : 전송된 신호의 세기를 강화한다.
    - 채널이 이미 nulling되었기 때문에 이러한 파워의 증가는 수신기의 ADC를 포화시키지 않는다.
    - 벽을 통과하는 전체 power가 증가하므로 벽 뒤 물체로 인한 신호의 SNR이 향상된다.

- **Iterative Nulling** : 정적 물체의 잔류 반사를 제거하기 위해 Power Boosting 후에 신호를 nulling 해야함
    - nulling 이후에는 합쳐진 채널만을 수신하기 때문에 각 채널을 분리해서 측정할 수 없다.
    - 다시 추정하려면 nulling을 없애야함 → ADC의 포화
    - Wi-Vi는 채널 추정치 자체보다 훨씬 작은 채널 추정치 속 오류를 사용하여 추정치를 개선
    - $\hat{h_{1}^{\prime}}=h_{1}=h_{res}+\hat{h_{1}}$ ($h_{res}$는 0에 근접, $h_1$에 대한 더 정확한 추정)
    - $\hat{h_{2}^{\prime}}=h_{2}=(1-\frac{h_{res}}{\hat{h_{1}}})\hat{h_{2}}$ ($h_2$에 대한 추정치를 얻을 수 있다)
    - $\hat{h_1}, \hat{h_2}$가 수렴할 때까지 Wi-Vi는 이 과정을 반복한다.

### 5.1 Tracking a Single Human

- Wi-Vi는 두가지 이유로 antenna array를 사용하는 것을 피한다
    - 좁은 antenna array beam으로 좋은 해상도를 얻기 위해선 큰 antenna array와 많은 antenna가 요구
    - 이미 Wi-Vi가 MIMO nulling을 통해 flash effect를 제거해서 추가된 receive antenna 또한 nulling이 또 요구된다.
- *Inverse Synthetic Aperture radar(ISAR)*을 사용해 antenna array를 가상화 한다.
    - 대상이 이동함에 따라 각 지점에 receiver antenna가 있는 것처럼 공간의 연속적인 위치에서 신호를 샘플링 한다.
    - 많은 antenaa 없이도 공간에서 수신하는 것을 시간에 효과적으로 수신한다.
    - 연속적인 시간 샘플을 공간 샘플로 처리함으로써 Wi-Vi는 antenna array를 가상화하고 이를 이용해서 벽 뒤의 움직임을 추적할 수 있다.

- $y[n]$은 시간 지점 $n$에서 Wi-Vi로 부터 받은 신호 샘플
- 대상을 추적하기 위해 **spatial angle** θ를 측정하게 된다.
- $A[ θ, n]$은 시간 $n$에 따른 공간 방향  ****θ를 측정하는 함수
- flash effect를 제거하고 $h[n]=y[n]/x[n]$을 통해 채널을 얻는다.

<img width="452" alt="스크린샷 2024-02-12 오후 1 35 02" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/bc5b5c3d-f022-4545-8712-e31428898b3b">

- $w$크기의 antenna array를 생성한다 → Figure 2
- $A[θ,n]=∑_{i=1}^n​h[n+i]e^{j\frac{2\Pi}{\lambda}i\Delta sinθ}$ 식을 통해 계산을 진행한다.
- $\lambda$는 wavelength, $\Delta$는 배열의 연속 안테나 사이의 공간적 분리

- $\Delta$를 추정하기 위해 $\Delta = vT$ (T는 Wi-Vi의 샘플링 기간이고, V는 운동 속도이다)
- 좁은 방속에서 가질 수 있는 속도는 한정적임으로 기본 v값으로 1m/s(편안한 걷기)를 설정
- 정확한 속도를 모르기 떄문에 사람의 위치를 정확히 알 순 없지만 상대적인 움직임을 추적할 수 있다.

<img width="441" alt="스크린샷 2024-02-12 오후 1 50 13" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/52ef24f8-7ab5-4c2e-8f8a-076a397c5890">

- 2개의 선중 zero line은 정적 요소의 평균적인 에너지를 나타낸다.
- curved line은 각도가 변하는 것을 나타낸다.
- 0초에서 사람은 Wi-Vi를 향해 다가오고 있고, θ가 0이 되었다가 점점 멀어지게 되면서 음수로 바뀐다.
- 3초 이후에는 몸을 돌려 안쪽으로 움직이기 시작하여 각도가 0이 되지만 상대적으로 멀어져 신호가 약하다

### **5.2 Tracking Multiple Humans**

- 각 사람은 분리된 antenna array를 가상화하게 된다.
    - 수신된 신호는 움직이는 사람의 신호가 중첩되어 있다

- 상관된 중첩 신호를 풀어 내는 것이 주요 관점 → Smoothed MUSIC algorithm
    - $A'[θ,n]$이라 불리는 특정 방향에서 받은 신호의 세기를 계산한다 → EQ.4와 비슷하다
    - 주어진 안테나 배열을 통해 상관 matrix $R[n] =E[hh^H]$를 계산한다
    - $R[n]$에서 고유 분해를 수행하여 잡음을 제거하고 가장 강한 고유 벡터를 유지 → 여러명이면 고유 벡터도 여러개
    - 공간 백터 matrix인 $U[n]$을 2개의 부분으로 나눈다
        - $U_s[n]$→ 신호 공간, $U_N[n]$→ 노이즈 공간
        - null인 공간에 모든 방향 θ을 투영하고 그 반대 방향을 취한다 →움직히는 사람에 해당하는 θ 급등
    - Smoothed MuSIC algorithm은 추가적인 단계를 수행하는데 다른 엔티티에서 도착하는 신호의 상관관계를 해제하기 위한 것이다.

<img width="454" alt="스크린샷 2024-02-12 오후 2 17 10" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/dd79705f-1afe-481c-903d-93a4de5e9378">

- 0.5초에선 두 사람 모두 Wi-Vi에서 멀어지고 있다.
- 1초와 2초 사이에선 1명의 움직임만 감지되며, 이는 한 사람이 정지해있거나, 너무 멀리 떨어져 있음
- 2초와 3초 사이에선 1명은 멀어지고 1명은 가까워지고 있다는 것을 알 수 있다.
- 다중 인식이 아니여도 단일 인식에서도 사용 가능하다.

- Wi-Vi가 자동적으로 사람의 수를 감지하게 하기 위해 machine learning을 사용한다.

### 6.1 Gesture Encoding

1. 0이든 1이든 각 비트에 끝에서 사람은 시작과 동일한 초기 상태로 돌아와야한다
2. 제스처는 인간이 쉽게 수행하고 구성할 수 있도록 단순해야 한다.
3. 제스처는 머신러닝 분류기와 같은 정교한 decoder없이도 쉽게 감지하고 디코딩할 수 있어야 한다.

<img width="441" alt="스크린샷 2024-02-12 오후 2 29 45" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/267cc475-62e5-4293-8397-f680f39eafcc">

- 한발짝 앞으로 오는 건 양수의 각도를, 한발짝 뒹로 가는 건 음수의 각도를 만든다
- 첫 2개의 제스처는 0, 다음 2개의 제스처는 1이다.

### **6.2 Gesture Decoding**

- Wi-Vi decoder는 input으로 $A'[θ,n]$를 받고, 맞는 필터를 적용한다

<img width="466" alt="스크린샷 2024-02-12 오후 5 10 36" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/832a27db-a899-48db-ab60-c5f33297ed22">

- Figure 5의 결과에 필터를 거치면 (a)처럼 결과가 도출되고, (b)처럼 bit 형식으로 decode 된다

### 7.3 Micro Benchmarks

<img width="965" alt="스크린샷 2024-02-12 오후 5 14 19" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/a8c15f51-f44d-4ae1-85a6-613354d77c7f">\

- (a)에선 단일 curved line만이 보인다
- (b)에선 같은 시간안에서 2개의 curved line이 보이는 것으로 보아 2명이 움직이고 있음을 알 수 있다
- (c)에선 같은 시간안에서 3개의 curved line이 보이는 것으로 보아 3명이 움직이고 있음을 알 수 있다

- 양수의 각도는 Wi-Vi를 향해 이동하는 것이고 음수의 각도는 Wi-Vi와 멀어지는 것을 알 수 있다.
- 선의 밝기에 따라 동일한 공간 각도라도 Wi-Vi와 멀수도 가까울 수도 있다
- 사람의 수가 많아질 수록(C처럼) 그것을 분리하는 것에 어려움이 생긴다.

### 7.4 **Automatic Detection of Moving Humans**

<img width="442" alt="스크린샷 2024-02-12 오후 5 26 21" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/aeb0fcbb-f951-4f81-95ef-977ad927a7fc">

- 사람이 없을때와 1명 있을 때의  spatial Variance의 차이가 1명과 2명, 2명과 3명 사이의 차이보다 굉장히 크다 → 사람이 많아질 수록 구별하기 어렵다

<img width="453" alt="스크린샷 2024-02-12 오후 5 28 12" src="https://github.com/UMC5th-PLANME/Android/assets/145656942/95627508-195b-4fcb-bd6b-f717f451a55d">

- 사람의 수가 늘어날 수록 오감지될 확률이 늘어나는 것을 볼 수 있다