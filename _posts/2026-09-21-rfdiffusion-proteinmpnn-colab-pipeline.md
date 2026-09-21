---
title: RFdiffusion으로 단백질 백본 생성 + ProteinMPNN으로 서열 설계, Colab에서 붙여본 기록
date: 2026-09-21 16:11:00 +0900
categories: [딥러닝, 실습]
tags: [aiffel, rfdiffusion, proteinmpnn, protein-design, diffusion-model, colab, conda, 학습정리]
---

오늘 과제는 "RFdiffusion으로 단백질 백본을 생성하고 ProteinMPNN으로 서열을
설계하라"였다. 이 둘을 이어 붙이면 **완전히 새로운 단백질 구조를 만들고, 그
구조가 실제로 접힐 수 있는 아미노산 서열까지 뽑아내는** 파이프라인이 된다.

## 두 모델의 역할을 집짓기에 비유하면

- **RFdiffusion** = 건축가. "몇 층짜리 건물을 어떤 모양으로 지을까"만 정한다.
  아미노산이 3차원 공간에서 어떻게 접힐지 **뼈대(backbone) 좌표**만 설계하고,
  어떤 아미노산인지는 아직 정하지 않는다. 확산 모델(diffusion model)이라는
  이름 그대로 — 이미지 생성 모델이 노이즈에서 사진을 만들어내는 것과 같은
  원리를 3D 좌표에 적용해서, 완전히 무작위인 점들에서 시작해 조금씩 "그럴듯한
  단백질 모양"으로 노이즈를 제거해 나간다.
- **ProteinMPNN** = 인테리어 시공업자. 건축가가 그린 뼈대(설계도)를 보고,
  "이 뼈대가 안정적으로 유지되려면 어느 자리에 어떤 아미노산(벽돌)을 놓아야
  하는가"를 계산해서 실제 서열(문자열)을 채워 넣는다.

## 왜 Colab인가 — 로컬 Mac은 애초에 안 된다

이 작업을 시작한 macOS(Apple M5 Pro)는 NVIDIA GPU가 없다. RFdiffusion은
NVIDIA의 SE3Transformer(DGL + CUDA 커스텀 커널)에 의존하는 구조라서, 로컬
실행이 원천적으로 불가능하다. `torch.backends.mps`는 되지만 이건 Apple
GPU용이라 CUDA 전용 SE3Transformer와는 무관하다. 그래서 무료 GPU를 주는
Google Colab(T4)에서 진행했다.

## 막힌 지점: "최신 스마트폰에서 2021년형 구형 앱을 돌리려는" 상황

Colab 커널 기본 사양을 확인하니 **python 3.13 + torch 2.11(cu128)**이었다.
그런데 RFdiffusion 공식 저장소가 요구하는 환경(`env/SE3nv.yml`)은:

```yaml
name: SE3nv
dependencies:
  - python=3.9
  - pytorch=1.9
  - cudatoolkit=11.1
  - dgl-cuda11.1
```

**python 3.9 + torch 1.9 + CUDA 11.1** — 2021년 기준 스택이다. 최신 DGL을
그냥 pip로 깔아봤더니 실제로는 python 3.10+ 에서 이미 제거된
`from collections import Mapping` 문법을 쓰는 훨씬 오래된 대체 휠이 잡혀서
바로 `ImportError`로 죽었다.

비유하자면 지금 커널은 최신 스마트폰인데 RFdiffusion은 구형 OS에서만 도는
앱인 상황. 앱을 억지로 최신 기기에 맞추는 대신, **기기 안에 구형 OS를
흉내내는 별도의 방(conda 가상환경)을 하나 더 만들어서** 그 안에서 앱을
돌리기로 했다.

## conda로 격리 환경 만들기, 그리고 또 한 번의 함정

RFdiffusion 저장소가 친절하게도 정확한 `SE3nv.yml`을 제공하기 때문에, 그걸
그대로 `mamba env create -f`로 설치했다. `defaults` 채널이 ToS 동의를
요구해서 한 번 막혔던 것만 빼면(`conda tos accept`로 해결), 여기까지는
순조로웠다 — python 3.9 환경에 `dgl-cuda11.1`, `cudatoolkit=11.1`까지 전부
깔렸다.

문제는 그 다음이었다. `se3-transformer`, `rfdiffusion` 패키지까지 다 설치하고
`torch.cuda.is_available()`을 찍어보니 **`False`**가 나왔다. GPU(T4)는
분명히 `nvidia-smi`에 잡히는데, 이 conda 환경의 torch만 CUDA를 못 봤다.

원인은 `torch.version.cuda`가 `None`으로 나온 것으로 확인했다 —
conda가 `SE3nv.yml`의 `pytorch=1.9` 스펙을 풀면서 **CPU 전용 빌드**를 골라버린
것이다(채널 우선순위 문제로 추정). 고친 방법은 conda 대신 pip으로 정확한
CUDA 11.1 빌드를 못박아 다시 까는 것이었다.

```bash
pip uninstall -y torch torchvision torchaudio
pip install torch==1.9.1+cu111 torchvision==0.10.1+cu111 torchaudio==0.9.1 \
  -f https://download.pytorch.org/whl/torch_stable.html
```

재설치 후 `torch.version.cuda`가 `11.1`로 잡히고 `is_available()`도
`True`로 바뀌었다. 다만 이 pip 설치 자체가 `-q`(quiet) 옵션 때문에 진행
표시줄이 전혀 안 보이는 상태로 몇 분간 조용히 돌았는데, 커널이 단일 스레드라
그 사이엔 다른 셀도 실행이 안 돼서 "멈춘 건가, 그냥 느린 건가" 헷갈리는
시간이 있었다. 결국은 그냥 큰 wheel(약 2GB) 다운로드가 조용히 진행 중이었던
것뿐이었다.

## ProteinMPNN은 의외로 순탄했다

ProteinMPNN은 순수 PyTorch로 짜여 있어서 DGL 같은 까다로운 의존성이 없다 —
Colab 기본 커널(torch 2.11)에서 conda 환경 없이 바로 돌아갔다. RFdiffusion
쪽에서 그렇게 씨름한 것과 대조적으로, 여기는 설치·실행 둘 다 문제가 없었다.

## 실제로 돌린 파이프라인

```bash
# 1) RFdiffusion — 조건 없는 100잔기 단일 사슬 백본 생성 (SE3nv 환경)
python RFdiffusion/scripts/run_inference.py \
  inference.output_prefix=outputs/design \
  inference.model_directory_path=params \
  inference.input_pdb=null \
  "contigmap.contigs=[100-100]" \
  inference.num_designs=1

# 2) ProteinMPNN — 그 백본에 맞는 서열 8개 설계 (기본 커널)
python ProteinMPNN/helper_scripts/parse_multiple_chains.py \
  --input_path outputs --output_path mpnn_out/parsed.jsonl
python ProteinMPNN/protein_mpnn_run.py \
  --jsonl_path mpnn_out/parsed.jsonl --out_folder mpnn_out \
  --num_seq_per_target 8 --sampling_temp "0.1"
```

결과: 백본 생성 **0.91분**(T=50 확산 스텝), 서열 설계 **4.1초**. 설계된
서열 8개 모두 ProteinMPNN 점수(낮을수록 뼈대에 더 잘 맞는 서열) **0.91~1.44**
범위로 나왔다.

```
>T=0.1, sample=3, score=0.9523, seq_recovery=0.0000
MEEEIKKALKELKEVVKKVIKEKNYSEEEKKKLEEVLKEVEKVAEEAKKAAEKNKEIKEATLKAFKEMIKVIKEEKDNVDKMVKKLKELIKKIKEEIKKA
```

`seq_recovery=0.0000`이 눈에 띄는데, 이건 원래 아미노산 서열과 비교해서
"몇 %를 맞췄나"를 재는 지표라서 — RFdiffusion 출력 자체가 아미노산 정보 없는
글리신(GLY) placeholder뿐이니 "원본"이 없고, 그래서 항상 0이 찍히는 게
정상이다.

## 느낀 점

1. **모델 자체보다 "환경을 어떻게 살리느냐"가 더 큰 산이었다.** RFdiffusion
   모델 코드나 논문 개념은 어렵지 않았는데, 2021년 스택을 2026년 Colab
   위에서 살리는 데 시간의 대부분을 썼다.
2. **"GPU가 잡히는데 `is_available()`이 False"는 conda가 CPU 빌드를 골랐을
   가능성을 의심하라.** `torch.version.cuda`가 `None`인지 먼저 찍어보면
   바로 확인된다.
3. **`pip install -q`로 큰 패키지를 깔 때는 진행 상황이 안 보인다는 걸
   감안해야 한다.** 조용히 오래 걸리는 걸 "멈췄다"고 착각하지 않으려면
   차라리 `-q`를 빼고 로그를 남기는 게 나을 뻔했다.

## 이번엔 왜 "콜랩 연결 끊김"이 안 일어났나

지난 [포즈 유지 이미지 생성 글]({% post_url 2026-09-20-pose-controlnet-colab-vs-local-two-track %})에서
콜랩 자동화가 4시간 넘게 실패한 핵심 이유는, 화면에 안 보이는 브라우저 창을
좌표 클릭·타이핑으로 흉내내야 했고 Colab이 그 탭을 "비활성"으로 판단해
GPU 런타임을 계속 끊었기 때문이었다. 이번엔 같은 문제를 한 번도 겪지 않았는데,
접근 방식 자체가 달랐다.

- **`colab-mcp`라는 전용 도구를 썼다.** 이건 화면 좌표를 클릭·타이핑으로
  흉내내는 게 아니라, 이미 열려 있는 Colab 탭에 "노트북 편집 API" 수준으로
  직접 붙는다. 셀 추가(`add_code_cell`)·수정(`update_cell`)·실행
  (`run_code_cell`)·조회(`get_cells`)가 전부 API 호출이라, 브라우저
  자동화 특유의 "안 보이는 창을 흉내내야 하는" 문제 자체가 없다.
- 그래도 바탕은 여전히 "사용자가 실제로 연 브라우저 탭"이다. 그래서 Colab
  입장에서는 사람이 보고 있는 정상 세션으로 취급됐고, **탭을 최소화·닫지
  않고 Mac이 잠자기 모드로 들어가지 않게만 유지**하면 됐다.
- 오래 걸리는 셀(conda 환경 생성처럼 5분 넘게 걸리는 것)은 도구 자체의
  클라이언트 타임아웃(30초)에 걸려서 에러 메시지가 뜨는데, 처음엔 이게
  진짜로 멈춘 건지 헷갈렸다. 하지만 실제로는 Colab 커널에서 계속 실행 중인
  경우가 대부분이었다 — `run_code_cell`을 반복 호출하는 대신
  `get_cells`(읽기 전용 조회, 재실행을 안 시킨다)로 상태만 조용히 확인하며
  기다리면, 끊지 않고도 진행 상황을 볼 수 있었다.

## Colab을 쓸 때 실제로 했던 것 (재현용 체크리스트)

1. Colab 새 노트북 탭을 열어두고, 그 탭을 최소화하거나 닫지 않는다.
2. 메뉴에서 `런타임 → 런타임 유형 변경 → 하드웨어 가속기: T4 GPU`를 선택한다
   (기본은 CPU라서 이 단계를 빼먹으면 `nvidia-smi`부터 실패한다).
3. 작업이 끝날 때까지 랩탑이 잠자기 모드로 들어가지 않게 한다(화면보호기·
   절전 설정 확인, 또는 `caffeinate` 류로 방지).
4. 오래 걸리는 설치·다운로드는 백그라운드(`&`)로 돌리고, 로그 파일에
   `tail`을 걸어 상태를 확인한다 — 셀 하나가 몇 분씩 블로킹되면 커널이
   단일 스레드라 다른 셀도 같이 멈춘 것처럼 보이기 때문이다.
5. 큰 패키지를 `pip install -q`로 깔 때는 진행 표시줄이 아예 안 보인다는
   걸 감안한다 — "출력이 없다"가 "멈췄다"는 뜻이 아니다.

## 산출물

- 노트북(설치 과정 + 실행 로그 포함): `ProteinDesign_test/colab/rfdiffusion_proteinmpnn.ipynb`
- 배경 설명: `ProteinDesign_test/README.md`

## 다음에 해볼 것

- `contigmap.contigs`를 바꿔서 다른 길이·모티프 고정(motif scaffolding) 조건으로도 생성해보기
- 매번 conda 환경을 새로 까는 대신, 완성된 `SE3nv` 환경을 통째로 압축해서
  Google Drive에 캐싱해두고 다음 실행부터는 그냥 풀어서 쓰기
- ProteinMPNN이 뽑은 서열을 다시 구조 예측(ESMFold 등)에 넣어서, RFdiffusion이
  의도한 뼈대 모양대로 실제로 접히는지 검증해보기
