# TypeScript 5.5

## 推断的类型谓词

TypeScript 的控制流分析在跟踪变量类型在代码中的变化时表现得非常出色：

```ts
interface Bird {
  commonName: string;
  scientificName: string;
  sing(): void;
}

// Maps country names -> national bird.
// Not all nations have official birds (looking at you, Canada!)
declare const nationalBirds: Map<string, Bird>;

function makeNationalBirdCall(country: string) {
  const bird = nationalBirds.get(country); // bird has a declared type of Bird | undefined
  if (bird) {
    bird.sing(); // bird has type Bird inside the if statement
  } else {
    // bird has type undefined here.
  }
}
```

通过让你处理 `undefined` 情况，TypeScript 促使你编写更健壮的代码。

在过去，这种类型细化在数组上更难应用。在所有以前的 TypeScript 版本中，这都会是一个错误：

```ts
function makeBirdCalls(countries: string[]) {
  // birds: (Bird | undefined)[]
  const birds = countries
    .map(country => nationalBirds.get(country))
    .filter(bird => bird !== undefined);

  for (const bird of birds) {
    bird.sing(); // error: 'bird' is possibly 'undefined'.
  }
}
```

代码是完全没有问题的：我们已经过滤掉了数组中所有的 `undefined` 值。
但是 TypeScript 却无法跟踪这些变化。

TypeScript 5.5 可以处理这种情况：

```ts
function makeBirdCalls(countries: string[]) {
  // birds: Bird[]
  const birds = countries
    .map(country => nationalBirds.get(country))
    .filter(bird => bird !== undefined);

  for (const bird of birds) {
    bird.sing(); // ok!
  }
}
```

注意 `birds` 变量的更精确类型。

因为 TypeScript 现在能够为 `filter` 函数推断出[类型谓词](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#using-type-predicates)，所以这段代码才能工作。
你可以将代码提出到独立的函数中以便能清晰地看出这些：

```ts
// function isBirdReal(bird: Bird | undefined): bird is Bird
function isBirdReal(bird: Bird | undefined) {
  return bird !== undefined;
}
```

`bird is Bird` 是类型谓词。
它表示，如果函数返回 `true`，那么结果为 `Bird` (如果函数返回 `false`，结果为 `undefined`)。
`Array.prototype.filter` 的类型声明能够识别类型谓词，所以最终的结果是你获得了一个更精确的类型，并且代码通过了类型检查器的验证。

如果以下条件成立，TypeScript 会推断一个函数返回一个类型谓词：

- 函数没有显式的返回类型或类型谓词注解。
- 函数只有一个返回语句，并且没有隐式返回。
- 函数不会改变其参数。
- 函数返回的布尔表达式与参数的类型细化有关。

通常，这种推断方式会如你所预期的那样工作。以下是一些推断类型谓词的更多示例：

```ts
// const isNumber: (x: unknown) => x is number
const isNumber = (x: unknown) => typeof x === 'number';

// const isNonNullish: <T>(x: T) => x is NonNullable<T>
const isNonNullish = <T>(x: T) => x != null;
```

从前，TypeScript 仅会推断出这类函数返回 `boolean`。
但现在会推断出带类型谓词的签名，例如 `x is number` 或 `x is NonNullable<T>`。

类型谓词具有“当且仅当”的语义。
如果函数返回 `x is T`，那就意味着：

1. 如果函数返回 `true`，那么 `x` 的类型为 `T`。
1. 如果函数返回 `false`，那么 `x` 的类型不为 `T`。

如果你期待得到一个类型谓词但却没有，那么有可能违反了第二条规则。
这通常出现在“真值”检查中：

```ts
function getClassroomAverage(
  students: string[],
  allScores: Map<string, number>
) {
  const studentScores = students
    .map(student => allScores.get(student))
    .filter(score => !!score);

  return studentScores.reduce((a, b) => a + b) / studentScores.length;
  //     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  // error: Object is possibly 'undefined'.
}
```

TypeScript 没有为 `score => !!score` 推断出类型谓词，这是有道理的：如果返回 `true`，那么 `score` 是一个数字。但如果返回 `false`，那么 `score` 可能是 `undefined` 或者是数字（特别是 `0`）。这确实是一个漏洞：如果有学生在测试中得了零分，那么过滤掉他们的分数会导致平均分上升。这样一来，少数人会高于平均水平，而更多的人会感到沮丧！

与第一个例子一样，最好明确地过滤掉 `undefined` 值：

```ts
function getClassroomAverage(
  students: string[],
  allScores: Map<string, number>
) {
  const studentScores = students
    .map(student => allScores.get(student))
    .filter(score => score !== undefined);

  return studentScores.reduce((a, b) => a + b) / studentScores.length; // ok!
}
```

当对象类型不存在歧义时，“真实性”检查会为对象类型推断出类型谓词。
请记住，函数必须返回一个布尔值才能成为推断类型谓词的候选者：`x => !!x` 可能会推断出类型谓词，但 `x => x` 绝对不会。

显式类型谓词依然像以前一样工作。TypeScript 不会检查它是否会推断出相同的类型谓词。
显式类型谓词（"is"）并不比类型断言（"as"）更安全。

如果 TypeScript 现在推断出的类型比你期望的更精确，那么这个特性可能会破坏现有的代码。例如：

```ts
// Previously, nums: (number | null)[]
// Now, nums: number[]
const nums = [1, 2, 3, null, 5].filter(x => x !== null);

nums.push(null); // ok in TS 5.4, error in TS 5.5
```

解决方法是使用显式类型注解告诉 TypeScript 你想要的类型：

```ts
const nums: (number | null)[] = [1, 2, 3, null, 5].filter(x => x !== null);
nums.push(null); // ok in all versions
```

更多详情请参考[PR](https://github.com/microsoft/TypeScript/pull/57465) 和 [Dan 的博客](https://effectivetypescript.com/2024/04/16/inferring-a-type-predicate/)。

## 常量索引访问的控制流细化

当 `obj` 和 `key` 是常量时，TypeScript 现在能够细化 obj[key] 形式的表达式。

```ts
function f1(obj: Record<string, unknown>, key: string) {
  if (typeof obj[key] === 'string') {
    // Now okay, previously was error
    obj[key].toUpperCase();
  }
}
```

如上，`obj` 和 `key` 都没有修改过，因此 TypeScript 能够在 `typeof` 检查后将 `obj[key]` 细化为 `string`。
更多详情请参考[PR](https://github.com/microsoft/TypeScript/pull/57847)。

## JSDoc `@import` 标签

如今，在 JavaScript 文件中，如果你只想为类型检查导入某些内容，这显得很繁琐。
JavaScript 开发者不能简单地导入一个名为 `SomeType` 的类型，如果在运行时该类型不存在的话。

```ts
// ./some-module.d.ts
export interface SomeType {
  // ...
}

// ./index.js
import { SomeType } from './some-module'; //  runtime error!

/**
 * @param {SomeType} myValue
 */
function doSomething(myValue) {
  // ...
}
```

`SomeType` 类型在运行时不存在，因此导入会失败。
开发者可以使用命名空间导入来替代。

```ts
import * as someModule from './some-module';

/**
 * @param {someModule.SomeType} myValue
 */
function doSomething(myValue) {
  // ...
}
```

但 `./some-module` 仍是在运行时导入 - 可能不是期望的行为。

为避免此问题，开发者通常需要在 JSDoc 里使用 `import(...)`。

```ts
/**
 * @param {import("./some-module").SomeType} myValue
 */
function doSomething(myValue) {
  // ...
}
```

如果你想在多处重用该类型，你可以使用 `typedef` 来减少重覆。

```ts
/**
 * @typedef {import("./some-module").SomeType} SomeType
 */

/**
 * @param {SomeType} myValue
 */
function doSomething(myValue) {
  // ...
}
```

对于本地使用 `SomeType` 的情况是没问题的，但是出现了很多重覆的导入并显得啰嗦。

因此，TypeScript 现在支持了新的 `@import` 注释标签，它与 ECMAScript 导入语句有相同的语法。

```ts
/** @import { SomeType } from "some-module" */

/**
 * @param {SomeType} myValue
 */
function doSomething(myValue) {
  // ...
}
```

此处，我们使用了命名导入。
我们也可将其写为命名空间导入。

```ts
/** @import * as someModule from "some-module" */

/**
 * @param {someModule.SomeType} myValue
 */
function doSomething(myValue) {
  // ...
}
```

因为它们是 JSDoc 注释，它们完全不影响运行时行为。

更多详情请参考[PR](https://github.com/microsoft/TypeScript/pull/57207)。
感谢 [Oleksandr Tarasiuk](https://github.com/a-tarasyuk) 的贡献。

## 正则表达式语法检查

直到现在，TypeScript 通常会跳过代码中的大多数正则表达式。
这是因为正则表达式在技术上具有可扩展的语法，TypeScript 从未努力将正则表达式编译成早期版本的 JavaScript。
尽管如此，这意味着许多常见问题可能会在正则表达式中被忽略，并且它们要么会在运行时转变为错误，要么会悄悄地失败。

但是现在，TypeScript 对正则表达式进行基本的语法检查了！

```ts
let myRegex = /@robot(\s+(please|immediately)))? do some task/;
//                                            ~
// error!
// Unexpected ')'. Did you mean to escape it with backslash?
```

这是一个简单的例子，但这种检查可以捕捉到许多常见的错误。
事实上，TypeScript 的检查略微超出了语法检查。
例如，TypeScript 现在可以捕捉到不存在的后向引用周围的问题。

```ts
let myRegex = /@typedef \{import\((.+)\)\.([a-zA-Z_]+)\} \3/u;
//                                                        ~
// error!
// This backreference refers to a group that does not exist.
// There are only 2 capturing groups in this regular expression.
```

这同样适用于捕获命名的分组。

```ts
let myRegex =
  /@typedef \{import\((?<importPath>.+)\)\.(?<importedEntity>[a-zA-Z_]+)\} \k<namedImport>/;
//                                                                                        ~~~~~~~~~~~
// error!
// There is no capturing group named 'namedImport' in this regular expression.
```

TypeScript 现在还会检测到当使用某些 RegExp 功能时，这些功能是否比您的 ECMAScript 目标版本更新。
例如，如果我们在 ES5 目标中使用类似上文中的命名捕获组，将会导致错误。

```ts
let myRegex =
  /@typedef \{import\((?<importPath>.+)\)\.(?<importedEntity>[a-zA-Z_]+)\} \k<importedEntity>/;
//                                  ~~~~~~~~~~~~         ~~~~~~~~~~~~~~~~
// error!
// Named capturing groups are only available when targeting 'ES2018' or later.
```

同样适用于某些正则表达式标志。

请注意，TypeScript 的正则表达式支持仅限于正则表达式字面量。
如果您尝试使用字符串字面量调用 `new RegExp`，TypeScript 将不会检查提供的字符串。

更多详情请参考[PR](https://github.com/microsoft/TypeScript/pull/55600)。
感谢 [graphemecluster](https://github.com/graphemecluster/) 的贡献。

## 支持新的 ECMAScript `Set` 方法

TypeScript 5.5 声明了新提议的 [ECMAScript Set](https://github.com/tc39/proposal-set-methods) 类型。

其中一些方法，比如 `union`、`intersection`、`difference` 和 `symmetricDifference`，接受另一个 `Set` 并返回一个新的 `Set` 作为结果。另一些方法，比如 `isSubsetOf`、`isSupersetOf` 和 `isDisjointFrom`，接受另一个 `Set` 并返回一个布尔值。这些方法都不会改变原始的 `Sets`。

示例：

```ts
let fruits = new Set(['apples', 'bananas', 'pears', 'oranges']);
let applesAndBananas = new Set(['apples', 'bananas']);
let applesAndOranges = new Set(['apples', 'oranges']);
let oranges = new Set(['oranges']);
let emptySet = new Set();

////
// union
////

// Set(4) {'apples', 'bananas', 'pears', 'oranges'}
console.log(fruits.union(oranges));

// Set(3) {'apples', 'bananas', 'oranges'}
console.log(applesAndBananas.union(oranges));

////
// intersection
////

// Set(2) {'apples', 'bananas'}
console.log(fruits.intersection(applesAndBananas));

// Set(0) {}
console.log(applesAndBananas.intersection(oranges));

// Set(1) {'apples'}
console.log(applesAndBananas.intersection(applesAndOranges));

////
// difference
////

// Set(3) {'apples', 'bananas', 'pears'}
console.log(fruits.difference(oranges));

// Set(2) {'pears', 'oranges'}
console.log(fruits.difference(applesAndBananas));

// Set(1) {'bananas'}
console.log(applesAndBananas.difference(applesAndOranges));

////
// symmetricDifference
////

// Set(2) {'bananas', 'oranges'}
console.log(applesAndBananas.symmetricDifference(applesAndOranges)); // no apples

////
// isDisjointFrom
////

// true
console.log(applesAndBananas.isDisjointFrom(oranges));

// false
console.log(applesAndBananas.isDisjointFrom(applesAndOranges));

// true
console.log(fruits.isDisjointFrom(emptySet));

// true
console.log(emptySet.isDisjointFrom(emptySet));

////
// isSubsetOf
////

// true
console.log(applesAndBananas.isSubsetOf(fruits));

// false
console.log(fruits.isSubsetOf(applesAndBananas));

// false
console.log(applesAndBananas.isSubsetOf(oranges));

// true
console.log(fruits.isSubsetOf(fruits));

// true
console.log(emptySet.isSubsetOf(fruits));

////
// isSupersetOf
////

// true
console.log(fruits.isSupersetOf(applesAndBananas));

// false
console.log(applesAndBananas.isSupersetOf(fruits));

// false
console.log(applesAndBananas.isSupersetOf(oranges));

// true
console.log(fruits.isSupersetOf(fruits));

// false
console.log(emptySet.isSupersetOf(fruits));
```

感谢 [Kevin Gibbons](https://github.com/bakkot) 的贡献。

## 孤立的声明

声明文件（即 `.d.ts` 文件）用于向 TypeScript 描述现有库和模块的结构。
这种轻量级描述包括库的类型签名，但不包含实现细节，例如函数体。
它们被发布出来，以便 TypeScript 在检查你对库的使用时无需分析库本身。
虽然可以手写声明文件，但如果你正在编写带类型的代码，使用 `--declaration` 让 TypeScript 从源文件自动生成它们会更安全、更简单。

TypeScript 编译器及其 API 一直以来都负责生成声明文件；
然而，有些情况下你可能希望使用其他工具，或者传统的构建流程无法满足需求。

### 用例：更快的声明生成工具

想象一下，如果你想创建一个更快的工具来生成声明文件，也许作为发布服务或一个新的打包工具的一部分。
虽然有许多快速工具生态系统可以将 TypeScript 转换为 JavaScript，但将 TypeScript 转换为声明文件的工具并不那么丰富。
原因在于 TypeScript 的推断能力允许我们编写代码而不需要显式声明类型，这意味着声明生成可能会变得复杂。

让我们考虑一个简单的例子，一个将两个导入变量相加的函数。

```ts
// util.ts
export let one = '1';
export let two = '2';

// add.ts
import { one, two } from './util';
export function add() {
  return one + two;
}
```

即使我们只想生成 `add.d.ts` 这个声明文件，TypeScript 也需要深入到另一个导入的文件（`util.ts`），推断出 `one` 和 `two` 的类型为字符串，然后计算出两个字符串上的 `+` 运算符将导致一个字符串返回类型。

```ts
// add.d.ts
export declare function add(): string;
```

虽然这种推断对开发人员体验很重要，但这意味着想要生成声明文件的工具需要复制类型检查器的部分内容，包括推断和解析模块规范器以跟踪导入。

### 用例：并行的声明生成和类型检查

想象一下，如果你拥有一个包含许多项目的单体库（monorepo）和一个渴望帮助你更快检查代码的多核 CPU。如果我们能够通过在每个核心上运行不同项目来同时检查所有这些项目，那不是太棒了吗？

不幸的是，我们不能自由地将所有工作并行处理。
原因是我们必须按照依赖顺序构建这些项目，因为每个项目都在对其依赖项的声明文件进行检查。
因此，我们必须首先构建依赖项以生成声明文件。
TypeScript 的项目引用功能也是以"拓扑"依赖顺序构建项目集合。

举个例子，如果我们有两个名为 `backend` 和 `frontend` 的项目，它们都依赖一个名为 `core` 的项目，TypeScript 在构建 `core` 并生成其声明文件之前，无法开始对 `frontend` 或 `backend` 进行类型检查。

![beta-isolated-declarations-deps](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2024/04/5-5-beta-isolated-declarations-deps.png)

在上面的图中，您可以看到我们有一个瓶颈。虽然我们可以并行构建 `frontend` 和 `backend`，但我们需要等待 `core` 完成构建，然后才能开始任何一个项目的构建。

我们该如何改进呢？
如果一个快速工具可以并行生成所有这些 `core` 的声明文件，那么 TypeScript 就可以立即跟进，通过并行检查 `core`、`frontend` 和 `backend`。

### 解决文案：显式的类型

在这两种用例中的共同要求是，我们需要一个跨文件类型检查器来生成声明文件。
这对于工具开发社区来说是一个很大的要求。

作为一个更复杂的例子，如果我们想要以下代码的声明文件…

```ts
import { add } from './add';

const x = add();

export function foo() {
  return x;
}
```

我们需要为 `foo` 生成一个签名。
这需要查看 `foo` 的实现。
`foo` 只是返回 `x`，所以获取 `x` 的类型需要查看 `add` 的实现。
但这可能需要查看 `add` 的依赖项的实现，依此类推。
我们在这里看到的是，生成声明文件需要大量逻辑来确定可能甚至不在当前文件中的不同位置的类型。

不过，对于寻求快速迭代时间和完全并行构建的开发人员来说，还有另一种思考这个问题的方式。
声明文件仅需要模块的公共 API 类型，换句话说，导出内容的类型。
如果开发人员愿意显式编写导出内容的类型，工具就可以生成声明文件，而无需查看模块的实现 - 也无需重新实现完整的类型检查器。

这就是新的 `--isolatedDeclarations` 选项发挥作用的地方。
`--isolatedDeclarations` 在模块无法在没有类型检查器的情况下被可靠转换时报告错误。
简而言之，如果您有一个没有足够注释其导出的文件，TypeScript 将报告错误。

这意味着在上面的例子中，我们将看到类似以下的错误：

```ts
export function foo() {
  //              ~~~
  // error! Function must have an explicit
  // return type annotation with --isolatedDeclarations.
  return x;
}
```

### 为什么错误是期望的？

因为这意味着 TypeScript 能够

1. 提前告知其它工具在生成声明文件时是否会有问题
1. 提供快速修复功能帮助添加缺失的类型注解

然而，这种模式并不要求在所有地方都进行注解。
对于局部变量，可以忽略这些注解，因为它们不会影响公共 API。
例如，以下代码不会产生错误：

```ts
import { add } from './add';

const x = add('1', '2'); // no error on 'x', it's not exported.

export function foo(): string {
  return x;
}
```

在某些表达式中，计算类型是“微不足道的”。

```ts
// No error on 'x'.
// It's trivial to calculate the type is 'number'
export let x = 10;

// No error on 'y'.
// We can get the type from the return expression.
export function y() {
  return 20;
}

// No error on 'z'.
// The type assertion makes it clear what the type is.
export function z() {
  return Math.max(x, y()) as number;
}
```

### 使用 `isolatedDeclarations`

`isolatedDeclarations` 要求设置 `declaration` 或 `composite` 标志之一。

请注意，`isolatedDeclarations` 不会改变 TypeScript 的输出方式，只会改变它报告错误的方式。
重要的是，与 `isolatedModules` 类似，启用 TypeScript 中的该功能不会立即带来本文讨论的潜在好处。
因此，请耐心等待，并期待这一领域的未来发展。
考虑到工具作者的需求，我们还应该认识到，如今，并非所有 TypeScript 的声明输出都能轻松地由其他希望使用它作为指南的工具复制。
这是我们正在积极致力于改进的事项。

此外，独立声明仍然是一个新功能，我们正在积极努力改进体验。
一些情景，比如在类和对象字面量中使用计算属性声明，尚不受 `isolatedDeclarations` 支持。
请留意这方面的进展，并随时提供反馈。

我们还认为值得指出的是，应该基于具体情况逐案采用 `isolatedDeclarations`。
在使用 `isolatedDeclarations` 时可能会失去一些开发人员友好性，因此，如果您的设置没有利用前面提到的两种情景，这可能不是正确的选择。
对于其他人来说，`isolatedDeclarations` 的工作已经揭示了许多优化和机会，可以解锁不同的并行构建策略。
同时，如果您愿意做出权衡，我们相信随着外部工具变得更广泛可用，`isolatedDeclarations` 可以成为加快构建流程的强大工具。

更多详情请参考[讨论](https://github.com/microsoft/TypeScript/issues/58944)。

### 信用

独立声明的工作是 TypeScript 团队与 Bloomberg 和 Google 内基础设施和工具团队之间长期的合作努力。
像 Google 的 Hana Joo 这样实现了独立声明错误快速修复的个人（更多相关信息即将发布），以及 Ashley Claymore、Jan Kühle、Lisa Velden、Rob Palmer 和 Thomas Chetwin 等人数个月以来一直参与讨论、规范和实施。
但我们特别要提到 Bloomberg 的 Titian Cernicova-Dragomir 提供的大量工作。
Titian 在推动独立声明实现方面发挥了关键作用，并在之前的多年里一直是 TypeScript 项目的贡献者。

更多详情请参考 [PR](https://github.com/microsoft/TypeScript/pull/58201)。

## 配置文件中的 `${configDir}` 模版变量

在许多代码库中都会重用某个 `tsconfig.json` 作为其它配置文件的“基础”。
这是通过在 `tsconfig.json` 文件中使用 `extends` 字段实现的。

```ts
{
    "extends": "../../tsconfig.base.json",
    "compilerOptions": {
        "outDir": "./dist"
    }
}
```

其中一个问题是，`tsconfig.json` 文件中的所有路径都是相对于文件本身的位置。
这意味着如果您有一个被多个项目使用的共享 `tsconfig.base.json` 文件，那么派生项目中的相对路径通常不会有用。
例如，请想象以下 `tsconfig.base.json`：

```ts
{
    "compilerOptions": {
        "typeRoots": [
            "./node_modules/@types"
            "./custom-types"
        ],
        "outDir": "dist"
    }
}
```

如果作者的意图是每个继承此文件的 `tsconfig.json` 都应：

1. 输出到相对于派生 `tsconfig.json` 的 `dist` 目录，并且
2. 有一个相对于派生 `tsconfig.json` 的 `custom-types` 目录，

那么这样做是行不通的。
`typeRoots` 路径将是相对于共享 `tsconfig.base.json` 文件的位置，而不是继承它的项目。
每个继承此共享文件的项目都需要声明自己的 `outDir` 和 `typeRoots`，并且内容相同。
这可能会让人沮丧，并且在项目之间保持同步可能会很困难。
虽然上面的示例使用了 `typeRoots`，但这对于路径和其他选项来说是一个常见问题。

为了解决这个问题，TypeScript 5.5 引入了一个新的模板变量 `${configDir}`。
当在 `tsconfig.json` 或 `jsconfig.json` 文件的某些路径字段中写入 `${configDir}` 时，此变量将在给定编译中替换为配置文件的所在目录。
这意味着上述 `tsconfig.base.json` 可以重写为：

```ts
{
    "compilerOptions": {
        "typeRoots": [
            "${configDir}/node_modules/@types"
            "${configDir}/custom-types"
        ],
        "outDir": "${configDir}/dist"
    }
}
```

现在，当一个项目继承此文件时，路径将相对于派生的 `tsconfig.json`，而不是共享的 `tsconfig.base.json` 文件。
这使得在项目之间共享配置文件变得更加容易，并确保配置文件更具可移植性。

如果您打算使一个 `tsconfig.json` 文件可继承，请考虑是否应该用 `${configDir}` 替代 `./`。

更多详情请参考 [设计](https://github.com/microsoft/TypeScript/issues/57485) 和 [PR](https://github.com/microsoft/TypeScript/pull/58042)。

## 参考 package.json 中的依赖项来生成声明文件

之前，TypeScript 可能经常抛出如下错误：

```txt
The inferred type of "X" cannot be named without a reference to "Y". This is likely not portable. A type annotation is necessary.
```

这通常是由于 TypeScript 的声明文件生成在从未在程序中显式导入的文件内容中发现自身。
如果路径最终变成相对路径，生成对这样的文件的导入可能存在风险。
然而，在 `package.json` 的依赖项（或 `peerDependencies` 和 `optionalDependencies`）中具有明确依赖关系的代码库中，在某些解析模式下生成这样的导入应该是安全的。
因此，在 TypeScript 5.5 中，当出现这种情况时，我们更加宽松，许多此类错误应该消失。

更多详情请参考 [PR](https://github.com/microsoft/TypeScript/issues/42873)。

## 编辑器和监视模式可靠性改进

TypeScript 已经添加了一些新功能或修复了现有逻辑，使得 `--watch` 模式和 TypeScript 的编辑器集成感觉更加可靠。
这希望能够转化为更少的 TSServer 或编辑器重新启动。

## 正确刷新配置文件中的编辑器错误

TypeScript 可以为 `tsconfig.json` 文件生成错误；
然而，这些错误实际上是在加载项目时生成的，编辑器通常不会直接请求针对 `tsconfig.json` 文件的这些错误。
虽然这听起来像一个技术细节，但这意味着当在 `tsconfig.json` 中修复了所有错误时，TypeScript 不会发出新的空错误集，用户将继续看到过时的错误，除非他们重新加载编辑器。

TypeScript 5.5 现在有意发出一个事件来清除这些错误。
更多详情请参考 [PR](https://github.com/microsoft/TypeScript/pull/58120)。

## 更好地处理删除操作后紧接着的写操作

一些工具选择删除文件而不是覆盖它们，然后从头开始创建新文件。
例如，在运行 `npm ci` 时就是这种情况。

尽管这对于那些工具可能是高效的，但对于 TypeScript 的编辑器场景可能会有问题，在这种情况下，删除一个被监视的文件可能会使其及其所有传递依赖项被丢弃。
快速连续删除和创建文件可能导致 TypeScript 拆除整个项目，然后从头开始重新构建。

TypeScript 5.5 现在采用了更加细致的方法，保留已删除项目的部分内容，直到它捕捉到新的创建事件。
这应该使像 `npm ci` 这样的操作与 TypeScript 协同工作更加顺畅。

更多详情请参考 [PR](https://github.com/microsoft/TypeScript/pull/57492)。

## 符号链接在解析失败时会被跟踪

当 TypeScript 无法解析一个模块时，它仍然需要监视所有失败的查找路径，以防该模块在之后被添加。
之前，这种情况不会发生在符号链接的目录中，这在 monorepo 场景中可能导致可靠性问题，例如当一个项目中发生构建但另一个项目中未检测到时。
这一问题应在 TypeScript 5.5 中得到修复，这意味着你不需要那么频繁地重启编辑器。

更多详情请参考 [PR](https://github.com/microsoft/TypeScript/pull/58139)。

## 项目引用有助于自动导入

在项目引用设置中，自动导入不再需要至少一个对依赖项目的显式导入。
相反，自动导入补全应适用于你在 `tsconfig.json` 的 `references` 字段中列出的任何内容。

更多详情请参考 [PR](https://github.com/microsoft/TypeScript/pull/55955)。
