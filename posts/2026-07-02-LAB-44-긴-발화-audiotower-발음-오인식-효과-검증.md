# [ LAB ] 44. 긴 발화 audio_tower 발음 오인식 효과 검증
- 작성일: 2026-07-02
- 원문: https://velog.io/@xorms/LAB-44.-긴-발화-audiotower-발음-오인식-효과-검증
- 태그: ASR, LLM, audiotower, 환각
---
이전에 audio_tower 학습이 발음 오인식에 효과가 없다고 했는데 그 경우는 샘플이 초단발화 샘플들이었다.

나는 실험을 하고 과연 이 학습이 _**긴 발화 샘플들의 발음 오인식 개선에도 영향이 없을까?**_ 라는 생각이 들었다.

그래서 긴 발화 샘플(40개)들에 대해서
- base 모델
- audio_embed ft 학습 모델 (통로)
- audio_tower ft 학습 모델 (귀)

위 3가지를 대조 비교하기로 하였다.

---
## 결과
![](https://velog.velcdn.com/images/xorms/post/0e654b55-e88f-40f1-baaf-6e052799f894/image.png)
- base -> embed_audio -> embed_audio + audio_tower 로 갈수록 CER이 줄어듦


긴 발화 40개 중 embed_audio -> audio_tower 개선폭 상위 6개를 나열
![](https://velog.velcdn.com/images/xorms/post/34eb7592-69fa-4c29-b923-4b19b682915a/image.png)
- 초단발화 샘플에서는 모두 똑같던 것에 비해 발음 오인식이 어느정도 해결된 것을 볼 수 있다.

## 결론
결론적으로, audio_tower 학습이 발음 오인식에 무력하다는 이전 결론은 **초단발화**에 한정된 것이었다.

즉, audio_tower의 음소 복구 능력은 발화 길이(정보량)에 의존한다.
초단발화(1~2초)는 신호에 담긴 음소 정보량이 부족해 귀를 학습해도 살리지 못하고 긴 발화는 정보가 충분해 학습 효과가 나타난다.

한계는 모듈 선택이 아닌 입력 정보량 x 인코더 용량의 상호작용에 있다고 판단한다.
