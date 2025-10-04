---
layout: post
title: LLM 서빙 환경 구축하기 + 모니터링
author: 'Juho'
date: 2025-10-04 09:00:00 +0900
categories: [LLM]
tags: [LLM, LiteLLM, vLLM, Prometheus, Grafana]
pin: True
toc : True
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
1. [Docker를 이용한 LLM 서빙 + 모니터링 환경 구축하기](#docker를-이용한-llm-서빙--모니터링-환경-구축하기)
2. [vLLM](#vllm)  
3. [LiteLLM](#litellm)
4. [Prometheus](#prometheus)
5. [Grafana](#grafana)
6. [실행 방법](#실행-방법)


## Docker를 이용한 LLM 서빙 + 모니터링 환경 구축하기  
LLM을 서비스 환경에 올릴 때는 효율적인 서빙과 함께 모니터링 체계를 구축하는 것이 필수적이다.   
vLLM과 LiteLLM으로 LLM을 서빙하고 Prometheus와 Grafana로 모니터링 환경을 통합하는 과정을 Docker 기반으로 정리했다.  
또한 GPU/CPU Exporter를 통해 자원 사용량까지 시각화해 안정적인 운영을 지원하려고 했다.  

## vLLM
vLLM은 OpenAI API 호환 인터페이스를 제공하면서도 고성능 LLM 서빙이 가능한 라이브러리다.  
특히 PagedAttention 기법을 활용해 GPU 메모리 효율이 뛰어나고 다양한 모델을 서빙하기 적합하다.  
```yml
services:
  vllm:
    image: vllm/vllm-openai:gptoss
    container_name: vllm
    command:
      - --model=openai/gpt-oss-20b
      - --host=0.0.0.0
      - --port=8000
      - --gpu-memory-utilization=0.8
      - --max-num-seqs=16
      - --max-model-len=4096
      - --max-num-batched-tokens=12288
      - --tensor-parallel-size=1
      - --dtype auto
      - --compile
      - --enable-prefix-caching
      - --async-scheduling
      - --enable-chunked-prefill
      - --kv-cache-dtype=fp16
      - --disable-log-stats
      - --disable-log-requests
      - --swap-space=16
    environment:
      - HF_TOKEN=${HUGGINGFACE_HUB_TOKEN}
    ports:
      - "8000:8000"
    volumes:
      - huggingface-cache:/root/.cache/huggingface
    networks:
      - monitoring
    gpus: all
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```
서빙하려는 모델이 `openai/gpt-oss-20b`여서 vllm 이미지도 `vllm/vllm-openai:gptoss`를 사용했다.  

## LiteLLM
LiteLLM은 여러 LLM 백엔드를 통합해 하나의 OpenAI 호환 API로 제공하는 라우터다.  
여러 모델 제공 시 단일 API 게이트웨이 역할을 하여 요청 라우팅 및 로깅을 할 수 있다. 
```yml
litellm:
    image: ghcr.io/berriai/litellm:main-stable
    container_name: litellm
    command: ["--port","4000","--config","/app/config.yaml", "--detailed_debug"]
    environment:
      - LITELLM_MASTER_KEY=${LITELLM_MASTER_KEY}
      - LOG_LEVEL=INFO
    volumes:
      - ./litellm/config.yaml:/app/config.yaml:ro
    ports:
      - "4000:4000"
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - vllm
``` 
그리고 `litellm` 디렉토리 하위에 `config.yml` 파일을 생성한 다음  
```yml
model_list:
  - model_name: gpt-oss-20b
    litellm_params:
      api_base: http://vllm:8000/v1
      api_key: "EMPTY"
      model: hosted_vllm/openai/gpt-oss-20b
      timeout: 120

litellm_settings:
  callbacks: ["prometheus"]
  prometheus_metrics_config:
    - group: "proxy"
      metrics:
        - "litellm_proxy_total_requests_metric"
        - "litellm_proxy_failed_requests_metric"
      include_labels:
        - "requested_model"

    - group: "deployment_health"
      metrics:
        - "litellm_deployment_success_responses"
        - "litellm_deployment_failure_responses"
      include_labels:
        - "requested_model"

    - group: "performance"
      metrics:
        - "litellm_request_total_latency_metric"
        - "litellm_llm_api_latency_metric"
      include_labels:
        - "model"
        - "requested_model"

  timeout: 60
  num_retries: 2

general_settings:
  logging: true
  enforce_openai_moderation_format: false
  disable_auth_check_paths: ["/health", "/metrics", "/docs"]
  public_routes: ["/health", "/metrics"]
  no_auth_routes: ["/health", "/metrics"]
  disable_spend_logging: true
```
이렇게 설정했다.  
서빙하는 LLM이 하나여서 model_list 하위에 하나의 model name만 있다.  
다수의 model을 사용할거라면 하위에 추가하면 된다.  
  
## Prometheus
시계열 기반의 모니터링 툴로, Exporter가 제공하는 메트릭을 수집한다.  
```yml
prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --web.console.libraries=/etc/prometheus/console_libraries
      - --web.console.templates=/etc/prometheus/consoles
      - --storage.tsdb.retention.time=15d
      - --web.enable-lifecycle
      - --web.enable-admin-api
    ports:
      - "9090:9090"
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - vllm
```
### node-exporter (CPU Exporter)
```yml
node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    command:
      - --path.rootfs=/host
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
    volumes:
      - /:/host:ro       
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
    ports:
      - "9100:9100"
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - prometheus
```
   
### dcgm-exporter (GPU Exporter)  
```yml
dcgm-exporter:
    image: nvidia/dcgm-exporter:4.4.1-4.5.2-ubuntu22.04
    container_name: dcgm-exporter
    privileged: true
    gpus: all
    cap_add:
      - SYS_ADMIN
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - /sys:/sys:ro
    pid:
      host
    ports:
      - "9400:9400"
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - prometheus
```
그 다음 `prometheus` 디렉토리 하위에 `rules` 디렉토리를 만들고 그 아래에 `prometheus.yml` 파일을 생성한다.  
```
global:
  scrape_interval: 10s
  evaluation_interval: 10s

scrape_configs:
 - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "vllm"
    metrics_path: /metrics
    static_configs:
      - targets: ["vllm:8000"]

  - job_name: "litellm"
    metrics_path: /metrics/
    static_configs:
      - targets: ["litellm:4000"]
        authorization:
        type: Bearer
        credentials: ${LITELLM_MASTER_KEY}

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
  
  - job_name: 'dcgm-exporter'
    static_configs:
      - targets: ['dcgm-exporter:9400']
```

## Grafana
Prometheus가 수집한 메트릭을 시각화하는 대시보드 툴이다.
```yml
grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=${GF_SECURITY_ADMIN_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    ports:
      - "3000:3000"
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - prometheus
```  
[Grafana dashboards](https://grafana.com/grafana/dashboards/){:target="_blank"}에서 원하는 것을 import해서 사용하거나 직접 대시보드를 만들면 된다.  
직접 만들기 보다는 Grafana에서 제공하는 [vLLM](https://grafana.com/grafana/dashboards/23991-vllm/){:target="_blank"} , [LiteLLM](https://grafana.com/grafana/dashboards/24055-litellm/){:target="_blank"}
2개의 대시보드를 import해서 사용했다.  


## 실행 방법
```
docker compose up -d
Grafana 접속: http://localhost:3000
Prometheus 접속: http://localhost:9090
LiteLLM API 호출: http://localhost:4000/v1
```
---  

알람 규칙(Alert Rule)을 설정해 이상 징후를 빠르게 감지하는 등 운영 환경을 고도화할 수 있다.  
이번 포스팅이 LLM 서빙 환경을 처음 구축하거나 모니터링 체계를 도입하려는 분들께 도움이 되길 바란다.  