# QRCode Generator API

API REST para geração de QR Codes a partir de qualquer texto ou URL, com armazenamento automático na **AWS S3** e execução via Docker em um único comando.

---

## Como funciona

```
Cliente → POST /qrcode { "text": "..." }
              ↓
       Spring Boot (porta 8080)
              ↓
       ZXing gera QR Code (PNG 200x200)
              ↓
       AWS SDK faz upload no S3
              ↓
       Retorna { "url": "https://..." }
```

---

## Stack

| Tecnologia | Versão | Papel |
|---|---|---|
| Java | 21 | Linguagem base |
| Spring Boot | 3.5.13 | Framework web e container IoC |
| Spring Web | — | Exposição dos endpoints REST |
| Google ZXing | 3.5.2 | Geração da matriz do QR Code e renderização PNG |
| AWS SDK for Java v2 | 2.24.12 | Upload dos arquivos para o bucket S3 |
| Docker | 24+ | Containerização e execução isolada |
| Maven | 3.9+ | Build e gerenciamento de dependências |

### Por que AWS S3?

Os QR Codes gerados são armazenados como objetos no S3 em vez de serem retornados como binário na resposta. Isso permite que a URL pública seja compartilhada, incorporada em e-mails ou impressa — sem depender da disponibilidade desta API. O SDK v2 do AWS é usado com autenticação via variáveis de ambiente, sem nenhuma credencial hardcoded no código.

---

## Pré-requisitos

- Docker e Docker Compose instalados
- Conta AWS com um **bucket S3 criado** e configurado com leitura pública nos objetos
- Credenciais AWS com permissão `s3:PutObject` no bucket

---

## Configuração

Crie um arquivo `.env` na raiz do projeto — ele **nunca deve ser commitado**:

```env
AWS_ACCESS_KEY_ID=sua_access_key
AWS_SECRET_ACCESS_KEY=sua_secret_key
```

O arquivo `.env` já está no `.gitignore`. As variáveis `AWS_REGION` (padrão: `us-east-1`) e `AWS_BUCKET_NAME` (padrão: `qrcode`) são definidas no `Dockerfile` e podem ser sobrescritas no `docker-compose.yml` se necessário.

---

## Executando com Docker

```bash
docker compose up --build
```

A API ficará disponível em `http://localhost:8080`.

---

## Testando com Insomnia

### Configuração da requisição

| Campo | Valor |
|---|---|
| Método | `POST` |
| URL | `http://localhost:8080/qrcode` |
| Content-Type | `application/json` |

### Body da requisição

```json
{
  "text": "https://github.com/seu-usuario"
}
```

O campo `text` aceita qualquer string — URL, texto livre, dados estruturados, etc.

### Resposta de sucesso — `200 OK`

```json
{
  "url": "https://qrcode.s3.us-east-1.amazonaws.com/f47ac10b-58cc-4372-a567-0e02b2c3d479"
}
```

A URL retornada aponta diretamente para o arquivo PNG no S3 e pode ser aberta em qualquer navegador.

### Resposta de erro — `500 Internal Server Error`

Retornado quando ocorre falha na geração do QR Code ou no upload ao S3 (ex: credenciais inválidas, bucket inexistente).

---

## Estrutura do projeto

```
src/
└── main/
    └── java/com/rechcosta/qrcode/generator/
        ├── controller/
        │   └── QrCodeController.java           # Recebe POST /qrcode e delega ao service
        ├── dto/
        │   ├── QrCodeGenerateRequest.java       # { "text": "..." }
        │   └── QrCodeGenerateResponse.java      # { "url": "..." }
        ├── ports/
        │   └── StoragePort.java                # Interface de abstração do storage
        ├── infrastructure/
        │   └── S3StorageAdapter.java           # Implementação concreta: upload para AWS S3
        ├── service/
        │   └── QrCodeGeneratorService.java     # Orquestra geração (ZXing) + upload (StoragePort)
        └── Application.java                    # Entry point Spring Boot
Dockerfile
docker-compose.yml
.env                                            # Credenciais AWS (não commitado)
```

A `StoragePort` isola o serviço da implementação concreta de storage — trocar de S3 para outro provider exige apenas uma nova implementação da interface, sem tocar no serviço.

---

## Segurança

- Credenciais AWS **nunca** são hardcoded; sempre injetadas via variáveis de ambiente
- O arquivo `.env` está no `.gitignore`
- O bucket S3 deve ter política de **leitura pública apenas para leitura de objetos** — escrita pública deve estar desabilitada
- O acesso de escrita é feito exclusivamente via credenciais IAM com permissão mínima (`s3:PutObject`)

---

## Licença

MIT — veja [LICENSE](./LICENSE)
