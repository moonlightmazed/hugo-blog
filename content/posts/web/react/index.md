---
title: React
date: 2021-06-02T22:13:02+08:00
lastmod: 2021-06-02T22:13:02+08:00
author: MoonlightMaze
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: image.png
images:
  - image.png
categories:
  - Web
tags:
  - React
# nolastmod: true
draft: fale
description: "React 是由 Facebook 开发并维护的一个用于构建用户界面的 JavaScript 库，因其高效、灵活和可复用性强等特点，在 Web 开发领域得到了广泛应用"
---



# 环境



## editorconfig

在项目根路径下创建`.editorconfig`

```toml
# https://editorconfig.org
root = true
[*]
charset = utf-8
indent_style = space
indent_size = 2
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
```



## prettier

```js
// 安装prettier
yarn add prettier -D 

//在根路径下创建.prettierrc.cjs
module.exports = {
  printWidth: 120,     //每行最大列，超过会换行
  tabWidth: 2,         //缩进
  semi: false,         //
}
```



# React基础

## React jsx语法讲解

### 变量声明

```jsx
import './App.css'

function App() {
  const name = <div>wg</div>
  const info = <h1>学习React18</h1>

    return (
   <>
     {name}
     {info}
   </>
  )
}

export default App

```

### 条件判断

```jsx
import './App.css'

function App() {
  const admin = <span>管理员</span>
  const member = <span>会员</span>
  const isAdmin = false
    return (
   <>
     {isAdmin ? admin: member}      {/*输出： 会员 */}
   </>
  )
}

export default App

```



### 样式

```jsx
import './App.css'

function App() {
  const admin = <span style={{color:'red',fontSize: 16}}>管理员</span>  {/* style */}
  const member = <span>会员</span>
  const isAdmin = true
  return (
    <>
      {isAdmin ? admin: member}      
    </>
  )
}

export default App

```



## 循环

### 使用 map 方法

```jsx
import React from 'react';

const numbers = [1, 2, 3, 4, 5];

const App = () => {
  return (
    <ul>
      {numbers.map((number) => (
        <li key={number}>{number}</li>
      ))}
    </ul>
  );
};

export default App;
```



### 使用 for 循环

```jsx
import React from 'react';

const numbers = [1, 2, 3, 4, 5];

const App = () => {
  const listItems = [];
  for (let i = 0; i < numbers.length; i++) {
    listItems.push(<li key={numbers[i]}>{numbers[i]}</li>);
  }

  return <ul>{listItems}</ul>;
};

export default App;
```

### 循环对象属性

```jsx
import React from 'react';

const person = {
  name: 'John',
  age: 30,
  occupation: 'Developer'
};

const App = () => {
  return (
    <ul>
      {/* 在这个例子中，我们使用 Object.entries 方法将对象的键值对转换为数组，然后使用 map 方法遍历该数组并渲染每个键值对。*/}
      {Object.entries(person).map(([key, value]) => (
        <li key={key}>{`${key}: ${value}`}</li>
      ))}
    </ul>
  );
};

export default App;
```



## 表单事件

### 文本输入框(input)事件

对于文本输入框，通常会用到 `onChange` 事件来实时捕获输入内容的变化，用 `onSubmit` 事件来处理表单提交。

```jsx
import React, { useState } from 'react';

const TextInputForm = () => {
    const [inputValue, setInputValue] = useState('');

    const handleChange = (e) => {
        setInputValue(e.target.value);
    };

    const handleSubmit = (e) => {
        e.preventDefault();
        console.log('提交的内容:', inputValue);
    };

    return (
        <form onSubmit={handleSubmit}>
            <input
                type="text"
                value={inputValue}
                onChange={handleChange}
                placeholder="请输入内容"
            />
            <button type="submit">提交</button>
        </form>
    );
};

export default TextInputForm;
```

在上述代码中，`useState` 用于创建一个状态变量 `inputValue` 来存储输入框的值。`handleChange` 函数会在输入内容变化时更新该状态。`handleSubmit` 函数则在表单提交时被调用，使用 `e.preventDefault()` 来阻止表单的默认提交行为。



### 复选框(input[type="checkbox"])事件

复选框通常使用 `onChange` 事件来处理选中状态的变化。

```jsx
import React, { useState } from 'react';

const CheckboxForm = () => {
    const [isChecked, setIsChecked] = useState(false);

    const handleCheckboxChange = (e) => {
        setIsChecked(e.target.checked);
    };

    return (
        <form>
            <input
                type="checkbox"
                checked={isChecked}
                onChange={handleCheckboxChange}
            />
            <label>是否选中</label>
        </form>
    );
};

export default CheckboxForm;
```

这里，`isChecked` 状态变量用来存储复选框的选中状态，`handleCheckboxChange` 函数会在复选框状态改变时更新该状态。



### 下拉框(select)事件

下拉框同样使用 `onChange` 事件来处理选项的选择变化。

```jsx
import React, { useState } from 'react';

const SelectForm = () => {
    const [selectedOption, setSelectedOption] = useState('option1');

    const handleSelectChange = (e) => {
        setSelectedOption(e.target.value);
    };

    return (
        <form>
            <select value={selectedOption} onChange={handleSelectChange}>
                <option value="option1">选项 1</option>
                <option value="option2">选项 2</option>
                <option value="option3">选项 3</option>
            </select>
        </form>
    );
};

export default SelectForm;
```

此例中，`selectedOption` 状态变量存储当前选中的选项值，`handleSelectChange` 函数会在选项改变时更新该状态。



### 多行文本框(textarea)事件

多行文本框和输入框类似，使用 `onChange` 事件处理内容变化。

```jsx
import React, { useState } from 'react';

const TextareaForm = () => {
    const [textareaValue, setTextareaValue] = useState('');

    const handleTextareaChange = (e) => {
        setTextareaValue(e.target.value);
    };

    return (
        <form>
            <textarea
                value={textareaValue}
                onChange={handleTextareaChange}
                placeholder="请输入多行文本"
            />
        </form>
    );
};

export default TextareaForm;
```

这里，`textareaValue` 状态变量存储多行文本框的内容，`handleTextareaChange` 函数会在内容改变时更新该状态。



## 属性传递

### 基本属性传递

基本属性传递是最常见的方式，你可以将数据作为属性传递给子组件，子组件通过 `props` 对象来接收这些数据。

```jsx
import React from 'react';

// 子组件
const ChildComponent = (props) => {
    return (
        <div>
            <p>接收到的名称: {props.name}</p>
            <p>接收到的年龄: {props.age}</p>
        </div>
    );
};

// 父组件
const ParentComponent = () => {
    const name = 'John';
    const age = 30;

    return (
        <div>
            <h1>父组件</h1>
            <ChildComponent name={name} age={age} />
        </div>
    );
};

export default ParentComponent;
```

在上述代码中，`ParentComponent` 作为父组件，将 `name` 和 `age` 作为属性传递给 `ChildComponent`。`ChildComponent` 通过 `props` 对象接收这些属性并进行渲染。



### 展开运算符传递属性

如果你有一个包含多个属性的对象，你可以使用展开运算符 `...` 来一次性传递所有属性

```jsx
import React from 'react';

// 子组件
const ChildComponent = (props) => {
    return (
        <div>
            <p>接收到的名称: {props.name}</p>
            <p>接收到的年龄: {props.age}</p>
        </div>
    );
};

// 父组件
const ParentComponent = () => {
    const person = {
        name: 'John',
        age: 30
    };

    return (
        <div>
            <h1>父组件</h1>
            <ChildComponent {...person} />
        </div>
    );
};

export default ParentComponent;
```

在这个例子中，`person` 对象包含 `name` 和 `age` 属性，使用展开运算符 `...` 将其所有属性传递给 `ChildComponent`。



### 函数作为属性传递

你可以将函数作为属性传递给子组件，子组件可以调用这个函数来与父组件进行通信。 

```jsx
import React, { useState } from 'react';

// 子组件
const ChildComponent = (props) => {
    const handleClick = () => {
        props.onClick('Hello from child!');
    };

    return (
        <button onClick={handleClick}>点击调用父组件函数</button>
    );
};

// 父组件
const ParentComponent = () => {
    const [message, setMessage] = useState('');

    const handleChildClick = (msg) => {
        setMessage(msg);
    };

    return (
        <div>
            <h1>父组件</h1>
            <p>接收到的消息: {message}</p>
            <ChildComponent onClick={handleChildClick} />
        </div>
    );
};

export default ParentComponent;
```

在这个示例中，`ParentComponent` 将 `handleChildClick` 函数作为 `onClick` 属性传递给 `ChildComponent`。`ChildComponent` 中的按钮点击事件调用 `props.onClick` 函数，并传递一个消息给父组件。



### 上下文(Context)传递属性

当你需要在多个层级的组件之间共享数据时，使用上下文（Context）是一个不错的选择。

```jsx
import React, { createContext, useContext, useState } from 'react';

// 创建上下文
const UserContext = createContext();

// 子组件
const ChildComponent = () => {
    const user = useContext(UserContext);

    return (
        <div>
            <p>接收到的用户名称: {user.name}</p>
            <p>接收到的用户年龄: {user.age}</p>
        </div>
    );
};

// 父组件
const ParentComponent = () => {
    const [user, setUser] = useState({
        name: 'John',
        age: 30
    });

    return (
        <UserContext.Provider value={user}>
            <div>
                <h1>父组件</h1>
                <ChildComponent />
            </div>
        </UserContext.Provider>
    );
};

export default ParentComponent;
```

在这个例子中，使用 `createContext` 创建了一个 `UserContext`，`ParentComponent` 使用 `UserContext.Provider` 提供数据，`ChildComponent` 使用 `useContext` 钩子获取上下文数据。

## useState

### 基本概念

在 React 中，状态是一种能在组件渲染过程中保存和更新数据的机制。在类组件里，状态通常通过 `this.state` 和 `this.setState` 来管理；而在函数组件中，就可以使用 `useState` 这个 Hook 来实现相同的功能。

### 语法

`useState` 函数接收一个初始状态值作为参数，并返回一个包含两个元素的数组：

```jsx
const [state, setState] = useState(initialState);
```

- `state`：当前状态的值。
- `setState`：用于更新状态的函数。
- `initialState`：状态的初始值，可以是任意数据类型，如数字、字符串、对象、数组等。



### 基本使用示例

以下是一个简单的计数器示例，展示了 `useState` 的基本用法：

```jsx
import React, { useState } from 'react';

const Counter = () => {
    // 初始化状态，count 的初始值为 0
    const [count, setCount] = useState(0);

    const increment = () => {
        // 更新状态
        setCount(count + 1);
    };

    const decrement = () => {
        setCount(count - 1);
    };

    return (
        <div>
            <p>计数: {count}</p>
            <button onClick={increment}>增加</button>
            <button onClick={decrement}>减少</button>
        </div>
    );
};

export default Counter;
```

在这个示例中，`useState(0)` 将 `count` 状态的初始值设为 0。`setCount` 函数用于更新 `count` 的值。每次点击 “增加” 或 “减少” 按钮时，相应的函数会被调用，从而更新 `count` 的值。



### 初始状态的延迟计算

如果初始状态是通过复杂计算得到的，你可以传入一个函数作为 `useState` 的参数。这个函数只会在组件的初始渲染时执行。

```jsx
import React, { useState } from 'react';

const getInitialValue = () => {
    // 模拟复杂计算
    return Math.random() * 100;
};

const ComplexCounter = () => {
    const [value, setValue] = useState(getInitialValue);

    return (
        <div>
            <p>初始值: {value}</p>
        </div>
    );
};

export default ComplexCounter;
```

在这个例子中，`getInitialValue` 函数只会在组件首次渲染时执行，用于计算初始状态的值。



### 函数式更新

当新的状态值依赖于之前的状态时，你可以向 `setState` 函数传入一个回调函数。这个回调函数接收前一个状态值作为参数，并返回新的状态值。

```jsx
import React, { useState } from 'react';

const DoubleCounter = () => {
    const [count, setCount] = useState(0);

    const doubleIncrement = () => {
        // 函数式更新
        setCount(prevCount => prevCount + 2);
    };

    return (
        <div>
            <p>计数: {count}</p>
            <button onClick={doubleIncrement}>增加 2</button>
        </div>
    );
};

export default DoubleCounter;
```

在这个例子中，`setCount` 接收一个回调函数，该函数使用前一个状态值 `prevCount` 来计算新的状态值。这种方式在处理多个状态更新时非常有用，因为它可以确保每次更新都基于最新的状态。



### 多个状态变量

你可以在一个组件中使用多个 `useState` 调用，以管理不同的状态

```jsx
import React, { useState } from 'react';

const MultipleStates = () => {
    const [name, setName] = useState('');
    const [age, setAge] = useState(0);

    const handleNameChange = (e) => {
        setName(e.target.value);
    };

    const handleAgeChange = (e) => {
        setAge(Number(e.target.value));
    };

    return (
        <div>
            <input
                type="text"
                value={name}
                onChange={handleNameChange}
                placeholder="输入姓名"
            />
            <input
                type="number"
                value={age}
                onChange={handleAgeChange}
                placeholder="输入年龄"
            />
            <p>姓名: {name}, 年龄: {age}</p>
        </div>
    );
};

export default MultipleStates;
```

在这个例子中，`name` 和 `age` 是两个独立的状态变量，分别使用不同的 `useState` 调用进行管理。

### 注意事项

- **状态更新是异步的**：`setState` 函数是异步的，多次调用 `setState` 可能会被合并。如果需要在状态更新后执行某些操作，可以使用 `useEffect` Hook。
- **状态的不可变性**：在更新状态时，应该避免直接修改状态对象，而是创建一个新的对象。例如，如果状态是一个数组或对象，应该使用展开运算符或其他方法创建一个新的副本进行更新。

## useEffect

`useEffect` 是 React 18 里一个十分关键的 Hook，它能让你在函数组件里执行副作用操作。副作用操作涵盖数据获取、订阅、手动修改 DOM 等。下面会详细解析 `useEffect` 的语法与使用场景。

### 基本语法

`useEffect` 函数接收两个参数：

```jsx
useEffect(effect, dependencies);
```

- `effect`：这是一个函数，其中包含了需要执行的副作用操作。此函数还可以返回一个清理函数，用于在组件卸载或者依赖项变更时执行清理工作。
- `dependencies`：这是一个可选的数组，包含了影响副作用执行的依赖项。当这些依赖项中的任意一个发生变化时，副作用函数就会重新执行。要是省略这个参数，副作用函数会在每次组件渲染之后都执行；若传入一个空数组，副作用函数仅会在组件挂载和卸载时执行。



### 常见使用场景

##### 1. 组件挂载和卸载时执行副作用

当传入一个空数组作为依赖项时，`useEffect` 里的副作用函数只会在组件挂载时执行一次，而返回的清理函数会在组件卸载时执行。

```jsx
import React, { useEffect } from 'react';

const MountAndUnmountEffect = () => {
    useEffect(() => {
        // 组件挂载时执行的操作
        console.log('组件已挂载');

        // 返回清理函数
        return () => {
            console.log('组件将卸载');
        };
    }, []);

    return <div>组件示例</div>;
};

export default MountAndUnmountEffect;
```

在这个例子中，`useEffect` 的依赖项为空数组，所以副作用函数仅在组件挂载时执行，清理函数在组件卸载时执行。



##### 2. 依赖项变化时执行副作用

当传入一个包含依赖项的数组时，副作用函数会在组件挂载时执行，并且在依赖项中的任意一个发生变化时重新执行。

```jsx
import React, { useState, useEffect } from 'react';

const DependencyChangeEffect = () => {
    const [count, setCount] = useState(0);

    useEffect(() => {
        // 依赖项变化时执行的操作
        console.log(`计数已更新为: ${count}`);
    }, [count]);

    const increment = () => {
        setCount(count + 1);
    };

    return (
        <div>
            <p>计数: {count}</p>
            <button onClick={increment}>增加</button>
        </div>
    );
};

export default DependencyChangeEffect;
```

在这个例子中，`useEffect` 的依赖项是 `[count]`，所以每当 `count` 的值发生变化时，副作用函数就会重新执行。



##### 3. 每次组件渲染时执行副作用

若省略依赖项参数，副作用函数会在每次组件渲染之后都执行。

```jsx
import React, { useState, useEffect } from 'react';

const EveryRenderEffect = () => {
    const [value, setValue] = useState('');

    useEffect(() => {
        // 每次组件渲染时执行的操作
        console.log('组件已渲染');
    });

    const handleChange = (e) => {
        setValue(e.target.value);
    };

    return (
        <div>
            <input
                type="text"
                value={value}
                onChange={handleChange}
                placeholder="输入内容"
            />
        </div>
    );
};

export default EveryRenderEffect;
```

在这个例子中，由于没有传入依赖项，副作用函数会在每次组件渲染之后执行。



### 清理函数

副作用函数可以返回一个清理函数，用于在组件卸载或者依赖项变更时执行清理工作。清理函数常用于取消订阅、清除定时器等操作。

```jsx
import React, { useEffect } from 'react';

const CleanupEffect = () => {
    useEffect(() => {
        const timer = setInterval(() => {
            console.log('定时器正在运行');
        }, 1000);

        // 返回清理函数
        return () => {
            clearInterval(timer);
            console.log('定时器已清除');
        };
    }, []);

    return <div>定时器组件</div>;
};

export default CleanupEffect;
```

在这个例子中，副作用函数创建了一个定时器，清理函数在组件卸载时清除了这个定时器，避免了内存泄漏。



### 注意事项

- **依赖项数组的准确性**：要确保依赖项数组中包含了所有会影响副作用执行的变量。若遗漏了某些依赖项，副作用可能不会在需要的时候重新执行；若包含了不必要的依赖项，副作用可能会过度执行。
- **清理函数的必要性**：如果副作用函数中涉及到需要清理的资源，如定时器、订阅等，一定要返回一个清理函数，以避免内存泄漏。





## useMemo和useCallback缓存方案

### useMemo

`useMemo` 是一个重要的 Hook，它主要用于性能优化，通过缓存计算结果，避免在每次渲染时都进行高开销的计算。

#### 基本语法

`useMemo` 函数接受两个参数，其基本语法如下：

```jsx
const memoizedValue = useMemo(() => computeValue(), dependencies);
```

- **第一个参数**：是一个函数，该函数返回一个需要被缓存的值，也就是你想要进行计算并缓存的结果。在上述代码中，`computeValue()` 就是进行具体计算的函数。
- **第二个参数**：是一个可选的依赖项数组 `dependencies`。当数组中的任何一个元素发生变化时，`useMemo` 会重新调用第一个参数中的函数来计算新的结果；如果依赖项数组为空，那么 `useMemo` 只会在组件挂载时计算一次结果，后续不会再重新计算；若省略这个参数，`useMemo` 会在每次组件渲染时都重新计算结果。
- **返回值**：`useMemo` 返回的是第一个参数函数计算得到的值的缓存版本，即 `memoizedValue`。



#### 使用场景及示例

##### 1. 缓存昂贵的计算结果

当你有一个计算量较大的函数时，使用 `useMemo` 可以避免在每次渲染时都进行该计算，只有在依赖项改变时才重新计算。

```jsx
import React, { useMemo, useState } from 'react';

const ExpensiveCalculation = () => {
    const [count, setCount] = useState(0);

    // 模拟一个昂贵的计算
    const expensiveValue = useMemo(() => {
        console.log('进行昂贵的计算');
        let sum = 0;
        for (let i = 0; i < 1000000; i++) {
            sum += i;
        }
        return sum;
    }, []);

    const increment = () => {
        setCount(count + 1);
    };

    return (
        <div>
            <p>计数: {count}</p>
            <button onClick={increment}>增加</button>
            <p>昂贵计算的结果: {expensiveValue}</p>
        </div>
    );
};

export default ExpensiveCalculation;
```

在这个例子中，由于依赖项数组为空，`useMemo` 中的计算函数只会在组件挂载时执行一次，后续点击按钮增加 `count` 时，不会重新进行昂贵的计算。



##### 2. 依赖项变化时重新计算

如果依赖项数组中有值发生变化，`useMemo` 会重新计算结果。

```jsx
import React, { useMemo, useState } from 'react';

const DependencyChangeCalculation = () => {
    const [a, setA] = useState(1);
    const [b, setB] = useState(2);

    const result = useMemo(() => {
        console.log('重新计算结果');
        return a + b;
    }, [a, b]);

    const incrementA = () => {
        setA(a + 1);
    };

    const incrementB = () => {
        setB(b + 1);
    };

    return (
        <div>
            <p>a 的值: {a}</p>
            <button onClick={incrementA}>增加 a</button>
            <p>b 的值: {b}</p>
            <button onClick={incrementB}>增加 b</button>
            <p>计算结果: {result}</p>
        </div>
    );
};

export default DependencyChangeCalculation;
```

在这个示例中，当 `a` 或 `b` 的值发生变化时，`useMemo` 会重新调用计算函数来更新 `result` 的值。



#### 注意事项

- **`useMemo` 仅用于优化**：不要过度使用 `useMemo`，因为它本身也有一定的开销。只有在确实需要避免重复计算时才使用。
- **引用相等性**：`useMemo` 缓存的值是基于引用相等性的。如果计算结果是一个对象或数组，即使对象或数组的内容相同，但引用不同，`useMemo` 也会认为是不同的值。
- **避免在 `useMemo` 中执行副作用**：`useMemo` 主要用于计算和缓存值，不应该在其中执行副作用操作（如数据获取、订阅等），副作用操作应该使用 `useEffect`。



### useCallback

它主要用于性能优化，特别是在处理函数引用时非常有用

#### 基本语法

`useCallback` 函数接收两个参数，其基本语法如下：

```jsx
const memoizedCallback = useCallback(
  callback,
  dependencies
);
```

- **第一个参数 `callback`**：这是一个需要被记忆（缓存）的函数。当组件重新渲染时，如果依赖项没有发生变化，`useCallback` 会返回之前缓存的函数引用；若依赖项有变化，则会返回一个新的函数引用。
- **第二个参数 `dependencies`**：这是一个可选的依赖项数组。它和 `useEffect` 中的依赖项数组类似，数组中的元素是一些变量，当这些变量中的任何一个发生变化时，`useCallback` 会重新创建一个新的函数。如果依赖项数组为空，那么 `useCallback` 只会在组件挂载时创建一次函数，后续不会重新创建；若省略这个参数，`useCallback` 会在每次组件渲染时都重新创建函数。
- **返回值 `memoizedCallback`**：`useCallback` 返回的是经过记忆化处理后的函数，也就是缓存的函数引用。

#### 使用场景及示例

##### 1. 避免子组件不必要的重新渲染

当你将一个函数作为属性传递给子组件时，如果这个函数在父组件每次渲染时都重新创建，可能会导致子组件不必要的重新渲染。使用 `useCallback` 可以缓存这个函数，只有在依赖项变化时才重新创建函数，从而避免子组件的不必要渲染。

```jsx
import React, { useCallback, useState } from 'react';

// 子组件
const ChildComponent = ({ onClick }) => {
    return (
        <button onClick={onClick}>点击我</button>
    );
};

// 父组件
const ParentComponent = () => {
    const [count, setCount] = useState(0);

    // 使用 useCallback 缓存函数
    const handleClick = useCallback(() => {
        setCount(count + 1);
    }, [count]);

    return (
        <div>
            <p>计数: {count}</p>
            <ChildComponent onClick={handleClick} />
        </div>
    );
};

export default ParentComponent;
```

在这个例子中，`handleClick` 函数使用 `useCallback` 进行了缓存。只有当 `count` 发生变化时，`handleClick` 函数才会重新创建。这样，即使父组件因为其他原因重新渲染，只要 `count` 不变，传递给 `ChildComponent` 的 `onClick` 函数引用就不会改变，从而避免了子组件的不必要重新渲染。

##### 2. 依赖项变化时重新创建函数

如果依赖项数组中有值发生变化，`useCallback` 会重新创建函数。

```jsx
import React, { useCallback, useState } from 'react';

const DependencyChangeCallback = () => {
    const [a, setA] = useState(1);
    const [b, setB] = useState(2);

    const calculateSum = useCallback(() => {
        return a + b;
    }, [a, b]);

    const incrementA = () => {
        setA(a + 1);
    };

    const incrementB = () => {
        setB(b + 1);
    };

    return (
        <div>
            <p>a 的值: {a}</p>
            <button onClick={incrementA}>增加 a</button>
            <p>b 的值: {b}</p>
            <button onClick={incrementB}>增加 b</button>
            <p>计算结果: {calculateSum()}</p>
        </div>
    );
};

export default DependencyChangeCallback;
```

在这个示例中，当 `a` 或 `b` 的值发生变化时，`calculateSum` 函数会重新创建。因为 `calculateSum` 函数依赖于 `a` 和 `b` 的值，所以当它们变化时需要重新创建函数以保证计算结果的正确性。

#### 注意事项

- **仅在必要时使用**：`useCallback` 本身也有一定的开销，因此不要过度使用。只有在确实需要避免函数的重复创建，以防止子组件不必要的重新渲染时才使用。
- **依赖项的准确性**：要确保依赖项数组中包含了所有会影响函数行为的变量。如果遗漏了某些依赖项，可能会导致函数使用到旧的变量值；如果包含了不必要的依赖项，会导致函数不必要地重新创建。
- **与 `useMemo` 的区别**：`useCallback` 缓存的是函数引用，而 `useMemo` 缓存的是函数的返回值。如果需要缓存一个函数，使用 `useCallback`；如果需要缓存一个计算结果，使用 `useMemo`。



## useContext和useReducer实现状态管理 

### useContext

`useContext` 是一个非常实用的钩子，其主要作用是让组件能够访问 React 上下文（Context）中的数据，而不用一级一级地手动传递 props

#### 上下文（Context）的概念

在 React 里，数据通常是自上而下（从父组件到子组件）传递的，不过对于某些共享数据（像用户登录状态、主题颜色、语言偏好等），若要把这些数据传递给多层嵌套的子组件，逐个传递 props 会显得很繁琐。上下文（Context）能够让你在组件树里共享这些数据，不用在每个层级手动传递 props。

#### `useContext` 的用法

`useContext` 钩子的用途是在函数式组件中读取上下文的值。它接收一个上下文对象（由 `React.createContext` 创建）作为参数，并且返回该上下文中当前的值。

```jsx

import React, { createContext, useContext } from 'react';

// 创建一个上下文对象
const ThemeContext = createContext();

// 定义一个提供主题的组件
const ThemeProvider = ({ children }) => {
    const theme = {
        color: 'blue',
        background: 'lightgray'
    };

    return (
        <ThemeContext.Provider value={theme}>
            {children}
        </ThemeContext.Provider> 
    );
};

// 定义一个使用主题的组件
const ThemedComponent = () => {
    // 使用 useContext 钩子获取上下文的值
    const theme = useContext(ThemeContext);

    return (
        <div style={{ color: theme.color, background: theme.background }}>
            This is a themed component.
        </div>
    );
};

// 定义根组件
const App = () => {
    return (
        <ThemeProvider>
            <ThemedComponent />
        </ThemeProvider>
    );
};

export default App;
```

1. **创建上下文对象**：借助 `React.createContext` 函数创建一个上下文对象 `ThemeContext`。这个函数会返回一个包含 `Provider` 和 `Consumer` 的对象。
2. **提供上下文值**：`ThemeProvider` 组件使用 `ThemeContext.Provider` 来提供上下文的值。`value` 属性定义了要共享的数据，在这个例子中是 `theme` 对象。
3. **使用上下文值**：`ThemedComponent` 组件运用 `useContext` 钩子来获取上下文的值。传入 `ThemeContext` 作为参数，就能得到 `ThemeProvider` 提供的 `theme` 对象。
4. **使用上下文值渲染组件**：在 `ThemedComponent` 里，使用从上下文中获取的 `theme` 对象来设置组件的样式。

#### 注意事项

- **上下文更新**：当 `Provider` 的 `value` 属性发生变化时，所有使用该上下文的组件都会重新渲染。
- **默认值**：`createContext` 函数可以接受一个默认值作为参数，当组件没有匹配到 `Provider` 时，就会使用这个默认值。
- **性能考量**：尽管上下文很方便，但过度使用可能会让组件的依赖关系变得复杂，影响性能。所以，仅在确实需要跨层级共享数据时使用上下文。



### useReducer

`useReducer` 是一个非常有用的 Hook，它提供了一种管理组件状态的方式，类似于 Redux 中的 reducer 概念，适用于处理复杂的状态逻辑。

#### 基本概念

`useReducer` 是 React 提供的一个 Hook，它接收一个 reducer 函数和一个初始状态，返回一个当前状态和一个用于触发状态更新的 dispatch 函数。reducer 是一个纯函数，它接收当前状态和一个 action 对象，根据 action 的类型返回一个新的状态。

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
```

- `reducer`：一个纯函数，接收当前状态 `state` 和一个 `action` 对象，返回一个新的状态。
- `initialState`：初始状态值。
- `state`：当前的状态值。
- `dispatch`：一个函数，用于触发状态更新，它接收一个 `action` 对象作为参数。

```jsx
import React, { useReducer } from 'react';

// 定义 reducer 函数
const counterReducer = (state, action) => {
    switch (action.type) {
        case 'increment':
            return { count: state.count + 1 };
        case 'decrement':
            return { count: state.count - 1 };
        default:
            return state;
    }
};

// 定义组件
const Counter = () => {
    // 使用 useReducer 钩子
    const [state, dispatch] = useReducer(counterReducer, { count: 0 });

    return (
        <div>
            <p>Count: {state.count}</p>
            <button onClick={() => dispatch({ type: 'increment' })}>Increment</button>
            <button onClick={() => dispatch({ type: 'decrement' })}>Decrement</button>
        </div>
    );
};

export default Counter;
```

1. **定义 reducer 函数**：`counterReducer` 是一个 reducer 函数，它接收当前状态 `state` 和一个 `action` 对象。根据 `action` 的类型（`increment` 或 `decrement`），返回一个新的状态。
2. **使用 `useReducer` 钩子**：在 `Counter` 组件中，使用 `useReducer` 钩子来管理状态。传入 `counterReducer` 作为 reducer 函数，`{ count: 0 }` 作为初始状态。
3. **获取状态和 dispatch 函数**：`useReducer` 返回一个数组，包含当前状态 `state` 和一个 `dispatch` 函数。
4. **触发状态更新**：通过调用 `dispatch` 函数并传入一个 `action` 对象来触发状态更新。在按钮的 `onClick` 事件中，分别调用 `dispatch({ type: 'increment' })` 和 `dispatch({ type: 'decrement' })` 来增加或减少计数器的值。

#### 高级用法

##### 初始化状态

可以使用第二个参数 `initialState` 来设置初始状态，也可以使用第三个参数来实现惰性初始化。

```jsx
const initialCount = 0;
const init = (initialCount) => {
    return { count: initialCount };
};

const [state, dispatch] = useReducer(reducer, initialCount, init);
```

##### 处理副作用

可以在 `useEffect` 中使用 `dispatch` 来处理副作用，例如在组件挂载时获取数据。

```jsx
import React, { useReducer, useEffect } from 'react';

const dataReducer = (state, action) => {
    switch (action.type) {
        case 'FETCH_SUCCESS':
            return { data: action.payload, loading: false };
        case 'FETCH_ERROR':
            return { error: action.error, loading: false };
        default:
            return state;
    }
};

const DataComponent = () => {
    const [state, dispatch] = useReducer(dataReducer, { data: null, loading: true, error: null });

    useEffect(() => {
        const fetchData = async () => {
            try {
                const response = await fetch('https://api.example.com/data');
                const data = await response.json();
                dispatch({ type: 'FETCH_SUCCESS', payload: data });
            } catch (error) {
                dispatch({ type: 'FETCH_ERROR', error: error.message });
            }
        };

        fetchData();
    }, []);

    if (state.loading) {
        return <p>Loading...</p>;
    }

    if (state.error) {
        return <p>Error: {state.error}</p>;
    }

    return <p>{JSON.stringify(state.data)}</p>;
};

export default DataComponent;
```

#### 适用场景

- **复杂状态逻辑**：当组件的状态逻辑比较复杂，包含多个子值或需要根据不同的 action 进行复杂的状态更新时，使用 `useReducer` 可以使状态管理更加清晰和可维护。
- **多个相关状态的更新**：当多个状态的更新相互关联时，使用 `useReducer` 可以将这些更新逻辑集中在一个 reducer 函数中，避免使用多个 `useState` 钩子带来的复杂性。

#### 注意事项

- **reducer 必须是纯函数**：reducer 函数不能有副作用，它应该根据当前状态和 action 同步返回一个新的状态。
- **状态不可变**：在 reducer 中，应该返回一个新的状态对象，而不是直接修改原状态对象。



## useRef

`useRef` 返回一个可变的 `ref` 对象，其 `.current` 属性被初始化为传入的参数（`initialValue`）。返回的 `ref` 对象在组件的整个生命周期内保持不变。

```jsx
const refContainer = useRef(initialValue);
```

- `initialValue`：`ref` 对象 `.current` 属性的初始值，可以是任意类型，比如 `null`、数字、字符串等。
- `refContainer`：返回的 `ref` 对象，它有一个 `.current` 属性，可用于存储和访问值。



### 使用场景

#### 1. 访问 DOM 元素

`useRef` 常被用于访问 DOM 元素，进而实现对元素的操作，像聚焦、滚动等。

```jsx
import React, { useRef } from 'react';

const TextInputWithFocusButton = () => {
    const inputEl = useRef(null);
    const onButtonClick = () => {
        // `current` 指向已挂载到 DOM 上的文本输入元素
        inputEl.current.focus();
    };
    return (
        <>
            <input ref={inputEl} type="text" />
            <button onClick={onButtonClick}>Focus the input</button>
        </>
    );
};

export default TextInputWithFocusButton;
```

在这个例子中，`useRef(null)` 创建了一个 `ref` 对象 `inputEl`，并将其赋值给 `input` 元素的 `ref` 属性。这样，`inputEl.current` 就指向了这个 `input` DOM 元素，点击按钮时就能调用 `focus` 方法让输入框获取焦点。

#### 2. 保存可变值

`useRef` 还能用来保存一些需要在组件渲染周期之间保持不变的值，比如定时器 ID、上一次的状态值等。

```jsx
import React, { useRef, useEffect, useState } from 'react';

const Counter = () => {
    const [count, setCount] = useState(0);
    const prevCountRef = useRef();

    useEffect(() => {
        prevCountRef.current = count;
    }, [count]);

    const prevCount = prevCountRef.current;

    return (
        <div>
            <p>Now: {count}, before: {prevCount}</p>
            <button onClick={() => setCount(count + 1)}>Increment</button>
        </div>
    );
};

export default Counter;
```

在这个例子中，`prevCountRef` 用于保存上一次的 `count` 值。每次 `count` 发生变化时，`useEffect` 会更新 `prevCountRef.current` 的值。

### 注意事项

- **`ref` 对象的 `.current` 属性是可变的**：你可以随时修改 `ref.current` 的值，并且修改不会触发组件重新渲染。
- **`ref` 对象在组件的整个生命周期内保持不变**：每次渲染时返回的 `ref` 对象都是同一个，这意味着你可以安全地在不同的渲染周期中访问和修改它。
- **不要在渲染期间修改 `ref` 对象**：虽然修改 `ref` 对象不会触发重新渲染，但在渲染期间修改它可能会导致意外的行为。通常，你应该在事件处理函数或副作用函数（如 `useEffect`）中修改 `ref` 对象。



## useTransition

它能帮助开发者处理紧急和非紧急的状态更新，以此优化用户体验和性能

### 基本概念

在 React 里，状态更新一般是紧急的，也就是一旦触发更新，React 会立刻重新渲染组件。但某些状态更新（例如搜索结果的过滤、切换视图等）并非紧急，即时更新可能会让页面卡顿，影响用户体验。`useTransition` 允许你把这些非紧急的更新标记为过渡（transition），从而让 React 可以在不阻塞 UI 的情况下处理它们。

```jsx
const [isPending, startTransition] = useTransition();
```



  - `isPending`：这是一个布尔值，用于表明过渡是否正在进行。在过渡期间，`isPending` 为 `true`，过渡结束后则变为 `false`。你可以利用这个值来显示加载状态，如加载指示器。
  - `startTransition`：这是一个函数，用于将非紧急的状态更新包装成过渡。它接收一个回调函数作为参数，在回调函数里进行的状态更新会被标记为非紧急。

### 使用示例

#### 示例 1：基本使用 

```jsx
import React, { useState, useTransition } from 'react';

const App = () => {
    const [inputValue, setInputValue] = useState('');
    const [list, setList] = useState([]);
    const [isPending, startTransition] = useTransition();

    const handleInputChange = (e) => {
        const value = e.target.value;
        setInputValue(value);

        startTransition(() => {
            // 模拟一个耗时的过滤操作
            const newList = Array.from({ length: 10000 }).map((_, index) => `Item ${index}`).filter(item => item.includes(value));
            setList(newList);
        });
    };

    return (
        <div>
            <input
                type="text"
                value={inputValue}
                onChange={handleInputChange}
                placeholder="Search..."
            />
            {isPending && <p>Loading...</p>}
            <ul>
                {list.map(item => (
                    <li key={item}>{item}</li>
                ))}
            </ul>
        </div>
    );
};

export default App;
```

在这个例子中：

- 当输入框内容改变时，`handleInputChange` 函数会立即更新 `inputValue`（紧急更新）。
- 然后使用 `startTransition` 将过滤列表的操作和更新 `list` 状态的操作包装起来，这些操作会被标记为非紧急。
- `isPending` 用于在过渡期间显示加载提示。



#### 示例 2：结合 `useEffect`

```jsx	
import React, { useState, useTransition, useEffect } from 'react';

const App = () => {
    const [isPending, startTransition] = useTransition();
    const [count, setCount] = useState(0);

    useEffect(() => {
        const intervalId = setInterval(() => {
            startTransition(() => {
                setCount(prevCount => prevCount + 1);
            });
        }, 1000);

        return () => clearInterval(intervalId);
    }, []);

    return (
        <div>
            {isPending && <p>Updating...</p>}
            <p>Count: {count}</p>
        </div>
    );
};

export default App;
```

在这个例子中，使用 `useEffect` 设置了一个定时器，每秒调用一次 `startTransition` 来更新 `count` 状态，把状态更新标记为非紧急。



### 注意事项

- **紧急更新优先**：紧急更新（如事件处理函数中的状态更新）会优先于过渡更新执行。
- **避免嵌套过渡**：虽然可以嵌套调用 `startTransition`，但通常不建议这样做，以免让代码逻辑变得复杂。
- **性能优化**：合理使用 `useTransition` 能避免不必要的渲染，提高应用性能，特别是在处理大量数据或复杂计算时。



# react-router-dom

## 基本用法
<BrowserRouter>
它是 React Router 的核心组件之一，用于包裹整个应用，创建一个基于 HTML5 history API 的路由系统。

```jsx
import { BrowserRouter } from 'react-router-dom';

const App = () => {
  return (
    <BrowserRouter>
      {/* 应用的其他部分 */}
    </BrowserRouter>
  );
};

export default App;
```

`<Routes>` 和 `<Route>`
<Routes> 组件是 react-router-dom v6 中的新特性，用于包裹多个 <Route> 组件，它会根据当前 URL 匹配并渲染第一个符合条件的 <Route>。
<Route> 组件用于定义路由规则，path 属性指定匹配的路径，element 属性指定匹配成功后要渲染的组件。

```jsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import Home from './Home';
import About from './About';

const App = () => {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
      </Routes>
    </BrowserRouter>
  );
};

export default App;
```

## 导航
**`<Link>`**
`<Link>` 组件用于在应用内创建链接，点击后会导航到指定的路径，它会阻止浏览器的默认跳转行为，使用 React Router 的路由系统进行导航。

```jsx
import { Link } from 'react-router-dom';

const Navbar = () => {
  return (
    <nav>
      <ul>
        <li><Link to="/">Home</Link></li>
        <li><Link to="/about">About</Link></li>
      </ul>
    </nav>
  );
};

export default Navbar;
```

**`<NavLink>`**
`<NavLink>` 和`<Link>`用法相同，`<link>`生成`<a>`标签，`<NavLink>`负责导航

**`<useNavigate>`**
useNavigate 是一个钩子函数，用于在函数组件中进行编程式导航

```jsx
import { useNavigate } from 'react-router-dom';

const LoginButton = () => {
  const navigate = useNavigate();

  const handleLogin = () => {
    // 模拟登录成功后导航到主页
    navigate('/');
  };

  return (
    <button onClick={handleLogin}>Login</button>
  );
};

export default LoginButton;
```

## 路由参数
可以在 path 属性中使用冒号（:）来定义路由参数，然后通过 useParams 钩子函数获取这些参数。

```jsx
import { BrowserRouter, Routes, Route, useParams } from 'react-router-dom';

const UserProfile = () => {
  const { id } = useParams();
  return (
    <div>
      <h1>User Profile: {id}</h1>
    </div>
  );
};

const App = () => {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/users/:id" element={<UserProfile />} />
      </Routes>
    </BrowserRouter>
  );
};

export default App;
```

## 路由守卫
可以通过自定义组件来实现路由守卫，根据条件决定是否允许用户访问某个路由。

```jsx
import { Navigate, useLocation } from 'react-router-dom';

const PrivateRoute = ({ children }) => {
  const isAuthenticated = false; // 模拟用户是否已登录
  const location = useLocation();

  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return children;
};

export default PrivateRoute;
```

然后在需要保护的路由中使用 PrivateRoute：

```jsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import PrivateRoute from './PrivateRoute';
import Dashboard from './Dashboard';
import Login from './Login';

const App = () => {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/dashboard" element={
          <PrivateRoute>
            <Dashboard />
          </PrivateRoute>
        } />
      </Routes>
    </BrowserRouter>
  );
};

export default App;

```

## 嵌套路由

```jsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import Dashboard from './Dashboard';
import Users from './Users';
import Orders from './Orders';
import ErrorPage from './ErrorPage';

const router = createBrowserRouter([
  {
    path: '/dashboard',
    element: <Dashboard />,
    errorElement: <ErrorPage />,
    children: [
      {
        path: 'users',
        element: <Users />
      },
      {
        path: 'orders',
        element: <Orders />
      }
    ]
  }
]);

const App = () => {
  return (
    <RouterProvider router={router} />
    <Outlet />       {/* 指定子路由组件的渲染位置*/}
  );
};

export default App;
```

## router hook
### createBrowserRouter
createBrowserRouter 是 react-router-dom v6 中用于创建路由配置的一个函数，它提供了一种更灵活、可扩展的方式来定义路由，相比于直接在组件树中使用 `<Routes>` 和 `<Route>` ，createBrowserRouter 更适合用于大型应用的路由管理

#### 基本使用步骤
createBrowserRouter 接收一个路由配置数组作为参数，每个配置对象定义了一个路由规则。

```jsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import Home from './Home';
import About from './About';

// 创建路由配置
const router = createBrowserRouter([
  {
    path: '/',
    element: <Home />
  },
  {
    path: '/about',
    element: <About />
  }
]);

const App = () => {
  return (
    // 使用 RouterProvider 来渲染路由
    <RouterProvider router={router} />
  );
};

export default App;
```

#### 路由配置对象详解
每个路由配置对象可以包含以下常用属性：
- path：用于匹配 URL 的路径，可以包含动态参数（如 :id ）。
- element：当路径匹配成功时要渲染的 React 元素。
- children：用于定义嵌套路由，是一个路由配置对象数组。
- errorElement：当该路由或其子路由发生错误时要渲染的元素。

以下是一个包含嵌套路由和错误处理的示例：

```jsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import Dashboard from './Dashboard';
import Users from './Users';
import Orders from './Orders';
import ErrorPage from './ErrorPage';

const router = createBrowserRouter([
  {
    path: '/dashboard',
    element: <Dashboard />,
    errorElement: <ErrorPage />,
    children: [
      {
        path: 'users',
        element: <Users />
      },
      {
        path: 'orders',
        element: <Orders />
      }
    ]
  }
]);

const App = () => {
  return (
    <RouterProvider router={router} />
  );
};

export default App;
```

### useRoutes
useRoutes 是 react-router-dom v6 中的一个钩子函数，它允许你以对象配置的方式来定义路由，这种方式更接近 createBrowserRouter 的配置风格，但 useRoutes 是在组件内部使用的，适合在某些需要动态生成路由或者在组件中根据条件渲染不同路由配置的场景

useRoutes 接收一个路由配置数组作为参数，该数组中的每个对象定义了一个路由规则。

```jsx
import { BrowserRouter, useRoutes } from 'react-router-dom';
import Home from './Home';
import About from './About';

const App = () => {
  const routes = [
    {
      path: '/',
      element: <Home />
    },
    {
      path: '/about',
      element: <About />
    }
  ];

  const element = useRoutes(routes);

  return element;
};

const Root = () => {
  return (
    <BrowserRouter>
      <App />
    </BrowserRouter>
  );
};

export default Root;
```

### useNavigate
可以在组件中使用 useNavigate 进行编程式导航：

```jsx

import { useNavigate } from 'react-router-dom';

const LoginButton = () => {
  const navigate = useNavigate();

  const handleLogin = () => {
    // 模拟登录成功后导航到主页
    navigate('/');
  };

  return (
    <button onClick={handleLogin}>Login</button>
  );
};

export default LoginButton;
```

## Data API
前置条件
- `createBrowserRouter`
- `createMemoryRouter`
- `createHashRouter`
- `createStaticRouter`

只有上面四个API创建的路由才有Data API功能

### Loader、useLoaderData
loader 和 useLoaderData 是两个非常重要的概念，它们为在路由渲染之前进行数据加载提供了便利

**Loader**
loader 是一个在路由匹配时执行的函数，其主要用途是在渲染路由组件之前获取所需的数据。这个函数返回一个 Promise，当 Promise 被解决后，其结果会作为数据传递给对应的路由组件。
在路由配置中，为路由对象添加 loader 属性，其值为一个异步函数。这个异步函数可以进行网络请求、访问本地存储等操作来获取数据。

```jsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import ProductList from './ProductList';

// 定义 loader 函数
const productListLoader = async () => {
    try {
        const response = await fetch('https://api.example.com/products');
        if (!response.ok) {
            throw new Error('Failed to fetch products');
        }
        return response.json();
    } catch (error) {
        console.error('Error fetching products:', error);
        throw error;
    }
};

const router = createBrowserRouter([
    {
        path: '/products',
        element: <ProductList />,
        loader: productListLoader
    }
]);

const App = () => {
    return <RouterProvider router={router} />;
};

export default App;    
```

**useLoaderData**
useLoaderData 是一个 React 钩子函数，用于在路由组件中获取 loader 函数返回的数据。它使得组件能够方便地使用在路由加载阶段获取到的数据。
在路由组件内部调用 useLoaderData 钩子，它会返回 loader 函数解决后的结果。
结合上面的 loader 示例，以下是使用 useLoaderData 在 ProductList 组件中获取商品列表数据的代码：

```jsx
import { useLoaderData } from 'react-router-dom';

const ProductList = () => {
    const products = useLoaderData();
    return (
        <div>
            <h1>Product List</h1>
            <ul>
                {products.map((product) => (
                    <li key={product.id}>{product.name}</li>
                ))}
            </ul>
        </div>
    );
};

export default ProductList;    
```

**总结**
- loader 函数负责在路由匹配时进行数据加载，它在路由组件渲染之前执行，确保组件所需的数据已经准备好。
- useLoaderData 钩子用于在路由组件中获取 loader 函数返回的数据，让组件能够方便地使用这些数据进行渲染。
  

通过使用 loader 和 useLoaderData，可以实现数据获取和组件渲染的分离，提高代码的可维护性和性能。


### action、useActionData
action 和 useActionData 是用于处理表单提交、数据修改等操作的重要工具

**action**
action 是一个在路由配置中定义的函数，通常用于处理表单提交或其他数据修改操作。当用户提交表单或触发某些交互时，action 函数会被调用，它可以执行诸如数据验证、发送请求到服务器进行数据创建、更新或删除等操作。

在路由配置对象中，为特定的路由添加 action 属性，该属性的值是一个异步函数。这个函数接收一个包含 request 对象的参数，request 对象可用于获取表单数据。

假设你有一个用于创建新文章的表单，以下是路由配置及 action 函数的示例:

```jsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import NewArticleForm from './NewArticleForm';

// 定义 action 函数
const createArticleAction = async ({ request }) => {
    const formData = await request.formData();
    const title = formData.get('title');
    const content = formData.get('content');

    // 模拟发送请求到服务器创建文章
    try {
        const response = await fetch('https://api.example.com/articles', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ title, content })
        });

        if (!response.ok) {
            throw new Error('Failed to create article');
        }

        const newArticle = await response.json();
        return newArticle;
    } catch (error) {
        console.error('Error creating article:', error);
        return { error: 'Failed to create article' };
    }
};

const router = createBrowserRouter([
    {
        path: '/articles/new',
        element: <NewArticleForm />,
        action: createArticleAction
    }
]);

const App = () => {
    return <RouterProvider router={router} />;
};

export default App;    
```


**useActionData**
useActionData 是一个 React 钩子函数，用于在组件中获取 action 函数返回的数据。通过它，组件可以根据 action 函数的执行结果进行不同的渲染或操作，比如显示成功消息、错误提示或进行页面跳转。

在使用 action 的路由对应的组件中调用 useActionData 钩子，它会返回 action 函数执行后返回的数据。

```jsx
import { useActionData, useNavigate } from 'react-router-dom';

const NewArticleForm = () => {
    const navigate = useNavigate();
    const actionData = useActionData();

    if (actionData && !actionData.error) {
        // 文章创建成功，跳转到文章列表页
        navigate('/articles');
    }

    const handleSubmit = (e) => {
        e.preventDefault();
        // 表单提交逻辑会触发 action 函数
    };

    return (
        <form onSubmit={handleSubmit}>
            <label>
                Title:
                <input type="text" name="title" />
            </label>
            <label>
                Content:
                <textarea name="content"></textarea>
            </label>
            <button type="submit">Create Article</button>
            {actionData && actionData.error && <p>{actionData.error}</p>}
        </form>
    );
};

export default NewArticleForm;    
```

**总结**
- action 函数负责处理表单提交等操作，执行数据的创建、更新或删除等逻辑，并返回处理结果。
- useActionData 钩子用于在组件中获取 action 函数的返回数据，以便根据结果进行相应的处理，如页面跳转、显示提示信息等。
  

通过 action 和 useActionData 的配合使用，可以方便地实现表单提交和数据修改的交互逻辑。





# 项目模块

## axios封装

```tsx
import axios from "axios";
import {message} from "antd";


const instance = axios.create({
  baseURL:'',
  timeout:3000,
  timeoutErrorMessage:"请求无效",
  withCredentials:true,
  headers: {
    'Content-Type': 'application/json',
    Authorization:'Bearer' + localStorage.getItem("token"),
    'X-Requested-With': 'XMLHttpRequest'
  },
})

// 请求拦截器
instance.interceptors.request.use(
  (config)=>{
    return config;
  },
  (error)=>{
    return Promise.reject(error);
  }
)

// 响应拦截器
instance.interceptors.response.use(
  (response)=>{
    const data =response.data;
    if (data.code === 40001){
      window.location.href = "/login"
    }else if(data.code !==200 ){
      message.error(data.message)
    }
    return data.data;
  },
  (error)=>{
    return Promise.reject(error);
  }
)

export default {
  get:(url:string,param?:object)=>{
      return instance.get(url,param);
  },
  post:(url:string,param?:object)=>{
    return instance.post(url,param);
  },
  put:(url:string,param?:object)=>{
    return instance.put(url,param);
  },
  delete:(url:string,param?:object)=>{
    return instance.delete(url,param);
  }
}


```



## Storeage

```tsx
// 本地存储
export default {
  get:(key:string)=>{
    return localStorage.getItem(key)
  },
  set:(key:string,value:any)=>{
    return localStorage.setItem(key,JSON.stringify(value));
  },
  delete:(key:string)=>{
    return localStorage.removeItem(key);
  },
  clear:()=>{
    return localStorage.clear();
  }
}

```



## 环境变量

根目录创建`.env.development` ,修改`package.json`中的编译命令

```jsx
// .env.development
VITE_BASE_URL=/api

// package.json
  "scripts": {
    "dev": "vite",      // 开发模式
    "build": "tsc -b && vite build --mode product",  // 打包.env.product
    "lint": "eslint .",
    "preview": "vite preview"
  },

```

在项目中调用

```tsx
const url = import.meta.env.VITE_BASE_URL
```



## 代理

`vite.config.ts` 文件

```tsx
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

// https://vite.dev/config/
export default defineConfig({
  plugins: [react()],
  server:{
    proxy:{
      '/api':{
        target: 'http://localhost:8000',
        changeOrigin: true,
      }
    }
  }
})

```



## CSS Module

css文件在打包后都会在一个文件中，容易出现污染,只需开启css module

```tsx
// 创建css文件名为如： index.module.css

// 导入: import style from 'index.module.css'

<h1 className={style.login}></h1>   

// 需要使用驼峰命名
```

