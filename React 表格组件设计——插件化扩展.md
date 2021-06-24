# React 表格组件设计——插件化扩展

## 前言

最近正好在整理一些项目，加上之前学习 Babel 插件，因此对 **插件化的思想** 有了进一步的体会，也启发我基于插件化的思想进行表格功能的解耦。

在之前的[文章](https://zhuanlan.zhihu.com/p/362473728)中我已经提到过，如果一个复杂的组件/系统没有一个完善的 **可扩展机制**，随着功能的增加，极有可能会陷入代码冗余、功能耦合、扩展困难的境地。这点大家在开发中想必深有体会——看着代码逐渐“坏掉”，改不动了索性将就用着吧。

在表格开发中，表格可能需要提供数据展示，排序和筛选，数据分页等功能，而这些功能可能需要冗余在一个 `columns` 中进行配置，也有可能要散落在代码中的多个地方配置。我们期望实现的效果是怎么样的呢：

- **高内聚**，代码以功能为维度进行组织和拆分，直观明确，易于修改
- **低耦合**，多个功能之间互不影响，易于扩展和插拔

而基于插件系统的开放性和灵活性，正好可以解决上述的问题。

## 插件系统的核心设计思想

在前端项目中，我们或多或少会接触到插件的概念，比如 [webpack 插件](https://www.webpackjs.com/concepts/plugins/)、[Babel 插件](https://babel.docschina.org/docs/en/7.0.0/plugins/)等，这些插件设计的基本思路都是通过预先配置并安装，随后在程序运行的某个时机/过程被触发，用于解决某组特定的问题。

总的来说，一个插件系统的设计可以分为以下几个部分：
![Pugin System](https://cdn.nlark.com/yuque/0/2021/png/474741/1624523937489-80d4ec9a-9c6e-491e-bab5-7f9858070955.png)

- **Program**：主程序，对外提供一些 Hooks，可供插件调用；
- **Plugin Loader**：加载器，提供注册的接口或者从系统文件中拉取配置，并将插件安装到 Program 中；
- **Plugin Interface**：基于 Program 提供的 Hooks，约定一个 Plugin 和 Program 都能理解的通用接口；
- **Plugin Implementation**：基于 Plugin Interface 实现具体的功能；

## 插件化的实践

本文的例子主要是基于 Ant Design Table 进行功能扩展，其他组件的开发也可借鉴类似的思路。接下来，让我们看看如何基于插件解耦与扩展表格的功能。

### 功能阐述

我们先来看看官网的一个受控排序的[例子](https://ant.design/components/table-cn/#components-table-demo-reset-filter)，我简化了下主体代码：

```tsx
// 1. 管理排序的状态
const [sortedInfo, setSortedInfo] = useState<{
  order: string;
  columnKey: string;
} | null>(null);

// 2. 处理排序回调
const handleChange = (pagination, filters, sorter, extra) => {
  console.log("params", pagination, filters, sorter, extra);
  if (extra.action === "sort") {
    // 处理排序变化
    setSortedInfo(sorter);
  }
  if (extra.action === "filter") {
    // 处理筛选变化
  }
  if (extra.action === "paginate") {
    // 处理分页变化
  }
};
// 3. 定义排序的配置
const columns = [
  {
    title: "Name",
    key: "name",
    sorter: (a, b) => a.name.length - b.name.length,
    sortOrder: sortedInfo.columnKey === "name" && sortedInfo.order,
  },
  {
    title: "Age",
    key: "age",
    sorter: (a, b) => a.age - b.age,
    sortOrder: sortedInfo.columnKey === "age" && sortedInfo.order,
  },
  {
    title: "Address",
    key: "address",
    sorter: (a, b) => a.address.length - b.address.length,
    sortOrder: sortedInfo.columnKey === "address" && sortedInfo.order,
  },
];

<Table
  sortDirections={["ascend", "descend"]}
  columns={columns}
  dataSource={data}
  onChange={handleChange}
/>;
```

我们可以看到，实现一个受控排序，我们需要在多个地方进行配置和修改。如果是复杂的中后台项目，表格往往有二三十列数据，`columns` 的配置也会越来越复杂。

这里，可以考虑把排序相关的功能进行整合，通过 `sortOptions` 进行配置，例如：

```tsx
// 定义表格排序配置
const [sorts, setSorts] = useState([
  {
    key: "name",
    sorter: (a, b) => a.name.length - b.name.length,
    sortOrder: true,
  },
  {
    key: "age",
    sorter: (a, b) => a.age - b.age,
    sortOrder: false,
  },
  {
    key: "address",
    sorter: (a, b) => a.address.length - b.address.length,
    sortOrder: false,
  },
]);

return (
  <Table
    sortOptions={{
      sortDirections: ["ascend", "descend"],
      sorts,
      onSortChange: (nextSorts) => {
        setSorts(nextSorts);
      },
    }}
    columns={columns}
    dataSource={dataSource}
  />
);
```

### 插件功能开发

在了解了插件思想和表格的封装需求之后，我们开始设计一个表格插件系统。

#### 入口程序

启动项目的第一步——思考插件和系统如何交互？这里为了确保始终拿到的是最新的 `props` 数据，需要考虑：

- **初始化**：`TablePlugin` 不处理任何与外界数据交互的行为，只是简单地进行插件的注册和合法性校验；
- **处理数据**：通过 `TablePlugin` 的 `getProps` 拿到最新的 `props`，在 `TablePlugin` 内部调用注册过的插件进行数据处理；
- **传递数据**：将包装过的 `pluginProps` 传递给 `AntTable`；

```js
const Table = <RecordType extends object = object>(
  props: TableProps<RecordType>,
) => {
  /**
   * 初始化并注册插件
   */
  const plugin = useMemo(() => {
    const plugin = new TablePlugin<RecordType>();
    plugin.register('plugin:sort', TableSortPlugin);
    return plugin;
  }, []);

  /**
   * 获取处理后的 props
   */
  const pluginProps = plugin.getProps(props);

  /**
   * 将数据传递给 Antd Table
   */
  return <AntTable<RecordType> {...pluginProps}></AntTable>;
};
```

#### 设计插件加载器

在上一步我们已经知道，`TablePlugin` 对外需要提供两个 API：`register` 和 `getProps`，主体代码如下：

```tsx
class TablePlugin<
  R extends object = object,
  Props extends TableProps<R> = TableProps<R>
> {
  private plugins: Record<string, Plugin<R, Props>> = {};

  /**
   * @description 注册插件
   */
  register(name: string, plugin: Plugin<R, Props>) {
    if (this.plugins[name]) {
      console.warn(`[${name}] already installed!`);
    }
    this.plugins[name] = plugin;
  }

  /**
   * @description 获取包装后的 Props
   */
  getProps(props: Props): TableProps<R> {
    const plugins = this.initPlugins(props);
    const pluginProps = this.processProps(plugins, props);
    return pluginProps;
  }
}
```

#### 定义插件接口

在本文的例子中，插件需要包装的主要是两个配置：`columns` 和 `onChange`。我们将 `Plugin` 设计为一个函数，可以接收传参，同时定义两个包装函数：

```tsx
export type Plugin<
  R extends object = object,
  P extends TableProps<R> = TableProps<R>
> = (props: P) => {
  /**
   * 插件名称
   */
  name: string;
  /**
   * 插件优先级
   */
  stage?: number;
  /**
   * 包装 columns
   */
  columns?: (columns: P["columns"]) => P["columns"];
  /**
   * 包装 onChange
   */
  onChange?: P["onChange"];
};
```

更进一步？`columns` 配置实际是树形结构，如果每个插件都需要遍历处理，可能会增加额外的计算。我们是不是可以参考 Babel 插件的写法呢？

Babel 在 `parse`阶段会生成一棵 AST 树，随后在 `transform` 阶段，Babel 会调用 `visitor` 函数来处理不同的 AST 节点：

```js
export default function () {
  return {
    visitor: {
      Identifier(path) {
        const name = path.node.name;
        // reverse the name: JavaScript -> tpircSavaJ
        path.node.name = name.split("").reverse().join("");
      },
    },
  };
}
```

同样地，如果我们把每个 `column` 当成一个节点，同样可以使用 `visitor` 函数进行处理。最终的插件接口定义如下：

```tsx
export type Plugin<
  R extends object = object,
  P extends TableProps<R> = TableProps<R>
> = (props: P) => {
  /**
   * 插件名称
   */
  name: string;
  /**
   * 插件优先级
   */
  stage?: number;
  /**
   * 包装 column
   */
  visitor?: ColumnVisitor<R>;
  /**
   * 包装 onChange
   */
  onChange?: P["onChange"];
};
```

#### 实现插件

接下来，我们就可以基于接口，自定义各种功能的插件啦~比如，我们定义一个 `TableSortPlugin` 用于处理 `sortOptions`：

```tsx
const TableSortPlugin = <
  R extends object = object,
  Props extends TableProps<R> = TableProps<R>
>({
  sortOptions = { sorts: [] },
}: Props) => {
  const { sorts } = sortOptions;
  const sortConfigs = sorts.reduce<{ [x: string]: SortItem<R> }>(
    (obj, item) => {
      // 处理数据，省略
      return obj;
    },
    {}
  );
  return {
    name: "plugin:sort",
    visitor(column: ColumnType<R>, index: number, depth: number) {
      if (!sorts.length) return column;
      const sortProps = sortConfigs[column.key as string] ?? {};
      return { ...column, ...sortProps };
    },
    onChange(...args: TableChangeParameters<R>) {
      const [_pagination, _filters, sorter, extra] = args;
      if (extra.action !== "sort" || !sorter) {
        return;
      }
      // 处理数据，省略
      sortOptions.onSortChange?.(nextSorts, changeSorts);
    },
  };
};
```

我们还可以定义一个 `TableTipPlugin` 用于给表格标题增加说明文案：

```tsx
const TableTipPlugin = <
  R extends object = object,
  Props extends TableProps<R> = TableProps<R>
>(
  props: Props
) => {
  return {
    name: "plugin:tips",
    visitor(column: ColumnType<R>, index: number, depth: number) {
      let title = column.title;
      if (column.tips) {
        title = (
          <Fragment>
            {column.title}
            <Tooltip placement="top" title={column.tips} autoAdjustOverflow>
              <QuestionCircleOutlined style={{ marginLeft: 4 }} />
            </Tooltip>
          </Fragment>
        );
      }
      return { ...column, title };
    },
  };
};
```

#### 完善插件加载器

最后一步，系统如何加载插件并包装数据。我们定义一个 `loadPlugins` 方法：

```tsx
class TablePlugin<
  R extends object = object,
  Props extends TableProps<R> = TableProps<R>
> {
  //...
  getProps(props: Props): TableProps<R> {
    const plugins = this.initPlugins(props);
    const pluginProps = this.processProps(plugins, props);
    return pluginProps;
  }

  /**
   * @description 加载插件
   */
  private loadPlugins(props: Props) {
    const plugins: ReturnType<Plugin<R, Props>>[] = [];
    Object.values(this.plugins).forEach((pluginFn) => {
      const plugin = pluginFn(props);
      plugins.push(plugin);
    });
    return plugins.sort(({ stage: a = 0 }, { stage: b = 0 }) => b - a);
  }
}
```

为了处理多个 `visitor`，我们定义一个 `pipe` 函数进行处理：

```tsx
export function pipe<R>(funcs: ColumnVisitor<R>[]): ColumnVisitor<R> {
  if (funcs.length === 0) {
    return (...args: Parameters<ColumnVisitor<R>>) => args[0];
  }
  if (funcs.length === 1) {
    return funcs[0];
  }
  return funcs.reduce((a, b) => (...args: Parameters<ColumnVisitor<R>>) => {
    const [column, ...rest] = args;
    return b(a(column, ...rest), ...rest);
  });
}
```

基于插件系统，我们保证了核心代码的精简，同时可以方便地扩展表格功能。完整的代码已经放在[github](https://github.com/shujer/enhance-component)仓库，有些实现比较简单。大家可以拉下来跑跑或者自己尝试实现一个功能插件。

## 推荐阅读

- [精读《插件化思维》](https://github.com/ascoders/weekly/blob/master/%E5%89%8D%E6%B2%BF%E6%8A%80%E6%9C%AF/53.%E7%B2%BE%E8%AF%BB%E3%80%8A%E6%8F%92%E4%BB%B6%E5%8C%96%E6%80%9D%E7%BB%B4%E3%80%8B.md)
- [如何设计开发一个 Web 插件系统？](https://juejin.cn/post/6969467348184989727)
