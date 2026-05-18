# Koopman Residual MPC 쉬운 설명

## 1. Residual이란?

`residual`은 모델이 틀린 만큼입니다.

```text
residual = 실제 다음 상태 - 선형모델이 예측한 다음 상태
```

차량 연구에서는 다음 4개 값에 대해 residual을 계산합니다.

```text
residual =
[
  실제 e_y_next - 선형모델 e_y_next,
  실제 e_psi_next - 선형모델 e_psi_next,
  실제 v_y_next - 선형모델 v_y_next,
  실제 yaw_rate_next - 선형모델 yaw_rate_next
]
```

즉 residual은 linear bicycle model이 `VehicleBody + ECorner + Fiala tire` plant를 못 맞힌 부분입니다.

## 2. 왜 그냥 residual 값을 쓰면 안 되는가?

이미 지나간 step에서는 실제 residual을 알 수 있습니다.

하지만 MPC는 지금 조향각을 정하기 전에 미래를 예측해야 합니다.

그래서 필요한 것은 과거 residual 자체가 아니라:

```text
현재 상황에서 이 조향각을 넣으면 다음 residual이 얼마나 생길까?
```

를 예측하는 모델입니다.

Koopman residual predictor는 이 역할을 합니다.

## 3. Linear bicycle model 식

MPC의 기본 상태는 4차원입니다.

```text
x = [e_y, e_psi, v_y, yaw_rate]^T
```

Linear bicycle predictor는 다음 형태입니다.

```text
x_next_linear = A_d x + B_d delta + d_d
```

여기서 `delta`는 조향 입력이고, `d_d`에는 경로 곡률 영향이 들어갑니다.

## 4. Koopman residual은 어디에 쓰는가?

Koopman 모델은 linear bicycle model이 틀릴 값을 예측합니다.

```text
residual_hat = Koopman(x, context, delta)
```

그 다음 linear model 예측에 더합니다.

```text
x_next_pred = x_next_linear + residual_hat
```

따라서 MPC는 단순 선형모델이 아니라, 보정된 다음 상태 예측을 사용합니다.

## 5. 22차원 feature와 105차원 observable

출력은 4차원 residual이지만, 그 residual을 맞히기 위한 힌트는 더 많습니다.

22차원 feature에는 다음 정보가 들어갑니다.

```text
e_y, e_psi, v_y, yaw_rate,
vx, delta_prev, steering_rate,
curvature, curvature_rate, curvature_preview_1~3,
yaw_rate_prev_1~2,
alpha_front_mean, alpha_rear_mean, alpha_front_diff,
Fy_front_sum, Fy_rear_sum,
Fz_front_sum, Fz_rear_sum,
steering_angle_front_mean
```

4개 상태만 보면 차가 지금 얼마나 벗어났는지만 알 수 있습니다.

22개 feature는 왜 벗어졌는지에 대한 힌트를 줍니다.

- 속도
- 곡률
- 조향 이력
- yaw-rate 이력
- 타이어 slip
- 타이어 횡력
- 수직하중

## 6. 왜 lift를 하는가?

실제 차량 residual은 단순한 직선 관계가 아닙니다.

예를 들어 residual은 다음 조합에 의해 달라질 수 있습니다.

```text
속도 * 곡률
yaw_rate * 타이어 힘
slip_angle^2
e_psi * vx
```

그래서 22개 feature를 다음 항들로 확장합니다.

```text
1
x_i
x_i^2
x_i * x_j
```

이렇게 만들어진 것이 105차원 lifted observable입니다.

쉽게 말하면:

```text
22차원 feature = 기본 힌트
105차원 observable = 기본 힌트 + 힌트끼리의 조합
```

## 7. EDMD는 어디서 들어가는가?

EDMD는 초기 training data로 Koopman 행렬을 학습하는 방법입니다.

학습 데이터는 다음 형태입니다.

```text
현재 observable z_k
현재 조향 입력 delta_k
다음 residual target
```

EDMD는 다음 관계를 가장 잘 만족하는 행렬을 찾습니다.

```text
z_{k+1} ≈ A z_k + B delta_k
```

그리고 `C`를 통해 105차원 lifted state에서 4차원 residual을 꺼냅니다.

```text
residual_hat = C z_{k+1}
```

## 8. Fixed와 Online의 차이

Fixed Residual Koopman MPC:

```text
초기 EDMD로 학습한 A, B, C를 그대로 사용
```

Online Residual Koopman MPC:

```text
초기 EDMD 모델에서 시작
주행 중 실제 plant residual을 보고 RLS로 A, B를 업데이트
```

즉 Fixed와 Online은 입력/출력 차원이 다른 것이 아니라, online update 여부가 다릅니다.

## 9. MPC에는 몇 차원이 들어가는가?

MPC QP 자체는 4차원 상태 문제입니다.

```text
x = [e_y, e_psi, v_y, yaw_rate]
u = delta
```

105차원은 MPC 최적화 변수로 직접 들어가지 않습니다.

105차원은 MPC 전에 Koopman residual을 계산하기 위한 내부 표현입니다.

최종적으로는 보정된 4차원 next-state prediction을 MPC에 넘깁니다.

```text
x_next_pred = x_next_linear + residual_hat
```

코드 구조에서는 이 보정된 예측함수를 현재 상태 주변에서 국소 선형화하여:

```text
x_next ≈ F x + G delta + d
```

형태로 만든 뒤 MPC QP에 사용합니다.

## 10. 계산 부담

105차원 observable 때문에 MPC가 105차원 최적화 문제가 되는 것은 아닙니다.

추가 계산은 주로 다음 부분입니다.

- 22차원 feature 생성
- 105차원 observable 계산
- Koopman 행렬-벡터 곱
- 국소 선형화를 위한 predictor 반복 호출
- Online RLS update

105 x 105 행렬-벡터 곱 자체는 큰 계산 부담은 아닙니다.

다만 Python/CVX 기반 prototype에서는 hard real-time을 보장하기 어렵습니다.

최종 실험 로그 기준으로 Online Residual Koopman MPC의 recorded solve time은:

```text
평균: 약 34.8 ms
p95: 약 63.0 ms
max: 약 71.9 ms
control period: 50 ms
50 ms 초과: 약 9.9%
```

따라서 표현은 다음처럼 하는 것이 안전합니다.

> 평균적으로는 20 Hz 제어 주기 근처에서 동작했지만, worst-case 기준의 hard real-time 보장은 아직 부족하다.

