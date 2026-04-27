# Project Name: Tactile-Augmented Hand-to-Robot Retargeting (R1)

## 1. Project Overview & Novelty Hook
- **Objective:** 사람의 손 포즈와 다지점 3축 촉각 센서 데이터를 기반으로 로봇 핸드(Shadow/Allegro 등)로 조작 기술을 리타겟팅(Retargeting)하는 알고리즘 개발.
- **Hardware:** MANUS Quantum Glove + 자체 부착 3축 촉각 센서(다중 측정 포인트, N 단위의 Force XYZ, Excel 파일로 센서의 Local Position 정의).
- **Core Novelty (vs. OSMO, TactAlign):** 단순한 Kinematic 매핑이나 오프라인 잠재 공간 정렬(Offline Latent Alignment)을 넘어선 **온라인 객체 중심 리타겟팅 옵티마이저(Online Object-Centric Retargeting Optimizer)**. 
  1. 손가락 변형(Deformation)을 고려한 미분 가능한 온라인 촉각 캘리브레이션.
  2. 인핸드 객체(In-hand Object)의 SE(3) 포즈를 명시적인 제약(Explicit Constraint)으로 보존.

## 2. System Architecture & Pipeline
### 2.1 Simulator: Isaac Lab 2.3+ (Primary)
- **Why Isaac Lab?** MANUS Native Streaming 지원 및 객체 포즈 트래킹 내장.
- **Tactile Implementation:** Excel에 정의된 Local Position에 Custom Contact Force Applicator를 생성하여 가상의 3축 센서 환경 구축.
- **Physics/Differentiability:** PyTorch Autograd 및 Warp를 활용하여 시뮬레이터 내 접촉 및 조인트 각도를 미분 가능하게(Differentiable) 최적화. (필요시 DiffTactile 확장은 제한적으로만 검토).

### 2.2 In-Hand Object Pose Estimation (Vision Module)
- **Sensor:** 외부 RGB-D 카메라 1~2대.
- **Pipeline:**
  1. **SAM3 (2026):** 실시간 손-물체 인터랙션 중 객체의 Mask/Segmentation 추출.
  2. **FoundationPose:** 추출된 Mask와 CAD 모델을 결합하여 Zero-shot 6D Pose Initialization 및 Temporal Tracking 수행.
  3. 추출된 SE(3) 포즈를 Isaac Lab 시뮬레이터 프레임으로 변환하여 리타겟팅 Loss의 제약 조건으로 주입.

## 3. Core Loss Function (The Retargeting Optimizer)
모델은 매 프레임마다 다음의 Multi-task Loss를 최소화하여 로봇의 Joint Angle(q)을 계산해야 합니다.

$$\mathcal{L} = \lambda_1 \|\mathbf{f}_\text{robot} - \mathbf{f}_\text{human}\|_2^2 + \lambda_2 \|\mathbf{T}_\text{obj}^\text{robot} - \mathbf{T}_\text{obj}^\text{human}\|_{\text{SE(3)}} + \lambda_3 \text{Penalty}_\text{penetration} + \lambda_4 \mathcal{L}_\text{contact invariance}$$

- **$\lambda_1$ (Tactile MSE):** 인간(Excel 기반)과 로봇(시뮬레이션)의 3축 힘 벡터 매칭.
- **$\lambda_2$ (Object Pose):** 비전 모듈에서 추론한 인간의 물체 제어 포즈와 로봇이 파지한 물체 포즈 간의 차이 최소화.
- **$\lambda_3$ (Penetration):** 메시(Mesh) 겹침 방지.
- **$\lambda_4$ (Contact Invariance):** 미끄러짐(Slip) 방지 및 힘의 방향성 일관성 유지.

## 4. Agent Guidelines (When writing code or searching)
1. **Always use PyTorch and Isaac Lab API:** 코드를 생성할 때 Isaac Sim 6.0 및 Isaac Lab 2.3+의 최신 API를 사용하세요. 
2. **Prioritize Differentiability:** Loss 함수 구현 시 모든 텐서 연산은 기울기(Gradient)가 흐를 수 있도록 작성해야 합니다.
3. **No Hallucinations on Integrations:** Isaac Lab과 DiffTactile의 네이티브 통합은 없음을 인지하고, 필요시 Custom Contact Sensor 클래스로 구현하세요.
4. **Modularity:** Vision 파트(SAM3+FoundationPose), Retargeting Optimizer(Loss+Torch), Sim 환경(Isaac Lab)을 철저히 분리하여 모듈화된 코드를 작성하세요.