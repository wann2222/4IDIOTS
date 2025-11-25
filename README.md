# Drone-based Real-time Person Search with Face Recognition

드론과 얼굴 인식 AI를 결합해 **야외 공간에서 실종자나 특정 인물을 실시간으로 탐색**하는 시스템입니다.  
사용자가 웹에서 대상자의 사진을 업로드하면, 드론이 촬영하는 실시간 영상에서 동일 인물을 자동으로 탐지해 웹 화면으로 보여줍니다.

> 본 연구는 과학기술정보통신부 및 정보통신기획평가원의  
> 소프트웨어중심대학사업(2021-0-01406)과 인공지능융합혁신인재양성사업(RS-2023-00256629)의  
> 연구결과로 수행되었습니다.

---

## 1. Overview

### Problem

- 놀이공원, 축제, 공원 등 **넓은 야외 공간에서 실종자 탐색**은 아직도 고정형 CCTV와 사람의 수작업에 크게 의존합니다.
- CCTV는
  - 사각지대가 많고
  - 설치 위치에 따라 각도·거리 제한이 있으며
  - 실시간 탐색보다는 사후 확인에 가깝다는 한계가 있습니다.

### Solution

- **드론**으로 상공에서 넓은 영역을 촬영하고
- **InsightFace(ArcFace 기반)** 얼굴 인식 모델로
  업로드된 대상 사진과 실시간 영상의 얼굴을 비교합니다.
- 웹 페이지에서
  - 실시간 드론 영상 스트리밍
  - 대상 사진 업로드
  - 인식된 프레임 하이라이트를 한 화면에서 제공합니다.

---

## 2. Key Features

- **Real-time Drone Video Streaming**
  - 드론 조종 앱 화면을 로컬 PC에서 캡처
  - 프레임 단위 JPEG 인코딩 후 TCP로 서버 전송

- **Face Detection & Identification**
  - InsightFace `FaceAnalysis`로 얼굴 검출 및 임베딩 추출
  - 대상 사진에서 얻은 기준 임베딩과 **코사인 유사도** 비교
  - 임계값(약 0.23) 이상일 경우 동일 인물로 판단하고 Bounding Box + 유사도 표시

- **Web-based UI**
  - 실시간 영상 스트리밍(`/video_feed`)
  - 대상 사진 업로드(`/upload`)
  - 인식된 프레임(`recognized.jpg`)을 주기적으로 리프레시하여 결과 영역에 표시

- **Empirical Threshold Tuning**
  - 거리(3~15m) × 높이(2~15m) × 환경(조도, 각도, 액세서리) 실험
  - 유사도 분포 기반으로 최적 임계값 도출

---

## 3. Architecture

    [Drone] ──(영상)──> [Local Client] ──(TCP)──> [GCP Server] ──(HTTP/MJPEG)──> [Web UI]

    Drone        : DJI MINI 2 SE (실시간 영상 촬영)
    Local Client : pyautogui로 화면 캡처, OpenCV로 인코딩 후 TCP 전송
    GCP Server   : Flask + SocketIO, InsightFace로 얼굴 인식, 결과 이미지 저장
    Web UI       : HTML/JS로 실시간 영상·업로드·결과 영역 표시

---

## 4. My Role

Role: **AI / Face Recognition Lead**

- **모델 후보 조사 및 선정**
  - InsightFace, FaceNet 등 얼굴 인식 모델 비교 검토
  - Open-set 인식 성능, ArcFace 지원, 구현 난이도 등을 기준으로 **InsightFace `buffalo_l`** 최종 채택

- **얼굴 인식 파이프라인 구현**
  - 업로드된 대상 이미지에서 얼굴 검출·정렬·임베딩 추출 로직 구현
  - 실시간 드론 프레임에서 얼굴 검출 → 임베딩 추출 → 기준 임베딩과 코사인 유사도 계산
  - 임계값 이상일 때 Bounding Box 및 유사도 점수 오버레이

- **임계값(Threshold) 튜닝 및 실험 설계**
  - 거리(3,5,7,9,11,13,15m) × 높이(2~15m) 격자 실험 설계
  - 각 조건에서 동일 인물 / 타인 데이터를 수집해 유사도 분포 분석
  - 오탐·미탐을 모두 고려하여 **최종 임계값 ≈ 0.23** 설정

- **성능 측정 및 보고**
  - 실험 환경에서
    - Face Detection ≈ 93%
    - Face Identification ≈ 98%
  - 인식 처리 속도 ≈ 52ms, 네트워크 포함 전체 응답 시간 ≈ 400ms 이내

- **파인튜닝 시도**
  - 자체 데이터 기반 파인튜닝 파이프라인 구성 및 학습 시도
  - 리소스·환경 제약으로 중단되었으나, 경험을 바탕으로
    “사전학습 모델 + 임계값 튜닝” 중심 운영 전략 수립

---

## 5. Tech Stack

- **Language**
  - Python 3
  - JavaScript

- **Backend / Streaming**
  - Flask
  - Flask-SocketIO
  - TCP Socket (`socket`)

- **AI / Computer Vision**
  - InsightFace (ArcFace 기반 Face Recognition)
  - OpenCV
  - NumPy
  - onnxruntime-gpu, CUDA

- **Infra**
  - Google Cloud Platform (Ubuntu, NVIDIA GPU VM)

- **Others**
  - pyautogui (스크린 캡처)
  - HTML, CSS (웹 UI)

---

## 6. How It Works

### 6.1 Local Client (Drone Screen Capture → TCP)

- 드론 조종 앱이 실행 중인 화면 영역을 지정하여 캡처
- 캡처한 이미지를 OpenCV BGR 포맷으로 변환 후 480×540으로 리사이즈
- JPEG로 인코딩하고, 크기 정보(4바이트) + 이미지 바이트를 TCP로 전송

    # pseudo-code
    TCP_IP = "SERVER_IP"
    TCP_PORT = 7777

    def send_video_frames(client_socket):
        left, top, width, height = 0, 0, 1920, 1080
        while True:
            screenshot = pyautogui.screenshot(region=(left, top, width, height))
            frame = cv2.cvtColor(np.array(screenshot), cv2.COLOR_RGB2BGR)
            frame = cv2.resize(frame, (480, 540))
            ret, buffer = cv2.imencode('.jpg', frame)
            frame_bytes = buffer.tobytes()
            size = len(frame_bytes)
            client_socket.sendall(size.to_bytes(4, 'big'))
            client_socket.sendall(frame_bytes)

### 6.2 Server (Flask + InsightFace)

- TCP로 JPEG 프레임 수신 후 OpenCV 이미지로 디코딩
- `FaceAnalysis`로 얼굴 검출 및 임베딩 추출
- 업로드된 기준 임베딩과 코사인 유사도 계산
- 임계값 이상이면 Bounding Box + 유사도 점수 표시 후 `recognized.jpg`로 저장
- `/video_feed`에서 MJPEG 스트리밍, `/recognized_feed`에서 결과 이미지 제공
- 웹 템플릿에서 일정 주기로 `recognized.jpg`를 리프레시해 인식 결과 영역에 반영

---

## 7. Results

- **Accuracy**
  - Face Detection: 약 93%
  - Face Identification: 약 98%

- **Latency**
  - Face detection & identification: 평균 ≈ 52ms
  - Network transfer (로컬 → GCP): 평균 ≈ 214ms
  - Web UI 표시까지 포함한 전체 응답 시간: 약 400ms 이내

- **Findings**
  - 거리·높이가 멀어질수록, 특히 드론이 위에서 내려다보는 각도가 커질수록 유사도 급감
  - 마스크·모자·안경 등 액세서리 착용이 많을수록 유사도 감소
  - 배경·환경보다는 얼굴 각도, 조명, 표정이 성능에 더 큰 영향을 미침

- **Dissemination**
  - 한국스마트미디어학회 춘계학술대회 포스터 발표
  - 동 학술대회 프로시딩에 논문으로 수록

---

## 8. Limitations & Future Work

- 현재는 사용자가 드론을 수동 조작해야 하며,
  향후 **자율 비행 및 최적 탐색 경로 계획** 기능이 필요합니다.
- 마스크 등으로 얼굴 일부가 가려진 상황에서 인식 성능이 저하되므로,
  해당 조건에 특화된 데이터 추가 학습 및 모델 개선이 요구됩니다.
- 개인정보 보호 강화를 위해
  영상·이미지 데이터 암호화, 접근 제어, 로그 관리 등 보안 기능을 고도화할 예정입니다.

---

## 9. Acknowledgement

본 연구는 과학기술정보통신부 및 정보통신기획평가원의  
소프트웨어중심대학사업(2021-0-01406)과  
인공지능융합혁신인재양성사업(RS-2023-00256629)의 연구결과로 수행되었습니다.
