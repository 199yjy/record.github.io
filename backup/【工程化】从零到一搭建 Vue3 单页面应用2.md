## 实现前端自动化部署

实现思路：在`git push`命令设置一个钩子函数，即`/.husky/pre-push`，获取当前分支，依据当前分支执行对应的`npm 脚本`，先进行项目打包构建，再用`ftp-deploy`库推送打包后的文件到指定的服务器，最后才将项目上传到远程仓库

1. 利用`ftp-deploy`库，实现上传函数

   - `/deploy/index.ts`——上传函数

     ```ts
     const FtpDeploy = require("ftp-deploy");
     const ftpDeploy = new FtpDeploy();
     const DeployConfig = require("./config.ts")
     
     const config = DeployConfig.getFtpDeployConfig()
     
     ftpDeploy
       .deploy(config)
       .then((res) => console.log("finished:", res))
       .catch((err) => console.log(err));
     ```

   - `/deploy/config.ts`——上传配置

     ```ts
     const deployConfig = require("./serverConfig.ts")
     
     const getNODE_ENV = function() {
       const script = process.env.npm_lifecycle_script;
       const reg = new RegExp('NODE_ENV=([a-z_\\d]+)');
       const result = reg.exec(script);
       return result[1]
     }
     
     module.exports.getFtpDeployConfig = function getFtpDeployConfig() {
       return {
         user: deployConfig.user,                    // 服务器登录账号
         password: deployConfig.password,            // 服务器密码
         host: deployConfig.host,                    // 服务器地址
         port: deployConfig.port.get(getNODE_ENV()), // ftp的服务器端口
         localRoot: deployConfig.localRoot,          // 上传的文件
         remoteRoot: "",                             // 远程服务器文件存储路径
         include: [".*"],                            // 这将上传除了点文件之外的所有文件
         // 排除sourcemaps和node_modules中的所有文件
         exclude: [
             "dist/**/*.map",
             "node_modules/**",
             "node_modules/**/.*",
             ".git/**",
         ],
         deleteRemote: false,                         // 如果为true，则在上传前删除目的地的所有现有文件
         forcePasv: true,                             // 主动模式/被动模式
         sftp: false,                                 // 使用 sftp协议 或 ftp协议
       }
     }
     ```

   - `/deploy/serverConfig.ts`——个人服务器配置，不上传到远程仓库，保障个人服务器安全

     ```ts
     module.exports = {
       user: "user",
       password: "password",
       host: "ftp.someserver.com",
       port: new Map([
         ["test", 8080],
         ["production", 80]
       ]),
       localRoot: __dirname + "/dist",
     }
     ```

2. 设置`npm`脚本命令

   - 安装`cross-env`

     ```shell
     npm i cross-env -D
     ```

   - 修改`package.json`

     ```json
     {
       "scripts": {
         "deploy:test": "cross-env NODE_ENV=test run-p build && node ./deploy/index.ts",
         "deploy:prod": "cross-env NODE_ENV=production run-p build && node ./deploy/index.ts",
       },
     }
     ```

3. 实现`git-push`钩子函数——`/.husky/pre-push`

   ```shell
   #!/bin/sh
   . "$(dirname "$0")/_/husky.sh"
   
   # 代码推送前依据当前分支进行打包构建并上传（仅对 master 分支 和 test 分支进行构建）
   function current_branch() {
     branch=""
     testBranch="test"
     masterBranch="master"
     cd $PWD
     if [ -d '.git' ]; then
       output=`sh -c 'git branch --no-color 2> /dev/null' | sed -e '/^[^*]/d' -e 's/* \(.*\)/\1/' -e 's/\//\_/g'`
       if [ "$output" ]; then
         branch="${output}"
       fi
     fi
   
     if [ $branch == $testBranch ]
     then
       echo "自动构建并上传 test 分支代码"
       npm run deploy:test
     elif [ $branch == $masterBranch ]
     then
       echo "自动构建并上传 master 分支代码"
       npm run deploy:prod
     fi
   }
   
   current_branch
   ```

# 框架配置管理

## 项目`TypeScript`配置 `tsconfig.ts`

```js
{
  "compileOnSave": true, //设置保存文件的时候自动编译
  "extends": "@vue/tsconfig/tsconfig.web.json", //引入其他配置文件，继承配置
  "exclude": ["node_modules", "dist"], //指定编译器需要排除的文件或文件夹
  "include": [
    //指定编译需要编译的文件或目录
    "src/**/*.ts",
    "src/**/*.d.ts",
    "src/**/*.vue",
    "types/**/*.d.ts",
    "types/**/*.ts",
    "build/**/*.ts",
    "build/**/*.d.ts",
    "vite.config.*"
  ],
  "compilerOptions": {
    //配置编译选项
    "diagnostics": true, // 打印诊断信息
    "removeComments": true, //移除代码中注释
    "strictNullChecks": true, //开启null、undefined检测
    "baseUrl": ".", // 解析非相对模块的基地址，默认是当前目录
    "target": "ES2015", // 目标语言的版本
    "moduleResolution": "node", // 模块解析策略，ts默认用node的解析策略，即相对的方式导入
    "allowJs": false, // 允许编译器编译JS，JSX文件
    "checkJs": false, // 允许在JS文件中报错，通常与allowJS一起使用
    "sourceMap": true, // 生成目标文件的sourceMap文件
    "strict": true, // 开启所有严格的类型检查
    "noUnusedLocals": true, // 检查只声明、未使用的局部变量(只提示不报错)
    "noUnusedParameters": true, // 检查未使用的函数参数(只提示不报错)
    "esModuleInterop": true, // 允许export=导出，由import from 导入
    "paths": {
      // 路径映射，相对于baseUrl
      "@/*": ["./src/*"],
      "#/*": ["./types/*"],
      "$locale": ["./src/locales/setupLocale.ts"],
      "$store/*": ["./src/store/modules/*"],
    },
    "suppressImplicitAnyIndexErrors": true,
    "jsx": "preserve",
    "types": ["node", "vite/client"]  //默认所有可见的”@types“包会在编译过程中被包含进来。 node_modules/@types文件夹下以及它们子文件夹下的所有									   包都是可见的；如果指定了types，只有被列出来的包才会被包含进来
  }
}
```

## 环境配置文件

`/.env`——环境基本配置

```shell
# port
VITE_PORT = 3001

# spa-title (spa——单页应用程序)
VITE_GLOB_APP_TITLE = 

# spa shortname
VITE_GLOB_APP_SHORT_NAME = 
```

`/.env.development`——开发环境配置

```shell
# public path
VITE_PUBLIC_PATH = /

# 跨域代理，可配置多个
VITE_PROXY = [["basic-api","http://192.168.1.103/PDP/public/index.php/"]]

# Delete console
VITE_DROP_CONSOLE = false

# Basic interface address SPA
VITE_GLOB_API_URL=/basic-api

# Whether to enable multiple languages
VITE_MULTIPLE_LANGUAGES = true
```

`/.env.production`——生产环境配置

```shell
# public path
VITE_PUBLIC_PATH = /

# Delete console
VITE_DROP_CONSOLE = true

# Whether to enable gzip or brotli compression  是否启用gzip或brotli压缩
# Optional: gzip | brotli | none
# If you need multiple forms, you can use `,` to separate  如果你需要多个形式，你可以用'，'来分隔
VITE_BUILD_COMPRESS = 'none'

# Whether to delete origin files when using compress, default false 使用压缩时是否删除源文件，默认为false
VITE_BUILD_COMPRESS_DELETE_ORIGIN_FILE = false

# Basic interface address SPA8
VITE_GLOB_API_URL=/basic-api

# Is it compatible with older browsers 是否兼容旧浏览器
VITE_LEGACY = false

# Whether to enable image compression 是否压缩图片
VITE_USE_IMAGEMIN= true

# Whether to enable multiple languages
VITE_MULTIPLE_LANGUAGES = false
```

`.env.test`——测试环境配置

```shell
NODE_ENV=production

# public path
VITE_PUBLIC_PATH = /

# Delete console
VITE_DROP_CONSOLE = true

# Whether to enable gzip or brotli compression
# Optional: gzip | brotli | none
# If you need multiple forms, you can use `,` to separate
VITE_BUILD_COMPRESS = 'none'

# Whether to delete origin files when using compress, default false
VITE_BUILD_COMPRESS_DELETE_ORIGIN_FILE = false


# Is it compatible with older browsers
VITE_LEGACY = false

# Whether to enable image compression 是否压缩图片
VITE_USE_IMAGEMIN= true

# Whether to enable multiple languages
VITE_MULTIPLE_LANGUAGES = true
```

## `.gitignore`

```
node_modules
dist

# Editor directories and files
.vscode/*
!.vscode/extensions.json
!.vscode/settings.json
./deploy/serverConfig.ts
```

## `Vite.config.ts`代理服务器配置化、插件模块单独管理及设置打包配置

### 代理服务器配置化

- 在`/build/vite/proxy.ts`创建函数`createProxy`，实现代理服务器由环境配置`VITE_PROXY`创建

  ```ts
  import type { ProxyOptions } from "vite";
  
  type ProxyItem = [string, string];
  
  type ProxyList = ProxyItem[];
  
  type ProxyTargetList = Record<string, ProxyOptions>;
  
  const httpsRE = /^http:\/\//;
  
  /**
   * Generate proxy
   * @param list
   */
  export function createProxy(list: ProxyList = []) {
    const ret: ProxyTargetList = {};
    for (const [prefix, target] of list) {
      const isHttps = httpsRE.test(target);
  
      ret["^/" + prefix] = {
        target: target,
        changeOrigin: true,
        ws: true,
        rewrite: (path) => path.replace(new RegExp(`^\/${prefix}`), ""),
        ...(isHttps ? { secure: false } : {}),
      };
    }
    return ret;
  }
  ```

- 修改`Vite.config.ts`

  ```js
  import { fileURLToPath, URL } from "node:url";
  import { resolve } from "node:path";
  import { loadEnv, type ConfigEnv, type UserConfig } from "vite";
  import { wrapperEnv } from "./build/utils";
  /*
    export function wrapperEnv(envConf: Recordable): ViteEnv {
      const ret: any = {};
  
      for (const envName of Object.keys(envConf)) {
        let realName = envConf[envName].replace(/\\n/g, "\n");
        realName = realName === "true" ? true : realName === "false" ? false : realName;
  
        if (envName === "VITE_PORT") {
          realName = Number(realName);
        }
        if (envName === "VITE_PROXY" && realName) {
          try {
            realName = JSON.parse(realName.replace(/'/g, '"'));
          } catch (error) {
            realName = "";
          }
        }
        ret[envName] = realName;
        if (typeof realName === "string") {
          process.env[envName] = realName;
        } else if (typeof realName === "object") {
          process.env[envName] = JSON.stringify(realName);
        }
      }
      return ret;
    }
  */
  import { createProxy } from "./build/vite/proxy";
  
  // https://vitejs.dev/config/
  export default ({ command, mode }: ConfigEnv): UserConfig => {
    const url = import.meta.url;
      
    // process.cwd()方法返回Node.js进程的当前工作目录。
    const root = process.cwd();
  
    // 加载 root 中的 .env 文件。
    const env = loadEnv(mode, root);
  
    // loadEnv读取的布尔类型是一个字符串。这个函数可以转换为布尔类型
    const viteEnv = wrapperEnv(env);
  
    const { VITE_PORT, VITE_PROXY } = viteEnv;
  
    return {
      // 需要用到的插件数组
      plugins: [vue()],
      // 解析
      resolve: {
        alias: {
          "@": fileURLToPath(new URL("./src", import.meta.url)),
        },
      },
      // 开发或生产环境服务的公共基础路径
      base: "./",
  
      // 设置代理服务器
      server: {
        host: true,
        port: VITE_PORT,
        open: true,
        cors: true,
        proxy: createProxy(VITE_PROXY),
      },
    };
  };
  ```

### 插件模块单独管理

- 在`/build/vite/plugin/index.ts`创建函数`createVitePlugin`，实现`Vite`插件模块`plugin`单独管理

  ```ts
  import type { PluginOption } from "vite";
  import vue from "@vitejs/plugin-vue";
  import vueJsx from "@vitejs/plugin-vue-jsx";
  import legacy from "@vitejs/plugin-legacy";
  import setting from "../../../src/settings/projectSetting";
  import { configElementPlugin } from "./element";
  /*
    import Components from "unplugin-vue-components/vite";
    import { ElementPlusResolver } from "unplugin-vue-components/resolvers";
  
    export function configElementPlugin() {
      const elementPlugin = Components({
        dts: false,
        resolvers: [ElementPlusResolver()],
      });
      return elementPlugin;
    }
  */
  import { configAutoImportPlugin } from "./autoImport";
  /*
    import AutoImport from "unplugin-auto-import/vite";
    import { ElementPlusResolver } from "unplugin-vue-components/resolvers";
  
    export function configAutoImportPlugin() {
      const autoImportPlugin = AutoImport({
        dts: false,
        resolvers: [ElementPlusResolver()],
      });
      return autoImportPlugin;
    }
  */
  import { configSvgIconsPlugin } from "./svgSprite";
  /*
    import { createSvgIconsPlugin } from "vite-plugin-svg-icons";
    import path from "path";
  
    export function configSvgIconsPlugin(isBuild: boolean) {
      const svgIconPlugin = createSvgIconsPlugin({
        iconDirs: [path.resolve(process.cwd(), "src/assets/icons")],
        svgoOptions: isBuild,
        // default
        symbolId: "icon-[dir]-[name]",
      });
      return svgIconPlugin;
    }
  */
  import { configImageminPlugin } from "./imagemin";
  /*
    import viteImagemin from "vite-plugin-imagemin";
  
    export function configImageminPlugin() {
      const plugin = viteImagemin({
        gifsicle: {
          optimizationLevel: 7,
          interlaced: false,
        },
        optipng: {
          optimizationLevel: 7,
        },
        mozjpeg: {
          quality: 20,
        },
        pngquant: {
          quality: [0.8, 0.9],
          speed: 4,
        },
        svgo: {
          plugins: [
            {
              name: "removeViewBox",
            },
            {
              name: "removeEmptyAttrs",
              active: false,
            },
          ],
        },
      });
      return plugin;
    }
  */
  import { configCompressPlugin } from "./compress";
  /*
    import type { Plugin } from "vite";
    import compressPlugin from "vite-plugin-compression";
  
    export function configCompressPlugin(
      compress: "gzip" | "brotli" | "none",
      deleteOriginFile = false,
    ): Plugin | Plugin[] {
      const compressList = compress.split(",");
  
      const plugins: Plugin[] = [];
  
      if (compressList.includes("gzip")) {
        plugins.push(
          compressPlugin({
            ext: ".gz",
            deleteOriginFile,
          }),
        );
      }
  
      if (compressList.includes("brotli")) {
        plugins.push(
          compressPlugin({
            ext: ".br",
            algorithm: "brotliCompress",
            deleteOriginFile,
          }),
        );
      }
      return plugins;
    }
  */
  import { configHtmlPlugin } from "./html";
  /*
    import type { PluginOption } from "vite";
    import { createHtmlPlugin } from "vite-plugin-html";
    import pkg from "../../../package.json";
    const GLOB_CONFIG_FILE_NAME = require("../../constant.ts").GLOB_CONFIG_FILE_NAME;
  
    export function configHtmlPlugin(env: ViteEnv, isBuild: boolean) {
      const { VITE_GLOB_APP_TITLE, VITE_PUBLIC_PATH } = env;
  
      const path = VITE_PUBLIC_PATH.endsWith("/") ? VITE_PUBLIC_PATH : `${VITE_PUBLIC_PATH}/`;
  
      const getAppConfigSrc = () => {
        return `${path || "/"}${GLOB_CONFIG_FILE_NAME}?v=${pkg.version}-${new Date().getTime()}`;
      };
  
      const htmlPlugin: PluginOption[] = createHtmlPlugin({
        minify: isBuild,
        inject: {
          // Inject data into ejs template
          data: {
            title: VITE_GLOB_APP_TITLE,
          },
          // Embed the generated app.config.js file
          tags: isBuild
            ? [
                {
                  tag: "script",
                  attrs: {
                    src: getAppConfigSrc(),
                  },
                },
              ]
            : [],
        },
      });
      return htmlPlugin;
    }
  */
  
  export function createVitePlugin(viteEnv: ViteEnv, isBuild: boolean) {
    const {
      VITE_LEGACY,
      VITE_USE_IMAGEMIN,
      VITE_BUILD_COMPRESS,
      VITE_BUILD_COMPRESS_DELETE_ORIGIN_FILE,
    } = viteEnv;
  
    const vitePlugins: (PluginOption | PluginOption[])[] = [
      // have to
      vue(),
      // have to
      vueJsx(),
    ];
  
    // @vitejs/plugin-legacy 兼容旧浏览器
    VITE_LEGACY && vitePlugins.push(legacy());
  
    // unplugin-vue-components 按需自动引入element-plus
    setting.loadOnDemandEl && vitePlugins.push(configElementPlugin());
    setting.loadOnDemandEl && vitePlugins.push(configAutoImportPlugin());
  
    // vite-plugin-svg-icons
    vitePlugins.push(configSvgIconsPlugin(isBuild));
  
    // vite-plugin-html
    vitePlugins.push(configHtmlPlugin(viteEnv, isBuild));
  
    if (isBuild) {
      // vite-plugin-imagemin 图片压缩
      VITE_USE_IMAGEMIN && vitePlugins.push(configImageminPlugin());
  
      // rollup-plugin-gzip 文件压缩
      vitePlugins.push(
        configCompressPlugin(VITE_BUILD_COMPRESS, VITE_BUILD_COMPRESS_DELETE_ORIGIN_FILE),
      );
    }
  
    return vitePlugins;
  }
  ```

- 修改`Vite.config.ts`

  ```ts
  import { fileURLToPath, URL } from "node:url";
  import { resolve } from "node:path";
  import { loadEnv, type ConfigEnv, type UserConfig } from "vite";
  import { wrapperEnv } from "./build/utils";
  import { createVitePlugin } from "./build/vite/plugin";
  
  // https://vitejs.dev/config/
  export default ({ command, mode }: ConfigEnv): UserConfig => {
    const url = import.meta.url;
  
    // process.cwd()方法返回Node.js进程的当前工作目录。
    const root = process.cwd();
  
    // 加载 root 中的 .env 文件。
    const env = loadEnv(mode, root);
  
    // loadEnv读取的布尔类型是一个字符串。这个函数可以转换为布尔类型
    const viteEnv = wrapperEnv(env);
  
    const isBuild = command === "build";
  
    const { VITE_PORT, VITE_PUBLIC_PATH, VITE_PROXY, VITE_DROP_CONSOLE } = viteEnv;
  
    return {
      base: VITE_PUBLIC_PATH,
  
      root,
  
      plugins: createVitePlugin(viteEnv, isBuild),
  
      resolve: {
        alias: {
          "@": fileURLToPath(new URL("./src", url)),
        },
      },
    };
  };
  ```

### 设置打包配置

`Vite.config.ts`

```ts
import { fileURLToPath, URL } from "node:url";
import { resolve } from "node:path";
import { loadEnv, type ConfigEnv, type UserConfig } from "vite";
import { buildAssetsFile, buildChunkFile } from "./build/utils";
/*
  import type { PreRenderedAsset, PreRenderedChunk } from "rollup";
  const ASSETS_DIR = require("./constant.ts").ASSETS_DIR;
  // contant.ts
  module.exports = {
 	OUTPUT_DIR: "dist",
    ASSETS_DIR: "assets",
  }
  
  export function buildAssetsFile(chunkInfo: PreRenderedAsset) {
    if (chunkInfo.name?.match(/\.(png|svg|jpg|jpeg|gif)$/i) !== null) {
      return ASSETS_DIR + "/images/[name]-[hash].[ext]";
    } else {
      return ASSETS_DIR + "/[ext]/[name]-[hash].[ext]";
    }
  }

  export function buildChunkFile(chunkInfo: PreRenderedChunk) {
    let fileName = chunkInfo.name?.replace("-legacy", "");
    return ASSETS_DIR + "/js/" + fileName + "/[name]-[hash].js";
  }
*/
const OUTPUT_DIR = require("./build/constant.ts").OUTPUT_DIR;

// https://vitejs.dev/config/
export default ({ command, mode }: ConfigEnv): UserConfig => {
  const url = import.meta.url;

  return {
    // 需要用到的插件数组
  	plugins: [vue()],
  	// 解析
 	 resolve: {
    	alias: {
    	  "@": fileURLToPath(new URL("./src", import.meta.url)),
   	 	},
  	},
  	// 开发或生产环境服务的公共基础路径
  	base: "./",

    build: {
      target: "es2015",
      outDir: OUTPUT_DIR,
      terserOptions: {
        compress: {
          keep_infinity: true,
          // Used to delete console in production environment
          drop_console: VITE_DROP_CONSOLE,
        },
      },
      rollupOptions: {
        // 设置项目打包入口文件
        input: {
          index: resolve(__dirname, "index.html"),
        },
        // 设置项目打包资源输出文件，包括JS、CSS、图片
        output: {
          chunkFileNames: (chunkInfo) => buildChunkFile(chunkInfo),
          entryFileNames: "[name]-[hash].js",
          assetFileNames: (chunkInfo) => buildAssetsFile(chunkInfo),
        },
      },
    },
  };
};
```

# 项目最终目录结构

```
PROJECT
├─ .husky
│  ├─ -
│  │  ├─ .gitnore
│  │  └─ husky.sh
│  ├─ commit-msg
│  ├─ common.sh
│  └─ pre-push
├─ .vscode
│  ├─ settings.json	# VSCode设置			
│  └─ extensions.json
├─ bulid			# 打包配置模块
│  ├─ vite
│  │  ├─ plugin		# 插件模块单独管理
│  │  │  ├─ index.ts
│  │  │  ├─ autoImport.ts
│  │  │  ├─ compress.ts
│  │  │  ├─ element.ts
│  │  │  ├─ html.ts
│  │  │  ├─ imagemin.ts
│  │  │  └─ svgSprite.ts
│  │  └─ proxy.ts	# 配置化代理服务器
│  ├─ constant.ts
│  ├─ getConfigFileName.ts
│  └─ utils.ts
├─ deploy			# 服务器上传脚本
│  ├─ index.ts
│  ├─ config.ts
│  └─ serverConfig.ts
├─ public			# 公共资源模块
│  └─ favicon.ico
├─ src
│  ├─ api			# 接口管理模块
│  ├─ assets		# 静态资源模块
│  │  └─ logo.png
│  ├─ common		# 自定义通用模块
│  ├─ components	# 公共组件模块
│  │  └─ HelloWorld.vue
│  ├─ enums			# 项目元组
│  ├─ layouts		# 公共自定义布局
│  ├─ locales		# 多语言配置模块
│  ├─ packages		# 项目包管理模块
│  ├─ router		# 路由
│  ├─ store			# pinia状态库
│  ├─ style			# 样式资源模块
│  ├─ utils			# 公共方法模块
│  ├─ views			# 视图模块
│  ├─ App.vue
│  ├─ env.d.ts
│  └─ main.ts		# 入口文件
├─ tests
├─ types			# 声明文件
├─ .env
├─ .env.development
├─ .env.production
├─ .env.test
├─ .eslintignore
├─ .eslintrc.cjs
├─ .gitignore
├─ .prettierignore
├─ .prettierrc.json
├─ .stylelintignore
├─ stylelint.config.js
├─ commitlint.config.js
├─ index.html
├─ package-lock.json
├─ package.json
├─ postcss.config.js
├─ README.md
├─ tsconfig.json
└─ vite.config.ts
```

`.vscode/settings.json`

```json
{
  "i18n-ally.localesPaths": ["src/locales", "src/locales/lang"],
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll": true
  },
  "eslint.validate": ["javascript", "javascriptreact", "vue"],
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[vue]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[jsonc]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[html]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[css]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "vetur.format.defaultFormatter.js": "prettier",
  "vetur.format.defaultFormatter.html": "prettyhtml",
  "vetur.format.defaultFormatterOptions": {
    "prettier": {
      "semi": true,
      "printWidth": 100,
      "singleQuote": false,
      "trailingComma": "none",
      "arrowParens": "avoid"
    },
    "prettyhtml": {
      "semi": true,
      "printWidth": 100,
      "singleQuote": false,
      "trailingComma": "none",
      "arrowParens": "avoid"
    }
  },
  "prettier.semi": true,
  "prettier.printWidth": 100,
  "prettier.trailingComma": "none",
  "prettier.singleQuote": false,
  "prettier.arrowParens": "avoid",
  "search.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/doc": true,
    "**/.git": true,
    "**/.gitignore": true,
    "**/.vscode": false,
    "**/yarn.lock": true,
    "dist": true,
    "node_modules": true,
    "yarn-error.log": true,
    "**/.yarn": true
  }
}
```

`.husky`

```shell
# husky.sh
#!/usr/bin/env sh
if [ -z "$husky_skip_init" ]; then
  debug () {
    if [ "$HUSKY_DEBUG" = "1" ]; then
      echo "husky (debug) - $1"
    fi
  }

  readonly hook_name="$(basename -- "$0")"
  debug "starting $hook_name..."

  if [ "$HUSKY" = "0" ]; then
    debug "HUSKY env variable is set to 0, skipping hook"
    exit 0
  fi

  if [ -f ~/.huskyrc ]; then
    debug "sourcing ~/.huskyrc"
    . ~/.huskyrc
  fi

  readonly husky_skip_init=1
  export husky_skip_init
  sh -e "$0" "$@"
  exitCode="$?"

  if [ $exitCode != 0 ]; then
    echo "husky - $hook_name hook exited with code $exitCode (error)"
  fi

  if [ $exitCode = 127 ]; then
    echo "husky - command not found in PATH=$PATH"
  fi

  exit $exitCode
fi
# ===========================================================
# commit-msg
#!/bin/sh

# shellcheck source=./_/husky.sh
. "$(dirname "$0")/_/husky.sh"


npx --no-install commitlint --edit "$1"
# ===========================================================
# common.sh
#!/bin/sh
command_exists () {
  command -v "$1" >/dev/null 2>&1
}

# Workaround for Windows 10, Git Bash and Yarn
if command_exists winpty && test -t 1; then
  exec < /dev/tty
fi
```



# `Vite.config.ts`最终配置

```ts
import { fileURLToPath, URL } from "node:url";
import { resolve } from "node:path";
import { loadEnv, type ConfigEnv, type UserConfig } from "vite";
import { wrapperEnv, buildAssetsFile, buildChunkFile } from "./build/utils";
import { createVitePlugin } from "./build/vite/plugin";
import { createProxy } from "./build/vite/proxy";
const OUTPUT_DIR = require("./build/constant.ts").OUTPUT_DIR;

// https://vitejs.dev/config/
export default ({ command, mode }: ConfigEnv): UserConfig => {
  const url = import.meta.url;

  // process.cwd()方法返回Node.js进程的当前工作目录。
  const root = process.cwd();

  // 加载 root 中的 .env 文件。
  const env = loadEnv(mode, root);

  // loadEnv读取的布尔类型是一个字符串。这个函数可以转换为布尔类型
  const viteEnv = wrapperEnv(env);

  const isBuild = command === "build";

  const { VITE_PORT, VITE_PUBLIC_PATH, VITE_PROXY, VITE_DROP_CONSOLE } = viteEnv;

  return {
    base: VITE_PUBLIC_PATH,

    root,

    plugins: createVitePlugin(viteEnv, isBuild),

    resolve: {
      alias: {
        "@": fileURLToPath(new URL("./src", url)),
        "#": fileURLToPath(new URL("./types", url)),
        $locale: fileURLToPath(new URL("./src/locales/setupLocale.ts", url)),
        $store: fileURLToPath(new URL("./src/store/modules", url)),
      },
    },

    server: {
      host: true,
      port: VITE_PORT,
      open: true,
      cors: true,
      proxy: createProxy(VITE_PROXY),
    },

    esbuild: {
      // 删除对控制台API方法(如console.log)的所有调用
      pure: VITE_DROP_CONSOLE ? ["console.log", "debugger"] : [],
    },

    build: {
      target: "es2015",
      outDir: OUTPUT_DIR,
      terserOptions: {
        compress: {
          keep_infinity: true,
          // Used to delete console in production environment
          drop_console: VITE_DROP_CONSOLE,
        },
      },
      rollupOptions: {
        // 设置项目打包入口文件
        input: {
          index: resolve(__dirname, "index.html"),
        },
        // 设置项目打包资源输出文件，包括JS、CSS、图片
        output: {
          chunkFileNames: (chunkInfo) => buildChunkFile(chunkInfo),
          entryFileNames: "[name]-[hash].js",
          assetFileNames: (chunkInfo) => buildAssetsFile(chunkInfo),
        },
      },
    },
  };
};
```



