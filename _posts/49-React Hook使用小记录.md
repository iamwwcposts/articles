---
title: React Hook使用小记录
date: 2020-10-11
updated: 2020-10-11
issueid: 49
tags:
- 前端
---
# UseMemo 的使用

FP 组件每次更新渲染都会被调用，我们希望更新时保存上次渲染的状态
比如如下组件 onChange触发之后组件更新，结果输入的值又没了

```jsx
const [formData, setFormData] = useState("");
const [formSchema, setFormSchema] = useState(defaultSchema);
return (
  <Row>
    <Col span={8}>
      <div className="pattern-editor">
        <Editor
          initialCode={JSON.stringify(defaultSchema)}
          onChange={() => {}}
        ></Editor>
      </div>
    </Col>
    <Col span={8}>
      <div className="WYSIWYG-editor">
        <SchemaForm
          schema={formSchema}
          onChange={(e) => {
            setFormData(JSON.stringify(e.formData));
          }}
        >
          <React.Fragment></React.Fragment>
        </SchemaForm>
      </div>
    </Col>
    <Col span={8}>
      <div className="config-output">
        <Editor
          initialCode={JSON.stringify(formData)}
          onChange={() => {}}
        ></Editor>
      </div>
    </Col>
  </Row>
);
```

而使用useMemo改造后，只要依赖数组不变化，那组件不会被重新渲染，状态还会被保留

```jsx
const [formData, setFormData] = useState("");
const [formSchema, setFormSchema] = useState(defaultSchema);
return (
  <Row>
    <Col span={8}>
      <div className="pattern-editor">
        <Editor
          initialCode={JSON.stringify(defaultSchema)}
          onChange={() => {}}
        ></Editor>
      </div>
    </Col>
    <Col span={8}>
      <div className="WYSIWYG-editor">
        {useMemo(
          () => (
            <SchemaForm
              schema={formSchema}
              onChange={(e) => {
                setFormData(JSON.stringify(e.formData));
              }}
            >
              <React.Fragment></React.Fragment>
            </SchemaForm>
          ),
          [formSchema]
        )}
      </div>
    </Col>
    <Col span={8}>
      <div className="config-output">
        {useMemo(
          () => (
            <Editor
              initialCode={JSON.stringify(formData)}
              onChange={() => {}}
            ></Editor>
          ),
          [formData]
        )}
      </div>
    </Col>
  </Row>
);
```
## useRef

函数组件每次调用函数都是新的引用，如果这个函数你传给了子组件，子组件不重新渲染那拿不到最新的函数，可以利用useRef保存函数状态

![image](https://user-images.githubusercontent.com/24750337/94576173-a4a4f480-02a7-11eb-838e-ddb4422a2136.png)

## useEffect

#### 最佳实践

看下面的代码，我们希望 createItem 创建完刷新页面，拉取最新的数据。
```js
 const [pattern, setPattern] = useState({});
useEffect(() => {
    (async () => {
        const items_temp = await api.items.getAllItems(key_id);
        const key_temp = await api.keys.getOne(key_id);
        setItems(items_temp);
        setKey(key_temp);
    })();
}, []);
```

如果我们将API请求和 setXXX 放到 `componentDidUpdate` 里会触发React无限更新

```js
const [x, SetX] = useState([]);
useEffect(() => {
    (async () => {
        const x = await api.get(id);
        // 这样会触发无限更新！
        setX(x);
    })();
})
```

解决办法是把API和SetX做成单独的函数，不要放到 `useEffect` 里，需要的时候调用一下就好了


```js
useEffect(() => {
    (async () => {
        fetchList();
    })();
}, []);
async function fetchList() {
    const items_temp = await api.items.getAllItems(key_id);
    const key_temp = await api.keys.getOne(key_id);
    setItems(items_temp);
    setKey(key_temp);
}
```

需要的拉取新数据时就调用一下

```js
try {
    await api.items.createItem(key_id, {
        Value: value,
        ...toUpperCase(data)
    });
} catch (e) {
    console.log(e);
} finally {
    setLoading(false);
    setModalVisiable(false);
    fetchList();
}
```