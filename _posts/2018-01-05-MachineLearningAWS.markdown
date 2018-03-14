---
layout: post
title:  "AWS의 머신 러닝 서비스 알아보기"
date:   2018-01-05 10:50:00 +0900
categories: dev
---

AWS의 머신 러닝 서비스에 대해 알아보겠습니다.  
머신 러닝을 이용한 서비스 개발시 적용한다면 생산성 향상을 기대할 수 있을 것으로 보입니다.

# 머신 러닝 관련 AWS 서비스
![머신러닝 관련 AWS 서비스](/assets/img/all_ml_services.png)

# Machine Learning


## Amazon SageMaker

대규모 기계 학습 모델 구축, 학습, 배포 서비스

![SageMaker 동작 과정](/assets/img/SageMaker_How_it_works.png)

- 기계 학습을 사용하려는 개발자가 일반적으로 빠른 속도를 내지 못하게 하는 모든 장벽을 제거
  - 모델 구축 → 학습 → 튜닝 → 배포 각 단계의 복잡성 제거
- 기계 학습 모델을 구축, 교육 및 배포할 수 있는 모듈이 포함되어 있음

**특징**

- 학습 데이터를 손쉽게 탐색하고 시각화 할 수 있는 호스팅 Jupyter 노트북이 포함됨
- 가장 일반적으로 사용하는 기계 학습 알고리즘 10가지가 미리 설치 되어있음
  - Linear Learner
  - Factorization Machines
  - XGBoost Algorithm
  - Image Classification Algorithm
  - Sequence to Sequence (seq2seq)
  - K-Means Algorithm
  - Principal Component Analysis (PCA)
  - Latent Dirichlet Allocation (LDA)
  - Neural Topic Model (NTM)
  - DeepAR
- 콘솔에서 클릭 한 번으로 교육 및 배포
- 기본 요금 없이 사용한 만큼 비용 지불


----------

## Amazon Lex

대화형 인터페이스(챗봇)를 구축하는 서비스, Alexa와 동일한 딥 러닝 기술 기반

![호텔 예약 예시](/assets/img/Diagrams_lex_bookhotel.png)

- 음성 인식, 언어 이해 등의 딥러닝 문제(speech to text, 자연어 처리) 해결을 위한 툴 제공
- 데이터를 로딩, 저장을 위한 로직을 실행하기 위해 사용 되는 AWS Lambda와 연동하여 사용
- 확장 가능하고, 안전하고, 사용하기 쉬운 챗봇 구축, 배포, 모니터링 솔루션 제공

**특징**

- Multi-turn 대화 지원
  - 의도가 파악된 다음 필요한 정보를 완전히 얻기 위해 사용자에게 질문해야 함
  - 얻기 원하는 파라미터 목록과 대응되는 질문을 간단히 지정해주면 Amazon Lex가 해결해줌
- 두 종류의 질문 제공
  - 확인 질문 : 비지니스 로직 실행 전 사용자의 의도를 최종 적으로 확인
  - 에러 핸들링 질문 : 사용자의 입력이 이해되지 않을 때 다시 묻거나 종료하는 과정
- AWS Lambda 연동
  - 데이터를 가져오고, 저장하거나 비지니스 로직 실행을 위한 AWS Lambda 연동을 기본적으로 제공
  - 봇 개발에만 집중할 수 있도록 서버리스 컴퓨팅을 제공
  - Amazon Lambda를 통해 AWS 서비스들을 사용 가능
    - 대화 상태 저장을 위해 Amazon DynamoDB를 사용
    - 사용자 알림을 위해 Amazon SNS를 사용

AWS Lambda

- 서버 구축이나 관리가 필요 없는 코드 실행이 가능함
- 사용 시간 만큼만 요금을 내면 되고 코드가 실행 중이 아닐 땐 요금을 내지 않아도 됨

**콜센터 봇**

![콜센터 봇 구성도](/assets/img/diagram_Lex_Connect_appointment-reschedule.png)


Amazon Connect

- 클라우드 기반 고객 센터 서비스로 보다 나은 고객 서비스를 저비용에 제공할 수 있게 함
- 전세계 Amazon 고객 서비스 직원들이 수 백만 건의 고객 대화에 사용 하는 것과 동일한 기술을 기반으로 함


1. 사용자가 예약 변경을 위해 고객 서비스에 전화
2. Connect가 Lex를 호출하고 Lambda가 데이터베이스를 호출하여 전화 번호를 이용하여 고객 정보를 조회
3. 고객이 예약 관련 질문을 하면 Lambda가 고객 예약 소프트웨어를 불러옴
4. 새로운 예약 날짜가 확인 되면 Connect가 확인 메시지를 SMS를 통해 사용자에게 보냄
5. 사용자가 새로운 예약 일정 세부 내용을 텍스트로 받게됨

**환자 예약**

![환자 예약을 위한 Amazon Lex 구성](/assets/img/Diagrams_lex_messaging-platform.png)


Amazon CloudWatch

- AWS 클라우드 리소스 및 AWS에서 실행 되는 응용프로그램을 위한 모니터링 서비스
- 메트릭, 로그, 알람을 수집하고 자동으로 AWS 리소스 변경에 자동으로 대응할 수 있음


1. 환자가 시설에 3시 예약을 요청
2. Lex가 일정 예약이 요청 되었음을 인식
3. Lex가 예약을 위해 사용자에게 선호하는 요일을 알려 달라고 함
4. 예약 시간을 전달 받음 (Lambda를 통해 DB 접근, SNS 전송, 기타 AWS 서비스 연동)
5. Lex가 문자로 환자에게 목요일 3시에 예약 되었음을 알려 줌


----------

# Analytics

**Amazon Athena - 서버 없는 Query 서비스**

- S3에 있는 데이터를 표준 SQL을 이용하여 쉽게 분석할 수 있으며 쿼리 실행에 대해서만 요금을 내면 됨

**Amazon EMR - Hadoop**

- 많은 데이터를 빠르고 비용 효율적으로 처리하기 위한 Hadoop 프레임워크를 제공
- Apache Spark, HBase, Presto, Flink와 같은 오픈 소스 프레임워크를 실행

**Amazon Elasticsearch Service - Elasticsearch**

- AWS에서 Elasticsearch 배포, 운영, 확장을 쉽게 만들어 줌

**Amazon Kinesis - Streaming Data**

- AWS에서 스트리밍 데이터로 작업하는 가장 쉬운 방법

**Amazon QuickSight - Business Analytics**

- 매우 빠르고 사용이 간편한 클라우드 기반 비지니스 분석 기능으로 기존 BI솔루션의 10분의 1 비용

**Amazon Redshift - Data Warehouse**

- 간단하고 비용효율적으로 분석 할 수 있도록 완벽하게 관리되는 페타바이트 급 데이터웨어하우스

**AWS Glue - ETL**

- 데이터를 만들고 저장

**AWS Data Pipeline - Data Workflow Orchestration**

- 다른 AWS 컴퓨팅 및 스토리지 서비스는 물론 사내 데이터 소스간에 데이터를 지정된 간격으로 안정적으로 처리하고 이동할 수 있음
