# [ LAB ] 19. 한국어 ASR 환각 판정 지표 + baseline 진단
- 작성일: 2026-06-01
- 원문: https://velog.io/@xorms/LAB-19
- 태그: ASR, CER, baseline, wer, 환각현상
---

이전 분석한 3가지의 방법 중에 Gemma4 프롬포트를 먼저 건드려보려 한다. 어조 분석을 하기 전에 기존의 문제점인 환각 현상에 대해서 집중해 볼 생각이다.

## 환각 판정 지표 설계

모델이 지금 얼마나, 어떻게 환각하나를 재는 진단을 먼저 해야한다고 판단했다.

환각을 어떻게 자동으로 가려내냐?

기존에 우리가 생각한 WER/CER 방식은 "얼마나 틀렸나"를 한 덩어리 숫자로 뭉쳐버려서, 환각 여부를 가려주지는 않는다.

### ASR 에러 3가지

- **환각**: 출력이 정답보다 길어짐 → 없는 말을 지어내기 때문에 삽입 폭증, CER이 1을 넘기도 함
- **대체**: 길이는 비슷하지만 단어만 틀림
- **누락**: 출력이 짧아짐 → 어려운 데를 그냥 빼버림

```python
ref = "오늘날씨가매우좋습니다"   # 공백 제거된 정답
cases = {
    "완벽":   "오늘날씨가매우좋습니다",
    "대체형": "오늘날씨가매우춥습니다", # 좋→춥, 길이 그대로
    "누락형": "오늘날씨가", # 뒷부분 통째 빠짐
    "환각형": "오늘날씨가매우좋습니다그리고내일은비가오고모레는눈이내릴거예요", # 없는 말 덧붙임
}
for name, hyp in cases.items():
    s = hallucination_signals(ref, hyp)
    print(f"[{name}] cer={s['cer']:.2f}  삽입률={s['ins_rate']:.2f}  "
          f"삭제율={s['del_rate']:.2f}  길이비={s['len_ratio']:.2f}")
```

세 에러 유형이 서로 다른 지문을 남김:

- 환각형: cer=1.82 / 삽입률=1.82 / 길이비=2.82 → 삽입 폭증 + 길이비>>1 + 삭제=0의 환각 시그니처
- 누락형: cer=0.55 / 삭제율=0.55 / 길이비=0.45 → 삭제 우세 + 길이비<1
- 대체형: cer=0.09 / 삽입·삭제=0 / 길이비=1.00 → 길이 보존, 내용만 틀림

## baseline 진단 (N=30)

```python
from datasets import load_dataset, Audio
import soundfile as sf
import io

N = 30

ds_n = (load_dataset("kresnik/zeroth_korean", split="test", streaming=True)
          .cast_column("audio", Audio(decode=False))
          .take(N))

rows = []
for i, ex in enumerate(ds_n):
    arr, _ = sf.read(io.BytesIO(ex["audio"]["bytes"]))
    pred, _ = transcribe(arr.astype("float32"))
    s = hallucination_signals(norm_for_cer(ex["text"]), norm_for_cer(pred))
    s.update(idx=i, ref=ex["text"], hyp=pred)
    rows.append(s)
```

결과:
- 코퍼스 CER = 0.117 (재현성 확인)
- 환각은 1건: 길이비 1.58 + INS 0.78으로 나머지 29개와 확실히 분리
- 이 1건의 환각률이 코퍼스 삽입률 0.036의 대부분을 기여
- 환각 성격: 실제 내용 오류(대체)와 없는 대화체 생성의 복합형, 배경 화자 목소리를 이어받아 생성한 것으로 추정
