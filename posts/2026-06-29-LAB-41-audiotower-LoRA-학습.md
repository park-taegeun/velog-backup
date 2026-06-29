# [ LAB ] 41. audio_tower LoRA 학습

- 작성일: 2026-06-29
- 원문: https://velog.io/@xorms/LAB-41.-audiotower-LoRA-학습
- 태그: ASR, Lora, audiotower, fine tunning

---

audio_tower의 구조를 살펴 보면

![](https://velog.velcdn.com/images/xorms/post/54fbfdef-b209-474a-bde1-849dc115fe30/image.png)

이 구조에서 본 실험은 12층 layers 부분을 학습할 것이다.

conformer 4부분의 역할을 나눠보면:

- feed_forward1/2: 각 시점 하나하나를 따로 정리(시점 간 관계 안 봄)
- self_attn: 시점들 서로의 관계를 봄
- lconv1d: 짧은 국소 패턴(이웃 몇 개)

조사해본 결과 발음 오인식의 범인이 어디인지의 직접적인 근거는 없었다.

그래서 처음부터 범인을 바로 찾기보다는 audio_tower의 LoRA 학습이 효과가 있냐를 먼저 확인해보려고 한다.

따라서 ** attn(q/k/v/post) + FFN(ffw_1/2), 12층, 96자리를 학습**시킬 것이다.

이에 대한 리스크: 96자리 = 과적합·ASR 저하 위험(audio_tower bf16 유지하는 민감 영역)

리스크에 대한 대응: disable/enable 토글 단일변수 비교 + short 262로 ASR 전반 확인 필수

---

## 학습 과정

### 1. 본 실험 핵심 구조

#### embed_audio

- 오디오와 텍스트 연결 지점
- 지난 실험에서 학습됨
- 본 실험에서는 보존(동결)

#### audio_tower

- 12층 conformer
- 음소를 인코딩하는 부분
- 본 실험에서 학습 (LoRA 96자리 = attn q/k/v/post + FFN ffw_1/2, 12층)

### 2. 구현 흐름

- embed_audio 가중치 추출 -> 디스크 백업
- embed_audio freeze + audio_tower 192만 학습 on

LoRA는 한 모듈당 가중치를 두 개로 쪼개서 붙임

- lora_A: 원래 차원 -> 저랭크로 압축
- lora_B: 저랭크 -> 원래 차원으로 복원

그래서 모듈 1개에 학습 파라미터가 A, B 한 쌍 = 96 x 2

## 3. 학습

![](https://velog.velcdn.com/images/xorms/post/4a95730c-5765-4e94-8a47-a0017f7826a1/image.png)

- val_loss 0.708(100) → 0.659(200) → 0.653(300) → 0.653(312) 단조 하락 후 수렴
- train도 0.725→0.663으로 val과 근접(과적합 없음)

---

## 실험 과정

### 결과 1

![](https://velog.velcdn.com/images/xorms/post/e6c9c6cc-43bc-43d3-86f5-b963a9988dba/image.png)

발음 오인식 경우 6개를 학습 전(통로만) / 후(통로 + 귀)로 추론해서 ref와 대조

- 정답 복귀 0/6
- "학습전 → 학습후"가 정답(ref) 쪽으로 갔나 비교

### 결과 2

![](https://velog.velcdn.com/images/xorms/post/119b3c1c-d9c7-4554-aa1c-972864fc838f/image.png)

(좌): 극단군 26개를 실패 유형별로 학습 전(통로만) / 후(통로 + 귀) CER 비교

(우): 전체/극단군/일반 short 그룹별 평균 CER 비교

- 빨강(학습 후)이 파랑(학습 전)보다 높으면 악화
- 우 패널 극단군에서 빨강이 크게 낮지만, 이는 도피 1개(전 "혹시 이 음성을..." CER 10.0 → 후 "90" CER 0.0)가 26개 평균을 끌어내린 이상치 효과
- 전체 평균 하락도 같은 이상치 때문이며 실제로는 악화 발화가 개선의 2배

## 결론

> audio_tower(귀)에 직접 LoRA를 학습했지만 발음 오인식은 0/6으로 전혀 복귀하지 않았고(결과 1), 전체적으로도 개선보다 악화가 2배였다(결과 2).
> 즉 attn+FFN 같은 선형 표현층에 LoRA를 붙이는 방식은 오디오 반응성(거시)은 강화해도 음소 변별 해상도(미시)는 끌어올리지 못한다.
> 이는 nb18(어텐션)·nb20(통로)와 함께, 표현층 개입이 음소 수준에서 일관되게 한계를 보임을 확인시켜준다. 발음 오인식의 뿌리는 LoRA가 닿지 않는 depthwise conv이거나, 초단발화 자체의 정보 부족일 가능성이 높다.
