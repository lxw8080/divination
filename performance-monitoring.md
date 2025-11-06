# 性能指标与监控方案

## 性能基准定义

### 1. 前端性能指标

#### 1.1 页面加载性能
```typescript
// utils/performance-metrics.ts
export interface PageLoadMetrics {
  // 页面首次渲染时间
  firstContentfulPaint: number;    // 目标: < 800ms
  // 页面完全加载时间
  domContentLoaded: number;        // 目标: < 1500ms
  // 资源加载完成时间
  loadComplete: number;            // 目标: < 3000ms
  // 小程序启动时间
  appLaunchTime: number;           // 目标: < 1000ms
}

export const PAGE_LOAD_THRESHOLDS = {
  EXCELLENT: { firstContentfulPaint: 600, domContentLoaded: 1200, loadComplete: 2500 },
  GOOD: { firstContentfulPaint: 1000, domContentLoaded: 2000, loadComplete: 4000 },
  POOR: { firstContentfulPaint: 1500, domContentLoaded: 3000, loadComplete: 6000 }
};
```

#### 1.2 动画性能指标
```typescript
export interface AnimationMetrics {
  // 硬币动画性能
  coinAnimation: {
    frameRate: number;             // 目标: ≥ 55 FPS
    frameDropCount: number;        // 目标: ≤ 2 次/动画
    animationDuration: number;     // 目标: 2000-3000ms
    memoryUsage: number;           // 目标: ≤ 50MB 增长
  };
  // 页面切换动画
  pageTransition: {
    duration: number;              // 目标: < 500ms
    smoothness: number;            // 目标: ≥ 90% 流畅度
  };
}

export const ANIMATION_THRESHOLDS = {
  coinAnimation: {
    minFrameRate: 55,
    maxFrameDrop: 2,
    maxMemoryIncrease: 50 * 1024 * 1024, // 50MB
    maxDuration: 3000
  }
};
```

#### 1.3 API响应性能
```typescript
export interface APIMetrics {
  // 云函数响应时间
  cloudFunction: {
    getHexagramData: {
      average: number;             // 目标: < 500ms
      p95: number;                 // 目标: < 1000ms
      p99: number;                 // 目标: < 2000ms
    };
    getAIInterpretation: {
      average: number;             // 目标: < 5000ms
      p95: number;                 // 目标: < 10000ms
      timeout: number;             // 目标: 30000ms
    };
    saveDivinationRecord: {
      average: number;             // 目标: < 300ms
      p95: number;                 // 目标: < 800ms
    };
  };
  // 网络请求成功率
  successRate: {
    target: number;                // 目标: ≥ 99.5%
    minimum: number;               // 最低: ≥ 98%
  };
}

export const API_THRESHOLDS = {
  responseTime: {
    fast: 500,      // < 500ms
    normal: 2000,   // < 2s
    slow: 5000,     // < 5s
    timeout: 30000  // 30s
  },
  successRate: {
    excellent: 99.9,  // ≥ 99.9%
    good: 99.5,       // ≥ 99.5%
    minimum: 98.0     // ≥ 98%
  }
};
```

### 2. 云函数性能指标

#### 2.1 执行性能
```typescript
export interface CloudFunctionMetrics {
  // 执行时间统计
  executionTime: {
    average: number;               // 平均执行时间
    max: number;                   // 最大执行时间
    min: number;                   // 最小执行时间
    p95: number;                   // 95分位执行时间
    p99: number;                   // 99分位执行时间
  };
  // 内存使用情况
  memoryUsage: {
    average: number;               // 平均内存使用 (MB)
    peak: number;                  // 峰值内存使用 (MB)
    limit: number;                 // 内存限制 (MB)
  };
  // 错误统计
  errors: {
    count: number;                 // 错误次数
    rate: number;                  // 错误率 (%)
    types: Record<string, number>; // 错误类型分布
  };
}

export const CLOUD_FUNCTION_THRESHOLDS = {
  executionTime: {
    getAIInterpretation: { max: 20000, target: 8000 },
    getHexagramData: { max: 5000, target: 1000 },
    saveDivinationRecord: { max: 3000, target: 500 }
  },
  memoryUsage: {
    getAIInterpretation: { limit: 256, warning: 200 },
    getHexagramData: { limit: 128, warning: 100 },
    saveDivinationRecord: { limit: 128, warning: 100 }
  },
  errorRate: {
    excellent: 0.1,    // < 0.1%
    good: 0.5,         // < 0.5%
    maximum: 2.0       // < 2%
  }
};
```

## 性能监控实现

### 3. 前端性能监控

#### 3.1 性能数据收集器
```typescript
// miniprogram/utils/performance-collector.ts
class PerformanceCollector {
  private metrics: Map<string, any> = new Map();
  private observers: PerformanceObserver[] = [];

  constructor() {
    this.initPerformanceObserver();
  }

  // 初始化性能观察器
  private initPerformanceObserver() {
    if (typeof PerformanceObserver !== 'undefined') {
      // 页面加载性能
      const pageObserver = new PerformanceObserver((list) => {
        const entries = list.getEntries();
        entries.forEach((entry) => {
          if (entry.entryType === 'navigation') {
            this.recordPageLoadMetrics(entry);
          }
        });
      });
      pageObserver.observe({ entryTypes: ['navigation'] });
      this.observers.push(pageObserver);

      // 资源加载性能
      const resourceObserver = new PerformanceObserver((list) => {
        const entries = list.getEntries();
        this.recordResourceMetrics(entries);
      });
      resourceObserver.observe({ entryTypes: ['resource'] });
      this.observers.push(resourceObserver);
    }
  }

  // 记录页面加载指标
  private recordPageLoadMetrics(entry: PerformanceNavigationTiming) {
    const metrics = {
      domContentLoaded: entry.domContentLoadedEventEnd - entry.domContentLoadedEventStart,
      loadComplete: entry.loadEventEnd - entry.loadEventStart,
      firstContentfulPaint: this.getFirstContentfulPaint(),
      appLaunchTime: this.getAppLaunchTime()
    };

    this.metrics.set('pageLoad', metrics);
    this.reportMetrics('pageLoad', metrics);
  }

  // 记录动画性能
  recordAnimationMetrics(animationType: string, startTime: number, endTime: number) {
    const frameCount = this.getFrameCount(startTime, endTime);
    const duration = endTime - startTime;
    const frameRate = (frameCount / duration) * 1000;

    const metrics = {
      frameRate,
      frameDropCount: this.getFrameDropCount(startTime, endTime),
      animationDuration: duration,
      memoryUsage: this.getMemoryUsage()
    };

    this.metrics.set(`${animationType}Animation`, metrics);
    this.reportMetrics(`${animationType}Animation`, metrics);
  }

  // 记录API调用性能
  recordAPICall(apiName: string, startTime: number, endTime: number, success: boolean, error?: any) {
    const duration = endTime - startTime;
    const key = `api_${apiName}`;

    const existingMetrics = this.metrics.get(key) || {
      callCount: 0,
      totalDuration: 0,
      successCount: 0,
      errorCount: 0,
      errors: {}
    };

    existingMetrics.callCount++;
    existingMetrics.totalDuration += duration;

    if (success) {
      existingMetrics.successCount++;
    } else {
      existingMetrics.errorCount++;
      const errorType = error?.code || 'UNKNOWN';
      existingMetrics.errors[errorType] = (existingMetrics.errors[errorType] || 0) + 1;
    }

    this.metrics.set(key, existingMetrics);
    this.reportMetrics(key, existingMetrics);
  }

  // 上报性能数据
  private async reportMetrics(type: string, metrics: any) {
    try {
      await wx.cloud.callFunction({
        name: 'reportPerformance',
        data: {
          type,
          metrics,
          timestamp: Date.now(),
          userAgent: wx.getSystemInfoSync()
        }
      });
    } catch (error) {
      console.warn('性能数据上报失败:', error);
      // 本地存储失败数据，稍后重试
      this.cacheFailedReport(type, metrics);
    }
  }

  // 获取性能摘要
  getPerformanceSummary() {
    const summary: any = {};

    // 页面加载性能评级
    const pageLoadMetrics = this.metrics.get('pageLoad');
    if (pageLoadMetrics) {
      summary.pageLoad = this.ratePageLoadPerformance(pageLoadMetrics);
    }

    // API性能评级
    const apiMetrics = this.getAPIMetricsSummary();
    summary.api = apiMetrics;

    // 动画性能评级
    const animationMetrics = this.getAnimationMetricsSummary();
    summary.animation = animationMetrics;

    return summary;
  }

  // 页面加载性能评级
  private ratePageLoadPerformance(metrics: any) {
    const { firstContentfulPaint, domContentLoaded, loadComplete } = metrics;

    if (firstContentfulPaint < 600 && domContentLoaded < 1200 && loadComplete < 2500) {
      return { rating: 'excellent', score: 95 };
    } else if (firstContentfulPaint < 1000 && domContentLoaded < 2000 && loadComplete < 4000) {
      return { rating: 'good', score: 80 };
    } else {
      return { rating: 'poor', score: 60 };
    }
  }
}

export default new PerformanceCollector();
```

#### 3.2 API性能监控装饰器
```typescript
// miniprogram/utils/api-monitor.ts
export function monitorAPICall(apiName: string) {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;

    descriptor.value = async function(...args: any[]) {
      const startTime = Date.now();
      let success = true;
      let error: any;

      try {
        const result = await originalMethod.apply(this, args);
        return result;
      } catch (err) {
        success = false;
        error = err;
        throw err;
      } finally {
        const endTime = Date.now();

        // 记录API调用性能
        const performanceCollector = require('./performance-collector').default;
        performanceCollector.recordAPICall(apiName, startTime, endTime, success, error);
      }
    };

    return descriptor;
  };
}

// 使用示例
class DivinationService {
  @monitorAPICall('getHexagramData')
  async getHexagramData(hexagramId: string) {
    return await wx.cloud.callFunction({
      name: 'getHexagramData',
      data: { hexagramId }
    });
  }

  @monitorAPICall('getAIInterpretation')
  async getAIInterpretation(question: string, hexagramData: any) {
    return await wx.cloud.callFunction({
      name: 'getAIInterpretation',
      data: { question, hexagramData }
    });
  }
}
```

### 4. 云函数性能监控

#### 4.1 云函数性能监控中间件
```javascript
// cloudfunctions/middleware/performance-monitor.js
const crypto = require('crypto');

class PerformanceMonitor {
  constructor() {
    this.metrics = new Map();
  }

  // 中间件函数
  middleware() {
    return (req, res, next) => {
      const startTime = Date.now();
      const requestId = this.generateRequestId();

      // 添加请求ID到上下文
      req.requestId = requestId;

      // 监听响应结束
      const originalEnd = res.end;
      res.end = function(...args) {
        const endTime = Date.now();
        const duration = endTime - startTime;

        // 记录性能指标
        req.performanceMonitor.recordFunctionMetrics(
          req.functionName,
          duration,
          req.memoryUsage,
          res.statusCode,
          req.error
        );

        originalEnd.apply(this, args);
      };

      next();
    };
  }

  // 记录云函数性能指标
  recordFunctionMetrics(functionName, duration, memoryUsage, statusCode, error) {
    const key = `function_${functionName}`;
    const existing = this.metrics.get(key) || {
      callCount: 0,
      totalDuration: 0,
      minDuration: Infinity,
      maxDuration: 0,
      errorCount: 0,
      memoryStats: {
        total: 0,
        min: Infinity,
        max: 0
      }
    };

    existing.callCount++;
    existing.totalDuration += duration;
    existing.minDuration = Math.min(existing.minDuration, duration);
    existing.maxDuration = Math.max(existing.maxDuration, duration);

    if (statusCode >= 400 || error) {
      existing.errorCount++;
    }

    if (memoryUsage) {
      existing.memoryStats.total += memoryUsage;
      existing.memoryStats.min = Math.min(existing.memoryStats.min, memoryUsage);
      existing.memoryStats.max = Math.max(existing.memoryStats.max, memoryUsage);
    }

    this.metrics.set(key, existing);

    // 异步上报到监控系统
    this.reportToCloudWatch(functionName, {
      duration,
      memoryUsage,
      statusCode,
      timestamp: Date.now()
    });
  }

  // 获取性能统计
  getMetricsSummary() {
    const summary = {};

    for (const [key, data] of this.metrics.entries()) {
      const functionName = key.replace('function_', '');
      const averageDuration = data.totalDuration / data.callCount;
      const errorRate = (data.errorCount / data.callCount) * 100;
      const averageMemory = data.memoryStats.total / data.callCount;

      summary[functionName] = {
        callCount: data.callCount,
        averageDuration: Math.round(averageDuration),
        minDuration: data.minDuration,
        maxDuration: data.maxDuration,
        errorRate: Math.round(errorRate * 100) / 100,
        averageMemory: Math.round(averageMemory),
        memoryPeak: data.memoryStats.max
      };
    }

    return summary;
  }

  // 性能健康检查
  healthCheck() {
    const summary = this.getMetricsSummary();
    const health = {
      overall: 'healthy',
      issues: [],
      recommendations: []
    };

    for (const [functionName, metrics] of Object.entries(summary)) {
      // 检查响应时间
      const thresholds = CLOUD_FUNCTION_THRESHOLDS.executionTime[functionName];
      if (thresholds && metrics.averageDuration > thresholds.target) {
        health.overall = 'warning';
        health.issues.push(`${functionName}平均响应时间过长: ${metrics.averageDuration}ms`);
        health.recommendations.push(`优化${functionName}算法或增加超时时间`);
      }

      // 检查错误率
      if (metrics.errorRate > CLOUD_FUNCTION_THRESHOLDS.errorRate.good) {
        health.overall = 'unhealthy';
        health.issues.push(`${functionName}错误率过高: ${metrics.errorRate}%`);
        health.recommendations.push(`检查${functionName}的错误处理逻辑`);
      }

      // 检查内存使用
      const memoryThreshold = CLOUD_FUNCTION_THRESHOLDS.memoryUsage[functionName];
      if (memoryThreshold && metrics.memoryPeak > memoryThreshold.warning) {
        health.overall = 'warning';
        health.issues.push(`${functionName}内存使用过高: ${metrics.memoryPeak}MB`);
        health.recommendations.push(`优化${functionName}内存使用或增加内存限制`);
      }
    }

    return health;
  }

  generateRequestId() {
    return crypto.randomBytes(16).toString('hex');
  }

  async reportToCloudWatch(functionName, metrics) {
    // 这里可以集成腾讯云监控或其他监控服务
    try {
      // 示例：上报到云开发日志系统
      console.log(`Performance: ${functionName}`, JSON.stringify(metrics));
    } catch (error) {
      console.error('性能监控上报失败:', error);
    }
  }
}

module.exports = PerformanceMonitor;
```

#### 4.2 云函数集成示例
```javascript
// cloudfunctions/getAIInterpretation/index.js
const cloud = require('wx-server-sdk');
const PerformanceMonitor = require('../middleware/performance-monitor');

cloud.init();

const performanceMonitor = new PerformanceMonitor();

// 使用性能监控中间件
const withPerformanceMonitoring = (handler) => {
  return async (event, context) => {
    context.performanceMonitor = performanceMonitor;
    context.functionName = 'getAIInterpretation';

    // 模拟req/res对象供中间件使用
    const mockReq = {
      functionName: 'getAIInterpretation',
      memoryUsage: context.memory_limit_in_mb,
      performanceMonitor
    };

    const mockRes = {
      statusCode: 200,
      end: function() {}
    };

    // 应用中间件
    const middleware = performanceMonitor.middleware();
    middleware(mockReq, mockRes, () => {});

    try {
      const result = await handler(event, context);
      mockRes.statusCode = 200;
      return result;
    } catch (error) {
      mockRes.statusCode = 500;
      mockReq.error = error;
      throw error;
    }
  };
};

exports.main = withPerformanceMonitoring(async (event, context) => {
  // 业务逻辑
  const { question, hexagramData } = event;

  // ... AI调用逻辑

  return {
    success: true,
    data: interpretation
  };
});
```

## 性能优化建议

### 5. 前端优化策略

#### 5.1 代码优化
```typescript
// miniprogram/utils/optimization.ts
export class OptimizationUtils {
  // 防抖函数
  static debounce<T extends (...args: any[]) => void>(
    func: T,
    wait: number
  ): (...args: Parameters<T>) => void {
    let timeout: NodeJS.Timeout;
    return function executedFunction(...args: Parameters<T>) {
      const later = () => {
        clearTimeout(timeout);
        func(...args);
      };
      clearTimeout(timeout);
      timeout = setTimeout(later, wait);
    };
  }

  // 节流函数
  static throttle<T extends (...args: any[]) => void>(
    func: T,
    limit: number
  ): (...args: Parameters<T>) => void {
    let inThrottle: boolean;
    return function executedFunction(...args: Parameters<T>) {
      if (!inThrottle) {
        func.apply(this, args);
        inThrottle = true;
        setTimeout(() => inThrottle = false, limit);
      }
    };
  }

  // 内存优化
  static optimizeMemory() {
    // 清理不需要的缓存
    if (typeof wx !== 'undefined') {
      wx.triggerGC();
    }

    // 清理事件监听器
    this.clearEventListeners();

    // 清理定时器
    this.clearTimers();
  }

  // 预加载资源
  static preloadResources(resources: string[]) {
    resources.forEach(resource => {
      if (typeof wx !== 'undefined') {
        wx.preloadRequest && wx.preloadRequest({ url: resource });
      }
    });
  }
}
```

#### 5.2 缓存策略
```typescript
// miniprogram/utils/cache-manager.ts
export class CacheManager {
  private static instance: CacheManager;
  private cache: Map<string, CacheItem> = new Map();
  private maxSize = 50; // 最大缓存项数
  private defaultTTL = 5 * 60 * 1000; // 5分钟

  static getInstance(): CacheManager {
    if (!CacheManager.instance) {
      CacheManager.instance = new CacheManager();
    }
    return CacheManager.instance;
  }

  set(key: string, value: any, ttl?: number): void {
    // 清理过期项
    this.cleanExpiredItems();

    // 如果缓存已满，删除最旧的项
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }

    const item: CacheItem = {
      value,
      timestamp: Date.now(),
      ttl: ttl || this.defaultTTL
    };

    this.cache.set(key, item);
  }

  get(key: string): any | null {
    const item = this.cache.get(key);

    if (!item) {
      return null;
    }

    // 检查是否过期
    if (Date.now() - item.timestamp > item.ttl) {
      this.cache.delete(key);
      return null;
    }

    return item.value;
  }

  // 智能缓存API响应
  async cacheAPICall<T>(
    key: string,
    apiCall: () => Promise<T>,
    ttl?: number
  ): Promise<T> {
    // 尝试从缓存获取
    const cached = this.get(key);
    if (cached !== null) {
      return cached;
    }

    // 执行API调用
    try {
      const result = await apiCall();
      this.set(key, result, ttl);
      return result;
    } catch (error) {
      // 如果API调用失败，尝试返回过期的缓存数据
      const expired = this.getExpiredItem(key);
      if (expired) {
        console.warn('API调用失败，返回过期缓存数据:', key);
        return expired;
      }
      throw error;
    }
  }

  private cleanExpiredItems(): void {
    const now = Date.now();
    for (const [key, item] of this.cache.entries()) {
      if (now - item.timestamp > item.ttl) {
        this.cache.delete(key);
      }
    }
  }

  private getExpiredItem(key: string): any | null {
    const item = this.cache.get(key);
    if (item && Date.now() - item.timestamp > item.ttl) {
      return item.value;
    }
    return null;
  }

  private clearEventListeners(): void {
    // 清理事件监听器的实现
  }

  private clearTimers(): void {
    // 清理定时器的实现
  }
}

interface CacheItem {
  value: any;
  timestamp: number;
  ttl: number;
}
```

### 6. 监控告警设置

#### 6.1 性能告警配置
```yaml
# monitoring/performance-alerts.yml
alerts:
  - name: "页面加载时间过长"
    condition: "page_load.first_contentful_paint > 1500"
    severity: "warning"
    threshold: 1500ms
    action: "优化页面资源加载"

  - name: "API响应时间过长"
    condition: "api.get_ai_interpretation.p95 > 10000"
    severity: "critical"
    threshold: 10000ms
    action: "检查AI服务状态或优化算法"

  - name: "云函数错误率过高"
    condition: "cloud_function.error_rate > 2.0"
    severity: "critical"
    threshold: 2.0%
    action: "立即检查错误日志"

  - name: "动画掉帧严重"
    condition: "animation.coin_animation.frame_rate < 30"
    severity: "warning"
    threshold: 30 FPS
    action: "优化动画性能"

  - name: "内存使用过高"
    condition: "cloud_function.memory_usage > 200"
    severity: "warning"
    threshold: 200MB
    action: "优化内存使用或增加内存限制"
```

#### 6.2 性能报告生成
```typescript
// utils/performance-report.ts
export class PerformanceReport {
  static generateDailyReport() {
    const performanceCollector = require('./performance-collector').default;
    const summary = performanceCollector.getPerformanceSummary();

    const report = {
      date: new Date().toISOString().split('T')[0],
      summary,
      recommendations: this.generateRecommendations(summary),
      trends: this.getTrendsData()
    };

    return report;
  }

  static generateRecommendations(summary: any) {
    const recommendations = [];

    // 页面性能建议
    if (summary.pageLoad?.score < 80) {
      recommendations.push({
        type: 'performance',
        priority: 'high',
        title: '优化页面加载性能',
        description: '考虑使用图片压缩、代码分割等优化手段'
      });
    }

    // API性能建议
    if (summary.api?.averageResponseTime > 2000) {
      recommendations.push({
        type: 'api',
        priority: 'medium',
        title: 'API响应时间优化',
        description: '检查云函数性能或增加缓存策略'
      });
    }

    // 动画性能建议
    if (summary.animation?.averageFrameRate < 50) {
      recommendations.push({
        type: 'animation',
        priority: 'medium',
        title: '动画性能优化',
        description: '减少动画复杂度或使用硬件加速'
      });
    }

    return recommendations;
  }

  static async sendReport(report: any) {
    try {
      await wx.cloud.callFunction({
        name: 'sendPerformanceReport',
        data: report
      });
    } catch (error) {
      console.error('性能报告发送失败:', error);
    }
  }
}
```

这个完整的性能监控方案提供了：

1. **明确的性能基准** - 定义了所有关键指标的目标值
2. **全面的监控实现** - 覆盖前端、云函数、API调用等各个环节
3. **智能的性能优化** - 提供缓存、防抖节流等优化策略
4. **实用的告警机制** - 及时发现和响应性能问题
5. **详细的性能报告** - 帮助持续优化应用性能

通过这套方案，可以确保微信小程序在各种使用场景下都能保持良好的性能表现。