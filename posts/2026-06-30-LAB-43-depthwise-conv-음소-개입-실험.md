# [ LAB ] 43. depthwise conv 음소 개입 실험

- 작성일: 2026-06-30
- 원문: https://velog.io/@xorms/LAB-43.-depthwise-conv-음소-개입-실험
- 태그: ASR, fine tunning, gemma, 인코더

---

이전 audio_tower 학습에서는 attn(q/k/v/post) + FFN(ffw_1/2) 해당 부분을 학습하여 발음 오인식 문제를 해결하려고 했지만 큰 효과를 보지 못했다.

본 실험에서는 depthwise_conv1d만 격리에서 학습시키려한다.

현재 데이터는,

- train 2485개
- val 275개
- test 828개

## 실험 개요

이며 본 실험에서 확인하려는 것은 발음 오인식 대표 샘플 6가지에 대한 학습 효과이다.
이 때, 발음 6개로 학습시키면 모델은 음소 변별을 잘하게 되는 것이 아닌 6문장 정답을 암기해버리기 때문에 full train 학습(train 2485개 전부 사용)으로 진행할 것이다.

또한 발음 오인식 대표 샘플 6개는 held-out eval 즉, 따로 떼어둔다.
학습에 한 번도 보여주지 않은 발음 6개를 학습이 끝난 모델에 처음 들려주고 CER을 잴 것이다.

본격적인 모델 학습을 시작하기 전에 train 2485 데이터셋에 held-out eval을 수행하기 위해 6개 샘플이 train 데이터에 들어있는지 확인하였다.

```
발음 6개: 6
  [0] 세균전에 대비해  | 000396.wav
  [1] 살림이 없어요  | 000029.wav
  [2] 거부감은 없어요  | 000154.wav
  [3] 동거 막 하고  | 000217.wav
  [4] 좋은 기회야  | 000260.wav
  [5] 황경택 씨도  | 000263.wav

train 섞임: 0 개 → held-out 깨끗
발음6 세션: {'S000017': 5, 'S000010': 1}
```

train 데이터셋에 포함되어 있지 않은 것을 알 수 있다.

## 학습 설계

이전 실험(audio_tower LoRA 학습)에서는 오디오와 텍스트의 연결 지점인 audio_embed를 동결한 상태에서 audio_tower를 학습해서 결과를 지켜보았다.

본 실험도 동일하게 audio_embed 통로를 동결로 깔되, 이번엔 귀 안에서 여태 건드리지 않은 국소 시간 연산자(depthwise_conv1d)만 full-FT한다.

평가는 ft(통로만)·after(통로+attn/FFN, nb23) 두 대조군과 같은 축에서 비교한다

```
conv 재해동 텐서: 12
학습 param 수: 61440

전체 trainable 텐서: 12
  그 중 lora(통로) trainable: 0 → 동결 유지
  그 중 conv trainable: 12
```

- 전체 trainable 텐서: 12
  - -> 모델 전체에서 학습으로 값이 바뀔 수 있는 텐서가 딱 conv 12개 뿐 = 변수가 격리됨
- lora(통로) trainable: 0
  - -> audio_embed 부분에 lora 어댑터가 깔려있지만 동결되어 학습되지 않는다.

### 학습량

```
train/val: 2485 / 275
유효 배치: 16

trainable 텐서: 12
  audio_tower conv: 12
  embed_audio(통로): 0
remove_unused_columns: False
총 스텝(약): 310
GPU used: 15906 MiB
```

- 스텝: 모델 weight를 한 번 업데이트하는 단위, 데이터 한 묶음(배치) 보고 conv 값을 한 번 조정
- 유효 배치: 16은 한 스텝에서 보는 샘플 수가 16개

따라서 465 스텝은 train 데이터 2485개를 16개씩 묶으면 155묶음이 나오고(1 에폭) 이걸 2 에폭(이전 audio_tower 학습 실험 때와 동일 상황을 가정하기 위해)을 돌리면 310스텝

## 학습

### 1. 학습

위와 같이 설계한 대로 학습을 진행하였다.

Training Loss와 Validation Loss 둘 다 거의 변화가 없음
-> conv가 거시 출력을 바꿀 여력이 작음
동일 조건의 audio_tower(attn(q/k/v/post) + FFN(ffw_1/2)) 학습 실험(변화 0.055)과 비교하면 학습량 부족이 아닌 모듈 표현력 한계

### 2. conv 학습 모델로 발음 6개 샘플 추론 + ft(audio_embed) + audio_tower 학습 + conv 학습 CER 비교

conv 열이 ft 열과 전부 동일 -> conv가 출력을 사실상 바꾸지 않음
표현층 개입이 음소에 무력하다라는 또 다른 증거

### 3. 재학습

이전 학습에서 conv loss가 거의 움직이지 않아서 학습량 및 강도 부족이 아닐까하는 생각을 하였다.

그래서 학습량만 4배로 늘려 동일 2 에폭으로 재학습을 시켰다.

그 결과,

국소 시간 연산자를 학습하고 강도까지 4배로 올려도 음소 오인식은 한 글자도 바뀌지 않음
conv 개입량이 사실상 0, 학습 강도와 무관

연구 맥락으로 살펴보면
audio_embed, 전역 attention-FFN에 이어 국소 conv까지 음소 무력
-> 인코더 안 모든 매커니즘이 가벼운 개입으로 음소 복구 불가
-> 한계는 인코더 용량으로 추정

## 결론

인코더 안 모든 메커니즘(audio_embed, audio_tower(전역 attn.FFN.국소 conv))이 가벼운 개입으로 발음 오인식 문제가 해결되지 않음
즉, 한계는 모듈 선택이 아닌 인코더 용량이며 풀파인튜닝 혹은 대규모 데이터로 검증 필요성이 있다고 생각한다.
