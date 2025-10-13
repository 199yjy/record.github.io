Webpack的编译过程就像一条精心设计的流水线，把我们的源代码一步步转化为最终可运行的代码。这个过程主要分为三个关键阶段：
1. 初始化阶段
2. 编译阶段
3. 输出阶段

## 初始化阶段
在初始化阶段，webpack会将`cli参数`、`配置文件`、`默认配置`进行融合，形成一个最终的配置对象。
初始化阶段主要用于产生一个最终的配置，为编译阶段做必要的准备，产生完整的配置基础。
- `cli参数`、配置文件与默认配置融合
	- cil参数：运行webpack命令时附加的参数（npx webpack --mode=development)
	- `配置文件`：
	- 默认文件：未执行时的默认值
- 融合机制：`cli参数` > `配置文件` > `默认配置`，最终生成完整的配置对象供后续阶段使用
- 特殊说明：配置文件必须使用commonjs规范，配置融合过程会实际执行配置文件中的代码。webpack的打包和编译需要执行webpack.config.js文件，执行过程是在node环境中运行的，node环境只能使用common JS模块化标准，不能识别ES6模块化标准；因此配置文件的导出必须使用上述的导出形式； 而除此之外，其他目录下的js模块可以使用common JS和ES6两种模块化标准是因为，在webpack打包和编译的过程中，他们不参与执行，只是作为字符串代码被编译和分析；
- 第三方库yargs的作用
	- 核心功能：专门用于处理命令行参数和配置融合
	- 实现特点：
		- 作为 webpack 的依赖库被引入（位于 node_modules 中）
		- 开发者通常不需要直接使用，除非进行底层开发
	- 工作流程
		- 解析 CLI 输入的原始参数
		- 合并不同来源的配置项
		- 处理配置冲突和默认值
## 编译
### 创建 `chunk`
`chunk` 是 webpack在内部构建过程中的一个概念，译为块，它表示通过入口模块找到的所有依赖模块的统称。
根据入口模块（默认为 ./src/index.js）创建一个 chunk，称为 main chunk。
每一个 chunk 是有名字的，有很多个入口文件，chunk 会有很多个，默认情况下有一个 chunk。
每个 chunk 都有至少两个属性：
	- name：默认main
	- id：唯一编号，开发环境和 name 相同，生产环境是一个数字，从 0 开始

### 依赖分析
1. 构建所有依赖模块
	- 确认入口模块
	```js
	module.exports = {
		entry: './src/index.js', // 这里是起点
	};
	```
	- 将代码转换为 AST：使用 @babel/parser 等工具将源代码字符串转换成一个抽象语法树。
	- 遍历 AST，收集依赖：使用 @babel/traverse 等工具遍历 AST，寻找所有的模块导入语句（import、require）。每找到一个，就把它添加到当前模块的依赖列表中。
	- 递归分析所有依赖模块
	```js
	// ./src/index.js
	import message from './message.js';
	console.log(message);

	//./src/message.js
	export default "Hello, Webpack!";
	```
	对于message.js，会重复步骤：
	- 读取 message 源码
	- 转换为AST
	- 遍历AST，收集 message 的依赖，直到项目中再也没有未被发现的模块
  
	在分析过程中，webpack会调用配置中对应的Loader对模块的源码进行转换，将非JS模块转化为JS代码，以便webpack处理。
	- 一个 .vue 文件会经过 vue-loader 处理，被拆解成 JS 模块。

    - 一个 .css 文件会经过 css-loader 处理，可能变成一个将样式插入 DOM 的 JS 模块。

    - 一个 .ts 文件会经过 ts-loader 处理，被编译成纯 JavaScript


2.  分组形成 Chunk：
Chunk 形成规则：
    - 每个入口模块会形成一个独立的 chunk
    - 动态导入 (import()) 创建新的异步 Chunk
    - SplitChunksPlugin 根据配置分割 Chunk
```js
// 形成的 Chunk 结构
const mainChunk = {
  name: 'main',
  entryModule: './src/index.js',
  modules: [
    './src/index.js',
    './src/message.js', 
  ],
  files: ['main.js'] // 最终输出的文件名
};
```

3. 代码生成与优化

Webpack 将每个模块包装成函数

代码优化：
- Tree Shaking：虽然 helper 函数导出了但没使用，可能被标记（但需要配置才能移除）

- Scope Hoisting：将模块尽可能合并到同一个作用域

- Minification：删除空白、缩短变量名等
4. 计算chunk hash
```js
const crypto = require('crypto');

// Webpack 获取 Chunk 的最终内容
const chunkContent = `(function(modules) { ... })([ ... ])`;

// 计算 Hash（简化表示）
const chunkHash = crypto.createHash('md4')
  .update(chunkContent)
  .digest('hex')
  .substring(0, 8); // 取前8位

console.log(chunkHash); // 例如: 'a1b2c3d4'
```
Hash 基于的内容包括：
- 所有模块的转换后代码

- 模块ID和依赖关系

- Webpack 运行时代码

- 模块映射表

5. 生成chunk assets
使用 Hash 生成文件名：
```js
// 根据 output.filename 配置生成最终文件名
const filenameTemplate = '[name].[chunkhash].js';
const finalFilename = filenameTemplate
  .replace('[name]', 'main')
  .replace('[chunkhash]', 'a1b2c3d4');

console.log(finalFilename); // 'main.a1b2c3d4.js'
```
创建 Chunk Asset 对象:
```js
const mainChunkAsset = {
  source: function() {
    return chunkContent; // 阶段4生成的完整代码
  },
  size: function() {
    return chunkContent.length;
  }
};

// Webpack 的 assets 对象
compilation.assets = {
  'main.a1b2c3d4.js': mainChunkAsset
};
```

   
6. 输出文件
```js
const fs = require('fs');
const path = require('path');

// 遍历所有 assets
for (const [filename, asset] of Object.entries(compilation.assets)) {
  const filePath = path.join(outputPath, filename);
  const content = asset.source();
  
  // 确保目录存在
  fs.mkdirSync(path.dirname(filePath), { recursive: true });
  
  // 写入文件
  fs.writeFileSync(filePath, content);
  
  console.log(`输出: ${filename} (${content.length} bytes)`);
}
```
```
dist/
  └── main.a1b2c3d4.js
```
|阶段|主要产出|数据形态|
|--|--|--|
|模块记录|单个模块信息|内存对象|
|依赖图|模块间关系|图数据结构|
|chunk|模块分组|包含多个模块的集合|
|chunk hash|内容标识|字符串哈希值|
|chunk assets|最终文件|文件内容+文件名|

### 产生chunk assets

||Chunk|Chunk assets|
|-----|-----|-----|
|特质|模块的集合|Chunk 经过最终处理（主要是由 Plugin）后，准备写入到输出目录的文件|
|主要内容|模块、模块信息|源信息（文件名、源内容）、最终形态|

## 其他
### chunk hash有什么用

缓存效果：
- 只修改业务代码 → 只有 main.a1b2c3d4.js 变为 main.m3n4o5p6.js
- react.*.js 和 vendors.*.js 的 hash 不变，浏览器继续使用缓存
- 用户只需下载变化了的 main 文件，大大提升加载速度

#### 注意事项：
- 模块标识符问题

早期 Webpack 版本中，即使模块内容没变，模块 ID 的变化也会导致 chunkhash 变化。

```js
// 第一次构建
// moduleA.js (ID: 1)
export const A = 'Module A';

// moduleB.js (ID: 2) 
export const B = 'Module B';

// 生成的Chunk内容大致如下：
/******/ ([
/* 0 */ function(module, exports, __webpack_require__) { /* 入口模块 */ },
/* 1 */ function(module, exports) { /* Module A 的代码 */ },
/* 2 */ function(module, exports) { /* Module B 的代码 */ }
/******/ ]);


// 第二次构建（只是修改了其他不相关的文件）
// moduleA.js (ID: 3 ← ID变了!)
export const A = 'Module A'; // 内容完全一样

// moduleB.js (ID: 4 ← ID也变了!)
export const B = 'Module B'; // 内容完全一样

// 生成的Chunk内容：
/******/ ([
/* 0 */ function(module, exports, __webpack_require__) { /* 入口模块 */ },
/* 3 */ function(module, exports) { /* Module A 的代码 */ }, // ID从1变成3
/* 4 */ function(module, exports) { /* Module B 的代码 */ }  // ID从2变成4
/******/ ]);
```
虽然模块内容完全没变，但由于模块ID从 [1, 2] 变成了 [3, 4]，Chunk的最终内容发生了变化，所以 chunkhash 也会改变。

解决方案：

```js
optimization: {
	moduleIds: 'deterministic', // Webpack 4+
	// 或
	chunkIds: 'deterministic',
}
```
- Webpack 运行时代码
运行时代码变化会影响所有包含它的 Chunk 的 hash。
早期 Webpack 中，运行时代码包含了所有 Chunk 的模块信息表。当你修改业务代码时：
	- 业务模块的 ID 或依赖关系可能发生变化

	- 运行时代码中的模块信息表随之更新

	- 由于运行时代码嵌在 main Chunk 中，它的内容变化导致 hash 变化

可以通过提取 runtime 来解决：
```js
optimization: {
	runtimeChunk: {
	name: 'runtime',
	},
}
```
- Hash 长度
```js
filename: '[name].[chunkhash:8].js' // 只使用前8位，平衡可读性与唯一性
```
