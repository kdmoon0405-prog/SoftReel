# SoftReel 숏폼 영상 점멸 검출과 국소 보정

[기술 보고서 PDF](./SoftReel_Technical_Report.pdf) · [보고서 원문](./SoftReel_Technical_Report.md) · [Zenodo DOI: 10.5281/zenodo.24150972](https://doi.org/10.5281/zenodo.24150972) · [Team L-EOD 프로젝트 저장소](https://github.com/SKT-FLY-AI-9-6/gumchulgi)

SoftReel은 업로드한 숏폼 영상에서 휘도와 적색 점멸 등을 검출하고, 후보 영역의 화소별 휘도와 색 변화량을 제한합니다. 워커는 강한 설정으로 보정한 영상을 다시 판정합니다. 위반이 남으면 기본 설정으로 보정한 뒤 두 출력의 위반 항목을 비교해 결과를 선택합니다. [워커 구현](https://github.com/SKT-FLY-AI-9-6/gumchulgi/blob/bbfb1a37140e9aa393719fe5e623165e048aa4ab/server/worker/pipeline.py)

**보고서 작성** Kyungdon Moon  
**프로젝트** SKT FLY AI Challenger 9기 Team L-EOD

## 검출·보정 구조

| 모듈 | 역할 |
| --- | --- |
| [점멸 검출기](https://github.com/SKT-FLY-AI-9-6/gumchulgi/blob/bbfb1a37140e9aa393719fe5e623165e048aa4ab/psepipe_v3_seam/pse_bt1702.py) | 분석 프레임을 선형광으로 변환하고, 휘도·적색 점멸과 패턴·화면전환 등의 위반을 판정합니다. |
| [CPU 보정 필터](https://github.com/SKT-FLY-AI-9-6/gumchulgi/blob/bbfb1a37140e9aa393719fe5e623165e048aa4ab/psepipe_v3_seam/pselive3.py) | 점멸 후보 마스크에 화소별 변화율 제한과 밝기 세부 복원을 적용합니다. |
| [GPU 이식판](https://github.com/SKT-FLY-AI-9-6/gumchulgi/blob/bbfb1a37140e9aa393719fe5e623165e048aa4ab/psepipe_v3_seam/psegpu_full.py) | CUDA 환경의 보정 경로입니다. |
| [업로드 워커](https://github.com/SKT-FLY-AI-9-6/gumchulgi/blob/bbfb1a37140e9aa393719fe5e623165e048aa4ab/server/worker/pipeline.py) | 정규화, 입력 판정, 보정, 출력 재판정을 실행합니다. |

## 실행

서버 실행과 모바일 앱 설치 절차는 [원본 프로젝트의 빠른 시작](https://github.com/SKT-FLY-AI-9-6/gumchulgi#빠른-시작-노트북--서버-폰--클라이언트)과 [서버 실행 안내](./server/README.md)를 참고하세요. Python, FFmpeg와 Flutter 설치 및 서버 환경 변수 설정이 필요합니다.

CPU 필터의 자기감시 보정은 공개 GPU 이식판에서 미구현 상태로 표시돼 있습니다. [GPU 이식판](https://github.com/SKT-FLY-AI-9-6/gumchulgi/blob/bbfb1a37140e9aa393719fe5e623165e048aa4ab/psepipe_v3_seam/psegpu_full.py)

## 실사 영상 검증

팀의 실사 개발 로그 209편 중 원본에 위반이 있던 41편을 비교했습니다. 아래 수치는 보정본을 선택할 때도 사용한 자체 검출기의 판정입니다. [영상별 판정 CSV](https://github.com/SKT-FLY-AI-9-6/gumchulgi/blob/bbfb1a37140e9aa393719fe5e623165e048aa4ab/psepipe_v3_seam/validation/fallback_209.csv)

| 처리 방식 | 적합 전환 | 위반 잔존 |
| --- | ---: | ---: |
| 기본 설정 | 14/41 | 27/41 |
| 강한 설정 | 19/41 | 22/41 |
| 재판정 후 선택 | 21/41 | 20/41 |

세부 복원 폭과 방향성 검사를 함께 바꾼 개발 실험에서 실사 6편의 판정이 나빠졌습니다. 설정별 재실행에서 휘도 점멸이 다시 나타난 3편은 모두 세부 복원 폭과 관련됐습니다. 209편 로그에서는 강한 설정으로 적합했던 영상이 19편이었고, 기본 설정에서만 적합한 영상 2편이 더 있었습니다. [회귀 실험 기록](https://github.com/SKT-FLY-AI-9-6/gumchulgi/blob/bbfb1a37140e9aa393719fe5e623165e048aa4ab/psepipe_v3_seam/validation/REGRESS_0820.md)

합성 영상 24편에서 EA IRIS와 판정을 비교한 결과, 선형광 적색 비율을 적용하기 전에는 18편, 적용 후에는 20편이 일치했습니다. 실사 비교에서는 두 검출기가 19편 중 17편에서 일치했습니다. 실사 비교군 19편은 모두 위 209편 로그에도 포함됩니다. [IRIS 비교 기록](https://github.com/SKT-FLY-AI-9-6/gumchulgi/blob/bbfb1a37140e9aa393719fe5e623165e048aa4ab/IRIS_%EA%B5%90%EC%B0%A8%EA%B2%80%EC%A6%9D_v0.1.md) · [실사 비교 기록](https://github.com/SKT-FLY-AI-9-6/gumchulgi/blob/bbfb1a37140e9aa393719fe5e623165e048aa4ab/psepipe_v3_seam/validation/README.md)

별도 실사 3편에서 BlazeBVD 재구현도 비교했습니다. 두 방식이 모두 적합 판정을 받은 한 영상의 경계 왜곡 점수는 자체 필터 2.542, BlazeBVD 29.849였습니다. 다른 영상에서는 BlazeBVD가 자체 필터 출력에 남은 화면전환 위반을 해소했습니다. [비교 로그](https://github.com/SKT-FLY-AI-9-6/gumchulgi/blob/bbfb1a37140e9aa393719fe5e623165e048aa4ab/psepipe_v3_seam/validation/ad_blazebvd_full.csv)

## 제가 맡은 작업

- 원본과 보정본의 휘도, 점멸 이벤트, 색도 변화를 측정하고, 워커에 저장된 결과를 대시보드 API와 Flutter 화면에 연결했습니다. [보정 영향 지표 및 대시보드 커밋](https://github.com/SKT-FLY-AI-9-6/gumchulgi/commit/55a136dbbe6ddb7b4d4e24704633b50b49695371)
- 실제 업로드 API와 보정 워커를 사용해 보정본의 시청 이벤트와 영향 지표가 대시보드 응답까지 전달되는지 검증했습니다. [종단 테스트 커밋](https://github.com/SKT-FLY-AI-9-6/gumchulgi/commit/52eebe5bd130f92c13c9e8201389d222477c3964)
- 업로더 스튜디오에 원본·보정본 전환과 위반 구간 표시를 구현하고, 측정 마커가 잔존 위반으로 잘못 표시되던 버그를 수정했습니다. [업로더 스튜디오 커밋](https://github.com/SKT-FLY-AI-9-6/gumchulgi/commit/321819cc1f54859e5812c1dad7c480f2a981f85d)
