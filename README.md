# QRCode Generator API

REST API for generating QR Codes from any text or URL, with automatic storage on **AWS S3** and Docker-based execution in a single command.

---

## How it works

```
Client → POST /qrcode { "text": "..." }
              ↓
       Spring Boot (port 8080)
              ↓
       ZXing generates QR Code (PNG 200x200)
              ↓
       AWS SDK uploads to S3
              ↓
       Returns { "url": "https://..." }
```

---

## Stack

| Technology | Version | Role |
|---|---|---|
| Java | 21 | Base language |
| Spring Boot | 3.5.13 | Web framework and IoC container |
| Spring Web | — | REST endpoint exposure |
| Google ZXing | 3.5.2 | QR Code matrix generation and PNG rendering |
| AWS SDK for Java v2 | 2.24.12 | File upload to S3 bucket |
| Docker | 24+ | Containerization and isolated execution |
| Maven | 3.9+ | Build and dependency management |

### Why AWS S3?

The generated QR Codes are stored as objects on S3 instead of being returned as binary in the response. This allows the public URL to be shared, embedded in emails, or printed — without depending on this API's availability. The AWS SDK v2 is used with environment-variable-based authentication, with no credentials hardcoded in the source.

---

## Prerequisites

- Docker and Docker Compose installed
- AWS account with an **S3 bucket created** and configured for public read access on objects
- AWS credentials with `s3:PutObject` permission on the bucket

---

## Configuration

Create a `.env` file at the project root — it **must never be committed**:

```env
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
```

The `.env` file is already listed in `.gitignore`. The `AWS_REGION` (default: `us-east-1`) and `AWS_BUCKET_NAME` (default: `qrcode`) variables are defined in the `Dockerfile` and can be overridden in `docker-compose.yml` when needed.

---

## Running with Docker

```bash
docker compose up --build
```

The API will be available at `http://localhost:8080`.

---

## Testing with Insomnia

### Request configuration

| Field | Value |
|---|---|
| Method | `POST` |
| URL | `http://localhost:8080/qrcode` |
| Content-Type | `application/json` |

### Request body

```json
{
  "text": "https://github.com/your-username"
}
```

The `text` field accepts any string — URL, free text, structured data, etc.

### Success response — `200 OK`

```json
{
  "url": "https://qrcode.s3.us-east-1.amazonaws.com/f47ac10b-58cc-4372-a567-0e02b2c3d479"
}
```

The returned URL points directly to the PNG file on S3 and can be opened in any browser.

### Error response — `500 Internal Server Error`

Returned when QR Code generation or S3 upload fails (e.g., invalid credentials, non-existent bucket).

---

## Project structure

```
src/
└── main/
    └── java/com/rechcosta/qrcode/generator/
        ├── controller/
        │   └── QrCodeController.java           # Receives POST /qrcode and delegates to the service
        ├── dto/
        │   ├── QrCodeGenerateRequest.java       # { "text": "..." }
        │   └── QrCodeGenerateResponse.java      # { "url": "..." }
        ├── ports/
        │   └── StoragePort.java                # Storage abstraction interface
        ├── infrastructure/
        │   └── S3StorageAdapter.java           # Concrete implementation: upload to AWS S3
        ├── service/
        │   └── QrCodeGeneratorService.java     # Orchestrates generation (ZXing) + upload (StoragePort)
        └── Application.java                    # Spring Boot entry point
Dockerfile
docker-compose.yml
.env                                            # AWS credentials (not committed)
```

The `StoragePort` interface isolates the service from the concrete storage implementation — swapping S3 for another provider only requires a new implementation of the interface, with no changes to the service.

---

## Security

- AWS credentials are **never** hardcoded; always injected via environment variables
- The `.env` file is listed in `.gitignore`
- The S3 bucket must be configured with **public read access for objects only** — public write must remain disabled
- Write access is performed exclusively through IAM credentials with minimal permissions (`s3:PutObject`)

---

## License

MIT — see [LICENSE](./LICENSE)
