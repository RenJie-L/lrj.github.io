---
title: React 19 更新了啥
date: 2025-05-09 17:29:20
tags: [学习, react]
---
# React 19 一些新特性
react 19 稳定版发布了,总结了一些可能会用到的新特性


### useTransition
React 中根据数据变更更新状态需要手动处理各种情况，比如用户提交表单，这些操作涉及的各种状态（加载、错误、更新等）和请求顺序都需要开发者手动管理，比较麻烦。在React 19中，可以使用useTransition简化状态和表单处理。

异步过渡会立即将 isPending 状态设置为 true，发出异步请求，然后在任何过渡后将 isPending 切换为 false。这使你能够在数据变化时保持当前 UI 的响应性和交互性。

举个栗子：
```jsx
// 使用 Actions 中的待定状态
function UpdateName({}) {
  const [name, setName] = useState("");
  const [error, setError] = useState(null);
  const [isPending, startTransition] = useTransition();

  const handleSubmit = () => {
    startTransition(async () => {
      const error = await updateName(name);
      if (error) {
        setError(error);
        return;～
      } 
      redirect("/path");
    })
  };

  return (
    <div>
      <input value={name} onChange={(event) => setName(event.target.value)} />
      <button onClick={handleSubmit} disabled={isPending}>
        Update
      </button>
      {error && <p>{error}</p>}
    </div>
  );
}
```

### 新的 Hook: useActionState 
[useActionState](https://zh-hans.react.dev/reference/react/useActionState) 是 React 19 新增的一个 Hook，用于处理表单提交和异步操作。它可以将表单数据和异步操作的状态（如加载、错误、成功等）合并到一个状态中，简化表单处理逻辑。

```jsx
const [error, submitAction, isPending] = useActionState(
  async (previousState, newName) => {
    const error = await updateName(newName);
    if (error) {
      // 你可以返回操作的任何结果。
      // 这里，我们只返回错误。
      return error;
    }

    // 处理成功的情况。
    return null;
  },
  null,
);
```
useActionState 接受一个函数（“Action”），并返回一个被包装的用于调用的 Action。这是因为 Actions 是可以组合的。当调用被包装的 Action 时，useActionState 将返回 Action 的最后结果作为 data，以及 Action 的待定状态作为 pending。

### 新 Hook: useOptimistic 

[useOptimistic](https://zh-hans.react.dev/reference/react/useOptimistic) 是 React 19 新增的一个 Hook，用于在异步操作进行时乐观地更新状态。它可以在异步操作进行时立即渲染乐观状态，然后在操作完成后切换回原始状态。

```jsx
function ChangeName({currentName, onUpdateName}) {
  const [optimisticName, setOptimisticName] = useOptimistic(currentName);

  const submitAction = async formData => {
    const newName = formData.get("name");
    setOptimisticName(newName);
    const updatedName = await updateName(newName);
    onUpdateName(updatedName);
  };

  return (
    <form action={submitAction}>
      <p>Your name is: {optimisticName}</p>
      <p>
        <label>Change Name:</label>
        <input
          type="text"
          name="name"
          disabled={currentName !== optimisticName}
        />
      </p>
    </form>
  );
}
```
useOptimistic Hook 会在 updateName 请求进行时立即渲染 optimisticName。当更新完成或出错时，React 将自动切换回 currentName 值。

### ref

#### 可以在函数组件中将 ref 作为 prop 进行访问：

```jsx
function MyInput({placeholder, ref}) {
  return <input placeholder={placeholder} ref={ref} />
}

//...
<MyInput ref={ref} />
```
新的函数组件将不再需要 forwardRef，将发布一个 codemod 来自动更新你的组件以使用新的 ref prop。

#### 支持清理函数

这将使得在 ref 改变时执行清理操作变得更加容易。例如，可以在 ref 改变时取消订阅事件：

```JSX
<input
  ref={(ref) => {
    // ref 创建

    // 新特性: 当元素从 DOM 中被移除时
    // 返回一个清理函数来重置 ref
    return () => {
      // ref cleanup
    };
  }}
/>
```
当组件卸载时，React 将调用从 ref 回调返回的清理函数。这适用于 DOM refs，类组件的 refs，以及 useImperativeHandle。