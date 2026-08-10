---
title: "Amazon SES이란 무엇인가요? - Amazon Simple Email Service"
source: "https://docs.aws.amazon.com/ko_kr/ses/latest/dg/Welcome.html"
author:
published:
created: 2026-08-07
description: "Amazon SES를 사용하여 특별 제안과 같은 마케팅 이메일, 주문 확인서와 같은 거래 이메일, 뉴스레터 및 시스템 알림과 같은 기타 유형의 통신문을 발송합니다. 또한 Amazon SES를 사용하여 이메일을 수신할 수 있습니다. Amazon SES를 사용하여 메일을 수신하면 다른 메일 서버와 통신, 스팸 및 바이러스 검사, 신뢰할 수 없는 소스의 메일 거부, 도메인 내의 수신자에 대한 메일 수락 등의 기본 메일 수신 작업이 Amazon SES에서 처리됩니다."
tags:
  - "clippings"
---
Amazon SES이란 무엇인가요? - Amazon Simple Email Service

기계 번역으로 제공되는 번역입니다. 제공된 번역과 원본 영어의 내용이 상충하는 경우에는 영어 버전이 우선합니다.

기계 번역으로 제공되는 번역입니다. 제공된 번역과 원본 영어의 내용이 상충하는 경우에는 영어 버전이 우선합니다.

[Amazon Simple Email Service(SES)](https://aws.amazon.com/ses) 는 사용자의 이메일 주소와 도메인을 사용해 이메일을 보내고 받기 위한 경제적이고 손쉬운 방법을 제공하는 이메일 플랫폼입니다.

예를 들어, 특별 행사 안내 등의 마케팅 이메일, 주문 확인서 등의 거래 이메일, 뉴스레터 등의 기타 통신문을 발송할 수 있습니다. Amazon SES를 사용하여 메일을 수신하면 이메일 자동 응답기, 이메일 구독 해제 시스템, 수신 이메일에서 고객 지원 티켓을 생성하는 애플리케이션과 같은 소프트웨어 솔루션을 개발할 수 있습니다.

Amazon SES에 관련된 다양한 주제에 대한 자세한 내용은 [AWS메시징 및 타겟팅 블로그](https://aws.amazon.com//blogs/messaging-and-targeting/) 를 참조하세요.

## 이점

대규모 이메일 솔루션을 구축하는 것은 비즈니스에 있어서 고비용의 복잡한 작업입니다. 이메일 서버 관리, 네트워크 구성, IP 주소 신뢰도 등의 인프라 문제를 해결해야 하기 때문입니다. 또한 많은 타사 이메일 솔루션에는 계약 및 가격 협상은 물론 상당한 초기 비용이 필요합니다. Amazon SES를 사용하면 이러한 문제들을 해결하여 Amazon.com이 대규모 고객층을 위해 구축한 고급 이메일 인프라와 다년간의 경험을 마음껏 이용할 수 있습니다.

## 관련 서비스

Amazon SES는 다른AWS제품과 원활하게 통합됩니다. 예를 들어, 다음을 수행할 수 있습니다.

- 애플리케이션에 이메일 전송 기능을 추가합니다.
- [AWS SDK](https://aws.amazon.com/tools/#sdk) 를 사용하거나, [Amazon SES SMTP 인터페이스](https://docs.aws.amazon.com/ko_kr/ses/latest/dg/send-email-smtp.html) 를 사용하거나, [Amazon SES API](https://docs.aws.amazon.com/ses/latest/APIReference/) 를 직접 호출하여 Amazon EC2에서 이메일을 보낼 수 있습니다,
- [AWS Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/) 를 사용하여 이메일 지원 애플리케이션(예를 들어 Amazon SES를 사용하여 고객에게 뉴스레터를 발송하는 프로그램)을 작성할 수 있습니다.
- 반송되었거나 수신 거부가 제출되었거나 수신자의 메일 서버로 성공적으로 전송된 이메일에 대한 알림을 수신하도록 [Amazon Simple Notification Service(Amazon SNS)](https://aws.amazon.com/sns/) 를 설정할 수 있습니다. Amazon SES로 이메일을 수신하면 이메일 콘텐츠를 Amazon SNS 주제에 게시할 수 있습니다.
- AWS Management Console를 사용하여 이메일을 인증하는 방법인 Easy DKIM을 설정합니다. 어떤 DNS 공급자에서도 Easy DKIM을 사용할 수 있지만, [Route 53](https://aws.amazon.com/route53/) 을 사용하여 도메인을 관리할 경우 설정이 특히 용이합니다.
- [AWS Identity and Access Management(IAM)](https://aws.amazon.com/iam/) 을 사용하여 이메일 전송에 대한 사용자 액세스를 제어할 수 있습니다.
- [Amazon Simple Storage Service(Amazon S3)](https://aws.amazon.com/s3/) 에서 수신한 이메일을 저장합니다.
- [AWS Lambda](https://aws.amazon.com/lambda/) 함수를 트리거하여 수신 이메일에 여러 가지 작업을 수행할 수 있습니다.
- 필요에 따라 [AWS Key Management Service(AWS KMS)](https://aws.amazon.com/kms/) 를 사용해 Amazon S3 버킷에 수신하는 메일을 암호화할 수 있습니다.
- [AWS CloudTrail](https://aws.amazon.com/cloudtrail/) 를 사용하여 콘솔 또는 Amazon SES API를 통해 생성한 Amazon SES API 호출을 기록할 수 있습니다.
- 이메일 전송 이벤트를 [Amazon CloudWatch](https://aws.amazon.com/cloudwatch/) 또는 [Amazon Data Firehose](https://aws.amazon.com/firehose/) 에 게시할 수 있습니다. 이메일 전송 이벤트를 Firehose에 게시하면 [Amazon Redshift](https://aws.amazon.com/redshift/), [Amazon OpenSearch Service](https://aws.amazon.com/elasticsearch-service/) 또는 [Amazon S3](https://aws.amazon.com/s3/) 에서 액세스할 수 있습니다.

## 가격 책정

Amazon SES를 사용하면 전송 및 수신된 이메일의 볼륨에 따라 비용을 지불할 수 있습니다. 자세한 내용은 [Amazon SES 요금](https://aws.amazon.com/ses/pricing/) 을 참조하세요.

- ### 이 페이지에서
- 예
	아니요