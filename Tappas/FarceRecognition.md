Hailo-ai의 **Tappas** 리포지토리에서 얼굴 인식(Face Recognition)은 단순한 단일 모델 추론이 아니라, **GStreamer 기반의 다단계 파이프라인(Pipeline)**으로 구현되어 있습니다.

소스 코드 수준의 핵심 구조와 내부 알고리즘을 분석하면 다음과 같습니다.

---

## 1. 하이레벨 아키텍처 (Multi-Stage Pipeline)

Tappas의 얼굴 인식은 'Detection -> Alignment -> Embedding Extraction -> Matching'의 4단계 과정을 거치며, 각 단계는 전용 신경망(NN)과 C++/Python 후처리 모듈이 결합된 형태입니다.

### 주요 구성 요소

* **hailonet**: Hailo HW 가속기에서 모델(HEF 파일)을 실행하는 GStreamer 요소.
* **hailofilter**: C++ 또는 Python으로 작성된 후처리 라이브러리(`.so`)를 로드하여 알고리즘을 수행.
* **hailocropper / hailoaggregator**: 탐지된 얼굴 영역을 잘라내어 다음 모델로 전달하고 결과를 다시 합치는 역할.

---

## 2. 소스 코드 수준의 핵심 단계별 알고리즘

### ① 얼굴 및 랜드마크 탐지 (Face Detection & Landmarks)

* **알고리즘**: 주로 **SCRFD**(Sample and Computation Redistribution for Efficient Face Detection) 모델을 사용합니다.
* **핵심 구조**:
* 입력 영상에서 얼굴의 Bounding Box뿐만 아니라 **5개의 핵심 랜드마크**(두 눈, 코, 입꼬리)를 동시에 추출합니다.
* `hailofilter`에서 실행되는 후처리 코드는 NMS(Non-Maximum Suppression)를 통해 중복된 탐지 결과를 제거합니다.



### ② 얼굴 정렬 (Face Alignment / Warping)

* **구현 위치**: `libvms_face_align.so` 또는 이와 유사한 이름의 C++ 라이브러리.
* **알고리즘**:
* 탐지된 5개 랜드마크를 기준으로 **Similarity Transformation(유사 변환)**을 수행합니다.
* 얼굴이 기울어져 있더라도 정면을 바라보는 표준화된 이미지(주로 112x112 크기)로 정규화합니다. 이 과정이 없으면 인식률이 급격히 떨어집니다.



### ③ 특징 추출 (Feature Extraction / Embedding)

* **알고리즘**: **ArcFace** 혹은 **MobileFaceNet** 계열의 모델을 사용합니다.
* **핵심 구조**:
* 정렬된 얼굴 이미지를 입력받아 **512차원(또는 128차원)의 벡터(Embedding)**를 생성합니다.
* 이 벡터는 해당 인물의 고유한 '지문' 역할을 하며, 조명이나 각도 변화에 강인하도록 학습되어 있습니다.



### ④ 신원 확인 및 매칭 (Recognition / Matching)

* **구현 방식**: `hailofilter` 내부의 Post-processing logic.
* **알고리즘**: **Cosine Similarity(코사인 유사도)** 비교.


* 추출된 임베딩 벡터와 데이터베이스(Gallery)에 저장된 벡터 간의 거리를 계산합니다.
* 임계값(Threshold, 예: 0.6) 이상인 경우 해당 인물로 식별(Identify)합니다.



---

## 3. 리포지토리 내 핵심 파일 경로 (Tappas 구조 기준)

구체적인 코드를 확인하시려면 다음 경로를 참고하십시오:

* **앱 실행 스크립트**: `apps/h8/gstreamer/general/face_recognition/face_recognition.sh`
* **파이프라인 구성**: `apps/h8/gstreamer/general/face_recognition/face_recognition.py`
* `hailonet`, `hailofilter`, `hailotracker` 등의 요소가 어떻게 체이닝되는지 확인할 수 있습니다.


* **후처리 C++ 소스**: `core/hailo/libs/post_processes/`
* 여기서 얼굴 인식과 관련된 `face_recognition` 혹은 `scrfd` 관련 후처리 로직 소스를 볼 수 있습니다.



---

## 4. 성능 최적화 전략

* **Cascading**: 모든 프레임에서 전체 인식을 돌리지 않고, **Hailo Tracker**를 사용하여 추적 중인 객체에 대해서만 간헐적으로 인식을 수행해 리소스를 절약합니다.
* **Multi-Stream**: Hailo-8의 병렬 처리 능력을 활용하여 여러 대의 카메라 입력을 동시에 처리하는 Muxing 구조가 포함되어 있습니다.

**혹시 특정 모델(예: SCRFD vs RetinaFace)의 파라미터 설정법이나, C++ 후처리 코드를 직접 수정하는 방법이 궁금하신가요?**
