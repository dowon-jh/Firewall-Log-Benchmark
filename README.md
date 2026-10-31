# Firewall Log Benchmark

방화벽 로그 1,000,000건을 Logstash, Elasticsearch, Kibana로 파싱/정제/집계하는 로컬 벤치마크 프로젝트입니다. Logstash worker 1·4·8 조건에서 처리 시간, 구간별 EPS, CPU·메모리와 집계 결과를 비교했습니다.

**수행 기간:** 2026.07.13 ~ 2026.07.19  
**작성자:** 장도원

## 1. 프로젝트 개요

### 목적

방화벽 로그의 파싱 오류를 해결하고, 같은 입력에서 worker 수에 따른 처리 시간과 자원 사용량을 비교했습니다. 처리 속도뿐 아니라 집계 결과가 같은지도 확인하는 것이 목적입니다.

### 주요 결과

- header 형식과 맞지 않는 grok 패턴을 실패 문서의 태그로 확인하고, 고정된 구분자·위치에 맞춰 dissect로 변경했습니다.
- 빈 값 때문에 뒤의 `src_ip` 구문이 앞 필드의 값으로 읽히는 문제를 kv 적용 전 입력 보정으로 처리했습니다.
- 측정 소요 시간은 worker 1·4·8에서 410초·172초·149초였습니다. 이 시간에는 컨테이너 시작·API 준비·집계 완료 대기가 포함됩니다.
- 세 조건 모두 집계 결과의 `sum(event_count)`가 1,000,000이고 실패 인덱스 건수는 0이었습니다.
- **총량 일치는 확인했지만, 키·시간 구간별 정확성 검증은 완료하지 않았습니다.** worker별 출력 행 수가 달랐고, worker 1 CSV에서도 동일 키가 여러 행으로 나뉜 것이 확인됐습니다.

자세한 측정 조건·수치·근거 범위는 [벤치마크 결과 문서](docs/benchmark-results.md)에 정리했습니다.

### 현재 공개 범위

기존 실험 보고서, 실행 화면, 집계 CSV를 대조해 처리 과정과 결과를 정리했습니다. 문서 정리 단계에서는 worker 1 CSV를 다시 검산했습니다.

현재 공개한 파일은 README와 벤치마크 결과 문서입니다. 실험 당시 Docker Compose 설정, Logstash 설정, 측정 스크립트 원본은 확보하지 못했습니다. 재실행하려면 해당 설정과 입력 파일이 필요합니다.

## 2. 실행 환경

| 항목 | 설정 |
|---|---|
| 실행 환경 | WSL·Docker Compose 기반 로컬 단일 노드 ELK |
| Elasticsearch / Logstash / Kibana | 각각 8.15.3 |
| 입력 | 방화벽 로그 파일 1,000,000건 |
| JVM heap | Logstash·Elasticsearch 각각 `-Xms1g -Xmx1g` |
| Pipeline batch size | 125 |
| 비교 조건 | Logstash worker 1 / 4 / 8 |
| 지표 수집 | Shell, Logstash API, Docker stats, Python JSON 값 추출 |

호스트 CPU·RAM, Docker·WSL 자원 제한, 입력 파일 해시는 기록에서 확인하지 못했습니다. 결과를 비교할 때 이 조건을 함께 확인해야 합니다.

## 3. 아키텍처

```text
방화벽 로그 파일
  → dissect: syslog header / key=value 본문 분리
  → mutate gsub: 값이 빈 항목 보정
  → kv: 본문을 필드로 분해
  → date: 로그 발생 시각을 @timestamp에 반영
  → mutate: source / destination 필드 정규화
  → 필수 필드 확인
  → aggregate: IP·포트·20초 구간 키로 누적
  → Elasticsearch: 집계 결과 저장

파싱·시각 변환·필수 필드 검증 실패
  → 원문·오류·실패 단계 보존
  → 실패 전용 인덱스
```

## 4. Logstash 정제 방식과 트러블슈팅

### 4.1 Header와 본문 분리

원본은 다음과 같은 구조였습니다. 아래는 구조를 설명하기 위한 축약 표기입니다.

```text
<priority>version event_time [log_type] [observer_ip] key=value body
```

처음 적용한 grok 패턴이 실제 timestamp/header 형식과 맞지 않아 header 파싱이 실패했습니다. 실패 인덱스에 문서가 저장되는지와 오류 태그를 확인하여, 입력·저장 경로와 header 파싱 문제를 나누어 확인했습니다.

Header의 구분자와 위치가 일정한 점을 확인한 뒤 dissect로 변경했습니다. 변경 후 header 파싱이 진행되는 것을 확인했습니다. grok와 dissect의 처리 속도는 별도로 비교하지 않았습니다.

### 4.2 빈 값으로 인한 key=value 파싱 오류 보정

다음 형태에서는 뒤의 `src_ip` 구문이 `nat_rule_name` 값으로 읽혀, 집계에 필요한 `source.ip`가 생성되지 않는 문제가 있었습니다.

```text
보정 전: nat_rule_name= src_ip=10.0.0.237
보정 후: nat_rule_name=- src_ip=10.0.0.237
```

kv 적용 전에 mutate gsub로 `nat_rule_name`, `fragment_info`의 빈 값을 `-`로 보정한 뒤 본문을 파싱했습니다. 보정 대상은 입력에서 빈 값이 확인된 두 필드입니다.

### 4.3 필드 정규화와 시각 처리

| 원본 / 입력값 | 정제 후 필드 | 용도 |
|---|---|---|
| `src_ip` | `source.ip` | 출발지 식별 |
| `dst_ip` | `destination.ip` | 목적지 식별 |
| `dst_port` | `destination.port` | 목적지 포트 구분 |
| 로그 발생 시각 | `@timestamp` | 처리 시각 대신 로그 시각 사용 |
| 로그 시각의 20초 구간 시작값 | `window_start_epoch` | 집계 키에 시간 구간 포함 |

파싱 결과 실행 화면에는 `source.ip`, `destination.ip`, 숫자형 포트와 변환된 `@timestamp`가 표시됩니다. 구체적인 정규식, date format 목록, 타입 변환 구문은 설정 원본을 확보한 뒤 별도로 확인해야 합니다.

### 4.4 집계 키와 결과

```text
source.ip | destination.ip | destination.port | window_start_epoch
```

예를 들어 `00:00:00Z`부터 시작하는 구간은 `window_start=00:00:00Z`, `window_end=00:00:20Z`로 표현했습니다. 결과에는 다음 필드가 포함됩니다.

- 키·구간: `aggregate_key`, `window_start`, `window_end`, `window_start_epoch`, `window_end_epoch`
- 이벤트 시각: `first_event_time`, `last_event_time`
- 분류: `source.ip`, `destination.ip`, `destination.port`, `protocol`, `fw_rule_name`, `firewall_log_type`
- 누적값: `event_count`, `bytes_total_sum`, `packets_total_sum`

키에 로그 발생 시각의 구간을 넣는 것과, 해당 구간의 모든 이벤트를 한 행으로 정확히 모으는 것은 별개의 검증입니다. 보고서에는 aggregate timeout도 20초로 기록돼 있지만, timeout을 event-time watermark 또는 지연 이벤트 처리 보장으로 해석하지 않습니다.

### 4.5 실패 분리와 사후 확인

실패 단계는 `header_parse`, `kv_parse`, `date_parse`, `required_field_validation`으로 구분했습니다. 실패 인덱스에는 원문과 태그, 오류 종류, 실패 단계 정보를 남겼습니다. Kibana Discover로 원문·오류를 확인하고, Elasticsearch count·sum 조회로 처리 건수를 대조했습니다.

집계 전 실행 화면에서 파싱 결과 인덱스 1,000,000건 적재를 확인했습니다. worker 비교 결과에서는 실패 인덱스 0건과 `sum(event_count)=1,000,000`을 확인했습니다. 실패 인덱스 0건만으로 모든 필드의 의미가 정확하다고 결론 내리지는 않습니다.

## 5. 측정 자동화와 트러블슈팅

- 완료 후 한 번의 통계 조회만으로는 구간별 EPS와 처리 중 자원 사용량을 확인하기 어려워, 반복 샘플링 스크립트를 작성했습니다.
- API 응답에서 `events.in`, `events.out`, `sum(event_count)`가 비어 나오는 문제는 JSON 전달·추출 방식, API 준비 대기와 재조회 절차를 보완하여 처리했습니다.
- 지표가 비어 있을 때 Elasticsearch 집계 결과를 먼저 확인해, 데이터 처리 실패와 지표 수집 실패를 구분했습니다.
- worker와 실행 시각별로 측정 결과를 저장했습니다. 현재 문서의 수치는 보고서·실행 화면에 남은 값이며, 결과 디렉토리 전체를 확보한 상태는 아닙니다.

## 6. 벤치마크 결과

| worker | 측정 소요 시간 | 샘플 구간 평균 EPS | 집계 결과 행 수 | event_count 합계 | 실패 건수 |
|---|---:|---:|---:|---:|---:|
| 1 | 410초 | 2,766.90 | 511,500 | 1,000,000 | 0 |
| 4 | 172초 | 8,208.19 | 261,502 | 1,000,000 | 0 |
| 8 | 149초 | 8,931.39 | 244,085 | 1,000,000 | 0 |

측정 소요 시간에는 컨테이너 시작과 완료 대기가 포함됩니다. EPS는 일부 샘플 구간의 요약값입니다.

worker 4에서 8로 늘렸을 때 소요 시간은 23초 줄었지만, 집계 결과 행 수도 달라졌습니다. worker 1 CSV에서는 76,500개 고유 키가 511,500개 행으로 나뉜 것을 확인했습니다. 따라서 총량 일치와 키·시간 구간별 집계 정확성을 구분했습니다.

상세 조건, 자원 사용량, CSV 검산 절차: [벤치마크 결과](docs/benchmark-results.md)

## 7. 대체 도구 검토

실측은 Logstash 조건에 한정했습니다. Vector와 Fluent Bit는 문서 비교와 전환 시 검증 항목 정리까지 수행했습니다.

| 후보 | 검토한 배치 역할 | 아직 측정하지 않은 항목 |
|---|---|---|
| Vector | 파싱·변환·집계 파이프라인 대체 후보 | 동일 입력의 파싱 결과, 구간별 집계, 순서 역전 처리, 처리 시간·자원 사용량 |
| Fluent Bit | 로그 수집·전처리 후보 | 빈 값 처리, 별도 집계 계층과의 조합, 버퍼링, CPU·메모리 사용량 |

대체 도구의 처리 시간과 자원 사용량은 추가 실측이 필요합니다.

## 8. 향후 검증 방향

1. 설정·측정 스크립트 원본과 입력 버전·장비 조건 확보
2. 독립적인 기준 집계와 키별 `event_count`·바이트·패킷 합계 대조
3. 동일 키가 여러 행으로 나뉜 원인 확인 및 이벤트 순서·timeout 조건별 실험
4. 동일 조건 반복 실행과 처리 구간 전체의 지표 수집
5. 정확성 기준을 통과한 조건끼리 성능 비교

이 항목들은 추가 검증안이며 완료한 작업이 아닙니다.

## 9. 문서 구성

```text
README.md
  → 프로젝트 개요, 정제 방식, 트러블슈팅, 결과 요약
docs/benchmark-results.md
  → 측정 조건, 지표 정의, worker별 수치, CSV 재검증과 한계
```

관련 프로젝트: [Security-Log-ETL](https://github.com/dowon-jh/Security-Log-ETL)
