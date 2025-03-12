# RTOS_LEDBLICK

```cpp
#include <Arduino.h>
#include <Arduino_FreeRTOS.h>
#include <semphr.h>

// 상수 정의
#define NUM_ADC_CHANNELS 4

// LED blink를 위한 데이터를 위한 구조체
typedef struct {
  int output_pin;          // output pin 번호
  uint32_t samplingInterval; // 샘플링 간격 (ms)
} LEDBlinkData_t;

// 각 채널별 LED 설정 데이터
LEDBlinkData_t blinkPinData[NUM_ADC_CHANNELS] = {
  {5, 100}, // 채널 0: 100ms 간격
  {6, 200}, // 채널 1: 200ms 간격
  {7, 300}, // 채널 2: 300ms 간격
  {8, 400}  // 채널 3: 400ms 간격
};

// 뮤텍스 핸들
SemaphoreHandle_t adcMutex;

// 스택 오버플로우 후크 함수
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
  Serial.print(F("스택 오버플로우 발생: "));
  Serial.println(pcTaskName);
  while (1); // 멈춤
}

// 각 채널별 LED 점멸 태스크
void TaskBlink(void *pvParameters) {
  // 매개변수에서 LED 데이터 구조체 가져오기
  LEDBlinkData_t *pLEDData = (LEDBlinkData_t *)pvParameters;
  
  // LED 핀 초기화
  pinMode(pLEDData->output_pin, OUTPUT);
  
  // LED 상태 변수
  bool ledState = false;
  
  // 무한 루프로 작업 실행
  for (;;) {
    // 뮤텍스 획득 시도
    if (xSemaphoreTake(adcMutex, (TickType_t)10) == pdTRUE) {
      // LED 상태 토글
      ledState = !ledState;
      digitalWrite(pLEDData->output_pin, ledState);

      // 디버깅을 위한 출력
      Serial.print(F("채널 "));
      Serial.print((pLEDData->output_pin) - 5);
      Serial.print(F(": LED "));
      Serial.println(ledState ? F("켜짐") : F("꺼짐"));

      // 뮤텍스 해제
      xSemaphoreGive(adcMutex);
    } else {
      Serial.println(F("뮤텍스 획득 실패"));
    }

    // 설정된 간격만큼 대기
    vTaskDelay(pdMS_TO_TICKS(pLEDData->samplingInterval));
  }
}

void setup() {
  // 시리얼 통신 초기화
  Serial.begin(115200);
  while (!Serial) {
    ; // 시리얼 포트 연결 대기
  }
  
  Serial.println(F("Arduino RTOS 4채널 LED 점멸 시작"));
  
  // 뮤텍스 생성
  adcMutex = xSemaphoreCreateMutex();
  
  if (adcMutex == NULL) {
    Serial.println(F("뮤텍스 생성 실패"));
    while (1); // 오류 시 정지
  }
  
  // Blink 변환 태스크 생성 (각 채널별로)
  for (uint8_t i = 0; i < NUM_ADC_CHANNELS; i++) {
    char taskName[12];
    sprintf(taskName, "Blink_Ch%d", i);
    
    if (xTaskCreate(
      TaskBlink,              // 태스크 함수
      taskName,               // 태스크 이름
      256,                    // 스택 크기 (늘림)
      (void*)&blinkPinData[i], // 파라미터
      1,                      // 우선순위
      NULL                    // 태스크 핸들
    ) == pdPASS) {
      Serial.print(F("태스크 생성: "));
      Serial.print(taskName);
      Serial.print(F(", 핀: "));
      Serial.print(blinkPinData[i].output_pin);
      Serial.print(F(", 간격: "));
      Serial.print(blinkPinData[i].samplingInterval);
      Serial.println(F("ms"));
    } else {
      Serial.print(F("태스크 생성 실패: "));
      Serial.println(taskName);
    }
  }

  Serial.println(F("스케줄러 시작"));
  vTaskStartScheduler();
  Serial.println(F("스케줄러 시작 실패"));  // 이 메시지가 보이면 메모리 문제
}

void loop() {
  // FreeRTOS가 태스크를 처리하므로 loop()는 실행되지 않음
}

```

# 결과

![image.png](attachment:6e0cced0-a7f1-46c7-b1a0-eae1628ceeea:image.png)

# 로직 애널라이저

각각 100ms, 200ms, 300ms,  400ms의 주기

[화면 녹화 중 2025-03-12 195927.mp4](attachment:419014e6-0f71-4a0e-b794-9d4bee377021:화면_녹화_중_2025-03-12_195927.mp4)

![image.png](attachment:320d0357-99a1-4354-a2e5-97de703cd5bd:image.png)

![image.png](attachment:f018a7b7-1e88-471c-8cf9-6d29d20b057b:image.png)

![image.png](attachment:902781c9-56ad-43c5-90e3-eb8944e01395:image.png)

![image.png](attachment:f5ecae82-338e-43df-84a3-258aa7f164c3:image.png)

![image.png](attachment:72d46cd8-1112-415e-b3d2-7284f03b0bb3:image.png)

# 코드 분석

### 1. 헤더 파일 포함

```cpp
#include <Arduino.h>
#include <Arduino_FreeRTOS.h>
#include <semphr.h>

```

- Arduino 기본 라이브러리 포함
- FreeRTOS 관련 함수와 매크로 사용을 위해 포함
- 뮤텍스와 세마포어 사용을 위해 `semphr.h` 포함

---

### 2. 상수 및 구조체 정의

```cpp
#define NUM_ADC_CHANNELS 4
```

- 4개의 ADC 채널(LED 제어)을 정의

### LED 점멸 데이터 구조체

```cpp
typedef struct {
  int output_pin;
  uint32_t samplingInterval;
} LEDBlinkData_t;

```

- LED 점멸에 필요한 정보를 저장
    - `output_pin` : LED가 연결된 핀 번호
    - `samplingInterval` : LED 점멸 주기 (밀리초)

### LED 설정 데이터 배열

```cpp
LEDBlinkData_t blinkPinData[NUM_ADC_CHANNELS] = {
  {5, 100},
  {6, 200},
  {7, 300},
  {8, 400}
};

```

- 각 채널의 LED 핀 번호와 점멸 주기를 구조체 배열로 초기화
- 채널 0~3까지 각기 다른 주기로 LED를 깜박이게 설정

---

### 3. 뮤텍스 핸들 선언

```cpp
SemaphoreHandle_t adcMutex;

```

- 뮤텍스 핸들을 전역 변수로 선언하여 모든 태스크에서 접근 가능

---

### 4. 스택 오버플로우 처리 함수

```cpp
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) 
{
  Serial.print(F("스택 오버플로우 발생: "));
  Serial.println(pcTaskName);
  while (1);
}

```

- 태스크가 할당된 스택을 초과하면 호출됨
- 오버플로우 발생 시 태스크 이름을 시리얼로 출력하고 무한 루프에 진입

---

### 5. LED 점멸 태스크

### 태스크 함수 정의

```cpp
void TaskBlink(void *pvParameters) 
{
  LEDBlinkData_t *pLEDData = (LEDBlinkData_t *)pvParameters;
  pinMode(pLEDData->output_pin, OUTPUT);
  bool ledState = false;

```

- 전달된 구조체 포인터를 사용하여 핀 번호와 간격을 가져옴
- 해당 핀을 출력 모드로 설정
- `ledState` 변수로 LED 상태 관리

---

### LED 점멸 루프

```cpp
  for (;;) 
  {
    if (xSemaphoreTake(adcMutex, (TickType_t)10) == pdTRUE) 
    {
      ledState = !ledState;
      digitalWrite(pLEDData->output_pin, ledState);

      Serial.print(F("채널 "));
      Serial.print((pLEDData->output_pin) - 5);
      Serial.print(F(": LED "));
      Serial.println(ledState ? F("켜짐") : F("꺼짐"));

      xSemaphoreGive(adcMutex);
    } 
    else 
    {
      Serial.println(F("뮤텍스 획득 실패"));
    }

    vTaskDelay(pdMS_TO_TICKS(pLEDData->samplingInterval));
  }
}

```

- 뮤텍스 획득 시도
    - `xSemaphoreTake(adcMutex, (TickType_t)10)`로 10 틱 동안 뮤텍스 대기
    - 뮤텍스 획득 성공 시 LED 상태를 반전하여 점멸
    - 직렬로 LED 상태를 출력하고, 뮤텍스를 해제
    - 실패 시 "뮤텍스 획득 실패" 메시지 출력
- LED 상태 제어
    - `ledState = !ledState`로 LED 상태를 반전
    - `digitalWrite()`로 LED를 켜거나 끔
- 대기
    - `vTaskDelay(pdMS_TO_TICKS(pLEDData->samplingInterval))`로 점멸 간격 조정

---

### 6. 설정 함수 (setup)

```cpp
void setup() 
{
  Serial.begin(115200);
  while (!Serial);

  Serial.println(F("Arduino RTOS 4채널 LED 점멸 시작"));
  adcMutex = xSemaphoreCreateMutex();

  if (adcMutex == NULL) 
  {
    Serial.println(F("뮤텍스 생성 실패"));
    while (1);
  }

```

- 시리얼 통신 초기화
- 뮤텍스 생성
    - 뮤텍스 생성 실패 시 무한 루프에 진입하여 실행 중단

---

### 태스크 생성

```cpp
 for (uint8_t i = 0; i < NUM_ADC_CHANNELS; i++) 
 {
    char taskName[12];
    sprintf(taskName, "Blink_Ch%d", i);

    if (xTaskCreate(
      TaskBlink,
      taskName,
      256,
      (void*)&blinkPinData[i],
      1,
      NULL
    ) == pdPASS) 
    {
      Serial.print(F("태스크 생성: "));
      Serial.println(taskName);
    } 
    else 
    {
      Serial.print(F("태스크 생성 실패: "));
      Serial.println(taskName);
    }
  }

  vTaskStartScheduler();
  Serial.println(F("스케줄러 시작 실패"));
}

```

- `xTaskCreate()`로 4개의 LED 제어 태스크를 각각 생성
    - 스택 크기: 256
    - 우선순위: 1
    - 각 채널별로 태스크 이름을 지정하여 생성
- 스케줄러 시작
    - `vTaskStartScheduler()`로 FreeRTOS 스케줄러를 시작
    - 시작 실패 시 메시지 출력

---

### 7. 메인 루프 (loop)

```cpp
void loop() 
{
  // FreeRTOS가 태스크를 처리하므로 loop()는 실행되지 않음
}

```

- FreeRTOS가 태스크를 주도하므로 메인 루프는 실행되지 않음
