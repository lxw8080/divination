# TypeScript 配置指南

## 项目配置文件

### 1. 根目录 tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ES2018",
    "module": "CommonJS",
    "lib": ["ES2018"],
    "allowJs": true,
    "checkJs": false,
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "node",
    "baseUrl": ".",
    "paths": {
      "@/*": ["./miniprogram/*"],
      "@/utils/*": ["./miniprogram/utils/*"],
      "@/components/*": ["./miniprogram/components/*"]
    },
    "typeRoots": [
      "./node_modules/@types",
      "./miniprogram/@types"
    ],
    "types": ["miniprogram-api-typings"]
  },
  "include": [
    "miniprogram/**/*.ts",
    "cloudfunctions/**/*.ts"
  ],
  "exclude": [
    "node_modules",
    "miniprogram/**/*.js"
  ]
}
```

### 2. 小程序专用配置
```json
// miniprogram/tsconfig.json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "target": "ES5",
    "module": "ES2015",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  },
  "include": [
    "**/*.ts",
    "**/*.wxml.ts" // 如果使用 WXML 类型声明
  ]
}
```

### 3. 云函数配置
```json
// cloudfunctions/tsconfig.json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "target": "ES2018",
    "module": "CommonJS",
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": [
    "**/*.ts"
  ]
}
```

## 类型声明文件

### 4. 微信小程序类型声明
```typescript
// miniprogram/@types/wx.d.ts
declare namespace WeChatMiniprogram {
  interface WxCloud {
    init(options: { env: string; traceUser?: boolean }): void
    callFunction(options: {
      name: string
      data?: any
      success?: (res: any) => void
      fail?: (error: any) => void
      complete?: () => void
    }): Promise<any>
    getOpenId(): Promise<string>
  }

  interface WxAnimation {
    rotateY(angle: number): WxAnimation
    step(): WxAnimation
    export(): any
  }

  interface Wx {
    cloud: WxCloud
    createAnimation(options?: { duration?: number; timingFunction?: string }): WxAnimation
    getSystemInfoSync(): any
    setStorageSync(key: string, data: any): void
    getStorageSync(key: string): any
  }
}

declare const wx: WeChatMiniprogram.Wx
```

### 5. 业务类型定义
```typescript
// miniprogram/types/index.ts
export interface CoinResult {
  headsCount: number
  isYang: boolean
  isChanging: boolean
  positionName: string
  coinResults: boolean[]
}

export interface HexagramLine extends CoinResult {
  position: number
  changingTo?: 'yin' | 'yang'
}

export interface TrigramInfo {
  name: string
  index: number
  lines: HexagramLine[]
}

export interface HexagramResult {
  lines: HexagramLine[]
  upperTrigram: TrigramInfo
  lowerTrigram: TrigramInfo
  guaNumber: number
  guaName: string
  guaMark: string
  guaTitle: string
  changingLines: HexagramLine[]
  changingHexagram?: HexagramResult
}

export interface DivinationRecord {
  _id?: string
  userId: string
  question: string
  primaryHexagram: number
  changingHexagram?: number
  changingLines: number[]
  aiSummary: string
  status: 'pending' | 'completed' | 'error'
  createdAt?: Date
  updatedAt?: Date
}

export interface AIInterpretationRequest {
  question: string
  hexagramData: HexagramResult
  taskId?: string
  poll?: boolean
}

export interface AIInterpretationResponse {
  success: boolean
  data?: string
  taskId?: string
  status?: 'processing' | 'completed' | 'error'
  error?: string
}
```

## ESLint 配置

### 6. ESLint 配置文件
```json
// .eslintrc.json
{
  "extends": [
    "@typescript-eslint/recommended",
    "prettier"
  ],
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "ecmaVersion": 2018,
    "sourceType": "module",
    "project": "./tsconfig.json"
  },
  "plugins": [
    "@typescript-eslint",
    "prettier"
  ],
  "rules": {
    "prettier/prettier": "error",
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/explicit-function-return-type": "warn",
    "@typescript-eslint/no-explicit-any": "warn",
    "prefer-const": "error",
    "no-var": "error"
  },
  "overrides": [
    {
      "files": ["*.wxml"],
      "rules": {
        "@typescript-eslint/no-explicit-any": "off"
      }
    }
  ]
}
```

### 7. Prettier 配置
```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "bracketSpacing": true,
  "arrowParens": "avoid"
}
```

## 编译脚本

### 8. package.json 脚本
```json
{
  "scripts": {
    "build": "tsc --noEmit",
    "build:watch": "tsc --watch",
    "lint": "eslint miniprogram/**/*.ts --fix",
    "type-check": "tsc --noEmit",
    "cloud-build": "cd cloudfunctions && npm run build",
    "prepare": "npm run build && npm run lint"
  },
  "devDependencies": {
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.0.0",
    "eslint-config-prettier": "^9.0.0",
    "eslint-plugin-prettier": "^5.0.0",
    "prettier": "^3.0.0",
    "typescript": "^5.0.0",
    "miniprogram-api-typings": "^3.12.0"
  }
}
```

## 云函数依赖管理

### 9. 云函数 package.json 模板
```json
// cloudfunctions/getAIInterpretation/package.json
{
  "name": "getAIInterpretation",
  "version": "1.0.0",
  "description": "AI解读云函数",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "build:watch": "tsc --watch",
    "dev": "npm run build && node dist/index.js",
    "lint": "eslint src/**/*.ts --fix"
  },
  "dependencies": {
    "wx-server-sdk": "~2.6.3",
    "openai": "^4.20.1",
    "node-fetch": "^3.3.2",
    "crypto": "^1.0.1"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0",
    "eslint": "^8.0.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0"
  }
}
```

### 10. 其他云函数配置
```json
// cloudfunctions/getHexagramData/package.json
{
  "name": "getHexagramData",
  "version": "1.0.0",
  "description": "卦象数据服务",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc"
  },
  "dependencies": {
    "wx-server-sdk": "~2.6.3"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0"
  }
}
```

```json
// cloudfunctions/saveDivinationRecord/package.json
{
  "name": "saveDivinationRecord",
  "version": "1.0.0",
  "description": "记录存储服务",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc"
  },
  "dependencies": {
    "wx-server-sdk": "~2.6.3",
    "moment": "^2.29.4"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/moment": "^2.13.0",
    "typescript": "^5.0.0"
  }
}
```

### 11. 批量依赖管理脚本
```bash
#!/bin/bash
# scripts/install-cloud-deps.sh

echo "安装云函数依赖..."

# 检查云函数目录
CLOUD_FUNCTIONS_DIR="cloudfunctions"
if [ ! -d "$CLOUD_FUNCTIONS_DIR" ]; then
    echo "错误: cloudfunctions 目录不存在"
    exit 1
fi

# 遍历所有云函数
for func_dir in "$CLOUD_FUNCTIONS_DIR"/*; do
    if [ -d "$func_dir" ] && [ -f "$func_dir/package.json" ]; then
        func_name=$(basename "$func_dir")
        echo "安装 $func_name 依赖..."
        cd "$func_dir"

        # 安装生产依赖
        npm install --production

        # 如果是开发环境，也安装开发依赖
        if [ "$NODE_ENV" = "development" ]; then
            npm install
        fi

        # 构建TypeScript
        if [ -f "tsconfig.json" ]; then
            npm run build
        fi

        cd ../..
        echo "✅ $func_name 依赖安装完成"
    fi
done

echo "所有云函数依赖安装完成!"
```

### 12. 云函数构建配置
```javascript
// cloudfunctions/build.config.js
module.exports = {
  // 云函数构建配置
  functions: [
    {
      name: 'getAIInterpretation',
      src: './cloudfunctions/getAIInterpretation/src',
      dist: './cloudfunctions/getAIInterpretation/dist',
      dependencies: ['openai', 'node-fetch'],
      timeout: 20000, // 20秒超时
      memorySize: 256  // 256MB内存
    },
    {
      name: 'getHexagramData',
      src: './cloudfunctions/getHexagramData/src',
      dist: './cloudfunctions/getHexagramData/dist',
      dependencies: [],
      timeout: 5000,
      memorySize: 128
    },
    {
      name: 'saveDivinationRecord',
      src: './cloudfunctions/saveDivinationRecord/src',
      dist: './cloudfunctions/saveDivinationRecord/dist',
      dependencies: ['moment'],
      timeout: 10000,
      memorySize: 128
    }
  ]
};
```

## 环境变量配置

### 13. 云函数环境变量配置
```bash
# scripts/setup-env.sh

# 设置云函数环境变量
echo "配置云函数环境变量..."

# OpenAI配置
wxcloud env set OPENAI_API_KEY="your-openai-api-key-here"
wxcloud env set OPENAI_BASE_URL="https://api.openai.com/v1"
wxcloud env set OPENAI_MODEL="gpt-4o-mini"

# 应用配置
wxcloud env set RATE_LIMIT_PER_MINUTE="10"
wxcloud env set CACHE_TTL="3600"
wxcloud env set MAX_RESPONSE_LENGTH="2000"

# 安全配置
wxcloud env set ENCRYPTION_KEY="your-encryption-key-here"
wxcloud env set JWT_SECRET="your-jwt-secret-here"

echo "环境变量配置完成!"
```

### 14. 本地开发环境配置
```bash
# .env.local
OPENAI_API_KEY=sk-your-development-key
OPENAI_BASE_URL=https://api.openai.com/v1
OPENAI_MODEL=gpt-4o-mini
RATE_LIMIT_PER_MINUTE=60
CACHE_TTL=1800
NODE_ENV=development
```

```javascript
// cloudfunctions/config/index.js
const config = {
  openai: {
    apiKey: process.env.OPENAI_API_KEY,
    baseURL: process.env.OPENAI_BASE_URL || 'https://api.openai.com/v1',
    model: process.env.OPENAI_MODEL || 'gpt-4o-mini',
    maxTokens: parseInt(process.env.MAX_RESPONSE_LENGTH) || 2000,
    temperature: 0.5
  },
  rateLimit: {
    perMinute: parseInt(process.env.RATE_LIMIT_PER_MINUTE) || 10,
    perHour: parseInt(process.env.RATE_LIMIT_PER_HOUR) || 100
  },
  cache: {
    ttl: parseInt(process.env.CACHE_TTL) || 3600,
    maxSize: parseInt(process.env.CACHE_MAX_SIZE) || 100
  },
  security: {
    encryptionKey: process.env.ENCRYPTION_KEY,
    jwtSecret: process.env.JWT_SECRET
  }
};

module.exports = config;
```
```