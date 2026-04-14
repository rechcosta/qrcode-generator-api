# QRCode Generator API

API REST para geração de QR Codes a partir de texto, com armazenamento na AWS S3 e containerização via Docker.

---

## 🧱 Stack

| Tecnologia | Versão | Função |
|---|---|---|
| Java | 21 | Linguagem base |
| Spring Boot | 3.5.13 | Framework web e IoC |
| Spring Web | — | Exposição dos endpoints REST |
| Google ZXing | 3.5.2 | Geração do QR Code |
| AWS SDK for Java | 2.24.12 | Upload para S3 |
| Docker | 24+ | Containerização |
| Maven | 3.9+ | Build e gerenciamento de dependências |

---

## 📋 Pré-requisitos

- Java 17+
- Maven 3.9+
- Docker e Docker Compose
- Conta AWS com bucket S3 criado
- Credenciais AWS configuradas (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)

---

## ⚙️ Configuração

### Variáveis de ambiente

Crie um arquivo `.env` na raiz do projeto (nunca commite este arquivo):

```env
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
```

### application.properties

```properties
spring.application.name=qrcode.generator
aws.s3.region=${AWS_REGION}
aws.s3.bucket.name=${AWS_BUCKET_NAME}
```

---

## 🚀 Como executar
 
### Com Docker Compose (recomendado)
 
```bash
docker compose up --build
```
 
A API ficará disponível em `http://localhost:8080`.
 
### Localmente com Maven
 
```bash
mvn clean package -DskipTests
java -jar target/qrcode-generator.jar
```

---

## 🔌 Endpoints
 
### `POST /api/qrcode`
 
Gera um QR Code a partir de um texto e retorna a URL pública do arquivo no S3.
 
**Request:**
 
```http
POST /api/qrcode
Content-Type: application/json
 
{
  "text": "https://exemplo.com"
}
```
 
**Response `201 Created`:**
 
```json
{
  "url": "https://your-bucket.s3.amazonaws.com/qrcodes/a1b2c3d4.png",
  "text": "https://exemplo.com",
  "createdAt": "2025-04-14T10:30:00Z"
}
```
 
**Response `400 Bad Request`** — quando o campo `text` está vazio ou nulo:
 
```json
{
  "error": "O campo 'text' é obrigatório e não pode estar vazio."
}
```

---

## 🏗️ Estrutura do projeto

```
src/
└── main/
    └── java/com/rechcosta/qrcode/generator/
        ├── controller/
        │   └── QrCodeController.java          # Camada de entrada REST
        ├── dto/
        │   ├── QrCodeGenerateRequest.java      # Payload da requisição
        │   └── QrCodeGenerateResponse.java     # Payload da resposta
        ├── infrastructure/
        │   └── ports/
        │       └── S3StorageAdapter.java       # Implementação do upload S3
        ├── ports/
        │   └── StoragePort.java               # Interface de abstração do storage
        ├── service/
        │   └── QrCodeGeneratorService.java    # Orquestração da lógica de negócio
        └── Application.java                   # Entry point Spring Boot
Dockerfile
docker-compose.yml
```

---

## 🐳 Docker

### Dockerfile

```dockerfile
FROM maven:3.9.6-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar

ARG AWS_ACESS_KEY_ID
ARG AWS_SECRET_ACESS_KEY

ENV AWS_REGION=us-east-1
ENV AWS_BUCKET_NAME=qrcode

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### docker-compose.yml

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    env_file:
      - .env
```

---

## 🔐 Segurança

- As credenciais AWS **nunca** são hardcoded; são sempre injetadas via variáveis de ambiente.
- O arquivo `.env` está listado no `.gitignore`.
- O bucket S3 deve ter política de acesso público apenas para leitura dos objetos gerados, nunca escrita pública.

---

## 📦 Dependências principais (pom.xml)

```xml
<!-- ZXing -->
<dependency>
    <groupId>com.google.zxing</groupId>
    <artifactId>core</artifactId>
    <version>3.5.2</version>
</dependency>
<dependency>
    <groupId>com.google.zxing</groupId>
    <artifactId>javase</artifactId>
    <version>3.5.2</version>
</dependency>

<!-- AWS SDK S3 -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
    <version>2.24.12</version>
</dependency>
```
