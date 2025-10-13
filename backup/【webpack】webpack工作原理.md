Webpack的编译过程就像一条精心设计的流水线，把我们的源代码一步步转化为最终可运行的代码。这个过程主要分为三个关键阶段：
1. 初始化阶段
2. 编译阶段
3. 输出阶段

### 初始化
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
### 编译
- 创建 `chunk`
`chunk` 是 webpack在内部构建过程中的一个概念，译为块，它表示通过入口模块找到的所有依赖模块的统称。
根据入口模块（默认为 ./src/index.js）创建一个 chunk，称为 main chunk。
每一个 chunk 是有名字的，有很多个入口文件，chunk 会有很多个，默认情况下有一个 chunk。
每个 chunk 都有至少两个属性：
	- name：默认main
	- id：唯一编号，开发环境和 name 相同，生产环境是一个数字，从 0 开始

- 依赖分析
	- 构建所有依赖模块
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


		- 生成模块列表：Webpack 已经收集了所有模块并完成了转换。它需要为这些模块创建一个“花名册”，以便在浏览器中能够准确地找到和执行它们。这个花名册就是我们所说的模块列表，它通常体现为一个对象或一个数组。
		这个列表中的每个条目都包含两个关键信息：
			- id
			- 模块转换后的代码：经过 Loader 处理、并且被 Webpack 封装在一个函数中的可执行代码

		```js
		// 这就是最终生成的一个简化到极致的模块列表结构示例（打包产物的一部分）
		var __webpack_modules__ = ({
		  // 模块 ID: 函数
		  "./src/index.js": (function(module, __webpack_exports__, __webpack_require__) {
		    // 这是转换后的 index.js 的代码
		    // Webpack 将 import 语句替换成了 __webpack_require__ 函数调用
		    const message = __webpack_require__("./src/message.js");
		    console.log(message);
		  }),
		  
		  "./src/message.js": (function(module, __webpack_exports__, __webpack_require__) {
		    // 这是转换后的 message.js 的代码
		    module.exports = "Hello, Webpack!";
		  })
		});
		```


	- 封装与输出，生成最终打包文件
	Webpack 不会直接输出上面的模块列表。它会将这个列表嵌入到一个自执行函数 中，这个函数被称为 Webpack Bootstrap（启动函数）。这个启动函数包含了 __webpack_require__ 等核心方法的实现，用于在浏览器中按需加载和执行模块。
	```js
	//最终打包产物（bundle.js）的简化结构
	(function(modules) { // webpackBootstrap
		  // 1. 初始化模块缓存（已加载的模块会在这里缓存）
		  var installedModules = {};
		  
		  // 2. 实现 __webpack_require__ 函数（这是核心！）
		  function __webpack_require__(moduleId) {
		    // 检查缓存
		    if(installedModules[moduleId]) {
		      return installedModules[moduleId].exports;
		    }
		    // 创建新模块并放入缓存
		    var module = installedModules[moduleId] = {
		      exports: {}
		    };
		    // 从模块列表中找到对应的函数并执行它
		    // 这就是“模块转换后的代码”被执行的地方！
		    modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
		    // 返回模块的导出内容
		    return module.exports;
		  }
		  
		  // 3. 从入口模块开始，启动整个应用程序
		  return __webpack_require__("./src/index.js");
		})
		// 将我们步骤4中生成的“模块列表”作为参数传入
		({
		  "./src/index.js": (function(module, exports, __webpack_require__) {
		    eval("...转换后的 index.js 代码...");
		  }),
		  "./src/message.js": (function(module, exports) {
		    eval("...转换后的 message.js 代码...");
		  })
	});
	```
- 计算chunk hash基于chunk的最终内容
在所有代码转换、优化完成后，在生成最终输出文件之前，基于 Chunk 的最终内容（包括模块代码、运行时代码、模板等）

	```js
	// webpack.config.js
	module.exports = {
	  output: {
	    filename: '[name].[chunkhash:8].js',
	    chunkFilename: '[name].[chunkhash:8].chunk.js',
	  },
	  plugins: [
	    new MiniCssExtractPlugin({
	      filename: '[name].[contenthash:8].css',
	      chunkFilename: '[name].[contenthash:8].chunk.css',
	    }),
	  ],
	  optimization: {
	    splitChunks: {
	      chunks: 'all',
	      cacheGroups: {
	        react: {
	          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
	          name: 'react',
	          priority: 20,
	        },
	        vendors: {
	          test: /[\\/]node_modules[\\/]/,
	          name: 'vendors',
	          priority: 10,
	        },
	      },
	    },
	  },
	};
	```
	输出结果：
	```js
	main.a1b2c3d4.js        // 业务代码 - 频繁变动
	react.e5f6g7h8.js       // React 相关 - 较少变动  
	vendors.i9j0k1l2.js     // 其他第三方库 - 较少变动
	```

	缓存效果：
	- 只修改业务代码 → 只有 main.a1b2c3d4.js 变为 main.m3n4o5p6.js
	- react.*.js 和 vendors.*.js 的 hash 不变，浏览器继续使用缓存
	- 用户只需下载变化了的 main 文件，大大提升加载速度

	注意事项：
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

- 产生chunk assets

||Chunk|Chunk assets|
|-----|-----|-----|
|特质|模块的集合|Chunk 经过最终处理（主要是由 Plugin）后，准备写入到输出目录的文件|
|主要内容|模块、模块信息|源信息（文件名、源内容）、最终形态|

### 输出
webpack 利用 node 中的 fs 模块（文件处理模块），根据编译产生的总的 assets，生成相应的文件。
- 输出过程：称为 emit 阶段
- 实现方式：使用 node.js 的 fs 模块
- 输出内容：将资源列表中的每个文件写入磁盘
- 输出结果：生成最终的打包文件