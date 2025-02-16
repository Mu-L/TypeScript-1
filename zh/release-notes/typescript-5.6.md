# TypeScript 5.6

## 禁止空值和真值检查

也许你曾经编写过正则表达式却忘记调用 `.test(...)`：

```typescript
if (/0x[0-9a-f]/) {
  // 哎呀！这个代码块总是会执行。
  // ...
}
```

或者你可能不小心写了 `=>`（创建一个箭头函数）而不是 `>=`（大于或等于运算符）：

```typescript
if (x => 0) {
  // 哎呀！这个代码块总是会执行。
  // ...
}
```

或者你可能尝试使用 `??` 设置默认值，但混淆了 `??` 和比较运算符（如 `<`）的优先级：

```typescript
function isValid(
  value: string | number,
  options: any,
  strictness: 'strict' | 'loose'
) {
  if (strictness === 'loose') {
    value = +value;
  }
  return value < options.max ?? 100;
  // 哎呀！这被解析为 (value < options.max) ?? 100
}
```

或者你可能在复杂表达式中放错了括号：

```typescript
if (
  isValid(primaryValue, 'strict') ||
  isValid(secondaryValue, 'strict') ||
  isValid(primaryValue, 'loose' || isValid(secondaryValue, 'loose'))
) {
  //                           ^^^^ 👀 我们忘记闭合 ')' 了吗？
}
```

这些示例都没有实现作者的意图，但它们都是有效的 JavaScript 代码。
此前，TypeScript 也默默地接受了这些示例。

通过一些实验，我们发现许多错误可以通过标记上述可疑示例来捕获。
在 TypeScript 5.6 中，当编译器可以从语法上确定真值或空值检查总是以特定方式求值时，它会报错。
因此，在上述示例中，你将开始看到错误：

```typescript
if (/0x[0-9a-f]/) {
  //  ~~~~~~~~~~~~
  // 错误：这种表达式总是为真。
}

if (x => 0) {
  //  ~~~~~~
  // 错误：这种表达式总是为真。
}

function isValid(
  value: string | number,
  options: any,
  strictness: 'strict' | 'loose'
) {
  if (strictness === 'loose') {
    value = +value;
  }
  return value < options.max ?? 100;
  //     ~~~~~~~~~~~~~~~~~~~
  // 错误：?? 的右操作数不可达，因为左操作数永远不会为空。
}

if (
  isValid(primaryValue, 'strict') ||
  isValid(secondaryValue, 'strict') ||
  isValid(primaryValue, 'loose' || isValid(secondaryValue, 'loose'))
) {
  //                    ~~~~~~~
  // 错误：这种表达式总是为真。
}
```

通过启用 ESLint 的 `no-constant-binary-expression` 规则也可以实现类似的效果，但 TypeScript 的新检查与 ESLint 规则并不完全重叠，我们认为将这些检查内置到 TypeScript 中具有很大的价值。

需要注意的是，某些表达式仍然被允许，即使它们总是为真或为空。
特别是 `true`、`false`、`0` 和 `1` 仍然被允许，尽管它们总是为真或假，因为像以下代码：

```typescript
while (true) {
  doStuff();

  if (something()) {
    break;
  }

  doOtherStuff();
}
```

仍然是惯用且有用的，而像以下代码：

```typescript
if (true || inDebuggingOrDevelopmentEnvironment()) {
  // ...
}
```

在迭代或调试代码时也很有用。

如果你对这个功能的实现或它捕获的错误类型感兴趣，可以查看实现该功能的 [Pull Request](https://github.com/microsoft/TypeScript/pull/51925)。

## 迭代器辅助方法

JavaScript 有可迭代对象（通过调用 `[Symbol.iterator]()` 并获取迭代器来遍历的对象）和迭代器（具有 `next()` 方法的对象，可以在遍历时调用以获取下一个值）。
通常，当你将它们放入 `for/of` 循环或将它们展开到新数组中时，你不需要考虑这些。
但 TypeScript 确实用 `Iterable` 和 `Iterator` 类型（甚至 `IterableIterator`，它同时兼具两者！）来建模这些对象，这些类型描述了 `for/of` 等结构所需的最小成员集。

可迭代对象（和 `IterableIterator`）很好，因为它们可以在 JavaScript 的许多地方使用——但许多人发现自己缺少像 `map`、`filter` 和 `reduce` 这样的数组方法。
这就是为什么 ECMAScript 最近提出了一项提案，将许多数组方法（以及更多）添加到 JavaScript 中生成的大多数 `IterableIterator` 上。

例如，每个生成器现在都会生成一个同时具有 `map` 和 `take` 方法的对象：

```typescript
function* positiveIntegers() {
  let i = 1;
  while (true) {
    yield i;
    i++;
  }
}

const evenNumbers = positiveIntegers().map(x => x * 2);

// 输出：
//    2
//    4
//    6
//    8
//   10
for (const value of evenNumbers.take(5)) {
  console.log(value);
}
```

对于 `Map` 和 `Set` 的 `keys()`、`values()` 和 `entries()` 方法也是如此：

```typescript
function invertKeysAndValues<K, V>(map: Map<K, V>): Map<V, K> {
  return new Map(map.entries().map(([k, v]) => [v, k]));
}
```

你还可以扩展新的 `Iterator` 对象：

```typescript
/**
 * 提供一个无限的 `0` 流。
 */
class Zeroes extends Iterator<number> {
  next() {
    return { value: 0, done: false } as const;
  }
}

const zeroes = new Zeroes();

// 转换为无限的 `1` 流。
const ones = zeroes.map(x => x + 1);
```

你可以使用 `Iterator.from` 将任何现有的可迭代对象或迭代器适配为这种新类型：

```typescript
Iterator.from(...).filter(someFunction);
```

所有这些新方法都可以在较新的 JavaScript 运行时中使用，或者你可以使用新的 `Iterator` 对象的 polyfill。

## 严格的内置迭代器检查（和 `--strictBuiltinIteratorReturn`）

当你调用 `Iterator<T, TReturn>` 的 `next()` 方法时，它会返回一个具有 `value` 和 `done` 属性的对象。这通过 `IteratorResult` 类型建模：

```typescript
type IteratorResult<T, TReturn = any> =
  | IteratorYieldResult<T>
  | IteratorReturnResult<TReturn>;

interface IteratorYieldResult<TYield> {
  done?: false;
  value: TYield;
}

interface IteratorReturnResult<TReturn> {
  done: true;
  value: TReturn;
}
```

这里的命名灵感来自生成器函数的工作方式。
生成器函数可以生成值，然后返回一个最终值——但两者之间的类型可能无关。

```typescript
function abc123() {
  yield 'a';
  yield 'b';
  yield 'c';
  return 123;
}

const iter = abc123();

iter.next(); // { value: "a", done: false }
iter.next(); // { value: "b", done: false }
iter.next(); // { value: "c", done: false }
iter.next(); // { value: 123, done: true }
```

随着新的 `IteratorObject` 类型的引入，我们发现允许安全实现 `IteratorObject` 存在一些困难。
同时，`IteratorResult` 在 `TReturn` 为 `any`（默认值！）的情况下存在长期的不安全性。
例如，假设我们有一个 `IteratorResult<string, any>`。
如果我们最终访问此类型的值，我们将得到 `string | any`，这实际上就是 `any`。

```typescript
function* uppercase(iter: Iterator<string, any>) {
  while (true) {
    const { value, done } = iter.next();
    yield value.toUppercase(); // 哎呀！忘记先检查 `done` 并且拼错了 `toUpperCase`

    if (done) {
      return;
    }
  }
}
```

目前，要在每个迭代器上修复此问题而不引入大量破坏是很困难的，但我们至少可以在大多数创建的 `IteratorObject` 上修复它。

TypeScript 5.6 引入了一个新的内置类型 `BuiltinIteratorReturn` 和一个新的 `--strict-mode` 标志 `--strictBuiltinIteratorReturn`。
每当在 `lib.d.ts` 中使用 `IteratorObject` 时，它们总是用 `BuiltinIteratorReturn` 类型表示 `TReturn`（尽管你会更常见到更具体的 `MapIterator`、`ArrayIterator`、`SetIterator` 等）。

```typescript
interface MapIterator<T>
  extends IteratorObject<T, BuiltinIteratorReturn, unknown> {
  [Symbol.iterator](): MapIterator<T>;
}

// ...

interface Map<K, V> {
  // ...

  /**
   * 返回一个包含 map 中每个条目的键值对的可迭代对象。
   */
  entries(): MapIterator<[K, V]>;

  /**
   * 返回 map 中键的可迭代对象。
   */
  keys(): MapIterator<K>;

  /**
   * 返回 map 中值的可迭代对象。
   */
  values(): MapIterator<V>;
}
```

默认情况下，`BuiltinIteratorReturn` 是 `any`，但当启用 `--strictBuiltinIteratorReturn` 时（可能通过 `--strict`），它是 `undefined`。
在这种新模式下，如果我们使用 `BuiltinIteratorReturn`，前面的示例现在会正确报错：

```typescript
function* uppercase(iter: Iterator<string, BuiltinIteratorReturn>) {
  while (true) {
    const { value, done } = iter.next();
    yield value.toUppercase();
    //    ~~~~~ ~~~~~~~~~~~
    // error!┃      ┃
    //       ┃      ┗━ 类型 'string' 上不存在属性 'toUppercase'。你是否指的是 'toUpperCase'？
    //       ┃
    //       ┗━ 'value' 可能为 'undefined'。

    if (done) {
      return;
    }
  }
}
```

你通常会在 `lib.d.ts` 中看到 `BuiltinIteratorReturn` 与 `IteratorObject` 配对使用。
一般来说，我们建议在你自己的代码中尽可能明确 `TReturn`。

更多信息，你可以在这里阅读该功能的详细信息。

## 支持任意模块标识符

JavaScript 允许模块以字符串字面量的形式导出无效标识符名称的绑定：

```typescript
const banana = "🍌";

export { banana as "🍌" };
```

同样，它也允许模块使用这些任意名称导入并将其绑定到有效标识符：

```typescript
import { "🍌" as banana } from "./foo"

/**
 * 吃吃吃
 */
function eat(food: string) {
    console.log("Eating", food);
};

eat(banana);
```

这看起来像是一个可爱的派对技巧（如果你像我们一样在派对上很有趣），但它对于与其他语言的互操作性（通常通过 JavaScript/WebAssembly 边界）很有用，因为其他语言可能对有效标识符的构成有不同的规则。它对于生成代码的工具（如 esbuild 的 `inject` 功能）也很有用。

TypeScript 5.6 现在允许你在代码中使用这些任意模块标识符！
我们要感谢 Evan Wallace 为 TypeScript 贡献了这一更改！

## `--noUncheckedSideEffectImports` 选项

在 JavaScript 中，可以在不实际导入任何值的情况下导入模块：

```typescript
import 'some-module';
```

这些导入通常被称为副作用导入，因为它们唯一有用的行为是通过执行某些副作用（如注册全局变量或将 polyfill 添加到原型）来提供的。

在 TypeScript 中，这种语法有一个相当奇怪的怪癖：如果导入可以解析为有效的源文件，那么 TypeScript 会加载并检查该文件。
另一方面，如果找不到源文件，TypeScript 会默默地忽略该导入！

这种行为令人惊讶，但它部分源于对 JavaScript 生态系统中模式的建模。
例如，这种语法也用于 bundler 中的特殊加载器来加载 CSS 或其他资源。
你的 bundler 可能配置为通过编写如下内容来包含特定的 `.css` 文件：

```typescript
import './button-component.css';

export function Button() {
  // ...
}
```

尽管如此，这掩盖了副作用导入中潜在的拼写错误。
这就是为什么 TypeScript 5.6 引入了一个新的编译器选项 `--noUncheckedSideEffectImports` 来捕获这些情况。
当启用 `--noUncheckedSideEffectImports` 时，如果 TypeScript 找不到副作用导入的源文件，它将报错。

```typescript
import 'oops-this-module-does-not-exist';
//     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
// 错误：找不到模块 'oops-this-module-does-not-exist' 或其对应的类型声明。
```

启用此选项后，一些原本可以工作的代码现在可能会收到错误，例如上面的 CSS 示例。
为了解决这个问题，只想为资源编写副作用导入的用户可能更适合编写带有通配符说明符的环境模块声明。
它会放在一个全局文件中，看起来像这样：

```typescript
// ./src/globals.d.ts

// 将所有 CSS 文件识别为模块导入。
declare module '*.css' {}
```

事实上，你的项目中可能已经有这样的文件了！
例如，运行类似 `vite init` 的命令可能会创建一个类似的 `vite-env.d.ts` 文件。

虽然此选项目前默认关闭，但我们鼓励用户尝试使用它！

更多信息，请查看此功能的实现。

## `--noCheck` 选项

TypeScript 5.6 引入了一个新的编译器选项 `--noCheck`，它允许你跳过对所有输入文件的类型检查。
这在执行生成输出文件所需的语义分析时避免了不必要的类型检查。

一个场景是将 JavaScript 文件生成与类型检查分开，以便两者可以作为单独的阶段运行。
例如，你可以在迭代时运行 `tsc --noCheck`，然后运行 `tsc --noEmit` 进行彻底的类型检查。
你还可以并行运行这两个任务，甚至在 `--watch` 模式下，但请注意，如果你真的同时运行它们，你可能需要指定一个单独的 `--tsBuildInfoFile` 路径。

`--noCheck` 对于以类似方式生成声明文件也很有用。
在指定了 `--noCheck` 的项目中，如果项目符合 `--isolatedDeclarations`，TypeScript 可以快速生成声明文件而无需进行类型检查。
生成的声明文件将完全依赖于快速的语法转换。

请注意，在指定了 `--noCheck` 但项目未使用 `--isolatedDeclarations` 的情况下，TypeScript 可能仍会执行尽可能多的类型检查以生成 `.d.ts` 文件。
从这个意义上说，`--noCheck` 有点用词不当；
然而，该过程将比完整的类型检查更懒，仅计算未注释声明的类型。
这应该比完整的类型检查快得多。

`noCheck` 也可以通过 TypeScript API 作为标准选项使用。
在内部，`transpileModule` 和 `transpileDeclaration` 已经使用 `noCheck` 来加速（至少在 TypeScript 5.5 中）。
现在，任何构建工具都应该能够利用该标志，采取各种自定义策略来协调和加速构建。

更多信息，请参阅 TypeScript 5.5 中为 `noCheck` 内部提速所做的工作，以及使其在命令行和 API 中公开的相关工作。

## 允许在中间错误时继续构建

TypeScript 的项目引用概念允许你将代码库组织为多个项目并创建它们之间的依赖关系。
在 `--build` 模式下运行 TypeScript 编译器（或简写为 `tsc -b`）是实际跨项目构建并确定需要编译的项目和文件的内置方式。

以前，使用 `--build` 模式会假设 `--noEmitOnError` 并在遇到任何错误时立即停止构建。
这意味着如果任何“上游”依赖项有构建错误，“下游”项目将永远不会被检查和构建。
理论上，这是一种非常合理的方法——如果一个项目有错误，它不一定处于其依赖项的一致状态。

实际上，这种僵化使得升级等工作变得痛苦。
例如，如果 `projectB` 依赖于 `projectA`，那么更熟悉 `projectB` 的人无法主动升级他们的代码，直到他们的依赖项升级完毕。
他们被 `projectA` 的升级工作所阻碍。

从 TypeScript 5.6 开始，`--build` 模式将继续构建项目，即使依赖项中存在中间错误。
在存在中间错误的情况下，它们将被一致地报告，并且输出文件将尽最大努力生成；但是，构建将继续完成指定的项目。

如果你想在第一个出现错误的项目上停止构建，你可以使用一个名为 `--stopOnBuildErrors` 的新标志。这在 CI 环境中运行时，或者在迭代一个被其他项目严重依赖的项目时非常有用。

请注意，为了实现这一点，TypeScript 现在总是为 `--build` 调用中的任何项目生成 `.tsbuildinfo` 文件（即使未指定 `--incremental/--composite`）。
这是为了跟踪 `--build` 的调用状态以及未来需要执行的工作。

你可以在这里阅读有关此更改的更多信息。

## 编辑器中的区域优先诊断

当 TypeScript 的语言服务被请求获取文件的诊断信息（如错误、建议和弃用）时，它通常需要检查整个文件。
大多数情况下这没问题，但在非常大的文件中，这可能会导致延迟。
这可能令人沮丧，因为修复拼写错误应该感觉是一个快速操作，但在足够大的文件中可能需要几秒钟。

为了解决这个问题，TypeScript 5.6 引入了一个名为区域优先诊断或区域优先检查的新功能。
编辑器现在不仅可以请求一组文件的诊断信息，还可以提供给定文件的相关区域——其意图是这通常是用户当前可见的文件区域。
TypeScript 语言服务器然后可以选择提供两组诊断信息：一组用于该区域，另一组用于整个文件。
这使得在大型文件中编辑感觉更加响应迅速，因为你不会等待那么长时间让红色波浪线消失。

对于一些具体数字，在我们的测试中，TypeScript 自己的 `checker.ts` 的完整语义诊断响应耗时 3330 毫秒。
相比之下，第一个基于区域的诊断响应耗时 143 毫秒！
而剩余的全文件响应耗时约 3200 毫秒，这对于快速编辑来说可以产生巨大的差异。

此功能还包括大量工作，以使诊断在整个体验中更一致地报告。
由于我们的类型检查器利用缓存来避免工作，相同类型之间的后续检查通常会有不同的（通常更短的）错误消息。
从技术上讲，懒惰的乱序检查可能会导致诊断在编辑器中的两个位置之间报告不同——甚至在此功能之前——但我们不想加剧这个问题。
通过最近的工作，我们已经消除了许多这些错误不一致。

目前，此功能在 Visual Studio Code 中可用于 TypeScript 5.6 及更高版本。

更多详细信息，请查看此功能的实现和说明。

## 细粒度的提交字符

TypeScript 的语言服务现在为每个补全项提供自己的提交字符。
提交字符是特定字符，当键入时，它们会自动提交当前建议的补全项。

这意味着随着时间的推移，当你键入某些字符时，你的编辑器将更频繁地提交当前建议的补全项。
例如，以下代码：

```typescript
declare let food: {
    eat(): any;
}

let f = (foo/**/
```

如果我们的光标位于 `/**/`，我们正在编写的代码可能是 `let f = (food.eat())` 或 `let f = (foo, bar) => foo + bar`。
你可以想象，编辑器可能会根据我们接下来键入的字符自动完成不同的内容。
例如，如果我们键入句点/点字符 (`.`)，我们可能希望编辑器用变量 `food` 完成；
但如果我们键入逗号字符 (`,`)，我们可能正在编写箭头函数中的参数。

不幸的是，以前 TypeScript 只是向编辑器发出信号，表明当前文本可能定义了一个新的参数名称，因此没有安全的提交字符。
因此，即使很明显编辑器应该用单词 `food` 自动完成，按下 `.` 也不会做任何事情。

TypeScript 现在明确列出了每个完成项的安全提交字符。
虽然这不会立即改变你的日常体验，但支持这些提交字符的编辑器应该会随着时间的推移看到行为改进。
要立即看到这些改进，你现在可以在 Visual Studio Code Insiders 中使用 TypeScript 夜间扩展。
在上面的代码中按下 `.` 会正确自动完成 `food`。

更多信息，请参阅添加提交字符的 Pull Request 以及我们根据上下文调整提交字符的工作。

## 自动导入的排除模式

TypeScript 的语言服务现在允许你指定一个正则表达式模式列表，这些模式将过滤掉某些说明符的自动导入建议。
例如，如果你想排除像 `lodash` 这样的包的所有“深层”导入，你可以在 Visual Studio Code 中配置以下首选项：

```json
{
  "typescript.preferences.autoImportSpecifierExcludeRegexes": ["^lodash/.*$"]
}
```

或者反过来，你可能希望禁止从包的入口点导入：

```json
{
  "typescript.preferences.autoImportSpecifierExcludeRegexes": ["^lodash$"]
}
```

你甚至可以通过以下设置避免 `node:` 导入：

```json
{
  "typescript.preferences.autoImportSpecifierExcludeRegexes": ["^node:"]
}
```

要指定某些正则表达式标志（如 `i` 或 `u`），你需要用斜杠包围你的正则表达式。
当提供包围斜杠时，你需要转义其他内部斜杠。

```json
{
  "typescript.preferences.autoImportSpecifierExcludeRegexes": [
    "^./lib/internal", // 不需要转义
    "/^.\\/lib\\/internal/", // 需要转义 - 注意前导和尾随斜杠
    "/^.\\/lib\\/internal/i" // 需要转义 - 我们需要斜杠来提供 'i' 正则表达式标志
  ]
}
```

相同的设置可以通过 VS Code 中的 `javascript.preferences.autoImportSpecifierExcludeRegexes` 应用于 JavaScript。

请注意，虽然此选项可能与 `typescript.preferences.autoImportFileExcludePatterns` 有一些重叠，但存在差异。
现有的 `autoImportFileExcludePatterns` 接受一个排除文件路径的 glob 模式列表。
这对于许多你想避免从特定文件和目录自动导入的场景可能更简单，但这并不总是足够的。
例如，如果你使用 `@types/node` 包，同一个文件声明了 `fs` 和 `node:fs`，所以我们不能使用 `autoImportExcludePatterns` 来过滤掉其中一个。

新的 `autoImportSpecifierExcludeRegexes` 特定于模块说明符（我们在导入语句中编写的特定字符串），所以我们可以编写一个模式来排除 `fs` 或 `node:fs` 而不排除另一个。
更重要的是，我们可以编写模式来强制自动导入更喜欢不同的说明符样式（例如，更喜欢 `./foo/bar.js` 而不是 `#foo/bar.js`）。

更多信息，请查看此功能的实现。

## 值得注意的行为变化

本节重点介绍一些值得注意的变化，这些变化应在任何升级中被确认和理解。有时它会突出显示弃用、移除和新限制。它还可以包含功能上改进的错误修复，但这些修复也可能通过引入新错误影响现有构建。

### `lib.d.ts`

为 DOM 生成的类型可能会影响你的代码库的类型检查。更多信息，请参阅与 DOM 和 `lib.d.ts` 更新相关的此版本 TypeScript 的问题。

### 始终写入 `.tsbuildinfo`

为了启用 `--build` 在依赖项中存在中间错误时继续构建项目，并支持命令行上的 `--noCheck`，TypeScript 现在总是为 `--build` 调用中的任何项目生成 `.tsbuildinfo` 文件。
无论是否实际启用了 `--incremental`，都会发生这种情况。更多信息请参见此处。

### 尊重 `node_modules` 中的文件扩展名和 `package.json`

在 Node.js 实现 ECMAScript 模块支持之前（v12），TypeScript 从来没有一个好的方法来知道它在 `node_modules` 中找到的 `.d.ts` 文件是代表作为 CommonJS 还是 ECMAScript 模块编写的 JavaScript 文件。当绝大多数 npm 只是 CommonJS 时，这并没有引起太多问题——如果有疑问，TypeScript 可以假设一切都像 CommonJS 一样运行。不幸的是，如果这种假设是错误的，它可能会允许不安全的导入：

```typescript
// node_modules/dep/index.d.ts
export declare function doSomething(): void;

// index.ts
// 如果 "dep" 是 CommonJS 模块，这是可以的，但如果
// 它是 ECMAScript 模块，则会失败——即使在捆绑器中！
import dep from 'dep';
dep.doSomething();
```

在实践中，这种情况并不常见。但在 Node.js 开始支持 ECMAScript 模块以来的几年里，npm 上 ESM 的份额有所增长。幸运的是，Node.js 还引入了一种机制，可以帮助 TypeScript 确定文件是 ECMAScript 模块还是 CommonJS 模块：`.mjs` 和 `.cjs` 文件扩展名以及 `package.json` 中的 `"type"` 字段。TypeScript 4.7 添加了对理解这些指示器的支持，以及编写 `.mts` 和 `.cts` 文件的支持；然而，TypeScript 只会在 `--module node16` 和 `--module nodenext` 下读取这些指示器，因此对于使用 `--module esnext` 和 `--moduleResolution bundler` 的人来说，上面的不安全导入仍然是一个问题。

为了解决这个问题，TypeScript 5.6 收集模块格式信息，并使用它来解析所有模块模式（除了 `amd`、`umd` 和 `system`）中的歧义。格式特定的文件扩展名（`.mts` 和 `.cts`）在任何地方都被尊重，并且 `package.json` 中的 `"type"` 字段在 `node_modules` 依赖项中被查阅，无论模块设置如何。以前，技术上可以将 CommonJS 输出生成到 `.mjs` 文件中，反之亦然：

```typescript
// main.mts
export default 'oops';

// $ tsc --module commonjs main.mts
// main.mjs
Object.defineProperty(exports, '__esModule', { value: true });
exports.default = 'oops';
```

现在，`.mts` 文件永远不会生成 CommonJS 输出，而 `.cts` 文件永远不会生成 ESM 输出。

请注意，这种行为的大部分在 TypeScript 5.5 的预发布版本中提供（实现细节在此），但在 5.6 中，此行为仅扩展到 `node_modules` 中的文件。

更多详细信息，请参阅此更改。

## 计算属性的正确覆盖检查

以前，标记为 `override` 的计算属性没有正确检查基类成员的存在。同样，如果你使用 `noImplicitOverride`，如果你忘记向计算属性添加 `override` 修饰符，你不会收到错误。

TypeScript 5.6 现在正确检查这两种情况下的计算属性。

```typescript
const foo = Symbol('foo');
const bar = Symbol('bar');

class Base {
  [bar]() {}
}

class Derived extends Base {
  override [foo]() {}
  //           ~~~~~
  // 错误：此成员不能有 'override' 修饰符，因为它未在基类 'Base' 中声明。

  [bar]() {}
  //  ~~~~~
  // 在 noImplicitOverride 下的错误：此成员必须具有 'override' 修饰符，因为它覆盖了基类 'Base' 中的成员。
}
```

此修复由 Oleksandr Tarasiuk 在此 Pull Request 中贡献。
