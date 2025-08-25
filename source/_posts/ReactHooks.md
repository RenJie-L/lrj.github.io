---
title: React Hooks 实践：useEffect vs useLayoutEffect & useRef vs createRef 深度解析
date: 2025-04-25 20:07:47
tags: React
---

## 1. 背景

在新人 onboarding 项目中，需要为智能视角组件注册位置信息以供引导弹窗使用。由于智能视角采用函数组件实现且 DOM 会动态更新，初期错误使用了 `createRef` 导致引导功能失效。

**问题现象**：引导弹窗无法正确定位到智能视角组件位置  
**根本原因**：函数组件重渲染时 `createRef` 会创建新的 ref 对象，导致位置信息丢失  
**解决方案**：改用 `useRef` 确保 ref 对象在组件生命周期内保持一致

借此机会，深入学习了这些 Hook 的区别和最佳实践。

## 2. useEffect 与 useLayoutEffect 的区别

`useEffect` 和 `useLayoutEffect` 是 React 中常用的两个 Hook，但在触发时机和执行顺序上有重要区别。

### 2.1 执行时机对比

**React 渲染流程**：

1. **Render 阶段**：执行函数组件，生成虚拟 DOM
2. **Commit 阶段**：更新真实 DOM
3. **Layout 阶段**：useLayoutEffect 执行（同步）
4. **Paint 阶段**：浏览器绘制页面
5. **Effect 阶段**：useEffect 执行（异步）

**关键区别**：

- `useLayoutEffect`：在浏览器绘制**之前**同步执行，可能阻塞渲染
- `useEffect`：在浏览器绘制**之后**异步执行，不阻塞渲染

### 2.2 代码示例

```typescript
function Example() {
  const [count, setCount] = useState(0);
  
  // 会在浏览器绘制前执行，可能阻塞渲染
  useLayoutEffect(() => {
    console.log('useLayoutEffect:', count);
    // 适合：DOM 测量、样式修改等需要在绘制前完成的操作
  }, [count]);
  
  // 会在浏览器绘制后异步执行，不阻塞渲染
  useEffect(() => {
    console.log('useEffect:', count);
    // 适合：API 调用、事件监听、数据获取等
  }, [count]);
  
  return <div>Count: {count}</div>;
}
```

### 2.3 使用建议

**优先使用 `useEffect`**：

- 不会阻塞页面渲染，提供更好的用户体验
- 适合处理不需要立即更新 UI 的副作用（网络请求、事件订阅等）

**特定场景使用 `useLayoutEffect`**：

- 需要在页面渲染前同步执行操作
- DOM 测量和样式修改，避免视觉闪烁
- 需要立即获取 DOM 元素尺寸或位置信息

## 3. React.createRef 和 React.useRef 的区别

**核心区别**：`useRef` 仅能用在函数组件，`createRef` 仅能用在类组件。

### 3.1 实际问题分析

**问题场景**：智能视角组件需要为引导弹窗提供位置信息

**错误用法**：

```typescript
function SmartView() {
  // ❌ 错误：每次渲染都创建新的 ref
  const viewRef = React.createRef<HTMLDivElement>();
  
  useEffect(() => {
    // 由于 viewRef 每次都是新对象，引导系统无法正确获取位置
    registerGuidePosition('smart-view', viewRef);
  });
  
  return <div ref={viewRef}>智能视角内容</div>;
}
```

**正确用法**：

```typescript
function SmartView() {
  // ✅ 正确：整个组件生命周期中保持同一个 ref 对象
  const viewRef = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    // viewRef 对象保持不变，引导系统可以正确获取位置
    registerGuidePosition('smart-view', viewRef);
  }, []); // 空依赖数组，只注册一次
  
  return <div ref={viewRef}>智能视角内容</div>;
}
```

### 3.2 详细对比

| 特性 | createRef | useRef |
|------|-----------|---------|
| **使用场景** | 类组件 | 函数组件 |
| **重新渲染行为** | 每次创建新对象 | 保持同一对象 |
| **初始值设置** | 不支持参数 | 支持初始值参数 |
| **性能** | 开销较大 | 更高效 |
| **推荐度** | 仅类组件使用 | 现代 React 推荐 |

### 3.3 原理解析

**createRef 在函数组件中的问题**：

- `createRef` 没有 Hooks 机制，每次函数组件执行都会重新创建
- 函数组件重渲染时，之前的 ref 引用会丢失

**useRef 的优势**：

- 基于 Hooks 机制，React 内部维护引用的一致性
- 在组件整个生命周期中返回同一个 ref 对象

## 4. ref 更新的最佳实践

由于 Ref 是贯穿函数组件所有渲染周期的实例，理论上在任何地方都可以修改，但需要遵循 React 的最佳实践。

### 4.1 推荐的更新时机

```typescript
function App() {
  const countRef = useRef(0);
  const [, forceUpdate] = useReducer(x => x + 1, 0);
  
  // ✅ 在事件处理器中更新
  const handleClick = () => {
    countRef.current += 1;
    forceUpdate(); // 手动触发重渲染
  };
  
  // ✅ 在 useEffect 中更新
  useEffect(() => {
    countRef.current = someCalculatedValue;
  }, [dependency]);
  
  // ❌ 避免在渲染期间直接更新
  // countRef.current += 1; // 不要这样做
  
  return <button onClick={handleClick}>Count: {countRef.current}</button>;
}
```

### 4.2 避免在 Render Phase 修改 ref

从 React 生命周期来看，Render Phase 阶段不允许副作用操作，因为这个阶段可能被 React 随时取消或重做。修改 ref 属于副作用操作，应该在 Commit Phase 阶段或回调函数中进行。

## 5. 使用决策指南

### 5.1 useEffect vs useLayoutEffect 选择流程

```text
需要修改DOM样式/测量尺寸？
├─ 是 → useLayoutEffect（避免视觉闪烁）
└─ 否 → useEffect（不阻塞渲染，性能更好）
```

### 5.2 useRef vs createRef 选择流程

```text

组件类型？
├─ 函数组件 → useRef
└─ 类组件 → createRef
```

## 6. 常见陷阱和注意事项

### 6.1 useLayoutEffect 注意事项

- **性能影响**：避免在其中执行耗时操作，会阻塞页面渲染
- **SSR 兼容性**：在服务端渲染中会有警告，可能需要条件使用
- **使用场景**：仅在需要同步 DOM 操作时使用，大多数情况下 useEffect 更合适

### 6.2 useRef 注意事项

- **不触发重渲染**：修改 `ref.current` 不会触发组件重渲染
- **更新时机**：不要在渲染期间修改 ref，应在副作用或事件处理器中修改
- **类型安全**：在 TypeScript 中注意 ref 的类型定义和 null 检查

```typescript
// 类型安全的 ref 使用
const inputRef = useRef<HTMLInputElement>(null);

const focusInput = () => {
  // 需要进行 null 检查
  if (inputRef.current) {
    inputRef.current.focus();
  }
};
```

## 7. 总结

1. **useEffect vs useLayoutEffect**：
   - 优先使用 `useEffect`，仅在需要同步 DOM 操作时使用 `useLayoutEffect`
   - 理解执行时机差异，选择合适的 Hook 避免性能问题

2. **useRef vs createRef**：
   - 函数组件必须使用 `useRef`，类组件使用 `createRef`
   - `useRef` 保证对象一致性，避免重渲染导致的引用丢失

3. **最佳实践**：
   - 遵循 React 生命周期规则，在合适的时机更新 ref
   - 注意类型安全和 null 检查
   - 理解每个 Hook 的特性，选择最适合的工具

通过这次问题排查和学习，不仅解决了引导弹窗的定位问题，更重要的是加深了对 React Hooks 机制的理解，为后续开发提供了坚实的基础。
