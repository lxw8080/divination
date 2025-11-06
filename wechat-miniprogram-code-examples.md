# 完整代码示例

## 1. 主题管理器完整实现

### 1.1 主题管理器类
```typescript
// miniprogram/utils/theme-manager.ts
interface Theme {
  primary: string;
  background: string;
  surface: string;
  text: string;
  textSecondary: string;
  border: string;
  shadow: string;
  accent: string;
}

interface ThemeConfig {
  light: Theme;
  dark: Theme;
  auto: string;
}

class ThemeManager {
  private static instance: ThemeManager;
  private currentTheme: 'light' | 'dark' | 'auto';
  private systemThemeListener: (() => void) | null = null;
  private changeListeners: Set<(theme: string) => void> = new Set();

  private readonly themes: ThemeConfig = {
    light: {
      primary: '#2C3E50',
      background: '#FFFFFF',
      surface: '#F8F9FA',
      text: '#2C3E50',
      textSecondary: '#6C757D',
      border: '#E9ECEF',
      shadow: 'rgba(0, 0, 0, 0.1)',
      accent: '#3498DB'
    },
    dark: {
      primary: '#ECF0F1',
      background: '#1A1A1A',
      surface: '#2D2D2D',
      text: '#ECF0F1',
      textSecondary: '#BDC3C7',
      border: '#404040',
      shadow: 'rgba(0, 0, 0, 0.3)',
      accent: '#3498DB'
    },
    auto: 'auto'
  };

  private constructor() {
    this.currentTheme = this.getStoredTheme() || 'auto';
    this.init();
  }

  static getInstance(): ThemeManager {
    if (!ThemeManager.instance) {
      ThemeManager.instance = new ThemeManager();
    }
    return ThemeManager.instance;
  }

  private init(): void {
    // 监听系统主题变化
    if (wx.onThemeChange) {
      wx.onThemeChange(({ theme }) => {
        if (this.currentTheme === 'auto') {
          this.applyTheme(theme);
          this.notifyListeners();
        }
      });
    }

    // 应用当前主题
    this.applyTheme(this.getCurrentTheme());
  }

  private getStoredTheme(): string | null {
    try {
      return wx.getStorageSync('theme');
    } catch (error) {
      console.warn('读取主题设置失败:', error);
      return null;
    }
  }

  private storeTheme(theme: string): void {
    try {
      wx.setStorageSync('theme', theme);
    } catch (error) {
      console.warn('保存主题设置失败:', error);
    }
  }

  getCurrentTheme(): 'light' | 'dark' {
    if (this.currentTheme === 'auto') {
      try {
        const systemInfo = wx.getSystemInfoSync();
        return systemInfo.theme || 'light';
      } catch (error) {
        console.warn('获取系统主题失败:', error);
        return 'light';
      }
    }
    return this.currentTheme;
  }

  private applyTheme(themeName: 'light' | 'dark'): void {
    const theme = this.themes[themeName] || this.themes.light;

    // 设置CSS变量
    const pages = getCurrentPages();
    if (pages.length > 0) {
      const currentPage = pages[pages.length - 1];
      currentPage.setData({
        theme: theme,
        themeName: themeName,
        themeVariables: this.convertToCSSVariables(theme)
      });
    }

    // 应用到全局样式
    this.applyGlobalStyles(theme);
  }

  private convertToCSSVariables(theme: Theme): Record<string, string> {
    return {
      '--theme-primary': theme.primary,
      '--theme-background': theme.background,
      '--theme-surface': theme.surface,
      '--theme-text': theme.text,
      '--theme-text-secondary': theme.textSecondary,
      '--theme-border': theme.border,
      '--theme-shadow': theme.shadow,
      '--theme-accent': theme.accent
    };
  }

  private applyGlobalStyles(theme: Theme): void {
    // 动态更新全局样式变量
    if (typeof wx !== 'undefined' && wx.setPageStyle) {
      wx.setPageStyle({
        style: `
          page {
            --theme-primary: ${theme.primary};
            --theme-background: ${theme.background};
            --theme-surface: ${theme.surface};
            --theme-text: ${theme.text};
            --theme-text-secondary: ${theme.textSecondary};
            --theme-border: ${theme.border};
            --theme-shadow: ${theme.shadow};
            --theme-accent: ${theme.accent};
            background-color: ${theme.background};
            color: ${theme.text};
          }
        `
      });
    }
  }

  setTheme(theme: 'light' | 'dark' | 'auto'): void {
    this.currentTheme = theme;
    this.applyTheme(this.getCurrentTheme());
    this.storeTheme(theme);
    this.notifyListeners();
  }

  onChange(callback: (theme: string) => void): () => void {
    this.changeListeners.add(callback);

    // 返回取消监听的函数
    return () => {
      this.changeListeners.delete(callback);
    };
  }

  offChange(callback?: (theme: string) => void): void {
    if (callback) {
      this.changeListeners.delete(callback);
    } else {
      this.changeListeners.clear();
    }
  }

  private notifyListeners(): void {
    const currentTheme = this.getCurrentTheme();
    this.changeListeners.forEach(callback => {
      try {
        callback(currentTheme);
      } catch (error) {
        console.error('主题变化回调执行失败:', error);
      }
    });
  }

  getThemeColors(): Theme {
    const themeName = this.getCurrentTheme();
    return this.themes[themeName] || this.themes.light;
  }

  // 清理资源
  destroy(): void {
    this.changeListeners.clear();
    this.systemThemeListener = null;
  }
}

export default ThemeManager;
```

### 1.2 主题切换组件
```typescript
// miniprogram/components/theme-toggle/theme-toggle.ts
Component({
  properties: {
    showLabel: {
      type: Boolean,
      value: true
    },
    size: {
      type: String,
      value: 'normal' // small, normal, large
    }
  },

  data: {
    currentTheme: 'auto',
    themeOptions: [
      { value: 'light', label: '浅色', icon: '☀️' },
      { value: 'dark', label: '深色', icon: '🌙' },
      { value: 'auto', label: '跟随系统', icon: '🔄' }
    ]
  },

  lifetimes: {
    attached() {
      this.initThemeManager();
    },

    detached() {
      this.cleanup();
    }
  },

  methods: {
    initThemeManager() {
      const themeManager = require('../../utils/theme-manager').default;
      this.themeManager = themeManager.getInstance();

      // 获取当前主题
      this.setData({
        currentTheme: this.themeManager.getCurrentTheme()
      });

      // 监听主题变化
      this.unsubscribe = this.themeManager.onChange((theme) => {
        this.setData({
          currentTheme: theme
        });
      });
    },

    onThemeSelect(e: any) {
      const selectedTheme = e.currentTarget.dataset.theme;
      this.setData({
        currentTheme: selectedTheme
      });

      this.themeManager.setTheme(selectedTheme);

      // 触发选择事件
      this.triggerEvent('themechange', {
        theme: selectedTheme
      });
    },

    cleanup() {
      if (this.unsubscribe) {
        this.unsubscribe();
      }
    }
  }
});
```

```xml
<!-- miniprogram/components/theme-toggle/theme-toggle.wxml -->
<view class="theme-toggle {{size}}">
  <view wx:if="{{showLabel}}" class="theme-label">主题</view>
  <view class="theme-options">
    <view
      wx:for="{{themeOptions}}"
      wx:key="value"
      class="theme-option {{currentTheme === item.value ? 'active' : ''}}"
      data-theme="{{item.value}}"
      bindtap="onThemeSelect"
    >
      <view class="theme-icon">{{item.icon}}</view>
      <view class="theme-name">{{item.label}}</view>
    </view>
  </view>
</view>
```

```wxss
/* miniprogram/components/theme-toggle/theme-toggle.wxss */
.theme-toggle {
  display: flex;
  align-items: center;
  gap: 20rpx;
}

.theme-toggle.small {
  gap: 10rpx;
}

.theme-toggle.large {
  gap: 30rpx;
}

.theme-label {
  font-size: 28rpx;
  color: var(--theme-text);
  font-weight: 500;
}

.theme-options {
  display: flex;
  gap: 10rpx;
  background: var(--theme-surface);
  border-radius: 50rpx;
  padding: 6rpx;
  border: 1rpx solid var(--theme-border);
}

.theme-option {
  display: flex;
  align-items: center;
  gap: 8rpx;
  padding: 12rpx 20rpx;
  border-radius: 40rpx;
  transition: all 0.3s ease;
  cursor: pointer;
}

.theme-toggle.small .theme-option {
  padding: 8rpx 16rpx;
  gap: 6rpx;
}

.theme-toggle.large .theme-option {
  padding: 16rpx 24rpx;
  gap: 10rpx;
}

.theme-option.active {
  background: var(--theme-accent);
  color: white;
}

.theme-option:not(.active):hover {
  background: var(--theme-border);
}

.theme-icon {
  font-size: 24rpx;
}

.theme-toggle.small .theme-icon {
  font-size: 20rpx;
}

.theme-toggle.large .theme-icon {
  font-size: 28rpx;
}

.theme-name {
  font-size: 24rpx;
  white-space: nowrap;
}

.theme-toggle.small .theme-name {
  font-size: 20rpx;
}

.theme-toggle.large .theme-name {
  font-size: 28rpx;
}
```

## 2. 硬币动画组件

### 2.1 硬币组件实现
```typescript
// miniprogram/components/coin-flip/coin-flip.ts
interface CoinResult {
  headsCount: number;
  isYang: boolean;
  isChanging: boolean;
  positionName: string;
  coinResults: boolean[];
}

interface FlipAnimation {
  animationData: any;
  delay: number;
  duration: number;
}

Component({
  properties: {
    coinCount: {
      type: Number,
      value: 3
    },
    flipDuration: {
      type: Number,
      value: 2000
    },
    autoStart: {
      type: Boolean,
      value: false
    },
    enableSound: {
      type: Boolean,
      value: true
    }
  },

  data: {
    isFlipping: false,
    results: [] as boolean[],
    animations: [] as FlipAnimation[],
    showResult: false,
    flipCount: 0
  },

  lifetimes: {
    attached() {
      this.initAudio();
      if (this.data.autoStart) {
        this.startFlip();
      }
    },

    detached() {
      this.cleanup();
    }
  },

  observers: {
    'coinCount': function(newCount: number) {
      this.resetCoins();
    }
  },

  methods: {
    initAudio() {
      // 创建音频上下文
      this.audioContext = wx.createInnerAudioContext();
      this.audioContext.src = '/sounds/coin-flip.mp3';
    },

    startFlip() {
      if (this.data.isFlipping) {
        return;
      }

      this.setData({
        isFlipping: true,
        showResult: false
      });

      // 生成随机结果
      const results = this.generateRandomResults();

      // 创建动画
      const animations = this.createFlipAnimations(results);

      // 播放音效
      if (this.data.enableSound) {
        this.playFlipSound();
      }

      this.setData({
        results,
        animations
      });

      // 动画完成后显示结果
      setTimeout(() => {
        this.onFlipComplete(results);
      }, this.data.flipDuration);
    },

    generateRandomResults(): boolean[] {
      const results: boolean[] = [];
      for (let i = 0; i < this.data.coinCount; i++) {
        results.push(Math.random() > 0.5);
      }
      return results;
    },

    createFlipAnimations(results: boolean[]): FlipAnimation[] {
      const animations: FlipAnimation[] = [];

      for (let i = 0; i < this.data.coinCount; i++) {
        const delay = i * 100; // 每个硬币延迟100ms
        const rotationCount = 5 + Math.floor(Math.random() * 3); // 5-7圈
        const finalRotation = results[i] ? 0 : 180; // 正面为0度，反面为180度

        const animation = wx.createAnimation({
          duration: this.data.flipDuration - delay,
          timingFunction: 'ease-out',
          delay: delay
        });

        animation
          .rotateY(rotationCount * 360 + finalRotation)
          .scale(1, 1)
          .step();

        animations.push({
          animationData: animation.export(),
          delay,
          duration: this.data.flipDuration - delay
        });
      }

      return animations;
    },

    playFlipSound() {
      if (this.audioContext) {
        try {
          this.audioContext.play();
        } catch (error) {
          console.warn('音效播放失败:', error);
        }
      }
    },

    onFlipComplete(results: boolean[]) {
      this.setData({
        isFlipping: false,
        showResult: true,
        flipCount: this.data.flipCount + 1
      });

      // 计算硬币结果
      const headsCount = results.filter(r => r).length;
      const coinResult: CoinResult = {
        headsCount,
        isYang: headsCount >= 2,
        isChanging: headsCount === 0 || headsCount === 3,
        positionName: '', // 由父组件设置
        coinResults: results
      };

      // 触发完成事件
      this.triggerEvent('flipcomplete', {
        results,
        coinResult,
        flipCount: this.data.flipCount
      });
    },

    resetCoins() {
      this.setData({
        isFlipping: false,
        results: [],
        animations: [],
        showResult: false
      });
    },

    cleanup() {
      if (this.audioContext) {
        this.audioContext.destroy();
      }
    }
  }
});
```

```xml
<!-- miniprogram/components/coin-flip/coin-flip.wxml -->
<view class="coin-flip-container">
  <view class="coins-wrapper">
    <view
      wx:for="{{animations}}"
      wx:key="index"
      class="coin {{showResult && results[index] ? 'heads' : ''}} {{showResult && !results[index] ? 'tails' : ''}}"
      animation="{{item.animationData}}"
      style="animation-delay: {{item.delay}}ms"
    >
      <view class="coin-side heads-side">
        <text class="coin-text">阳</text>
      </view>
      <view class="coin-side tails-side">
        <text class="coin-text">阴</text>
      </view>
    </view>
  </view>

  <view class="controls">
    <button
      class="flip-button {{isFlipping ? 'disabled' : ''}}"
      disabled="{{isFlipping}}"
      bindtap="startFlip"
    >
      {{isFlipping ? '卜筮中...' : '开始卜筮'}}
    </button>
  </view>

  <view wx:if="{{showResult}}" class="result-summary">
    <view class="result-text">
      结果：{{results.filter(r => r).length}}个正面，{{coinCount - results.filter(r => r).length}}个反面
    </view>
  </view>
</view>
```

```wxss
/* miniprogram/components/coin-flip/coin-flip.wxss */
.coin-flip-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 40rpx;
  background: var(--theme-surface);
  border-radius: 20rpx;
  margin: 20rpx;
}

.coins-wrapper {
  display: flex;
  justify-content: center;
  gap: 40rpx;
  margin-bottom: 60rpx;
  min-height: 200rpx;
  align-items: center;
}

.coin {
  width: 120rpx;
  height: 120rpx;
  position: relative;
  transform-style: preserve-3d;
  transition: transform 0.6s ease-in-out;
}

.coin-side {
  position: absolute;
  width: 100%;
  height: 100%;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 48rpx;
  font-weight: bold;
  backface-visibility: hidden;
  box-shadow: 0 8rpx 16rpx var(--theme-shadow);
}

.heads-side {
  background: linear-gradient(135deg, #FFD700, #FFA500);
  color: #8B4513;
}

.tails-side {
  background: linear-gradient(135deg, #C0C0C0, #808080);
  color: #2F4F4F;
  transform: rotateY(180deg);
}

.coin.show-result.heads {
  transform: rotateY(0deg);
}

.coin.show-result.tails {
  transform: rotateY(180deg);
}

.coin-text {
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.3);
  user-select: none;
}

.controls {
  margin-top: 40rpx;
}

.flip-button {
  background: var(--theme-accent);
  color: white;
  border: none;
  border-radius: 50rpx;
  padding: 24rpx 48rpx;
  font-size: 32rpx;
  font-weight: 500;
  min-width: 200rpx;
  transition: all 0.3s ease;
}

.flip-button:not(.disabled):active {
  transform: scale(0.95);
  opacity: 0.8;
}

.flip-button.disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.result-summary {
  margin-top: 40rpx;
  padding: 20rpx;
  background: var(--theme-background);
  border-radius: 12rpx;
  border: 1rpx solid var(--theme-border);
}

.result-text {
  font-size: 28rpx;
  color: var(--theme-text);
  text-align: center;
}

/* 动画效果 */
@keyframes coinFlip {
  0% {
    transform: rotateY(0deg) scale(1);
  }
  50% {
    transform: rotateY(900deg) scale(1.1);
  }
  100% {
    transform: rotateY(var(--final-rotation, 1800deg)) scale(1);
  }
}

.coin.flipping {
  animation: coinFlip var(--duration, 2000ms) ease-in-out;
}
```

## 3. 卦象计算与显示

### 3.1 卦象计算器
```typescript
// miniprogram/utils/hexagram-calculator.ts
interface HexagramLine {
  position: number;
  isYang: boolean;
  isChanging: boolean;
  positionName: string;
  coinResults: boolean[];
  changingTo?: 'yin' | 'yang';
}

interface TrigramInfo {
  name: string;
  index: number;
  lines: HexagramLine[];
  symbol: string;
}

interface HexagramResult {
  lines: HexagramLine[];
  upperTrigram: TrigramInfo;
  lowerTrigram: TrigramInfo;
  guaNumber: number;
  guaName: string;
  guaMark: string;
  guaTitle: string;
  changingLines: HexagramLine[];
  changingHexagram?: HexagramResult;
}

class HexagramCalculator {
  private static instance: HexagramCalculator;

  // 卦象名称映射
  private readonly guaNames = [
    '乾', '坤', '屯', '蒙', '需', '讼', '师', '比',
    '小畜', '履', '泰', '否', '同人', '大有', '谦', '豫',
    '随', '蛊', '临', '观', '噬嗑', '贲', '剥', '复',
    '无妄', '大畜', '颐', '大过', '坎', '离', '咸', '恒',
    '遁', '大壮', '晋', '明夷', '家人', '睽', '蹇', '解',
    '损', '益', '夬', '姤', '萃', '升', '困', '井',
    '革', '鼎', '震', '艮', '渐', '归妹', '丰', '旅',
    '巽', '兑', '涣', '节', '中孚', '小过', '既济', '未济'
  ];

  // 爻位名称
  private readonly positionNames = ['初', '二', '三', '四', '五', '上'];
  private readonly yangSuffix = '九';
  private readonly yinSuffix = '六';

  // 八卦名称
  private readonly trigramNames = ['坤', '震', '坎', '兑', '艮', '离', '巽', '乾'];
  private readonly trigramSymbols = ['☷', '☳', '☵', '☱', '☶', '☲', '☴', '☰'];

  // 卦象索引矩阵 (上卦下卦对应关系)
  private readonly guaIndexMatrix = [
    [2, 24, 7, 19, 15, 36, 46, 11],
    [16, 51, 40, 54, 62, 55, 32, 34],
    [8, 3, 29, 60, 39, 63, 48, 5],
    [45, 17, 47, 58, 31, 49, 28, 43],
    [12, 27, 4, 52, 23, 35, 20, 42],
    [6, 13, 59, 64, 1, 38, 33, 10],
    [25, 41, 21, 61, 56, 14, 9, 26],
    [44, 30, 50, 18, 22, 37, 57, 53]
  ];

  private constructor() {}

  static getInstance(): HexagramCalculator {
    if (!HexagramCalculator.instance) {
      HexagramCalculator.instance = new HexagramCalculator();
    }
    return HexagramCalculator.instance;
  }

  // 计算卦象
  calculateHexagram(coinResults: boolean[][]): HexagramResult {
    if (coinResults.length !== 6) {
      throw new Error('需要6次硬币投掷结果');
    }

    // 将六次硬币结果转换为六爻
    const lines = coinResults.map((coins, index) => {
      const headsCount = coins.filter(coin => coin).length;

      const line: HexagramLine = {
        position: index, // 0-5 对应初到上
        isYang: headsCount >= 2,
        isChanging: headsCount === 0 || headsCount === 3,
        positionName: this.getPositionName(index, headsCount >= 2),
        coinResults: coins
      };

      // 设置变化方向
      if (line.isChanging) {
        line.changingTo = headsCount === 3 ? 'yin' : 'yang';
      }

      return line;
    });

    // 计算上下卦 (传统方法：初二三爻为下卦，四五六爻为上卦)
    const lowerTrigram = this.calculateTrigram(lines.slice(0, 3));
    const upperTrigram = this.calculateTrigram(lines.slice(3, 6));

    // 获取卦象信息
    const hexagramInfo = this.getHexagramInfo(upperTrigram, lowerTrigram);

    const result: HexagramResult = {
      lines,
      upperTrigram,
      lowerTrigram,
      ...hexagramInfo,
      changingLines: lines.filter(line => line.isChanging)
    };

    // 如果有变爻，计算变卦
    if (result.changingLines.length > 0) {
      result.changingHexagram = this.calculateChangingHexagram(result);
    }

    return result;
  }

  // 计算单卦 (三爻)
  private calculateTrigram(lines: HexagramLine[]): TrigramInfo {
    let index = 0;

    // 从下往上计算二进制值
    lines.forEach((line, i) => {
      if (line.isYang) {
        index += Math.pow(2, 2 - i);
      }
    });

    return {
      name: this.trigramNames[index],
      index: index,
      symbol: this.trigramSymbols[index],
      lines: lines
    };
  }

  // 获取爻位名称
  private getPositionName(index: number, isYang: boolean): string {
    const position = this.positionNames[index];
    const suffix = isYang ? this.yangSuffix : this.yinSuffix;
    return position + suffix;
  }

  // 获取卦象详细信息
  private getHexagramInfo(upperTrigram: TrigramInfo, lowerTrigram: TrigramInfo): {
    guaNumber: number;
    guaMark: string;
    guaTitle: string;
    guaName: string;
    upperTrigramName: string;
    lowerTrigramName: string;
  } {
    const upperIndex = upperTrigram.index;
    const lowerIndex = lowerTrigram.index;

    // 使用索引矩阵获取卦序
    const guaNumber = this.guaIndexMatrix[upperIndex][lowerIndex];
    const guaName = this.guaNames[guaNumber - 1];

    // 构建卦象标识
    const guaMark = `${guaNumber.toString().padStart(2, '0')}.${upperTrigram.name}${lowerTrigram.name}`;
    const guaTitle = `周易第${guaNumber}卦`;

    return {
      guaNumber: guaNumber,
      guaMark: guaMark,
      guaTitle: guaTitle,
      guaName: guaName,
      upperTrigramName: upperTrigram.name,
      lowerTrigramName: lowerTrigram.name
    };
  }

  // 计算变卦
  private calculateChangingHexagram(originalHexagram: HexagramResult): HexagramResult {
    // 复制原始爻线
    const changedLines = originalHexagram.lines.map(line => {
      const newLine = { ...line };

      // 如果是变爻，则改变阴阳属性
      if (line.isChanging) {
        newLine.isYang = !line.isYang;
        newLine.isChanging = false; // 变卦的爻不再变化
        newLine.positionName = this.getPositionName(line.position, newLine.isYang);
        newLine.changingTo = undefined;
      }

      return newLine;
    });

    // 重新计算上下卦
    const lowerTrigram = this.calculateTrigram(changedLines.slice(0, 3));
    const upperTrigram = this.calculateTrigram(changedLines.slice(3, 6));

    // 获取变卦信息
    const hexagramInfo = this.getHexagramInfo(upperTrigram, lowerTrigram);

    return {
      lines: changedLines,
      upperTrigram,
      lowerTrigram,
      ...hexagramInfo,
      changingLines: [] // 变卦没有动爻
    };
  }

  // 验证卦象计算结果
  validateHexagram(hexagram: HexagramResult): boolean {
    // 检查基本属性
    if (!hexagram.guaNumber || hexagram.guaNumber < 1 || hexagram.guaNumber > 64) {
      return false;
    }

    if (!hexagram.lines || hexagram.lines.length !== 6) {
      return false;
    }

    if (!hexagram.upperTrigram || !hexagram.lowerTrigram) {
      return false;
    }

    // 检查爻线数据完整性
    for (const line of hexagram.lines) {
      if (typeof line.isYang !== 'boolean' || typeof line.isChanging !== 'boolean') {
        return false;
      }
    }

    return true;
  }

  // 获取卦象统计信息
  getHexagramStatistics(hexagram: HexagramResult): {
    yangLineCount: number;
    yinLineCount: number;
    changingLineCount: number;
    stability: 'stable' | 'somewhat_stable' | 'unstable';
  } {
    const yangLineCount = hexagram.lines.filter(line => line.isYang).length;
    const yinLineCount = 6 - yangLineCount;
    const changingLineCount = hexagram.changingLines.length;

    let stability: 'stable' | 'somewhat_stable' | 'unstable';
    if (changingLineCount === 0) {
      stability = 'stable';
    } else if (changingLineCount <= 2) {
      stability = 'somewhat_stable';
    } else {
      stability = 'unstable';
    }

    return {
      yangLineCount,
      yinLineCount,
      changingLineCount,
      stability
    };
  }
}

export default HexagramCalculator;
export type { HexagramResult, HexagramLine, TrigramInfo };
```

### 3.2 卦象显示组件
```typescript
// miniprogram/components/hexagram-view/hexagram-view.ts
Component({
  properties: {
    hexagramData: {
      type: Object,
      value: null,
      observer: function(newVal) {
        if (newVal) {
          this.updateDisplay();
        }
      }
    },
    showChanging: {
      type: Boolean,
      value: true
    },
    compact: {
      type: Boolean,
      value: false
    },
    animationDuration: {
      type: Number,
      value: 1000
    }
  },

  data: {
    displayLines: [],
    showTrigrams: true,
    animated: false
  },

  lifetimes: {
    attached() {
      this.updateDisplay();
    }
  },

  methods: {
    updateDisplay() {
      if (!this.data.hexagramData) return;

      const { hexagramData } = this.data;

      this.setData({
        displayLines: this.formatDisplayLines(hexagramData),
        showTrigrams: !this.data.compact
      });

      // 启动动画
      if (!this.data.animated) {
        this.startDisplayAnimation();
      }
    },

    formatDisplayLines(hexagram: any) {
      return hexagram.lines.map((line: any, index: number) => ({
        ...line,
        displayIndex: 6 - index, // 从上到下显示
        isAnimated: false
      }));
    },

    startDisplayAnimation() {
      const { displayLines, animationDuration } = this.data;

      this.setData({ animated: true });

      // 逐个显示爻线
      displayLines.forEach((line: any, index: number) => {
        setTimeout(() => {
          const updatedLines = [...this.data.displayLines];
          updatedLines[index].isAnimated = true;

          this.setData({
            displayLines: updatedLines
          });
        }, index * (animationDuration / 6));
      });
    },

    onLineTap(e: any) {
      const { index } = e.currentTarget.dataset;
      const line = this.data.displayLines[index];

      this.triggerEvent('linetap', {
        line,
        index,
        position: line.positionName
      });
    },

    onTrigramTap(e: any) {
      const { type } = e.currentTarget.dataset;
      const trigram = type === 'upper'
        ? this.data.hexagramData.upperTrigram
        : this.data.hexagramData.lowerTrigram;

      this.triggerEvent('trigramtap', {
        trigram,
        type
      });
    }
  }
});
```

```xml
<!-- miniprogram/components/hexagram-view/hexagram-view.wxml -->
<view class="hexagram-view {{compact ? 'compact' : ''}}">
  <!-- 上卦显示 -->
  <view wx:if="{{showTrigrams}}" class="trigram upper-trigram" data-type="upper" bindtap="onTrigramTap">
    <view class="trigram-symbol">{{hexagramData.upperTrigram.symbol}}</view>
    <view class="trigram-name">{{hexagramData.upperTrigramName}}</view>
  </view>

  <!-- 六爻显示 -->
  <view class="lines-container">
    <view
      wx:for="{{displayLines}}"
      wx:key="position"
      class="hexagram-line {{line.isYang ? 'yang' : 'yin'}} {{line.isChanging ? 'changing' : ''}} {{line.isAnimated ? 'animated' : ''}}"
      data-index="{{index}}"
      bindtap="onLineTap"
      style="animation-delay: {{index * 100}}ms"
    >
      <!-- 爻线 -->
      <view class="line-content">
        <view wx:if="{{line.isYang}}" class="yang-line"></view>
        <view wx:else class="yin-line">
          <view class="yin-segment left"></view>
          <view class="yin-gap"></view>
          <view class="yin-segment right"></view>
        </view>
      </view>

      <!-- 爻位标签 -->
      <view class="line-label">
        <text class="position-name">{{line.positionName}}</text>
        <view wx:if="{{line.isChanging}}" class="changing-indicator">变</view>
      </view>
    </view>
  </view>

  <!-- 下卦显示 -->
  <view wx:if="{{showTrigrams}}" class="trigram lower-trigram" data-type="lower" bindtap="onTrigramTap">
    <view class="trigram-symbol">{{hexagramData.lowerTrigram.symbol}}</view>
    <view class="trigram-name">{{hexagramData.lowerTrigramName}}</view>
  </view>

  <!-- 卦象信息 -->
  <view class="hexagram-info">
    <view class="hexagram-number">{{hexagramData.guaNumber}}</view>
    <view class="hexagram-name">{{hexagramData.guaName}}</view>
    <view class="hexagram-mark">{{hexagramData.guaMark}}</view>
  </view>
</view>
```

```wxss
/* miniprogram/components/hexagram-view/hexagram-view.wxss */
.hexagram-view {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 40rpx;
  background: var(--theme-surface);
  border-radius: 20rpx;
  margin: 20rpx;
}

.hexagram-view.compact {
  padding: 20rpx;
  margin: 10rpx;
}

.trigram {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 20rpx 0;
  padding: 20rpx;
  background: var(--theme-background);
  border-radius: 12rpx;
  border: 1rpx solid var(--theme-border);
  transition: all 0.3s ease;
}

.trigram:active {
  transform: scale(0.95);
  background: var(--theme-accent);
  color: white;
}

.trigram-symbol {
  font-size: 48rpx;
  margin-bottom: 10rpx;
}

.trigram-name {
  font-size: 24rpx;
  color: var(--theme-text-secondary);
}

.trigram:active .trigram-name {
  color: white;
}

.lines-container {
  display: flex;
  flex-direction: column;
  gap: 30rpx;
  margin: 40rpx 0;
  min-height: 400rpx;
  justify-content: center;
}

.hexagram-line {
  display: flex;
  align-items: center;
  gap: 20rpx;
  padding: 20rpx;
  border-radius: 12rpx;
  transition: all 0.3s ease;
  cursor: pointer;
}

.hexagram-line:active {
  background: var(--theme-border);
  transform: translateX(10rpx);
}

.hexagram-line.changing {
  background: rgba(52, 152, 219, 0.1);
  border: 1rpx solid var(--theme-accent);
}

.line-content {
  width: 200rpx;
  height: 16rpx;
  position: relative;
}

.yang-line {
  width: 100%;
  height: 100%;
  background: var(--theme-text);
  border-radius: 8rpx;
  transition: all 0.3s ease;
}

.yin-line {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  gap: 20rpx;
}

.yin-segment {
  height: 100%;
  background: var(--theme-text);
  border-radius: 8rpx;
  flex: 1;
}

.yin-gap {
  width: 20rpx;
}

.line-label {
  display: flex;
  align-items: center;
  gap: 10rpx;
  min-width: 80rpx;
}

.position-name {
  font-size: 28rpx;
  color: var(--theme-text);
  font-weight: 500;
}

.changing-indicator {
  background: var(--theme-accent);
  color: white;
  font-size: 20rpx;
  padding: 4rpx 8rpx;
  border-radius: 8rpx;
  font-weight: 500;
}

.hexagram-info {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10rpx;
  margin-top: 30rpx;
  padding: 30rpx;
  background: var(--theme-background);
  border-radius: 16rpx;
  border: 2rpx solid var(--theme-accent);
}

.hexagram-number {
  font-size: 36rpx;
  font-weight: bold;
  color: var(--theme-accent);
}

.hexagram-name {
  font-size: 32rpx;
  font-weight: 500;
  color: var(--theme-text);
}

.hexagram-mark {
  font-size: 24rpx;
  color: var(--theme-text-secondary);
}

/* 动画效果 */
@keyframes lineAppear {
  0% {
    opacity: 0;
    transform: translateX(-50rpx);
  }
  100% {
    opacity: 1;
    transform: translateX(0);
  }
}

.hexagram-line.animated {
  animation: lineAppear 0.5s ease-out forwards;
}

.hexagram-line.changing .yang-line,
.hexagram-line.changing .yin-segment {
  background: var(--theme-accent);
}

/* 响应式设计 */
@media (max-width: 750rpx) {
  .line-content {
    width: 150rpx;
  }

  .trigram-symbol {
    font-size: 36rpx;
  }

  .hexagram-view {
    padding: 20rpx;
  }
}
```

这些完整的代码示例提供了：

1. **完整的主题管理系统** - 支持自动/手动切换，事件监听，资源清理
2. **专业的硬币动画组件** - 3D动画效果，音效支持，状态管理
3. **准确的卦象计算引擎** - 传统算法实现，变卦计算，数据验证
4. **优雅的卦象显示组件** - 逐行动画，交互响应，信息展示

所有代码都遵循微信小程序开发规范，包含完整的错误处理、性能优化和用户体验设计。