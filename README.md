# 페달 오조작 방지 보조 시스템

> **텔레칩스 차량용 반도체 임베디드 스쿨 최종 프로젝트 | 2023.11.01 ~ 2023.12.17**  
> **팀 프로젝트 | 실시간 운영체제를 활용한 차량 제어 시스템**

<br>

**페달 오조작 방지 보조 시스템**은 페달 오조작 발생 시, `차량의 구동력을 제어하여 차량의 충돌을 방지하는 보조 시스템`입니다. 페달 오조작으로 인한 사고 발생률이 증가함에 따라, 전 세계적으로 페달 오조작 방지 장치의 도입이 적극 추진되고 있습니다. 이에 따라 **페달 오조작 방지 보조 시스템**을 개발하고자 하였으며, 2024년에 출시된 **현대자동차 캐스퍼 일렉트릭 차량에 탑재된 PMSA(페달 오조작 안전 보조)를 참고**하여 해당 프로젝트를 진행하였습니다.

![overview](https://github.com/junghyun21/pedal-misapplication-prevention/blob/main/img/overview.png)

`페달 오조작`: 운전자의 의도와 다르게 발생한 비정상적인 페달 입력
  
  - 정차 또는 정차 후 출발하여 저속 주행 중일 때, 전/후방에 장애물이 존재하는 상황에서 가속 페달이 강하게 밟힌 경우
  - 오토 홀드 기능이 활성화된 상태에서 브레이크를 밟지 않고 정지 중일 때, 전/후방에 장애물이 있거나 적색 신호등이 점등된 상태에서 가속 페달이 조금이라도 밟힌 경우

<br>

[📜 프로젝트 최종 발표 자료](https://github.com/junghyun21/pedal-misapplication-prevention/blob/main/presentation.pdf)

> [!IMPORTANT]  
> 본 프로젝트에는 텔레칩스 사의 내부 디바이스 드라이버 코드가 포함되어 있어, 해당 코드의 라이선스 및 보안 정책에 따라 전체 소스 코드를 공개할 수 없습니다.

<br>

## 프로젝트 개요

| 항목 | 세부내용 |
|:--------------:|--------------------------------------|
| **사용 보드**  | 3개의 TOPST D3 보드 <br>(TOPST D3: Cortex-A72, Cortex-A53, Cortex-R5를 포함하는 멀티코어 보드) |
| **운영체제**  | Cortex-R5: FreeRTOS <br>Cortex-A72: Embedded Linux (Linux version: 5.4.159) |
| **통신 방식**  | 코어 간 통신: IPC (Mailbox) <br>보드 간 통신: UART |
| **센서 (입력 장치)** | 가속/브레이크 조이스틱 (ADC 변환 후 입력값 처리)<br>3단 토글 스위치 (GPIO 입력)<br>푸쉬 버튼 스위치 (GPIO 입력)<br>초음파 센서 (장애물 거리 측정)<br>카메라 모듈 (적색 신호등 감지) |
| **출력 장치** | 모터 (PWM 기반 속도 제어)<br>속도값 출력 7-Segment LED<br>오토홀드/주행모드 상태 LED<br>경고음 부저 (오조작 발생 시 알람) |
| **주 담당 업무** | IPC 통신 구현, UART 통신 구현, RTOS 태스크 설계, 오조작 알고리즘 설계, 코드 통합 |

<br>

## 개발 환경 세팅

**Yocto Project 기반의 autolinux 활용**

텔레칩스(Telechips) 사에서 제공하는 임베디드 리눅스 개발을 위한 도구인 `autolinux`를 활용하여 각 코어 별 개발 환경을 셋팅하였습니다. autolinux란, **Yocto Project를 기반으로 하는 빌드 자동화 시스템**입니다.

- **Cortex-A72**: autolinux를 통해 U-Boot, Kernel, Device Tree, Root File System, Home Directory를 생성
- **Cortex-R5**: autolinux를 통해 Image(`cr5_snor.rom`) 생성

<br>

## 시스템 아키텍처

**Zonal Architecture**

자동차 산업이 SDV(Software Defined Vehicle) 시대로 전환됨에 따라, 개별적으로 작동하는 ECU를 통합하고 중앙 집중식 SoC로 제어하는 Zonal Architecture로의 전환이 필요하다고 업계에서 평가하고 있습니다. 이에 따라 향후 필수적으로 요구될 설계 개념을 경험해보고자 **Zonal Architecture**를 모사하여 시스템 아키텍처를 설계하였습니다.

![architecture](https://github.com/junghyun21/pedal-misapplication-prevention/blob/main/img/architecture.png)

### 전방 보드

전방의 모터 속도를 제어하며, 오조작 발생 상황 판단에 필요한 정보를 중앙 보드로 전달합니다.

![front_board](https://github.com/junghyun21/pedal-misapplication-prevention/blob/main/img/front_board.png)

| 코어 | 역할 |
|:---:| --- |
| **Cortex-R5** | - 중앙 보드로부터 수신한 모터 속도(PWM Duty)와 오조작 발생 여부 정보를 바탕으로 모터를 제어<br>- 초음파 센서를 이용해 전방 장애물과의 거리를 측정하고, 장애물이 위험 거리 내에 있는지 판단하여 그 결과를 중앙 보드로 전송 <br>- Cortex-A72(Main Core)에서 받은 적색 신호등 감지 결과를 중앙 보드로 전달 |
| **Cortex-A72** | - 카메라를 이용해 적색등 감지 여부를 판단하고, 해당 정보를 Cortex-R5(MICOM)으로 전송 |

### 후방 보드

후방의 모터 속도를 제어하며, 오조작 발생 상황 판단에 필요한 정보를 중앙 보드로 전달합니다.

![rear_board](https://github.com/junghyun21/pedal-misapplication-prevention/blob/main/img/rear_board.png)

| 코어 | 역할 |
|:---:| --- |
| **Cortex-R5** | - 중앙 보드로부터 수신한 모터 속도(PWM Duty)와 오조작 발생 여부 정보를 바탕으로 모터를 제어<br>- 초음파 센서를 이용해 후방 장애물과의 거리를 측정하고, 장애물이 위험 거리 내에 있는지 판단하여 그 결과를 중앙 보드로 전송 |

### 중앙 보드

다양한 입력 장치 및 전후방보드에서 전달받은 정보를 종합하여 오조작 상황 최종 판단하고, 전후방의 모터 속도를 조절하는 핵심 제어 역할을 수행합니다.

![mid_board](https://github.com/junghyun21/pedal-misapplication-prevention/blob/main/img/mid_board.png)

| 코어 | 역할 |
|:---:| --- |
| **Cortex-R5** | - 다양한 입력 장치를 통해 모터 속도(PWM Duty), 주행 모드(D,R,P), 오토홀드 기능 활성화 여부를 업데이트<br>- 업데이트된 값과 전후방 보드로부터 수신한 데이터를 바탕으로 오조작 발생 여부 판단<br>- 모터 속도와 오조작 발생 여부 등 여러 데이터를 전후방 보드 및 Cortex-A72(main core)로 전송 |
| **Cortex-A72** | - Cortex-R5(MICOM)로부터 수신한 데이터를 다양한 출력 장치를 통해 사용자에게 시각/청각적으로 표시 |

<br>

## 오조작 판단 알고리즘

`페달 오조작`: 운전자의 의도와 다르게 발생한 비정상적인 페달 입력
  
  - 정차 또는 정차 후 출발하여 저속 주행 중일 때, 전/후방에 장애물이 존재하는 상황에서 가속 페달이 강하게 밟힌 경우
  - 오토 홀드 기능이 활성화된 상태에서 브레이크를 밟지 않고 정지 중일 때, 전/후방에 장애물이 있거나 적색 신호등이 점등된 상태에서 가속 페달이 조금이라도 밟힌 경우

![abnormal](https://github.com/junghyun21/pedal-misapplication-prevention/blob/main/img/abnormal.png)

페달 오조작이 발생하면 즉시 모터를 제어하여 차량을 정지시키고, 부저를 통해 경고음을 출력하여 운전자에게 이를 알립니다. 오조작 상황을 해제하기 위해 운전자는 브레이크를 강하게 밟아야합니다.  
(오조작 발생 여부를 판단하는 태스크에 가장 높은 우선순위를 할당하여, 정확한 주기마다 태스크가 안정적으로 실행되도록 하였습니다.)

### 오조작 상황 발생 확인 (`check_start_abnormal()`)

**오토홀드 기능 활성화 여부**에 따라 오조작 상황 판단 방식이 달라집니다.

![start_abnormal](https://github.com/junghyun21/pedal-misapplication-prevention/blob/main/img/start_abnormal.png)

> 오토홀드 기능이 활성화되지 않은 경우

1. 장애물이 일정 범위 내 존재할 때, 가속 페달이 강하게 밟혔는지 확인 
2. 0.25초 내에 차량이 정차 또는 정차 후 저속 주행 상태인 적이 있는지 확인
3. 위 두 개의 조건이 모두 충족한다면 오조작이 발생한 것으로 판단

> 오토홀드 기능이 활성화된 경우

1. 차량이 정차 후, 브레이크를 밟지 않아도 정차를 유지하는 상태인지 확인
2. 장애물이 일정 범위 내 존재하거나, 주행모드가 D이며 전방에 적색 신호등이 점등되었을 때, 가속 페달이 살짝이라도 밟혔는지 확인
3. 위 두 개의 조건이 모두 충족한다면 오조작이 발생한 것으로 판단

### 오조작 상황 종료 확인 (`check_finish_abnormal()`)

운전자가 **브레이크를 강하게 밟으면** 오조작 상황이 해제되어, 차량이 정상적으로 움직이고 경고음도 종료됩니다.

![finish_abnormal](https://github.com/junghyun21/pedal-misapplication-prevention/blob/main/img/finish_abnormal.png)

<br>

## 협업 과정

### 담당 업무

| 담당 업무 | 세부 사항 |
| :-----: | ------- |
| 코어 간 통신 | - **IPC(Mailbox)**를 통해 Cortex-A72와 Cortex-R5 간의 양방향 통신 구현<br>- Cortex-A72: mailbox 초기화 시 등록되는 **수신 콜백 함수** 및 **`ioctl()` 시스템콜 기반의 전송** 방식 구현<br>- Cortex-R5: SAL(System Adaptation Layer) 계층에서 mailbox를 통해 송수신하는 **API를 구현**|
| 보드 간 통신 | - 각 보드에 내장된 **Cortex-R5 코어 간 UART 연결** 구성<br>- 중앙 보드에서 UART 채널 2개를 사용해 전방/후방 보드와 각각 통신<br>- 기존의 UART 디바이스 드라이버를 사용하여 송수신 **API를 구현** |
| RTOS 태스크 설계 | - 태스크의 종류, 주기, 우선 순위를 종합적으로 설계<br>- **이벤트 기반 태스크를 구현**하여, 오조작 발생 시 전후방 보드의 모터를 즉각 제어하도록 설계 |
| 오조작 알고리즘 설계 | - 원형 큐를 사용해 0.25초 내의 차량 속도를 특정 시간 간격으로 샘플링하여 최근 상태 추적<br>- 이전 속도와 현재 속도의 차이가 일정 임계값 이상 급변하면 오조작으로 판단하고, 즉시 안전 제어 로직(모터 정지, 경고음 출력) 실행 |
| 코드 통합 | - 세 개의 보드에 대한 프로젝트 구조(디렉토리/빌드 스크립트)를 통일하여, 유지보수 및 확장성 확보 |

### 협업 툴

- **Notion**: 회의록, 멘토링, 분석 내용 정리 등의 협업 자료 정리 
- **[GitHub](https://github.com/junghyun21/topst_d3)**: 버전 별로 코드 관리 (라이선스 및 보안 정책에 따라 코드 공개 불가)
  
  ![github](https://github.com/junghyun21/pedal-misapplication-prevention/blob/main/img/github.png)

### 소스 코드 관리 (보드 별 빌드 방법)

Cortex-A72는 기존에 빌트인 되어 있는 커널 모듈의 소스 코드를 수정하여 컴파일 및 빌드를 진행하였습니다.  
Cortex-R5는 [프로젝트만을 위한 디렉토리](https://github.com/junghyun21/pedal-misapplication-prevention/blob/main/cr5_src/project)를 생성하여, 해당 디렉토리 내에서 세 가지 보드(전방, 중앙, 후방)에 대한 개발 및 빌드가 모두 가능하도록 소프트웨어를 관리하였습니다. 그 이유는 다음과 같습니다.

- 3개의 보드에서 공통적으로 사용하는 기능 존재 (ex. 모터 제어, 보드 간 통신)
- 구현 기능을 기준으로 역할을 분배하였기 때문에, 각 팀원들을 여러 보드에 대해 작업 진행

다음은 Cortex-R5에서 사용한 프로젝트 디렉토리의 구조 및 빌드 방법입니다.

![project](https://github.com/junghyun21/pedal-misapplication-prevention/blob/main/img/project.png)

![build](https://github.com/junghyun21/pedal-misapplication-prevention/blob/main/img/build.png)

<br>

## 참고 자료

- [TOPST D3](https://topst.ai/product/p/d3)
