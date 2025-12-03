# 📋 PROJECT SPECIFICATION - Signature Integration Service

## 🎯 Visão Geral do Projeto

Sistema de integração com provedores de assinatura digital (Certisign e Docusign) usando Spring Boot 3.2, Kotlin, PostgreSQL, GCP Cloud Tasks e Google Cloud Storage.

### Objetivo
Criar um serviço que:
- Recebe eventos de assinatura com múltiplos documentos
- Envia para provedores de assinatura digital (Certisign/Docusign)
- Processa retries automáticos via Cloud Tasks
- Armazena documentos assinados no GCS
- Consulta status periodicamente
- Expõe webhooks para receber atualizações dos provedores

---

## 🏗️ Arquitetura

### Princípios SOLID Aplicados

1. **Single Responsibility**: Cada classe tem uma responsabilidade única
   - `SignatureEventService`: Gerencia ciclo de vida dos eventos
   - `GcsStorageService`: Responsável apenas por storage
   - `CloudTasksService`: Gerencia apenas filas

2. **Open/Closed**: Extensível para novos provedores sem modificar código existente
   - Interface `SignatureProvider` permite adicionar novos provedores
   - `SignatureProviderFactory` resolve dinamicamente o provider

3. **Liskov Substitution**: Qualquer implementação de `SignatureProvider` pode substituir outra

4. **Interface Segregation**: Interfaces específicas e coesas

5. **Dependency Inversion**: Dependências via interfaces, não implementações concretas

### Stack Tecnológica

- **Backend**: Spring Boot 3.2.1, Kotlin 1.9.21, JDK 17
- **Database**: PostgreSQL com Flyway migrations
- **Cloud**: GCP (Cloud Tasks, Cloud Storage, Secret Manager)
- **Container**: Docker, Kubernetes
- **Build**: Maven
- **Testes**: JUnit 5, MockK, Testcontainers

---

## 📊 Modelo de Dados

### Tabela: signature_events

```sql
CREATE TABLE signature_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    campaign_id VARCHAR(100) NOT NULL,
    cnpj VARCHAR(14) NOT NULL,
    provider VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Campos metadata (JSONB)

```json
{
  "envelope_id": "string",
  "documents": [
    {"fileName": "contrato.pdf"}
  ],
  "signer": {
    "name": "João Silva",
    "email": "joao@example.com",
    "cpf": "12345678900"
  },
  "documents_gcs_path": "gs://bucket/campaign/cnpj/uuid/documents.zip",
  "signed_documents_gcs_path": "gs://bucket/campaign/cnpj/uuid/signed_documents.zip",
  "sent_at": "2025-12-02T22:00:00",
  "signed_at": "2025-12-03T10:30:00",
  "error_message": "string",
  "error_at": "2025-12-02T22:05:00",
  "expired_at": "2026-01-02T22:00:00",
  "requests": [
    {
      "timestamp": "2025-12-02T22:00:00",
      "payload": {}
    }
  ],
  "responses": [
    {
      "timestamp": "2025-12-02T22:00:05",
      "payload": {}
    }
  ]
}
```

### Status Enum

- `PENDING`: Aguardando envio
- `SENT`: Enviado ao provider
- `SIGNED`: Assinado
- `REJECTED`: Rejeitado
- `UPLOADED`: Documentos assinados salvos no GCS
- `ERROR`: Erro após 3 tentativas
- `EXPIRED`: Mais de 30 dias sem assinatura

### Provider Enum

- `CERTISIGN`
- `DOCUSIGN`

---

## 🔄 Fluxo de Dados

### 1. Criação de Evento

```
POST /api/assinaturas/eventos
{
  "campaignId": "CAMP-2025-001",
  "cnpj": "12345678000190",
  "provider": "CERTISIGN",
  "documents": [
    {
      "fileName": "contrato.pdf",
      "base64Content": "JVBERi0xLjQK..."
    }
  ],
  "signerName": "João Silva",
  "signerEmail": "joao@example.com",
  "signerCpf": "12345678900"
}

1. Cria evento no banco (status: PENDING)
2. Upload documents.zip para GCS: campaign_id/cnpj/event_id/documents.zip
3. Cria Cloud Task na fila send-signature-queue
4. Retorna 201 Created com event_id
```

### 2. Envio ao Provider (Cloud Task)

```
Cloud Task executa: POST /api/internal/assinaturas/tasks/send
{
  "eventId": "uuid"
}

1. Busca evento no banco
2. Obtém SignatureProvider via Factory
3. Provider transforma request para formato específico (Certisign/Docusign)
4. Salva request em metadata.requests[]
5. Envia para API do provider
6. Salva response em metadata.responses[]
7. Atualiza status para SENT e metadata.envelope_id
8. Se falhar: Cloud Tasks retenta (max 3x, intervalo 1min)
9. Na 3ª falha: marca status=ERROR via header X-CloudTasks-TaskRetryCount
```

### 3. Webhook de Status

```
POST /api/webhook/assinaturas/certisign (com Basic Auth)
{
  "envelope_id": "ENV-123",
  "status": "completed"
}

1. Valida autenticação
2. Busca evento por envelope_id
3. Atualiza status conforme mapping do provider
4. Se status=SIGNED, cria Cloud Task para upload
```

### 4. Verificação Agendada (01:00 AM)

```
Scheduler executa às 01:00 (America/Sao_Paulo):

1. Busca eventos com status=SENT (paginado 100/página)
2. Para cada evento, cria Cloud Task na fila check-signature-status-queue
3. Task executa: consulta API do provider
4. Atualiza status conforme resposta

Também executa:
1. Busca eventos SENT criados há > 30 dias
2. Marca como EXPIRED
```

### 5. Download e Upload (Cloud Task)

```
Cloud Task executa: POST /api/internal/assinaturas/tasks/upload
{
  "eventId": "uuid"
}

1. Busca evento no banco
2. Provider baixa documentos assinados (retorna List<SignedDocumentData>)
3. Cria signed_documents.zip
4. Upload para GCS: campaign_id/cnpj/event_id/signed_documents.zip
5. Atualiza metadata.signed_documents_gcs_path
6. Atualiza status para UPLOADED
7. Se falhar: retenta até 5x
```

---

## 🔌 APIs REST

### Rotas Públicas (Rede Interna)

```kotlin
// Criar evento
POST /api/assinaturas/eventos
Body: CreateSignatureEventRequest
Response: 201 SignatureEventResponse

// Buscar evento
GET /api/assinaturas/eventos/{id}
Response: 200 SignatureEventResponse

// Listar eventos
GET /api/assinaturas/eventos?page=0&size=50
Response: 200 Page<SignatureEventResponse>
```

### Webhooks (Autenticação Obrigatória)

```kotlin
// Webhook genérico por provider
POST /api/webhook/assinaturas/{provider}
Headers: Authorization: Basic base64(username:password)
Body: Provider-specific payload
Response: 200 {"success": true}
```

### Rotas Internas (Cloud Tasks)

```kotlin
// Enviar para provider
POST /api/internal/assinaturas/tasks/send
Headers: X-CloudTasks-TaskRetryCount: 0
Body: {"eventId": "uuid"}

// Verificar status
POST /api/internal/assinaturas/tasks/check-status
Body: {"eventId": "uuid"}

// Upload documentos assinados
POST /api/internal/assinaturas/tasks/upload
Body: {"eventId": "uuid"}
```

---

## 🔧 GCP Cloud Tasks

### Configuração das Filas

#### Fila 1: send-signature-queue
```bash
gcloud tasks queues create send-signature-queue \
  --location=us-central1 \
  --max-dispatches-per-second=10 \
  --max-attempts=3 \
  --min-backoff=60s \
  --max-backoff=180s \
  --max-doublings=0
```
- **Rate**: 10 req/s (limite Certisign)
- **Retries**: 3 tentativas
- **Intervalo**: 1 minuto fixo entre retries

#### Fila 2: check-signature-status-queue
```bash
gcloud tasks queues create check-signature-status-queue \
  --location=us-central1 \
  --max-dispatches-per-second=5 \
  --max-attempts=3 \
  --min-backoff=30s \
  --max-backoff=120s
```

#### Fila 3: upload-signed-queue
```bash
gcloud tasks queues create upload-signed-queue \
  --location=us-central1 \
  --max-dispatches-per-second=5 \
  --max-attempts=5 \
  --min-backoff=10s \
  --max-backoff=300s \
  --max-doublings=3
```

### Detecção de Última Tentativa

```kotlin
@PostMapping("/send")
fun sendTask(
    @RequestHeader("X-CloudTasks-TaskRetryCount", defaultValue = "0") retryCount: Int
) {
    val isLastAttempt = (retryCount + 1) >= MAX_ATTEMPTS_SEND

    try {
        // processa
    } catch (e: Exception) {
        if (isLastAttempt) {
            signatureEventService.markAsError(eventId, e.message)
            return ResponseEntity.ok(mapOf("status" to "ERROR"))
        }
        throw e // Cloud Tasks retenta automaticamente
    }
}
```

---

## 📦 Provider Pattern (SOLID)

### Interface SignatureProvider

```kotlin
interface SignatureProvider {
    fun sendEnvelope(event: SignatureEvent): ProviderResponse
    fun checkStatus(providerEnvelopeId: String): StatusCheckResponse
    fun downloadSignedDocuments(providerEnvelopeId: String): List<SignedDocumentData>
    fun getProviderType(): SignatureProvider
}
```

### CertisignProvider

**API Base**: `https://api.certisign.com.br`

#### Criar Envelope
```http
POST /api/v1/envelopes
Authorization: Bearer {token}
{
  "documents": [
    {"name": "contrato.pdf", "order": 1}
  ],
  "signers": [
    {
      "name": "João Silva",
      "email": "joao@example.com",
      "cpf_cnpj": "12345678000190"
    }
  ],
  "reference": "CAMP-2025-001"
}

Response:
{
  "envelope_id": "ENV-CERT-123",
  "status": "pending"
}
```

#### Consultar Status
```http
GET /api/v1/envelopes/{envelope_id}
Response:
{
  "envelope_id": "ENV-CERT-123",
  "status": "completed",
  "signed_at": "2025-12-03T10:30:00Z"
}
```

#### Download Documentos
```http
GET /api/v1/envelopes/{envelope_id}/documents
Response:
{
  "documents": [
    {
      "id": "DOC-1",
      "name": "contrato.pdf",
      "content": "base64..."
    }
  ]
}
```

#### Mapeamento de Status
```kotlin
COMPLETED, SIGNED -> SIGNED
REJECTED, CANCELLED -> REJECTED
PENDING, WAITING -> SENT
```

### DocusignProvider

**API Base**: `https://demo.docusign.net`

#### Criar Envelope
```http
POST /restapi/v2.1/accounts/{accountId}/envelopes
Authorization: Bearer {token}
{
  "emailSubject": "Documento para assinatura - CAMP-2025-001",
  "documents": [
    {
      "documentId": "1",
      "name": "contrato.pdf",
      "fileExtension": "pdf"
    }
  ],
  "recipients": {
    "signers": [
      {
        "email": "joao@example.com",
        "name": "João Silva",
        "recipientId": "1",
        "routingOrder": "1"
      }
    ]
  },
  "status": "sent"
}

Response:
{
  "envelopeId": "a1b2c3d4-e5f6-7890",
  "status": "sent"
}
```

#### Consultar Status
```http
GET /restapi/v2.1/accounts/{accountId}/envelopes/{envelopeId}
Response:
{
  "envelopeId": "a1b2c3d4-e5f6-7890",
  "status": "completed",
  "completedDateTime": "2025-12-03T10:30:00Z"
}
```

#### Mapeamento de Status
```kotlin
completed -> SIGNED
declined, voided -> REJECTED
sent, delivered -> SENT
```

---

## 💾 GCS Storage Pattern

### Estrutura de Diretórios

```
bucket-name/
├── {campaign_id}/
│   └── {cnpj}/
│       └── {event_id}/
│           ├── documents.zip       # Documentos originais
│           └── signed_documents.zip # Documentos assinados
```

### Exemplo
```
my-signatures-bucket/
├── CAMP-2025-001/
│   └── 12345678000190/
│       └── a1b2c3d4-e5f6-7890-abcd-1234567890ab/
│           ├── documents.zip
│           └── signed_documents.zip
```

### Formato ZIP
```
documents.zip:
├── contrato.pdf
├── anexo_1.pdf
└── anexo_2.pdf
```

**Razão**: Base64 não deve ser armazenado em banco de dados. Documentos são enviados via request e imediatamente salvos em GCS como ZIP.

---

## ⏰ Schedulers

### 1. Check Status (Diário 01:00 AM)

```kotlin
@Scheduled(cron = "0 0 1 * * *", zone = "America/Sao_Paulo")
fun checkPendingDocumentsStatus() {
    // Busca eventos SENT paginados (100/página)
    // Cria Cloud Task para cada um
}
```

### 2. Mark Expired (Diário 01:00 AM)

```kotlin
@Scheduled(cron = "0 0 1 * * *", zone = "America/Sao_Paulo")
fun markExpiredEvents() {
    val thirtyDaysAgo = LocalDateTime.now().minusDays(30)
    // Busca eventos SENT criados antes de thirtyDaysAgo
    // Marca como EXPIRED
}
```

### 3. Process Signed (A cada 30 min)

```kotlin
@Scheduled(fixedDelay = 1800000, zone = "America/Sao_Paulo")
fun processSignedDocuments() {
    // Busca eventos SIGNED paginados
    // Cria Cloud Task para upload
}
```

---

## 🔐 Segurança

### Spring Security Configuration

```kotlin
@Bean
fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
    http {
        csrf { disable() }
        authorizeHttpRequests {
            authorize("/api/webhook/**", authenticated)      // Basic Auth obrigatório
            authorize("/actuator/health", permitAll)
            authorize("/api/internal/**", permitAll)         // Rede interna
            authorize("/api/assinaturas/**", permitAll)      // Rede interna
            authorize(anyRequest, authenticated)
        }
        httpBasic { }
    }
    return http.build()
}
```

### Webhooks
- **Autenticação**: HTTP Basic Auth
- **Usuário/Senha**: Configurável via Secret Manager
- **Validação**: Opcional - validar assinatura HMAC do provider

---

## 📝 Variáveis de Ambiente

```yaml
# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=signature_db
DB_USER=postgres
DB_PASSWORD=postgres

# GCP
GCP_PROJECT_ID=your-project-id
GCP_LOCATION=us-central1
GCS_BUCKET_NAME=your-bucket-name
GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json

# Application
INTERNAL_URL=http://signature-integration-service.default.svc.cluster.local

# Webhook
WEBHOOK_USERNAME=webhook-user
WEBHOOK_PASSWORD=secure-password

# Certisign
CERTISIGN_BASE_URL=https://api.certisign.com.br
CERTISIGN_API_TOKEN=your-certisign-token

# Docusign
DOCUSIGN_BASE_URL=https://demo.docusign.net
DOCUSIGN_ACCOUNT_ID=your-account-id
DOCUSIGN_ACCESS_TOKEN=your-access-token
```

---

## 🐳 Docker

### Dockerfile

```dockerfile
FROM eclipse-temurin:17-jdk-alpine AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN ./mvnw clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### docker-compose.yml

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: signature_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
    depends_on:
      - postgres

volumes:
  postgres_data:
```

---

## ☸️ Kubernetes

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: signature-integration
spec:
  replicas: 2
  selector:
    matchLabels:
      app: signature-integration
  template:
    metadata:
      labels:
        app: signature-integration
    spec:
      serviceAccountName: signature-sa
      containers:
      - name: app
        image: gcr.io/PROJECT_ID/signature-integration:latest
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: signature-config
        - secretRef:
            name: signature-secrets
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 45
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
```

### Service (ClusterIP - Interno)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: signature-integration-service
spec:
  type: ClusterIP
  selector:
    app: signature-integration
  ports:
  - port: 80
    targetPort: 8080
```

### Ingress (Kong - Apenas Webhooks)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: signature-webhook-ingress
  annotations:
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - host: api.yourdomain.com
    http:
      paths:
      - path: /api/webhook
        pathType: Prefix
        backend:
          service:
            name: signature-integration-service
            port:
              number: 80
```

---

## 🧪 Testes

### Testes Unitários (MockK)

```kotlin
@ExtendWith(MockKExtension::class)
class SignatureEventServiceTest {
    @MockK
    private lateinit var repository: SignatureEventRepository

    @MockK
    private lateinit var providerFactory: SignatureProviderFactory

    @MockK
    private lateinit var gcsStorage: GcsStorageService

    @InjectMockKs
    private lateinit var service: SignatureEventService

    @Test
    fun `should send event to provider successfully`() {
        // given
        val event = SignatureEvent(...)
        val provider = mockk<SignatureProvider>()

        every { providerFactory.getProvider(any()) } returns provider
        every { provider.sendEnvelope(event) } returns ProviderResponse(...)
        every { repository.save(any()) } returns event

        // when
        val result = service.sendToProvider(event)

        // then
        assertEquals(SignatureStatus.SENT, result.status)
        verify { provider.sendEnvelope(event) }
    }
}
```

### Testes de Integração (Testcontainers)

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class SignatureEventControllerIntegrationTest {
    @Container
    val postgres = PostgreSQLContainer<Nothing>("postgres:16-alpine")

    @Autowired
    lateinit var restTemplate: TestRestTemplate

    @Test
    fun `should create signature event`() {
        val request = CreateSignatureEventRequest(...)

        val response = restTemplate.postForEntity(
            "/api/assinaturas/eventos",
            request,
            SignatureEventResponse::class.java
        )

        assertEquals(HttpStatus.CREATED, response.statusCode)
        assertNotNull(response.body?.id)
    }
}
```

---

## 📚 Estrutura de Pastas Completa

```
signature-integration/
├── src/
│   ├── main/
│   │   ├── kotlin/com/yourcompany/signature/
│   │   │   ├── SignatureIntegrationApplication.kt
│   │   │   ├── config/
│   │   │   │   ├── SecurityConfig.kt
│   │   │   │   ├── CloudTasksConfig.kt
│   │   │   │   ├── GcpConfig.kt
│   │   │   │   ├── SchedulerConfig.kt
│   │   │   │   └── JacksonConfig.kt
│   │   │   ├── domain/
│   │   │   │   ├── entity/
│   │   │   │   │   └── SignatureEvent.kt
│   │   │   │   ├── enums/
│   │   │   │   │   ├── SignatureStatus.kt
│   │   │   │   │   └── SignatureProvider.kt
│   │   │   │   └── repository/
│   │   │   │       └── SignatureEventRepository.kt
│   │   │   ├── service/
│   │   │   │   ├── SignatureEventService.kt
│   │   │   │   ├── CloudTasksService.kt
│   │   │   │   ├── GcsStorageService.kt
│   │   │   │   └── provider/
│   │   │   │       ├── SignatureProvider.kt
│   │   │   │       ├── SignatureProviderFactory.kt
│   │   │   │       ├── certisign/
│   │   │   │       │   ├── CertisignProvider.kt
│   │   │   │       │   └── CertisignApiClient.kt
│   │   │   │       └── docusign/
│   │   │   │           ├── DocusignProvider.kt
│   │   │   │           └── DocusignApiClient.kt
│   │   │   ├── controller/
│   │   │   │   ├── SignatureEventController.kt
│   │   │   │   ├── SignatureWebhookController.kt
│   │   │   │   └── InternalTasksController.kt
│   │   │   ├── dto/
│   │   │   │   ├── request/
│   │   │   │   │   └── CreateSignatureEventRequest.kt
│   │   │   │   └── response/
│   │   │   │       └── SignatureEventResponse.kt
│   │   │   ├── scheduler/
│   │   │   │   └── SignatureStatusScheduler.kt
│   │   │   └── util/
│   │   │       └── Base64Util.kt
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       └── db/migration/
│   │           └── V1__create_signature_events_table.sql
│   └── test/
│       └── kotlin/com/yourcompany/signature/
│           ├── service/
│           │   ├── SignatureEventServiceTest.kt
│           │   └── CloudTasksServiceTest.kt
│           └── controller/
│               └── SignatureEventControllerIntegrationTest.kt
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── k8s/
│   ├── deployment.yml
│   ├── service.yml
│   ├── ingress.yml
│   ├── configmap.yml
│   └── secret.yml
├── docs/
│   ├── ARQUITETURA.md
│   └── API.md
├── pom.xml
├── README.md
└── CHANGELOG.md
```

---

## 🚀 Build e Deploy

### Local
```bash
# Build
./mvnw clean package

# Run
./mvnw spring-boot:run

# Docker
docker-compose up --build
```

### GCP
```bash
# Build image
docker build -t gcr.io/PROJECT_ID/signature-integration:latest .

# Push
docker push gcr.io/PROJECT_ID/signature-integration:latest

# Deploy
kubectl apply -f k8s/
```

---

## 📊 Monitoramento

### Actuator Endpoints
- `/actuator/health` - Health check
- `/actuator/metrics` - Métricas
- `/actuator/info` - Informações da aplicação

### Logs
- Usar SLF4J + Logback
- Formato JSON para Cloud Logging
- Níveis: DEBUG para com.yourcompany.signature, INFO para root

### Métricas GCP
- Taxa de sucesso/erro por fila
- Latência de processamento
- Tamanho das filas
- Tempo médio de assinatura

---

## 🎓 Conceitos Importantes

### Por que JSONB e não tabelas?
- **Flexibilidade**: Metadata varia por provider
- **Rastreabilidade**: Requests/responses completos
- **Evolução**: Fácil adicionar campos sem migrations
- **Performance**: Índice GIN permite queries eficientes

### Por que ZIP no GCS?
- **Custo**: Storage é mais barato que banco
- **Performance**: Base64 em TEXT é ineficiente
- **Escalabilidade**: Banco não deve armazenar binários
- **Simplicidade**: Um arquivo por evento

### Por que Cloud Tasks?
- **Retries automáticos**: Sem código adicional
- **Rate limiting**: 10 req/s nativamente
- **Idempotência**: Headers permitem detecção de retry
- **Escalabilidade**: Processa 29.000 eventos automaticamente

### Por que Provider Pattern?
- **SOLID**: Open/Closed, dependency inversion
- **Testabilidade**: Mock de providers
- **Manutenção**: Adicionar novo provider = nova classe
- **Isolamento**: Bugs em um provider não afetam outros

---

Essa especificação deve ser suficiente para gerar o projeto completo usando GitHub Copilot ou qualquer LLM!
