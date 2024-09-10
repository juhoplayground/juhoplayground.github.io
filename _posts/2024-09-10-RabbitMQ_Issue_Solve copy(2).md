---
layout: post
title: RabbitMQ - "All stable feature flags must be enabled after completing an upgrade." 
author: 'Juho'
date: 2024-06-04 09:00:00 +0900
categories: [RabbitMQ]
tags: [RabbitMQ, Celery, Python]
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
1. [문제 및 해결 방법](#문제-및-해결-방법)

## 문제 및 해결 방법
`All stable feature flags must be enabled after completing an upgrade.` 라는 경고창을 확인했다.<br/>
그래서 `sudo rabbitmqctl list_feature_flags` 명령어를 통해서 `feature_flag`를 확인해봤다.<br/>
그랬더니 몇 가지 `feature_flag`가 `enabled`가 아닌 `disabled`로 되어 있는 것을 확인할 수 있었다.<br/>
해결 방법은 `disabled`로 되어 있는 `feature_flag`를 `sudo rabbitmqctl enable_feature_flag <feature_flag_name>`으로 `enabled``하게 해주면 된다.<br/>
그럼 위에서 확인했던 경고 문구가 rabbitmq에서 제거된 것을 확인할 수 있다.<br/>
