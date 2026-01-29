# T01: Project Setup

## Overview
Initialize the Node.js project with all dependencies, TypeScript configuration, and development tooling.

## Dependencies
None - this is the foundation task.

## Deliverables
- `package.json` with all dependencies
- `tsconfig.json` with strict TypeScript config
- `.env.example` with all environment variables
- `docker-compose.yml` for local development
- `.gitignore` for Node.js project
- `README.md` with setup instructions

---

## Acceptance Criteria

- [ ] `npm install` completes without errors
- [ ] `npm run build` compiles TypeScript without errors
- [ ] `docker-compose up -d` starts Redis and MySQL containers
- [ ] `npm run dev` starts the development server (can fail on missing modules)
- [ ] All required dependencies are pinned to specific versions

---

## Test Specifications

### TS-01-01: Package Installation
```bash
# Test: Fresh install succeeds
rm -rf node_modules package-lock.json
npm install
echo $?  # Must be 0
```

### TS-01-02: TypeScript Compilation
```bash
# Test: TypeScript compiles
npm run build
echo $?  # Must be 0
```

### TS-01-03: Docker Services
```bash
# Test: Containers start and are healthy
docker-compose up -d
sleep 10
docker-compose ps | grep -E "(redis|mysql).*Up"
```

---

## Implementation Details

### package.json
```json
{
  "name": "arla-engine",
  "version": "0.1.0",
  "description": "ARLA Question Engine - Low-latency voice agent API",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "test": "jest",
    "test:unit": "jest --testPathPattern=tests/unit",
    "test:integration": "jest --testPathPattern=tests/integration",
    "test:load": "npx ts-node tests/load/runner.ts",
    "lint": "eslint src/**/*.ts",
    "proto:gen": "grpc_tools_node_protoc --plugin=protoc-gen-ts=./node_modules/.bin/protoc-gen-ts --ts_out=grpc_js:./src/grpc/generated --grpc_out=grpc_js:./src/grpc/generated -I ./proto ./proto/*.proto"
  },
  "dependencies": {
    "@gorules/zen-engine": "^0.26.0",
    "@grpc/grpc-js": "^1.9.14",
    "@grpc/proto-loader": "^0.7.10",
    "@aws-sdk/client-sqs": "^3.500.0",
    "@msgpack/msgpack": "^3.0.0-beta2",
    "ioredis": "^5.3.2",
    "mysql2": "^3.9.1",
    "yaml": "^2.3.4",
    "pino": "^8.19.0",
    "dotenv": "^16.4.1"
  },
  "devDependencies": {
    "@types/node": "^20.11.0",
    "typescript": "^5.3.3",
    "ts-node-dev": "^2.0.0",
    "jest": "^29.7.0",
    "@types/jest": "^29.5.12",
    "ts-jest": "^29.1.2",
    "pino-pretty": "^10.3.1",
    "grpc-tools": "^1.12.4",
    "grpc_tools_node_protoc_ts": "^5.3.3",
    "eslint": "^8.56.0",
    "@typescript-eslint/parser": "^6.19.0",
    "@typescript-eslint/eslint-plugin": "^6.19.0"
  }
}
```

### tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "sourceMap": true,
    "moduleResolution": "node",
    "noImplicitAny": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

### .env.example
```bash
# Server
PORT=50051
NODE_ENV=development

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# Aurora MySQL
AURORA_HOST=localhost
AURORA_PORT=3306
AURORA_USER=arla
AURORA_PASSWORD=arla_password
AURORA_DATABASE=octane
AURORA_POOL_SIZE=20

# AWS
AWS_REGION=us-east-1
SQS_QUEUE_URL=

# Config
CONFIG_PATH=./config

# Performance
MAX_EVAL_TIME_MS=8
MAX_RULES_PER_REQUEST=200
```

### docker-compose.yml
```yaml
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3

  mysql:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: octane
      MYSQL_USER: arla
      MYSQL_PASSWORD: arla_password
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 3s
      retries: 5
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

### Directory Structure to Create
```bash
mkdir -p src/{config,grpc,rules,cache,db,questions,queue,utils}
mkdir -p proto config/sections config/questions
mkdir -p tests/{unit,integration,load}
```

---

## Security Considerations

- [ ] No secrets in committed files
- [ ] `.env` in `.gitignore`
- [ ] Docker containers use non-root users in production
- [ ] Dependency versions pinned (no `^` in production)

---

## Performance Requirements

N/A - Setup task only.

---

## Error Handling

N/A - Setup task only.

---

## Notes

- Use `msgpack` instead of JSON for Redis serialization (added to deps)
- Include ESLint for code quality
- Jest configured for unit, integration, and load test separation
