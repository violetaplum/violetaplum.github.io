---
title: "Event-Driven 뱅킹 서비스 구현하기: Go와 CQRS 패턴"
date: 2025-02-27T09:00:00+09:00
description: kafka, opentelemetry controller, jaeger 맛보기
menu:
  sidebar:
    name: Event-Driven 뱅킹 서비스
    identifier: Event-Driven 뱅킹 서비스
    weight: 40
hero: hero.jpg
mermaid: true
tags: ["Go", "CQRS", "Event Sourcing", "Kafka", "Microservices", "OpenTelemetry", "Docker"]
categories: ["Backend Development", "Architecture"]
---

## 소개

CQRS(Command Query Responsibility Segregation)와 Event Sourcing 패턴에 관심이 많았는데, 이번 토이 프로젝트를 통해 이 두 패턴을 함께 구현해 봤다. '초' 간단한 뱅킹 서비스를 예시로 삼아, Go 언어로 개발하고 Kafka, PostgreSQL, OpenTelemetry와 같은 현대적인 기술 스택을 사용해 봤다.

## 프로젝트 아키텍처

이 프로젝트는 크게 다음과 같은 컴포넌트로 구성되어 있다:

1. **Account API**: 계좌 생성, 입금, 출금 등의 명령(Command)을 처리하는 서비스
2. **Event Processor**: 이벤트를 소비하고 조회(Query) 모델을 업데이트하는 서비스
3. **Postgres**: 이벤트 저장소 및 조회 모델 데이터베이스
4. **Kafka**: 이벤트 메시징 시스템 (분산 처리)
5. **OpenTelemetry**: 분산 추적 및 모니터링 시스템
6. **Jaeger**: 트레이싱 데이터 시각화 (UI 용도)

전체 아키텍처는 다음과 같은 흐름으로 동작한다:

``` 
사용자 요청 -> Account API -> 이벤트 생성 -> Kafka -> Event Processor -> 조회 모델 업데이트
                      |                                     |
                      v                                     v
                  이벤트 저장 -------------------------> 이벤트 읽기
                  (PostgreSQL)                        (PostgreSQL)
```

## 도메인 설계

도메인 설계는 DDD(Domain-Driven Design)의 원칙을 따라 구현했다. 핵심 도메인 객체는 다음과 같다:

### Account

```go
// domain/account.go (일부)
type Account struct {
    ID        string
    OwnerName string
    Balance   int64
    CreatedAt time.Time
    UpdatedAt time.Time
}

type AccountRepository interface {
    Save(ctx context.Context, account *Account) error
    FindByID(ctx context.Context, id string) (*Account, error)
    // ... 기타 메서드
}
```

### Transaction

```go
// domain/transaction.go (일부)
type TransactionType string

const (
    Deposit  TransactionType = "DEPOSIT"
    Withdraw TransactionType = "WITHDRAW"
)

type Transaction struct {
    ID          string
    AccountID   string
    Amount      int64
    Type        TransactionType
    Description string
    CreatedAt   time.Time
}
```

### Event

```go
// domain/event.go (일부)
type EventType string

const (
    AccountCreated EventType = "ACCOUNT_CREATED"
    MoneyDeposited EventType = "MONEY_DEPOSITED"
    MoneyWithdrawn EventType = "MONEY_WITHDRAWN"
)

type Event struct {
    ID        string
    Type      EventType
    Data      []byte
    Timestamp time.Time
    Version   int
}
```

## CQRS 패턴 구현

CQRS 패턴은 명령(Command)과 조회(Query)의 책임을 분리하는 패턴이다. 이 프로젝트에서는 다음과 같이 구현했다:

### Command 서비스

```go
// application/command/account_service.go (일부)
type AccountService struct {
    accountRepo  domain.AccountRepository
    eventStore   domain.EventStore
    eventProducer domain.EventProducer
}

func (s *AccountService) CreateAccount(ctx context.Context, ownerName string) (string, error) {
    // 계좌 생성 로직
    account := &domain.Account{
        ID:        uuid.New().String(),
        OwnerName: ownerName,
        Balance:   0,
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }
    
    // 이벤트 생성 및 저장
    event := &domain.Event{
        ID:        uuid.New().String(),
        Type:      domain.AccountCreated,
        Data:      // marshal account data
        Timestamp: time.Now(),
        Version:   1,
    }
    
    // 트랜잭션으로 계좌 및 이벤트 저장
    err := s.accountRepo.Save(ctx, account)
    if err != nil {
        return "", err
    }
    
    // 이벤트 발행
    err = s.eventProducer.Produce(ctx, event)
    if err != nil {
        return "", err
    }
    
    return account.ID, nil
}
```

### Query 서비스

```go
// application/query/account_service.go (일부)
type AccountQueryService struct {
    accountRepo domain.AccountRepository
}

func (s *AccountQueryService) GetAccount(ctx context.Context, id string) (*domain.Account, error) {
    return s.accountRepo.FindByID(ctx, id)
}

func (s *AccountQueryService) ListAccounts(ctx context.Context) ([]*domain.Account, error) {
    return s.accountRepo.FindAll(ctx)
}
```

## 이벤트 처리 파이프라인

이벤트 처리 파이프라인은 Kafka를 통해 구현했다. 계좌 서비스에서 생성된 이벤트는 Kafka 토픽에 발행되고, 이벤트 프로세서가 이를 소비하여 조회 모델을 업데이트한다.

### 이벤트 프로듀서

```go
// infrastructure/kafka/producer.go (일부)
type KafkaProducer struct {
    producer *kafka.Producer
    topic    string
}

func (p *KafkaProducer) Produce(ctx context.Context, event *domain.Event) error {
    // 이벤트 직렬화
    eventData, err := json.Marshal(event)
    if err != nil {
        return err
    }
    
    // Kafka 메시지 생성 및 발행
    message := &kafka.Message{
        TopicPartition: kafka.TopicPartition{
            Topic:     &p.topic,
            Partition: kafka.PartitionAny,
        },
        Key:   []byte(event.ID),
        Value: eventData,
    }
    
    // 메시지 전송
    return p.producer.Produce(message, nil)
}
```

### 이벤트 컨슈머

```go
// infrastructure/kafka/consumer.go (일부)
type KafkaConsumer struct {
    consumer    *kafka.Consumer
    handlers    map[domain.EventType]EventHandler
    eventStore  domain.EventStore
}

func (c *KafkaConsumer) Start(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return nil
            
        default:
            msg, err := c.consumer.ReadMessage(time.Second)
            if err != nil {
                if err.(kafka.Error).Code() == kafka.ErrTimedOut {
                    continue
                }
                return err
            }
            
            // 이벤트 역직렬화
            var event domain.Event
            if err := json.Unmarshal(msg.Value, &event); err != nil {
                // 에러 처리
                continue
            }
            
            // 이벤트 처리
            if handler, ok := c.handlers[event.Type]; ok {
                span, ctx := c.tracer.Start(ctx, "ProcessEvent")
                if err := handler.Handle(ctx, &event); err != nil {
                    // 에러 처리
                }
                span.End()
            }
        }
    }
}
```

## OpenTelemetry를 활용한 분산 추적

시스템의 모니터링과 디버깅을 위해 OpenTelemetry를 구현했다. 이를 통해 요청의 전체 경로를 추적하고, 각 서비스 간의 연계성을 시각화할 수 있다.

```go
// interface/telemetry/tracer.go (일부)
func InitTracer(serviceName string, config Config) (*sdktrace.TracerProvider, error) {
    ctx := context.Background()
    
    // OTLP 익스포터 설정
    exporter, err := otlptraceexporter.New(
        ctx,
        otlptraceexporter.WithGRPCEndpoint(config.Endpoint),
        otlptraceexporter.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }
    
    // 리소스 설정
    resource := resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceNameKey.String(serviceName),
    )
    
    // 트레이서 프로바이더 생성
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(resource),
        sdktrace.WithSampler(sdktrace.AlwaysSample()),
    )
    
    // 글로벌 트레이서 등록
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))
    
    return tp, nil
}
```

## HTTP 인터페이스

RESTful API를 통해 외부 시스템과 통신하며, 미들웨어를 통해 요청 로깅과 트레이싱을 구현했다.

```go
// interface/http/account_handler.go (일부)
type AccountHandler struct {
    commandService *command.AccountService
    queryService   *query.AccountQueryService
}

func (h *AccountHandler) RegisterRoutes(r *mux.Router) {
    r.HandleFunc("/accounts", h.CreateAccount).Methods("POST")
    r.HandleFunc("/accounts/{id}", h.GetAccount).Methods("GET")
    r.HandleFunc("/accounts/{id}/deposit", h.Deposit).Methods("POST")
    r.HandleFunc("/accounts/{id}/withdraw", h.Withdraw).Methods("POST")
}

func (h *AccountHandler) CreateAccount(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    var req struct {
        OwnerName string `json:"ownerName"`
    }
    
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    id, err := h.commandService.CreateAccount(ctx, req.OwnerName)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]string{"id": id})
}
```

## Docker 구성

프로젝트의 배포를 위해 Docker와 Docker Compose를 사용했다. 멀티 스테이지 빌드를 통해 경량화된 이미지를 생성하고, 의존성 관리를 간소화했다.

### Dockerfile

```dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /app

RUN apk add --no-cache \
    git \
    librdkafka-dev \
    pkgconfig \
    build-base \
    bash \
    gcc \
    musl-dev \
    linux-headers \
    libc-dev

COPY . .

RUN go mod download

# http router 테스트 실행
RUN go test -v ./...

# CGO_ENABLED=1을 명시적으로 설정하고, 빌드 시 추가 플래그 사용
RUN CGO_ENABLED=1 GOOS=linux go build -tags musl -o account ./cmd/account/main.go
RUN CGO_ENABLED=1 GOOS=linux go build -tags musl -o event ./cmd/event/main.go

FROM alpine:3.18 AS account-app
RUN apk add --no-cache librdkafka-dev
WORKDIR /app
COPY --from=builder /app/account /account
EXPOSE 8080

CMD ["/account"]

FROM alpine:3.18 AS event-app
RUN apk add --no-cache librdkafka-dev
WORKDIR /app
COPY --from=builder /app/event /event

CMD ["/event"]
```

### Docker Compose

Docker Compose를 통해 모든 서비스를 한 번에 실행하고 관리할 수 있다.

```yaml
services:
  postgres:
    image: postgres:latest
    environment:
      POSTGRES_DB: eventstore
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d eventstore"]
      interval: 5s
      timeout: 5s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_NODE_ID: 1
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka:29093'
      KAFKA_LISTENERS: 'PLAINTEXT://kafka:9092,CONTROLLER://kafka:29093'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka:9092'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      CLUSTER_ID: 'ciWo7IWazngRchmPES6q5A=='
    ports:
      - "9092:9092"
    healthcheck:
      test: ["CMD-SHELL", "kafka-topics --bootstrap-server kafka:9092 --list"]
      interval: 5s
      timeout: 5s
      retries: 5
    # ... 이하 생략 ...

  account-api:
    build:
      context: ..
      dockerfile: deployments/app/Dockerfile
      target: account-app
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_started
      otel-collector:
        condition: service_started
    environment:
      DB_HOST: postgres
      DB_NAME: user
      DB_PASSWORD: password
      KAFKA_BROKERS: kafka:9092
      KAFKA_TOPIC: account-events
      OTEL_EXPORTER_OTLP_ENDPOINT: "otel-collector:4317"
      OTEL_SERVICE_NAME: "account-api"
    ports:
      - "8080:8080"

  event-processor:
    build:
      context: ..
      dockerfile: deployments/app/Dockerfile
      target: event-app
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
      account-api:
        condition: service_started
      otel-collector:
        condition: service_started
    environment:
      DB_HOST: postgres
      DB_NAME: eventstore
      DB_USER: user
      DB_PASSWORD: password
      KAFKA_BROKERS: kafka:9092
      KAFKA_TOPIC: account-events
      KAFKA_GROUP_ID: event-processor-group
      OTEL_EXPORTER_OTLP_ENDPOINT: "otel-collector:4317"
      OTEL_SERVICE_NAME: "event-processor"

  # ... 모니터링 서비스 생략 ...
```

## 개발 과정에서 배운 점

이번 프로젝트를 통해 다음과 같은 점들을 배울 수 있었다:

1. **CQRS와 Event Sourcing 패턴의 실제 적용**: 이론적으로만 알고 있던 패턴들을 실제로 구현해보면서 이해도가 크게 향상됐다.

2. **분산 시스템 디버깅**: OpenTelemetry와 Jaeger를 통해 분산 시스템에서의 디버깅이 얼마나 중요한지 깨달았다. 특히 비동기 처리가 많은 이벤트 기반 시스템에서는 추적성이 무엇보다 중요하다.

3. **Go와 관련 라이브러리 경험**: Go 언어의 동시성 모델이 이러한 시스템을 구축하는 데 매우 적합하다는 것을 확인했다. 특히 컨텍스트(Context) API를 활용한 취소 및 데드라인 처리가 유용했다.

4. **Docker 및 컨테이너 오케스트레이션**: 멀티 스테이지 Dockerfile을 사용하여 이미지 크기를 최적화하고, Docker Compose로 복잡한 의존성을 관리하는 방법을 익혔다.

5. **CGO 활용 경험**: librdkafka와 같은 네이티브 라이브러리를 사용하기 위해 CGO를 활성화해야 했고, 이 과정에서 크로스 컴파일 문제 해결 등 여러 도전적인 상황을 경험했다.

## 향후 개선 사항

이 프로젝트는 계속해서 개선해 나갈 예정이다. 주요 개선 계획은 다음과 같다:

1. **테스트 커버리지 향상**: 현재는 기본적인 단위 테스트만 구현되어 있지만, 통합 테스트와 E2E 테스트를 추가할 예정이다.

2. **성능 최적화**: 현재 시스템은 기본적인 성능만 갖추고 있으나, 벤치마킹을 통해 병목 지점을 식별하고 최적화할 계획이다.

3. **멀티 노드 지원**: Kafka 및 PostgreSQL의 클러스터링을 통해 고가용성을 확보할 예정이다.

4. **보안 강화**: 현재는 간단한 인증만 구현되어 있으나, OAuth나 JWT를 활용한 보안 강화가 필요하다.

5. **UI 구현**: 관리자용 대시보드 및 사용자용 인터페이스를 React로 구현할 계획이다.

## 결론

이 토이 프로젝트를 통해 현대적인 백엔드 아키텍처와 기술 스택을 실제로 경험해볼 수 있었다. 특히 CQRS와 Event Sourcing 패턴이 실제 프로덕션 환경에서 어떻게 활용될 수 있는지에 대한 인사이트를 얻을 수 있었다.

비록 간단한 뱅킹 서비스였지만, 이 패턴들이 복잡한 비즈니스 요구사항에도 효과적으로 대응할 수 있다는 것을 확인했다. 더 많은 기능과 최적화를 통해 실제 프로덕션에서도 사용 가능한 수준으로 발전시켜 나갈 예정이다.

프로젝트 코드는 제 GitHub에서 확인하실 수 있다: [GitHub 저장소 링크]