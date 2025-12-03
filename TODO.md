# ✅ TODO.md - Plano de Implementação

## 📋 Guia de Implementação Passo a Passo

Este documento fornece um checklist detalhado para implementar o projeto Signature Integration Service do zero.

---

## 🎯 FASE 1: Setup Inicial do Projeto

### 1.1 Configuração do Repositório
- [ ] Criar repositório no GitLab/GitHub
- [ ] Inicializar estrutura de pastas
- [ ] Adicionar `.gitignore` para Java/Kotlin/Maven
- [ ] Criar branch `develop` para desenvolvimento

### 1.2 Configuração Maven
- [ ] Criar `pom.xml` com dependências:
  - [ ] Spring Boot 3.2.1 (parent)
  - [ ] spring-boot-starter-web
  - [ ] spring-boot-starter-security
  - [ ] spring-boot-starter-data-jpa
  - [ ] spring-boot-starter-validation
  - [ ] spring-boot-starter-actuator
  - [ ] spring-boot-starter-webflux (para WebClient)
  - [ ] PostgreSQL driver
  - [ ] Flyway (core + database-postgresql)
  - [ ] Kotlin reflect e stdlib
  - [ ] Jackson module Kotlin
  - [ ] Spring Cloud GCP (BOM 5.0.0)
  - [ ] spring-cloud-gcp-starter
  - [ ] spring-cloud-gcp-starter-secretmanager
  - [ ] spring-cloud-gcp-starter-storage
  - [ ] google-cloud-tasks 2.40.0
  - [ ] hypersistence-utils-hibernate-63 3.7.0
  - [ ] MockK 1.13.8 (test)
  - [ ] Testcontainers (test)
  - [ ] springmockk (test)
- [ ] Configurar Kotlin Maven Plugin com:
  - [ ] allopen plugin (Spring)
  - [ ] jpa plugin
  - [ ] sourceDirectory: src/main/kotlin
  - [ ] testSourceDirectory: src/test/kotlin

### 1.3 Application Main
- [ ] Criar `SignatureIntegrationApplication.kt`
- [ ] Adicionar anotação `@SpringBootApplication`
- [ ] Criar função `main` com `runApplication`

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/SignatureIntegrationApplication.kt`

---

## 🗄️ FASE 2: Database e Migrations

### 2.1 Configuração Flyway
- [ ] Criar pasta `src/main/resources/db/migration`
- [ ] Configurar Flyway no `application.yml`:
  ```yaml
  spring:
    flyway:
      enabled: true
      locations: classpath:db/migration
  ```

### 2.2 Migration V1 - Criar Tabela
- [ ] Criar `V1__create_signature_events_table.sql`
- [ ] Adicionar `CREATE EXTENSION IF NOT EXISTS "uuid-ossp";`
- [ ] Criar tabela `signature_events` com campos:
  - [ ] id UUID PRIMARY KEY
  - [ ] campaign_id VARCHAR(100) NOT NULL
  - [ ] cnpj VARCHAR(14) NOT NULL
  - [ ] provider VARCHAR(20) NOT NULL
  - [ ] status VARCHAR(20) NOT NULL
  - [ ] metadata JSONB NOT NULL DEFAULT '{}'
  - [ ] created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
  - [ ] updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
- [ ] Criar índices:
  - [ ] idx_signature_events_status
  - [ ] idx_signature_events_campaign_cnpj
  - [ ] idx_signature_events_metadata (GIN)
  - [ ] idx_signature_events_created_at

**Arquivo**: `src/main/resources/db/migration/V1__create_signature_events_table.sql`

### 2.3 Configuração Database
- [ ] Configurar `application.yml`:
  ```yaml
  spring:
    datasource:
      url: jdbc:postgresql://\${DB_HOST:localhost}:5432/\${DB_NAME:signature_db}
      username: \${DB_USER:postgres}
      password: \${DB_PASSWORD:postgres}
    jpa:
      hibernate:
        ddl-auto: validate
      show-sql: false
  ```

---

## 🏗️ FASE 3: Domain Layer

### 3.1 Enums
- [ ] Criar `SignatureStatus.kt` com valores:
  - [ ] PENDING
  - [ ] SENT
  - [ ] SIGNED
  - [ ] REJECTED
  - [ ] UPLOADED
  - [ ] ERROR
  - [ ] EXPIRED
- [ ] Criar `SignatureProvider.kt` com valores:
  - [ ] CERTISIGN
  - [ ] DOCUSIGN

**Arquivos**: 
- `src/main/kotlin/com/yourcompany/signature/domain/enums/SignatureStatus.kt`
- `src/main/kotlin/com/yourcompany/signature/domain/enums/SignatureProvider.kt`

### 3.2 Entity
- [ ] Criar `SignatureEvent.kt` com:
  - [ ] Anotação `@Entity` e `@Table(name = "signature_events")`
  - [ ] Campos mapeados conforme migration
  - [ ] `@Type(JsonBinaryType::class)` para metadata
  - [ ] `@CreationTimestamp` e `@UpdateTimestamp`
  - [ ] `@GeneratedValue(strategy = GenerationType.UUID)`
- [ ] Adicionar métodos auxiliares:
  - [ ] `fun addRequest(request: Map<String, Any>)`
  - [ ] `fun addResponse(response: Map<String, Any>)`
  - [ ] `fun getEnvelopeId(): String?`
  - [ ] `fun setEnvelopeId(envelopeId: String)`

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/domain/entity/SignatureEvent.kt`

### 3.3 Repository
- [ ] Criar `SignatureEventRepository` interface
- [ ] Extender `JpaRepository<SignatureEvent, UUID>`
- [ ] Adicionar métodos:
  - [ ] `findByStatus(status: SignatureStatus, pageable: Pageable): Page<SignatureEvent>`
  - [ ] `findByStatusInForStatusCheck(statuses: List<SignatureStatus>, pageable: Pageable): Page<SignatureEvent>` com `@Query`

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/domain/repository/SignatureEventRepository.kt`

---

## 📦 FASE 4: DTOs

### 4.1 Request DTOs
- [ ] Criar `CreateSignatureEventRequest.kt` com:
  - [ ] campaignId: String (NotBlank)
  - [ ] cnpj: String (NotBlank, Size 14)
  - [ ] provider: String (NotBlank)
  - [ ] documents: List<DocumentData> (NotEmpty)
  - [ ] signerName: String
  - [ ] signerEmail: String
  - [ ] signerCpf: String?
  - [ ] metadata: Map<String, Any>?
  - [ ] Método `fun toEntity(): SignatureEvent`
- [ ] Criar data class `DocumentData` com:
  - [ ] fileName: String
  - [ ] base64Content: String
  - [ ] Método `fun toMap()`

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/dto/request/CreateSignatureEventRequest.kt`

### 4.2 Response DTOs
- [ ] Criar `SignatureEventResponse.kt` com:
  - [ ] id: UUID
  - [ ] campaignId: String
  - [ ] cnpj: String
  - [ ] provider: SignatureProvider
  - [ ] status: SignatureStatus
  - [ ] envelopeId: String?
  - [ ] documentsGcsPath: String?
  - [ ] signedDocumentsGcsPath: String?
  - [ ] createdAt: LocalDateTime
  - [ ] updatedAt: LocalDateTime
- [ ] Criar `ProviderResponse.kt` com:
  - [ ] envelopeId: String
  - [ ] status: String
  - [ ] rawResponse: Map<String, Any>
- [ ] Criar `StatusCheckResponse.kt` com:
  - [ ] status: String
  - [ ] signedAt: String?
  - [ ] rawResponse: Map<String, Any>

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/dto/response/SignatureEventResponse.kt`

---

## ⚙️ FASE 5: Configurações

### 5.1 Security Config
- [ ] Criar `SecurityConfig.kt`
- [ ] Adicionar `@Configuration` e `@EnableWebSecurity`
- [ ] Criar bean `securityFilterChain`:
  - [ ] Desabilitar CSRF
  - [ ] `/api/webhook/**` -> authenticated (Basic Auth)
  - [ ] `/actuator/health**` -> permitAll
  - [ ] `/api/internal/**` -> permitAll
  - [ ] `/api/assinaturas/**` -> permitAll
  - [ ] anyRequest -> authenticated
  - [ ] Habilitar httpBasic
- [ ] Criar bean `userDetailsService`:
  - [ ] Criar usuário webhook com credenciais do environment

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/config/SecurityConfig.kt`

### 5.2 GCP Config
- [ ] Criar `GcpConfig.kt`
- [ ] Criar bean `Storage` do GCS:
  - [ ] Configurar credenciais via `GOOGLE_APPLICATION_CREDENTIALS`
  - [ ] Usar StorageOptions.newBuilder()

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/config/GcpConfig.kt`

### 5.3 Cloud Tasks Config
- [ ] Criar `CloudTasksConfig.kt`
- [ ] Criar bean `CloudTasksClient`:
  - [ ] Usar `CloudTasksClient.create()`

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/config/CloudTasksConfig.kt`

### 5.4 Scheduler Config
- [ ] Criar `SchedulerConfig.kt`
- [ ] Adicionar `@EnableScheduling`

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/config/SchedulerConfig.kt`

### 5.5 Jackson Config (Opcional)
- [ ] Criar `JacksonConfig.kt`
- [ ] Configurar ObjectMapper para Kotlin
- [ ] Configurar serialização de LocalDateTime

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/config/JacksonConfig.kt`

### 5.6 Application Properties
- [ ] Criar `application.yml` com todas as configurações:
  - [ ] Spring datasource
  - [ ] JPA/Hibernate
  - [ ] Flyway
  - [ ] GCP (project-id, location, bucket)
  - [ ] Signature providers (Certisign, Docusign URLs e tokens)
  - [ ] Webhook credentials
  - [ ] Actuator endpoints
  - [ ] Logging levels
- [ ] Criar `application-dev.yml` (opcional)
- [ ] Criar `application-prod.yml` (opcional)

**Arquivos**: `src/main/resources/application*.yml`

---

## 🎯 FASE 6: Provider Pattern (SOLID)

### 6.1 Interface SignatureProvider
- [ ] Criar interface `SignatureProvider`
- [ ] Declarar métodos:
  - [ ] `fun sendEnvelope(event: SignatureEvent): ProviderResponse`
  - [ ] `fun checkStatus(providerEnvelopeId: String): StatusCheckResponse`
  - [ ] `fun downloadSignedDocuments(providerEnvelopeId: String): List<SignedDocumentData>`
  - [ ] `fun getProviderType(): SignatureProvider`
- [ ] Criar data class `SignedDocumentData`:
  - [ ] documentId: String
  - [ ] documentName: String
  - [ ] base64Content: String

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/service/provider/SignatureProvider.kt`

### 6.2 SignatureProviderFactory
- [ ] Criar `SignatureProviderFactory` classe
- [ ] Adicionar `@Component`
- [ ] Injetar `List<SignatureProvider>` via construtor
- [ ] Criar map: `providerMap = providers.associateBy { it.getProviderType() }`
- [ ] Criar método `fun getProvider(providerType: ProviderType): SignatureProvider`

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/service/provider/SignatureProviderFactory.kt`

### 6.3 CertisignApiClient
- [ ] Criar `CertisignApiClient` classe
- [ ] Adicionar `@Component`
- [ ] Injetar configurações via `@Value`:
  - [ ] baseUrl
  - [ ] apiToken
- [ ] Criar WebClient no construtor
- [ ] Implementar métodos:
  - [ ] `fun createEnvelope(payload: Map<String, Any>): Map<String, Any>`
    - POST /api/v1/envelopes
  - [ ] `fun getEnvelopeStatus(envelopeId: String): Map<String, Any>`
    - GET /api/v1/envelopes/{id}
  - [ ] `fun downloadDocuments(envelopeId: String): Map<String, Any>`
    - GET /api/v1/envelopes/{id}/documents

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/service/provider/certisign/CertisignApiClient.kt`

### 6.4 CertisignProvider
- [ ] Criar `CertisignProvider` classe
- [ ] Adicionar `@Service`
- [ ] Implementar `SignatureProvider`
- [ ] Injetar `CertisignApiClient`
- [ ] Implementar `sendEnvelope`:
  - [ ] Extrair documentos de metadata
  - [ ] Montar payload Certisign
  - [ ] Chamar `createEnvelope`
  - [ ] Adicionar request/response ao metadata do evento
  - [ ] Retornar `ProviderResponse`
- [ ] Implementar `checkStatus`:
  - [ ] Chamar `getEnvelopeStatus`
  - [ ] Mapear status Certisign -> interno
- [ ] Implementar `downloadSignedDocuments`:
  - [ ] Chamar `downloadDocuments`
  - [ ] Transformar em `List<SignedDocumentData>`
- [ ] Implementar `getProviderType()`: retornar `CERTISIGN`
- [ ] Criar método privado `mapCertisignStatus`:
  - COMPLETED, SIGNED -> SIGNED
  - REJECTED, CANCELLED -> REJECTED
  - PENDING, WAITING -> SENT

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/service/provider/certisign/CertisignProvider.kt`

### 6.5 DocusignApiClient
- [ ] Criar `DocusignApiClient` classe
- [ ] Adicionar `@Component`
- [ ] Injetar configurações via `@Value`:
  - [ ] baseUrl
  - [ ] accountId
  - [ ] accessToken
- [ ] Criar WebClient no construtor
- [ ] Implementar métodos:
  - [ ] `fun createEnvelope(payload: Map<String, Any>): Map<String, Any>`
    - POST /restapi/v2.1/accounts/{accountId}/envelopes
  - [ ] `fun getEnvelopeStatus(envelopeId: String): Map<String, Any>`
    - GET /restapi/v2.1/accounts/{accountId}/envelopes/{id}
  - [ ] `fun getEnvelopeDocuments(envelopeId: String): List<Map<String, Any>>`
    - GET /restapi/v2.1/accounts/{accountId}/envelopes/{id}/documents
  - [ ] `fun downloadDocument(envelopeId: String, documentId: String): String`
    - GET /restapi/v2.1/accounts/{accountId}/envelopes/{id}/documents/{docId}
    - Retorna Base64

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/service/provider/docusign/DocusignApiClient.kt`

### 6.6 DocusignProvider
- [ ] Criar `DocusignProvider` classe
- [ ] Adicionar `@Service`
- [ ] Implementar `SignatureProvider`
- [ ] Injetar `DocusignApiClient`
- [ ] Implementar métodos similares ao CertisignProvider
- [ ] Implementar `getProviderType()`: retornar `DOCUSIGN`
- [ ] Criar método privado `mapDocusignStatus`:
  - completed -> SIGNED
  - declined, voided -> REJECTED
  - sent, delivered -> SENT

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/service/provider/docusign/DocusignProvider.kt`

---

## 🔧 FASE 7: Services

### 7.1 GcsStorageService
- [ ] Criar `GcsStorageService` classe
- [ ] Adicionar `@Service`
- [ ] Injetar `Storage` e `bucketName` via `@Value`
- [ ] Implementar `uploadDocumentsZip`:
  - [ ] Receber campaignId, cnpj, eventId, documents
  - [ ] Criar ZIP com método privado
  - [ ] Path: `{campaignId}/{cnpj}/{eventId}/documents.zip`
  - [ ] Upload para GCS
  - [ ] Retornar `gs://bucket/path`
- [ ] Implementar `uploadSignedDocumentsZip`:
  - [ ] Similar ao anterior
  - [ ] Path: `{campaignId}/{cnpj}/{eventId}/signed_documents.zip`
- [ ] Criar método privado `createZipFromBase64Documents`:
  - [ ] Usar ZipOutputStream
  - [ ] Para cada documento: decode Base64 e adicionar ao ZIP
  - [ ] Retornar ByteArray

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/service/GcsStorageService.kt`

### 7.2 CloudTasksService
- [ ] Criar `CloudTasksService` classe
- [ ] Adicionar `@Service`
- [ ] Injetar:
  - [ ] CloudTasksClient
  - [ ] ObjectMapper
  - [ ] projectId via `@Value`
  - [ ] location via `@Value`
  - [ ] internalUrl via `@Value`
- [ ] Implementar `createSendTask(eventId: UUID, delaySeconds: Long = 0)`:
  - [ ] Queue: "send-signature-queue"
  - [ ] Endpoint: "{internalUrl}/api/internal/assinaturas/tasks/send"
  - [ ] Payload: {"eventId": "uuid"}
- [ ] Implementar `createCheckStatusTask(eventId: UUID, delaySeconds: Long = 0)`:
  - [ ] Queue: "check-signature-status-queue"
  - [ ] Endpoint: "{internalUrl}/api/internal/assinaturas/tasks/check-status"
- [ ] Implementar `createUploadTask(eventId: UUID, delaySeconds: Long = 0)`:
  - [ ] Queue: "upload-signed-queue"
  - [ ] Endpoint: "{internalUrl}/api/internal/assinaturas/tasks/upload"
- [ ] Criar método privado `createTask`:
  - [ ] Montar HttpRequest com JSON body
  - [ ] Montar Task com scheduleTime
  - [ ] Chamar `client.createTask(queuePath, task)`
  - [ ] Retornar task.name
- [ ] Adicionar `@PreDestroy` para fechar client

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/service/CloudTasksService.kt`

### 7.3 SignatureEventService
- [ ] Criar `SignatureEventService` classe
- [ ] Adicionar `@Service`
- [ ] Injetar:
  - [ ] SignatureEventRepository
  - [ ] SignatureProviderFactory
  - [ ] GcsStorageService
- [ ] Implementar `createSignatureEvent`:
  - [ ] `@Transactional`
  - [ ] Converter request para entity
  - [ ] Salvar no banco
  - [ ] Upload documents.zip para GCS
  - [ ] Atualizar metadata com GCS path
  - [ ] Retornar response
- [ ] Implementar `sendToProvider`:
  - [ ] `@Transactional`
  - [ ] Obter provider via factory
  - [ ] Chamar `provider.sendEnvelope`
  - [ ] Atualizar status para SENT
  - [ ] Salvar envelope_id em metadata
  - [ ] Salvar no banco
- [ ] Implementar `markAsError`:
  - [ ] `@Transactional`
  - [ ] Buscar evento por ID
  - [ ] Atualizar status para ERROR
  - [ ] Salvar error_message e error_at em metadata
- [ ] Implementar `checkAndUpdateStatus`:
  - [ ] `@Transactional`
  - [ ] Obter provider via factory
  - [ ] Chamar `provider.checkStatus`
  - [ ] Atualizar status se mudou
  - [ ] Se SIGNED, salvar signed_at em metadata
- [ ] Implementar `markExpiredEvents`:
  - [ ] `@Transactional`
  - [ ] Calcular thirtyDaysAgo
  - [ ] Buscar eventos SENT criados antes disso
  - [ ] Atualizar status para EXPIRED
  - [ ] Retornar quantidade
- [ ] Implementar `downloadAndUploadSignedDocuments`:
  - [ ] `@Transactional`
  - [ ] Obter provider via factory
  - [ ] Chamar `provider.downloadSignedDocuments`
  - [ ] Upload signed_documents.zip para GCS
  - [ ] Atualizar status para UPLOADED
  - [ ] Salvar GCS path em metadata
- [ ] Implementar métodos de busca:
  - [ ] `findSentEventsForStatusCheck`
  - [ ] `findSignedEvents`
  - [ ] `findById`
- [ ] Criar método privado `toResponse`: converte entity para DTO

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/service/SignatureEventService.kt`

---

## 🎮 FASE 8: Controllers

### 8.1 SignatureEventController
- [ ] Criar `SignatureEventController` classe
- [ ] Adicionar `@RestController`
- [ ] Adicionar `@RequestMapping("/api/assinaturas/eventos")`
- [ ] Injetar:
  - [ ] SignatureEventService
  - [ ] CloudTasksService
- [ ] Implementar `POST /` (createEvent):
  - [ ] Receber `@Valid @RequestBody CreateSignatureEventRequest`
  - [ ] Chamar `service.createSignatureEvent`
  - [ ] Criar Cloud Task para envio
  - [ ] Retornar `201 CREATED` com response
- [ ] Implementar `GET /{id}` (getEvent):
  - [ ] Buscar evento por ID
  - [ ] Retornar `200 OK` ou `404 NOT FOUND`
- [ ] Implementar `GET /` (listEvents):
  - [ ] Receber `Pageable` via `@PageableDefault(size = 50)`
  - [ ] Retornar Page<SignatureEventResponse>

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/controller/SignatureEventController.kt`

### 8.2 SignatureWebhookController
- [ ] Criar `SignatureWebhookController` classe
- [ ] Adicionar `@RestController`
- [ ] Adicionar `@RequestMapping("/api/webhook/assinaturas")`
- [ ] Injetar:
  - [ ] SignatureEventService
  - [ ] CloudTasksService
- [ ] Implementar `POST /{provider}` (handleWebhook):
  - [ ] Receber `@PathVariable provider`
  - [ ] Receber `@RequestBody Map<String, Any>`
  - [ ] Extrair envelope_id conforme provider
  - [ ] Mapear status conforme provider
  - [ ] Chamar `service.updateStatus`
  - [ ] Se status = SIGNED, criar Cloud Task para upload
  - [ ] Retornar `200 OK` com {"success": true}
  - [ ] Em caso de erro, retornar `500` com {"error": "message"}

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/controller/SignatureWebhookController.kt`

### 8.3 InternalTasksController
- [ ] Criar `InternalTasksController` classe
- [ ] Adicionar `@RestController`
- [ ] Adicionar `@RequestMapping("/api/internal/assinaturas/tasks")`
- [ ] Injetar `SignatureEventService`
- [ ] Implementar `POST /send` (sendTask):
  - [ ] Receber `@RequestHeader("X-CloudTasks-TaskRetryCount", defaultValue = "0") retryCount`
  - [ ] Receber `@RequestBody Map<String, Any>` com eventId
  - [ ] Calcular `isLastAttempt = (retryCount + 1) >= 3`
  - [ ] Try-catch:
    - [ ] Buscar evento
    - [ ] Chamar `service.sendToProvider`
    - [ ] Retornar `200 OK`
  - [ ] Catch:
    - [ ] Se isLastAttempt: chamar `service.markAsError` e retornar 200
    - [ ] Senão: throw exception (Cloud Tasks retenta)
- [ ] Implementar `POST /check-status` (checkStatusTask):
  - [ ] Receber eventId
  - [ ] Buscar evento
  - [ ] Chamar `service.checkAndUpdateStatus`
  - [ ] Retornar `200 OK`
- [ ] Implementar `POST /upload` (uploadTask):
  - [ ] Similar ao /send
  - [ ] MAX_ATTEMPTS = 5
  - [ ] Chamar `service.downloadAndUploadSignedDocuments`

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/controller/InternalTasksController.kt`

---

## ⏰ FASE 9: Schedulers

### 9.1 SignatureStatusScheduler
- [ ] Criar `SignatureStatusScheduler` classe
- [ ] Adicionar `@Component`
- [ ] Injetar:
  - [ ] SignatureEventService
  - [ ] CloudTasksService
- [ ] Definir constantes:
  - [ ] PAGE_SIZE = 100
  - [ ] TIME_ZONE = "America/Sao_Paulo"
- [ ] Implementar `checkPendingDocumentsStatus`:
  - [ ] Adicionar `@Scheduled(cron = "0 0 1 * * *", zone = TIME_ZONE)`
  - [ ] Loop paginado:
    - [ ] Buscar eventos SENT
    - [ ] Para cada: criar Cloud Task check-status
  - [ ] Logar quantidade processada
- [ ] Implementar `markExpiredEvents`:
  - [ ] Adicionar `@Scheduled(cron = "0 0 1 * * *", zone = TIME_ZONE)`
  - [ ] Chamar `service.markExpiredEvents()`
  - [ ] Logar quantidade expirada
- [ ] Implementar `processSignedDocuments`:
  - [ ] Adicionar `@Scheduled(fixedDelay = 1800000, zone = TIME_ZONE)` (30 min)
  - [ ] Loop paginado:
    - [ ] Buscar eventos SIGNED
    - [ ] Para cada: criar Cloud Task upload
  - [ ] Logar quantidade processada

**Arquivo**: `src/main/kotlin/com/yourcompany/signature/scheduler/SignatureStatusScheduler.kt`

---

## 🧪 FASE 10: Testes

### 10.1 Testes Unitários - SignatureEventServiceTest
- [ ] Criar `SignatureEventServiceTest` classe
- [ ] Adicionar `@ExtendWith(MockKExtension::class)`
- [ ] Criar mocks com `@MockK`:
  - [ ] repository
  - [ ] providerFactory
  - [ ] gcsStorageService
- [ ] Injetar com `@InjectMockKs`: service
- [ ] Criar teste `should send event to provider successfully`:
  - [ ] Mock do provider
  - [ ] Mock do repository.save
  - [ ] Verificar status = SENT
  - [ ] Verificar chamada ao provider
- [ ] Criar teste `should mark event as error after 3 attempts`:
  - [ ] Verificar status = ERROR
- [ ] Criar teste `should mark expired events`:
  - [ ] Mock de eventos antigos
  - [ ] Verificar status = EXPIRED

**Arquivo**: `src/test/kotlin/com/yourcompany/signature/service/SignatureEventServiceTest.kt`

### 10.2 Testes Unitários - CloudTasksServiceTest
- [ ] Criar `CloudTasksServiceTest`
- [ ] Mockar CloudTasksClient
- [ ] Testar criação de tasks
- [ ] Verificar payload e headers

**Arquivo**: `src/test/kotlin/com/yourcompany/signature/service/CloudTasksServiceTest.kt`

### 10.3 Testes Unitários - CertisignProviderTest
- [ ] Criar `CertisignProviderTest`
- [ ] Mockar CertisignApiClient
- [ ] Testar sendEnvelope
- [ ] Testar checkStatus
- [ ] Testar mapeamento de status

**Arquivo**: `src/test/kotlin/com/yourcompany/signature/service/provider/certisign/CertisignProviderTest.kt`

### 10.4 Testes de Integração - SignatureEventControllerIntegrationTest
- [ ] Criar `SignatureEventControllerIntegrationTest`
- [ ] Adicionar `@SpringBootTest(webEnvironment = RANDOM_PORT)`
- [ ] Adicionar `@Testcontainers`
- [ ] Criar container PostgreSQL:
  ```kotlin
  @Container
  val postgres = PostgreSQLContainer<Nothing>("postgres:16-alpine")
  ```
- [ ] Injetar `TestRestTemplate`
- [ ] Criar teste `should create signature event`:
  - [ ] POST /api/assinaturas/eventos
  - [ ] Verificar status 201
  - [ ] Verificar ID retornado
- [ ] Criar teste `should get event by id`:
  - [ ] GET /api/assinaturas/eventos/{id}
  - [ ] Verificar status 200

**Arquivo**: `src/test/kotlin/com/yourcompany/signature/controller/SignatureEventControllerIntegrationTest.kt`

### 10.5 Testes Utilitários (Opcional)
- [ ] Criar `Base64UtilTest` se houver classe utilitária
- [ ] Testar encode/decode

---

## 🐳 FASE 11: Docker

### 11.1 Dockerfile
- [ ] Criar `Dockerfile` na raiz
- [ ] Stage 1 - Build:
  - [ ] FROM eclipse-temurin:17-jdk-alpine AS build
  - [ ] WORKDIR /app
  - [ ] COPY pom.xml e src
  - [ ] RUN mvn clean package -DskipTests
- [ ] Stage 2 - Runtime:
  - [ ] FROM eclipse-temurin:17-jre-alpine
  - [ ] COPY jar do stage build
  - [ ] EXPOSE 8080
  - [ ] ENTRYPOINT java -jar

**Arquivo**: `docker/Dockerfile` ou raiz

### 11.2 docker-compose.yml
- [ ] Criar `docker-compose.yml`
- [ ] Service postgres:
  - [ ] image: postgres:16-alpine
  - [ ] environment: POSTGRES_DB, USER, PASSWORD
  - [ ] ports: 5432
  - [ ] volumes
- [ ] Service app:
  - [ ] build: .
  - [ ] ports: 8080
  - [ ] environment: DB_HOST=postgres
  - [ ] depends_on: postgres

**Arquivo**: `docker/docker-compose.yml` ou raiz

### 11.3 .dockerignore
- [ ] Criar `.dockerignore`
- [ ] Adicionar: target/, .git/, .idea/, *.iml

---

## ☸️ FASE 12: Kubernetes

### 12.1 ConfigMap
- [ ] Criar `configmap.yml`
- [ ] Adicionar variáveis não-sensíveis:
  - [ ] GCP_PROJECT_ID
  - [ ] GCP_LOCATION
  - [ ] GCS_BUCKET_NAME
  - [ ] CERTISIGN_BASE_URL
  - [ ] DOCUSIGN_BASE_URL
  - [ ] INTERNAL_URL

**Arquivo**: `k8s/configmap.yml`

### 12.2 Secret
- [ ] Criar `secret.yml`
- [ ] Adicionar variáveis sensíveis (base64):
  - [ ] DB_PASSWORD
  - [ ] WEBHOOK_USERNAME
  - [ ] WEBHOOK_PASSWORD
  - [ ] CERTISIGN_API_TOKEN
  - [ ] DOCUSIGN_ACCESS_TOKEN

**Arquivo**: `k8s/secret.yml`

### 12.3 Deployment
- [ ] Criar `deployment.yml`
- [ ] Definir:
  - [ ] replicas: 2
  - [ ] selector com labels
  - [ ] template com:
    - [ ] serviceAccountName
    - [ ] container com image GCR
    - [ ] ports: 8080
    - [ ] envFrom: configMap e secret
    - [ ] livenessProbe: /actuator/health/liveness
    - [ ] readinessProbe: /actuator/health/readiness
    - [ ] resources: requests e limits

**Arquivo**: `k8s/deployment.yml`

### 12.4 Service
- [ ] Criar `service.yml`
- [ ] Tipo: ClusterIP (interno)
- [ ] Selector: app label
- [ ] Ports: 80 -> 8080

**Arquivo**: `k8s/service.yml`

### 12.5 Ingress
- [ ] Criar `ingress.yml`
- [ ] Annotations: kubernetes.io/ingress.class: kong
- [ ] Rules:
  - [ ] host: api.yourdomain.com
  - [ ] path: /api/webhook (apenas webhooks públicos)
  - [ ] backend: service port 80

**Arquivo**: `k8s/ingress.yml`

---

## 🚀 FASE 13: GCP Setup

### 13.1 Criar Filas Cloud Tasks
- [ ] Executar comandos gcloud:
```bash
# Fila 1: send-signature-queue
gcloud tasks queues create send-signature-queue \
  --location=us-central1 \
  --max-dispatches-per-second=10 \
  --max-attempts=3 \
  --min-backoff=60s \
  --max-backoff=180s \
  --max-doublings=0

# Fila 2: check-signature-status-queue
gcloud tasks queues create check-signature-status-queue \
  --location=us-central1 \
  --max-dispatches-per-second=5 \
  --max-attempts=3 \
  --min-backoff=30s

# Fila 3: upload-signed-queue
gcloud tasks queues create upload-signed-queue \
  --location=us-central1 \
  --max-dispatches-per-second=5 \
  --max-attempts=5 \
  --min-backoff=10s \
  --max-backoff=300s
```

### 13.2 Criar Bucket GCS
- [ ] Executar:
```bash
gsutil mb -l us-central1 gs://your-bucket-name
gsutil versioning set on gs://your-bucket-name
```

### 13.3 Configurar Service Account
- [ ] Criar service account:
```bash
gcloud iam service-accounts create signature-sa
```
- [ ] Adicionar roles:
  - [ ] Cloud Tasks Enqueuer
  - [ ] Storage Object Creator
  - [ ] Storage Object Viewer
  - [ ] Secret Manager Secret Accessor

### 13.4 Workload Identity (GKE)
- [ ] Vincular service account K8s ao GCP:
```bash
gcloud iam service-accounts add-iam-policy-binding \
  signature-sa@PROJECT.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT.svc.id.goog[default/signature-sa]"
```

---

## 📚 FASE 14: Documentação

### 14.1 README.md
- [ ] Criar `README.md` com:
  - [ ] Descrição do projeto
  - [ ] Tecnologias utilizadas
  - [ ] Pré-requisitos
  - [ ] Como executar localmente
  - [ ] Como executar com Docker
  - [ ] Como fazer deploy no GKE
  - [ ] Variáveis de ambiente
  - [ ] Exemplos de uso da API
  - [ ] Licença

### 14.2 ARQUITETURA.md
- [ ] Criar `docs/ARQUITETURA.md` com:
  - [ ] Diagrama de arquitetura
  - [ ] Fluxo de dados
  - [ ] Decisões técnicas
  - [ ] Princípios SOLID aplicados
  - [ ] Padrões de projeto utilizados

### 14.3 API.md
- [ ] Criar `docs/API.md` com:
  - [ ] Documentação de todas as rotas
  - [ ] Exemplos de request/response
  - [ ] Códigos de status HTTP
  - [ ] Autenticação

### 14.4 CHANGELOG.md
- [ ] Criar `CHANGELOG.md`
- [ ] Adicionar versão inicial 1.0.0

---

## ✅ FASE 15: Testes End-to-End e Deploy

### 15.1 Testes Locais
- [ ] Executar `./mvnw clean test` (todos os testes devem passar)
- [ ] Executar `./mvnw spring-boot:run`
- [ ] Testar endpoints com Postman/curl
- [ ] Verificar logs

### 15.2 Testes com Docker
- [ ] Executar `docker-compose up --build`
- [ ] Testar aplicação
- [ ] Verificar conectividade com PostgreSQL

### 15.3 Deploy GCP
- [ ] Build da imagem:
```bash
docker build -t gcr.io/PROJECT_ID/signature-integration:1.0.0 .
docker push gcr.io/PROJECT_ID/signature-integration:1.0.0
```
- [ ] Deploy no GKE:
```bash
kubectl apply -f k8s/
```
- [ ] Verificar pods:
```bash
kubectl get pods
kubectl logs -f deployment/signature-integration
```

### 15.4 Testes de Integração
- [ ] Criar evento via API
- [ ] Verificar Cloud Task criada
- [ ] Verificar logs de processamento
- [ ] Verificar documento no GCS
- [ ] Testar webhook
- [ ] Verificar scheduler

---

## 🎉 FASE 16: Finalização

### 16.1 Code Review
- [ ] Revisar código
- [ ] Verificar nomenclaturas
- [ ] Verificar tratamento de erros
- [ ] Verificar logs

### 16.2 Documentação Final
- [ ] Atualizar README com aprendizados
- [ ] Documentar problemas encontrados
- [ ] Adicionar troubleshooting guide

### 16.3 Monitoramento
- [ ] Configurar alertas no GCP
- [ ] Configurar dashboard de métricas
- [ ] Documentar SLOs

---

## 📋 Checklist de Verificação Final

### Funcionalidades Core
- [ ] ✅ Criar evento de assinatura
- [ ] ✅ Upload de documentos para GCS (ZIP)
- [ ] ✅ Envio para Certisign via Cloud Task
- [ ] ✅ Envio para Docusign via Cloud Task
- [ ] ✅ Retry automático (3x com 1 min)
- [ ] ✅ Detecção de última tentativa e marcação ERROR
- [ ] ✅ Webhook recebe status de providers
- [ ] ✅ Scheduler verifica status diariamente (01:00 AM)
- [ ] ✅ Marcar eventos expirados (30 dias)
- [ ] ✅ Download documentos assinados
- [ ] ✅ Upload documentos assinados para GCS (ZIP)
- [ ] ✅ Rate limiting 10 req/s (Certisign)

### Qualidade
- [ ] ✅ Testes unitários > 80% cobertura
- [ ] ✅ Testes de integração funcionando
- [ ] ✅ Código segue SOLID
- [ ] ✅ Provider pattern implementado
- [ ] ✅ Logs estruturados
- [ ] ✅ Tratamento de erros adequado

### Infraestrutura
- [ ] ✅ Docker funcionando
- [ ] ✅ Kubernetes manifests corretos
- [ ] ✅ Cloud Tasks configuradas
- [ ] ✅ GCS bucket criado
- [ ] ✅ Secrets configurados
- [ ] ✅ Health checks funcionando

### Documentação
- [ ] ✅ README completo
- [ ] ✅ Arquitetura documentada
- [ ] ✅ API documentada
- [ ] ✅ CHANGELOG atualizado

---

## 🚨 Troubleshooting Comum

### Problema: Flyway migration falha
- [ ] Verificar se PostgreSQL está rodando
- [ ] Verificar credenciais no application.yml
- [ ] Verificar se extensão uuid-ossp está disponível

### Problema: Cloud Tasks não executam
- [ ] Verificar se filas foram criadas
- [ ] Verificar URL do endpoint interno
- [ ] Verificar service account permissions
- [ ] Verificar logs da task no GCP Console

### Problema: Webhook não autentica
- [ ] Verificar credenciais no Secret
- [ ] Verificar header Authorization: Basic
- [ ] Testar com: echo -n "user:pass" | base64

### Problema: GCS upload falha
- [ ] Verificar bucket existe
- [ ] Verificar service account tem role Storage Object Creator
- [ ] Verificar GOOGLE_APPLICATION_CREDENTIALS

---

**Total de Tarefas**: ~200 itens

**Tempo Estimado**: 40-60 horas para desenvolvedor experiente

**Ordem Recomendada**: Seguir as fases de 1 a 16 sequencialmente

---

✨ **Dica Final**: Use este TODO.md junto com o PROJECT_SPECIFICATION.md. Copie ambos para o GitHub Copilot ou Claude e peça para implementar fase por fase!
