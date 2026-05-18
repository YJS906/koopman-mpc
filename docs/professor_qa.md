# 교수님 질문 대응 Q&A

## Q1. residual이 무엇인가?

Residual은 선형모델이 틀린 만큼입니다.

```text
residual = 실제 plant 다음 상태 - 선형모델 다음 상태
```

본 연구에서는 다음 4개 상태에 대해 residual을 계산합니다.

```text
[e_y, e_psi, v_y, yaw_rate]
```

## Q2. Koopman 없이 residual만 쓰면 안 되는가?

지나간 step의 residual은 Koopman 없이도 계산할 수 있습니다.

하지만 MPC는 미래 조향 후보를 평가해야 하므로, 아직 일어나지 않은 미래 residual을 알아야 합니다.

Koopman 모델은 현재 상태, 도로 곡률, 조향 이력, 타이어 정보를 보고 미래 residual을 예측하는 보정기입니다.

## Q3. 4차원 residual을 예측하는데 왜 105차원 observable이 필요한가?

4차원 residual은 맞혀야 하는 답입니다.

105차원 observable은 그 답을 맞히기 위한 힌트입니다.

4개 상태만 보면 차가 지금 얼마나 벗어났는지만 알 수 있습니다.

하지만 residual은 속도, 곡률, 조향 이력, slip angle, 타이어 힘, 수직하중 같은 조건에 따라 달라집니다.

따라서 22개 물리 기반 feature를 만들고, 제곱항과 교차항으로 lift해서 105차원 observable을 구성했습니다.

## Q4. 그럼 105차원이 차원 낭비 아닌가?

무조건 낭비라고 보기는 어렵습니다.

105차원은 출력 차원이 아니라 nonlinear residual을 선형 predictor로 표현하기 위한 basis expansion입니다.

다만 초기 학습 데이터가 적으면 과적합 위험은 있습니다.

그래서 ridge regularization, feature scaling, clipping, online RLS update guard를 함께 사용했습니다.

교수님께는 이렇게 답하는 것이 안전합니다.

> 4-state Koopman만으로도 기본적인 residual 보정은 가능하지만, 본 연구의 시나리오는 속도 변화, 마찰 변화, Fiala tire saturation이 포함되어 있어 4개 error state만으로는 plant-model mismatch의 원인을 충분히 구분하기 어렵습니다. 따라서 tire/load/curvature/steering-history 정보를 포함한 augmented observable을 사용했습니다.

## Q5. MPC 자체가 105차원 문제인가?

아닙니다.

MPC QP 자체는 4차원 상태 문제입니다.

```text
x = [e_y, e_psi, v_y, yaw_rate]
u = delta
```

105차원은 MPC 전에 Koopman residual을 계산하기 위한 내부 표현입니다.

최종적으로 MPC에는 보정된 4차원 next-state prediction이 들어갑니다.

```text
x_next_pred = x_next_linear + residual_hat
```

## Q6. Koopman 출력이 MPC에 어떻게 들어가는가?

Koopman 출력은 `residual_hat`입니다.

```text
residual_hat = [residual_e_y, residual_e_psi, residual_v_y, residual_yaw_rate]
```

이 값은 linear bicycle model 예측에 더해집니다.

```text
x_next_pred = x_next_linear + residual_hat
```

그 다음 이 보정된 예측함수를 현재 상태 주변에서 4차원 선형 모델로 국소 선형화해서 MPC QP에 넣습니다.

```text
x_next ≈ F x + G delta + d
```

## Q7. EDMD는 어디에서 들어가는가?

EDMD는 초기 Koopman residual predictor를 학습하는 단계에서 들어갑니다.

EDMD는 105차원 observable 사이의 선형 관계를 맞춥니다.

```text
z_{k+1} ≈ A z_k + B delta_k
```

이렇게 얻은 `A`, `B`, `C`가 Fixed/Online Koopman MPC의 초기 predictor가 됩니다.

## Q8. Fixed와 Online의 차이는 무엇인가?

Fixed Koopman은 초기 EDMD 모델을 그대로 사용합니다.

Online Koopman은 같은 초기 EDMD 모델에서 시작하지만, 주행 중 새로 관측되는 residual을 이용해 RLS로 모델을 업데이트합니다.

즉 차이는 차원이 아니라 online adaptation 여부입니다.

## Q9. 계산 부담은 없는가?

계산 부담은 있습니다.

하지만 105차원이 MPC 최적화 변수로 직접 들어가는 것은 아니므로 부담이 폭발적으로 커지지는 않습니다.

주요 부담은 다음 부분입니다.

- Koopman residual 계산
- 국소 선형화 과정
- Online RLS update
- QP solver

최종 Python/CVX prototype에서는 평균 solve time이 50 ms 제어 주기 안에 들어왔지만, worst-case에서는 일부 초과가 있었습니다.

따라서 hard real-time 보장보다는 prototype-level real-time feasibility로 표현하는 것이 안전합니다.

## Q10. 연구의 핵심 기여를 한 문장으로 말하면?

> 제한된 초기 데이터로 학습한 EDMD-Koopman residual predictor와 online RLS update를 이용해, nominal linear bicycle model이 설명하지 못하는 nonlinear tire/vehicle plant mismatch를 보정한 경로 추종 MPC를 구성했다.

