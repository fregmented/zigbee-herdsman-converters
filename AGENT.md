# AGENT.md - Zigbee-Herdsman-Converters → SmartThings Edge Driver 지침

이 문서는 zigbee-herdsman-converters(이하 ZHC)의 **디바이스 정의(device definition)**를 입력으로 받아 SmartThings Edge Driver(이하 ST Edge)용 **Device Profile/Capability/Cluster 매핑**을 생성하는 작업을 지시하기 위한 최소 컨텍스트 지침이다.

## 목표
- ZHC의 `src/devices/*.ts`에 있는 **정의 객체**를 읽고, ST Edge 드라이버의 **프로필(capabilities)**, **클러스터 핸들러**, **명령/속성 매핑**으로 변환한다.
- 가능한 한 **ZHC의 modern extend 결과**를 기준으로 기능을 파악하고, **중복/불필요한 변환**을 줄인다.

---

## 입력(소스) 구조 요약
ZHC 디바이스 정의는 대략 다음 구조를 가진다:

```ts
{
    zigbeeModel: ["model_id"],
    model: "PRODUCT_CODE",
    vendor: "Vendor",
    description: "...",
    extend: [ ... ],
    fromZigbee: [ ... ],
    toZigbee: [ ... ],
    exposes: [ ... ],
    endpoint: (device) => ( ... ),
    configure: async (device, coordinatorEndpoint, logger) => { ... },
}
```

> **우선순위:** `extend` → `exposes` → `fromZigbee`/`toZigbee` 순으로 해석한다. `extend`가 있으면 먼저 이를 전개(풀어쓰기)하여 capabilities를 결정한다.

---

## 전개(extend) 규칙
- `extend`는 기능의 **정의 템플릿**이다. ST Edge에서는 **capability 셋** 및 **클러스터 핸들러**로 변환한다.
- 주요 매핑 예시:

| ZHC extend | 의미 | ST Edge 매핑 |
| --- | --- | --- |
| `m.onOff()` | On/Off 스위치 | `switch` capability, OnOff 클러스터(0x0006) |
| `m.levelControl()` | 밝기 | `switchLevel` capability, LevelControl(0x0008) |
| `m.light({color: true})` | 컬러 조명 | `colorControl`, `switch`, `switchLevel`, ColorControl(0x0300) |
| `m.light({colorTemp: true})` | 색온도 조명 | `colorTemperature`, ColorControl(0x0300) |
| `m.temperature()` | 온도 센서 | `temperatureMeasurement` (0x0402) |
| `m.humidity()` | 습도 센서 | `relativeHumidityMeasurement` (0x0405) |
| `m.occupancy()` | 모션/재실 | `motionSensor` (0x0406) |
| `m.contact()` | 접촉 센서 | `contactSensor` (0x0500) |
| `m.battery()` | 배터리 | `battery` (0x0001) |

> extend에 옵션이 있는 경우(예: 범위/스케일)는 ST Edge capability의 **state range/units**에 반영한다.

---

## exposes 해석 규칙
- `exposes`는 **기능의 출력 스키마**에 해당한다.
- `exposes.presets.*` 혹은 개별 `exposes.numeric/binary/enum`은 다음으로 변환한다:

| exposes 타입 | ST Edge 변환 |
| --- | --- |
| `binary` | `switch`, `contactSensor`, `motionSensor` 등 상태형 capability |
| `numeric` | `temperatureMeasurement`, `illuminanceMeasurement`, `powerMeter` 등 측정형 capability |
| `enum` | `mode`, `fanMode` 등 단일값 제어 capability |

- `exposes.access` 플래그를 참고해 **읽기 전용/쓰기 가능** 여부를 결정한다.

---

## fromZigbee / toZigbee 매핑 규칙
- `fromZigbee`: Zigbee → 상태 업데이트
  - 해당 cluster/attribute를 읽어 **ST Edge capability 이벤트**로 변환.
- `toZigbee`: ST Edge 명령 → Zigbee write/command
  - capability command → cluster command로 매핑.

### Tuya 전용 매핑 지침
- Tuya 디바이스는 `extend`에 **tuya 계열 modern extend**가 포함될 수 있다(예: `tuya.modernExtend.*`).
- Tuya 계열 extend가 감지되면, **Zigbee 표준 클러스터 외에 DP(Data Point) 기반 제어/상태**를 사용하는지 확인한다.
- DP 기반인 경우:
  - `toZigbee`의 DP write/command → ST Edge command 핸들러에 매핑
  - `fromZigbee`의 DP report → ST Edge capability 이벤트로 매핑
- Tuya-specific `meta` 옵션(전압 스케일, 값 변환 등)이 있으면 capability 단위/스케일링에 반영한다.

### 대표 클러스터 매핑
| Cluster | ZHC 처리 | ST Edge 처리 |
| --- | --- | --- |
| `genOnOff` (0x0006) | on/off | switch on/off command + state |
| `genLevelCtrl` (0x0008) | level | setLevel + level event |
| `lightingColorCtrl` (0x0300) | hue/sat/ct | setColor, setColorTemperature |
| `msTemperatureMeasurement` (0x0402) | temperature | temperatureMeasurement event |
| `msRelativeHumidity` (0x0405) | humidity | relativeHumidityMeasurement event |
| `msOccupancySensing` (0x0406) | occupancy | motionSensor event |
| `ssIasZone` (0x0500) | contact/motion | contactSensor/motionSensor event |

---

## 함수/속성 처리 지침
- `configure`: 바인딩/리포트 설정. ST Edge에서는 **binding/attribute reporting 설정 단계**로 변환.
- `endpoint`: 엔드포인트 매핑. ST Edge의 **component/endpoint binding**에 반영.
- `fingerprint`/`zigbeeModel`: 디바이스 식별 키로 사용.
- `meta` 설정은 변환 과정에서 **기능 제한/옵션**으로 고려.

---

## 출력(Edge Driver) 산출물
1. **profiles/*.yaml**: capabilities 및 components 정의.
2. **clusters/*.lua** 또는 **handlers**: ZCL cluster 리스너/명령 매핑.
3. **device init/bind**: configure 대응.

각 산출물은 다음 기준으로 최소화한다:
- ZHC 정의의 기능에 **직접 대응되는 capability만 생성**
- `extend`/`exposes`에서 제공되지 않은 기능은 생성하지 않음

---

## 작업 절차(요약)
1. 대상 디바이스 정의 로드
2. `extend` 전개 → 기능 목록 수집
3. `exposes`로 기능 보강/검증
4. `fromZigbee`/`toZigbee`로 command/attribute 매핑 생성
5. ST Edge profiles + handlers 출력

---

## 구현 팁(최소 컨텍스트)
- ZHC는 `extend`가 가장 신뢰할 수 있는 기능 선언이다.
- `exposes`는 UI/기능 스키마를 결정하는 보조 정보다.
- `fromZigbee`/`toZigbee`는 프로토콜 레벨 매핑이며, ST Edge에서는 ZCL Cluster handler로 대응된다.
