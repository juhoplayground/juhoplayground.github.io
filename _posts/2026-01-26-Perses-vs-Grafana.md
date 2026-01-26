---
layout: post
title: Perses vs Grafana - 오픈소스 대시보드 도구 비교
author: 'Juho'
date: 2026-01-26 00:00:00 +0900
categories: [DevOps]
tags: [Grafana, Prometheus, Kubernetes, DevOps]
pin: True
toc: True
---

<style>
  th{
    font-weight: bold;
    text-align: center;
    background-color: white;
  }
  td{
    background-color: white;
  }

</style>

## 목차
1. [개요](#개요)
2. [Perses란](#perses란)
3. [Grafana란](#grafana란)
4. [핵심 비교](#핵심-비교)
5. [Dashboard-as-Code](#dashboard-as-code)
6. [데이터 소스 지원](#데이터-소스-지원)
7. [Kubernetes 통합](#kubernetes-통합)
8. [마이그레이션](#마이그레이션)
9. [언제 무엇을 선택할까](#언제-무엇을-선택할까)

## 개요

Observability 대시보드 도구 선택은 현대 DevOps 환경에서 중요한 결정이다.
Grafana는 오랫동안 업계 표준이었지만 CNCF 샌드박스 프로젝트인 Perses가 새로운 대안으로 주목받고 있다.
두 도구의 특징과 차이점을 비교해본다.

## Perses란

Perses는 CNCF 샌드박스 프로젝트로 등록된 오픈소스 대시보드 도구다.
Prometheus 창시자인 Julius Volz가 "GitOps 우선 접근 방식을 가진 새로운 OSS 대시보드 빌더 프로젝트"라고 소개했다.

### 핵심 특징

Perses의 주요 특징은 다음과 같다.

- 대시보드를 위한 오픈 스펙 제공
- 플러그인 기반 아키텍처로 확장 가능
- Dashboard-as-Code 지원 (CUE, Go SDK)
- Kubernetes 네이티브 배포 지원
- Apache 2.0 라이선스

### 지원하는 Observability 영역

Perses는 네 가지 Observability 영역을 통합 지원한다.

- Prometheus - 메트릭 (PromQL 디버거 포함)
- Tempo - 분산 트레이싱 (산점도, 간트 차트, 테이블)
- Loki - 로그 검색 및 필터링
- Pyroscope - 프로파일링 (인터랙티브 플레임 그래프)

### 도입 기업

Amadeus, Red Hat, SAP, Chronosphere 등이 Perses를 도입했다.

## Grafana란

Grafana는 가장 널리 사용되는 오픈소스 시각화 및 모니터링 솔루션이다.
복잡한 데이터셋을 아름다운 대시보드로 시각화하여 시스템 성능을 모니터링한다.

### 핵심 특징

Grafana의 주요 특징은 다음과 같다.

- 150개 이상의 데이터 소스 플러그인 지원
- 시계열 그래프, 히트맵, 3D 차트 등 다양한 시각화
- Explore 탭을 통한 데이터 탐색
- Grafana Assistant (AI 기반 대시보드 지원)
- 알림 및 통합 기능

### 라이선스 변경

Grafana는 AGPLv3 라이선스로 변경되었다.
이는 Apache 2.0보다 제한적인 라이선스다.
상용 환경에서 임베딩하려면 라이선스 조건을 확인해야 한다.

## 핵심 비교

두 도구의 핵심 차이점을 비교한다.

### 기본 정보

| 항목 | Perses | Grafana |
|------|--------|---------|
| 라이선스 | Apache 2.0 | AGPLv3 |
| 관리 주체 | CNCF 샌드박스 | Grafana Labs |
| GitHub 스타 | 1.9K+ | 60K+ |
| 성숙도 | 상대적으로 신규 | 10년 이상 운영 |

### 기능 비교

| 기능 | Perses | Grafana |
|------|--------|---------|
| 데이터 소스 플러그인 | 4개 (Prometheus, Tempo, Loki, Pyroscope) | 150개 이상 |
| Dashboard-as-Code | 네이티브 지원 (CUE, Go SDK) | 제한적 |
| Kubernetes CRD | 네이티브 지원 | 별도 도구 필요 |
| Explore 탭 | 미지원 | 지원 |
| AI 지원 | 미지원 | Grafana Assistant |
| 커스터마이징 | 제한적 | 매우 광범위 |

### 장단점 요약

Perses는 GitOps 워크플로우와 Dashboard-as-Code에 강점이 있다.
반면 Grafana는 풍부한 기능과 광범위한 생태계를 제공한다.

## Dashboard-as-Code

Dashboard-as-Code는 대시보드를 코드로 정의하고 버전 관리하는 접근 방식이다.
이 영역에서 두 도구는 큰 차이를 보인다.

### Perses의 접근 방식

Perses는 Dashboard-as-Code를 핵심 철학으로 삼는다.
CUE와 Go SDK를 통해 대시보드를 프로그래밍 방식으로 정의할 수 있다.

```go
// Go SDK 예시
dashboard := &v1.Dashboard{
    Kind: "Dashboard",
    Metadata: v1.ProjectMetadata{
        Name:    "my-dashboard",
        Project: "my-project",
    },
    Spec: v1.DashboardSpec{
        Duration:  "1h",
        Variables: []dashboard.Variable{},
        Panels:    map[string]v1.Panel{},
    },
}
```

percli 도구로 CI/CD 파이프라인에 통합할 수 있다.
재사용 가능한 컴포넌트 라이브러리를 만들어 대규모 대시보드를 관리한다.

### Grafana의 접근 방식

Grafana에서 Dashboard-as-Code는 달성하기 어렵다.
JSON 파일로 대시보드를 정의할 수 있지만 프로그래밍적 생성이 제한적이다.
Grafonnet 같은 외부 도구를 사용해야 한다.

### 정적 검증

Perses는 대시보드 형식에 대한 완전한 정적 검증을 제공한다.
조직별 커스텀 린트 규칙도 확장할 수 있다.
Grafana는 이런 수준의 검증을 기본 제공하지 않는다.

## 데이터 소스 지원

데이터 소스 지원 범위에서 두 도구는 큰 차이가 있다.

### Perses

Perses는 4개의 핵심 데이터 소스에 집중한다.

- Prometheus (메트릭)
- Tempo (트레이스)
- Loki (로그)
- Pyroscope (프로파일링)

이 조합으로 네 가지 Observability 영역을 모두 커버한다.
향후 더 많은 데이터 소스가 추가될 예정이다.

### Grafana

Grafana는 150개 이상의 데이터 소스 플러그인을 제공한다.
Prometheus, InfluxDB, Elasticsearch, MySQL, PostgreSQL 등 거의 모든 데이터 소스를 지원한다.
다양한 시스템을 통합해야 한다면 Grafana가 유리하다.

## Kubernetes 통합

Kubernetes 환경에서의 통합 방식을 비교한다.

### Perses

Perses는 Kubernetes 네이티브 모드를 제공한다.
대시보드 정의를 CRD(Custom Resource Definition)로 배포할 수 있다.
개별 애플리케이션 네임스페이스에서 대시보드를 관리한다.

```yaml
apiVersion: perses.dev/v1alpha1
kind: PersesDashboard
metadata:
  name: my-dashboard
  namespace: my-app
spec:
  display:
    name: My Dashboard
  duration: 1h
  panels: {}
```

Perses Operator를 통해 Kubernetes 환경에 쉽게 배포할 수 있다.
GitOps 워크플로우와 자연스럽게 통합된다.

### Grafana

Grafana도 Kubernetes에 배포할 수 있지만 네이티브 CRD 지원은 없다.
Grafana Operator나 Helm 차트를 사용한다.
대시보드는 ConfigMap이나 외부 저장소에서 관리한다.

## 마이그레이션

기존 Grafana 사용자가 Perses로 전환하는 방법이다.

### 지원하는 마이그레이션 방식

Perses는 Grafana에서의 마이그레이션을 지원한다.

- Web UI에서 JSON 파일 직접 가져오기
- percli를 사용한 배치 마이그레이션

### 마이그레이션 시 고려사항

모든 Grafana 기능이 Perses에서 지원되지는 않는다.
Explore 탭 같은 기능은 Perses에 없다.
복잡한 플러그인을 사용한다면 호환성을 확인해야 한다.

### percli 사용 예시

```bash
# Grafana 대시보드 마이그레이션
percli migrate grafana --input dashboard.json --output perses-dashboard.yaml
```

## 언제 무엇을 선택할까

두 도구 중 선택은 요구사항에 따라 달라진다.

### Perses를 선택해야 할 때

다음 경우에 Perses가 적합하다.

- GitOps 워크플로우를 적극 활용하는 환경
- Dashboard-as-Code가 핵심 요구사항인 경우
- Kubernetes 네이티브 통합이 필요한 경우
- Apache 2.0 라이선스가 필요한 경우
- Prometheus, Tempo, Loki, Pyroscope만 사용하는 경우

### Grafana를 선택해야 할 때

다음 경우에 Grafana가 적합하다.

- 다양한 데이터 소스 통합이 필요한 경우
- 풍부한 시각화 옵션이 필요한 경우
- Explore 탭을 통한 데이터 탐색이 필요한 경우
- 성숙한 생태계와 커뮤니티 지원이 필요한 경우
- AI 기반 대시보드 지원이 필요한 경우

### 하이브리드 접근

두 도구를 함께 사용할 수도 있다.
Perses는 Grafana 대시보드를 가져올 수 있어 점진적 전환이 가능하다.
핵심 모니터링은 Perses로, 복잡한 분석은 Grafana로 분리할 수 있다.

## Reference

- [Perses](https://perses.dev/)
- [Grafana](https://grafana.com/)
- [GitHub - perses/perses](https://github.com/perses/perses)
- [Perses - CNCF](https://www.cncf.io/projects/perses/)
- [Perses Closes the Observability Gap with Declarative Dashboards - The New Stack](https://thenewstack.io/perses-closes-the-observability-gap-with-declarative-dashboards/)
- [Getting Started with Perses - AppCode](https://appscode.com/blog/post/getting-started-with-perses-the-free-open-source-grafana-alternative/)
