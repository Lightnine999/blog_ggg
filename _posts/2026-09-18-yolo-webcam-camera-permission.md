---
title: YOLO 웹캠 탐지기를 만들다가 카메라 권한이 "앱마다 따로"라는 걸 배웠다
date: 2026-09-18 11:34:28 +0900
categories: [Computer Vision, 기초]
tags: [aiffel, yolo, opencv, uv, macos, tcc, 카메라권한, 트러블슈팅]
---

Claude Code한테 "YOLO로 웹캠 잡아서 실시간 객체 탐지 프로그램 만들어줘"라고 시켰다.
프로그램 자체는 금방 나왔는데, 정작 막힌 건 코드가 아니라 **macOS 카메라 권한**이었다.
왜 같은 코드가 어디서 실행하느냐에 따라 되고 안 되고가 갈렸는지 정리한다.

---

## 만든 것

`realtime_detect.py` 하나. 구조는 단순하다.

```python
while True:
    ok, frame = cap.read()               # 웹캠에서 프레임 한 장
    results = model.predict(frame, conf=args.conf)  # YOLOv8n으로 탐지
    annotated = results[0].plot()        # 박스 + 클래스명 그리기
    cv2.imshow("...", annotated)
    if cv2.waitKey(1) & 0xFF == ord("q"):
        break
```

YOLO가 "You Only Look Once"인 이유: 옛날 2-stage 탐지기는 후보 영역을 먼저 찍고
그 후보들을 다시 하나씩 판정했는데, YOLO는 사진 전체를 **한 번의 신경망 통과**로
"위치 + 종류"를 동시에 뽑는다. 그래서 프레임마다 다시 돌려도 실시간 속도가 나온다.

## 환경은 uv로

`pip install -r requirements.txt`는 "이 버전대로 대충 맞춰줘" 수준이라 나중에
똑같이 재현이 안 될 수 있다. `uv add ultralytics opencv-python`을 쓰면
`pyproject.toml`(뭘 쓸지) + `uv.lock`(정확히 어떤 버전이 깔렸는지)가 같이
남아서 `uv run python realtime_detect.py` 한 줄로 언제든 똑같은 환경이 만들어진다.

```console
$ uv init --bare --no-readme --no-workspace --app
$ uv add "ultralytics>=8.3.0" "opencv-python>=4.10.0"
```

## 진짜 문제: 카메라가 안 열린다

코드를 Claude Code가 대신 실행해줬는데 매번 이렇게 죽었다.

```
OpenCV: not authorized to capture video (status 0), requesting...
OpenCV: camera failed to properly initialize!
RuntimeError: 카메라(index=0)를 열 수 없습니다.
```

시스템 설정 → 개인정보 보호 및 보안 → 카메라에서 "Claude" 권한을 켜고 재시도해도
**똑같이** 실패했다. 두 번이나.

그런데 내가 직접 터미널을 열어서 같은 명령(`uv run python realtime_detect.py`)을
치니까 바로 macOS 권한 팝업이 뜨고, 허용을 누르니 정상적으로 카메라 창이 열렸다.
사람도, 옆에 있던 물건도 다 인식했다.

## 왜 이런 차이가 생겼나

macOS의 TCC(카메라·마이크·연락처 등 개인정보 접근 통제)는 **권한을 프로세스 단위가
아니라 "그 프로세스를 실행한 앱" 단위로** 기억한다.

- 내가 터미널에서 실행 → TCC는 "Terminal.app이 카메라를 쓰려 한다"로 인식 →
  Terminal에 이미 준 권한이 있으면 통과
- Claude Code가 대신 실행 → TCC는 "Claude(에이전트) 앱이 카메라를 쓰려 한다"로 인식 →
  이건 Terminal 권한과 완전히 별개의 항목

그래서 "시스템 설정에서 Claude 권한을 켰다"고 했는데도 안 됐던 이유는, 켠 항목이
실제로 이 프로세스를 실행하는 정확한 앱/헬퍼 프로세스와 이름이 안 맞았을 가능성이
높다. Electron 기반 앱은 GUI 앱 본체와 실제 명령을 실행하는 헬퍼 프로세스가 분리돼
있는 경우가 많아서, 사용자가 보는 앱 이름과 TCC가 붙잡는 프로세스가 다를 수 있다.

**결론적으로 가장 확실한 방법은: 카메라처럼 OS가 앱별로 관리하는 하드웨어는, 내가
직접 여는 셸에서 실행하는 것.** 에이전트가 대신 실행해주는 건 권한이 얽히는 순간
디버깅 비용이 더 커진다.

## 그 밖의 교훈

- 이 작업 이후로 **"카메라는 내가 명시적으로 실행해달라고 할 때만 접근한다"**는
  규칙을 세웠다. 웹캠·마이크 같은 사생활 민감 하드웨어는 에이전트가 확인 없이
  선제적으로 켜면 안 된다고 판단했다.
- Claude Code가 헤드리스 환경(카메라 없는 원격 샌드박스)에서 먼저 코드를
  검증(`--help`, 모델 로드, 예외 처리)해준 덕에, 실제 카메라 문제가 "코드 버그"가
  아니라 "권한 문제"라는 걸 빠르게 좁힐 수 있었다. 증거 없이 "될 겁니다"라고
  넘어가지 않고 한 단계씩 확인하는 게 디버깅 시간을 줄였다.

## 정리

1. YOLO는 한 번의 신경망 통과로 위치+클래스를 동시에 뽑아서 실시간이 가능하다.
2. `uv`는 lock 파일로 "누가 언제 돌려도 같은 환경"을 보장한다.
3. macOS 카메라 권한은 **"어떤 앱이 실행했는가"** 기준이다 — 터미널 권한과
   에이전트(Claude Code) 권한은 서로 다른 항목이라 따로 관리된다.
4. 사생활 민감 하드웨어는 에이전트에게 미리 켜두라고 시키지 않는다.
