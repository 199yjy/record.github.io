| 插件 | 功能 | 
|-------|-------|
| vue3 |  | 
| ts |  |
| pinia | 状态管理 |
| vue-router |  |
| vue-i18n |  |
| element-plus | UI框架 |
| axios |  |
| eslint | 规范 |
| less | css处理器 |
| commitlint | git提交规范|
| husky | git钩子触发器 |

## 初始化框架
### 安裝vue
```js 
npm init vue@latest
```
### 初始化
```js vite.config.ts
import { fileURLToPath, URL } from "node:url";

import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";
// 如果编辑器提示 path 模块找不到，则可以安装一下 @types/node -> npm i @types/node -D
import path from "path";
// https://vitejs.dev/config/
export default defineConfig({
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
  // 服务器选项
  server: {
    // 指定开发服务器端口
    port: 4000,
    // 在开发服务器启动时自动在浏览器中打开应用程序
    open: true,
    // 为开发服务器配置 CORS
    cors: true,
  },
});
```
### 初始化目录
```js
├─ .vscode
│  └─ extensions.json
├─ bulid			# 打包配置模块
├─ public			# 公共资源模块
│  └─ favicon.ico
├─ src
│  ├─ api			# 接口管理模块
│  ├─ assets		# 静态资源模块
│  │  └─ logo.png
│  ├─ common		# 自定义通用模块
│  ├─ components	# 公共组件模块
│  │  └─ HelloWorld.vue
│  ├─ layouts		# 公共自定义布局
│  ├─ router		# 路由
│  ├─ stores		# pinia状态库
│  ├─ style			# 样式资源模块
│  ├─ utils			# 公共方法模块
│  ├─ views			# 视图模块
│  ├─ App.vue
│  ├─ env.d.ts
│  └─ main.ts		# 入口文件
├─ tests
├─ types			# 声明文件
├─ .gitignore
├─ index.html
├─ package-lock.json
├─ package.json
├─ README.md
├─ tsconfig.json
└─ vite.config.ts

```
## 初始化路由
### 安装路由
```js
npm install vue-router
```
### 配置路由文件
### 挂载路由
```js
import { createApp } from "vue";
import App from "./App.vue";
import router from "@/router/index";

const app = createApp(App);

app.use(router);

app.mount("#app");
```
## 初始化状态
### 安装pinia
```js
npm install pinia

```
### 配置状态仓库
```js index.ts
import type { App } from "vue";
import { createPinia } from "pinia";

const store = createPinia();

export function setupStore(app: App<Element>) {
  app.use(store);
}

export { store };

```
```js userStore.ts
import { store } from "@/store";
import { defineStore } from "pinia";

export const useUserStore = defineStore({
  id: "userStore",
  state: () => ({}),
  getters: {},
  actions: {},  
});

export function useUserStoreHook() {
  return useUserStore(store);
}
```
### 挂载
```js
import { createApp } from "vue";
import App from "./App.vue";
import { setupStore } from "@/store";

const app = createApp(App);

// 配置状态仓库
setupStore(app);

app.mount("#app");

```
## 集成element-plus组件库
### 安装
```js
npm install element-plus --save

```
### 选择导入方式
- 完全导入
```
import type { App } from "vue";

/**
 * 完整导入 element-plus 组件
 * @param app {App}
 */
import ElementPlus from "element-plus";
import "element-plus/dist/index.css";
import "@/style/element-plus.css";
import Modal from "@/components/Dialog";

/**
 * 完整导入 element-plus 图标
 * @param app {App}
 */
import * as ElementPlusIconsVue from "@element-plus/icons-vue";

export default function fullLoadEl(app: App, params: Object) {
  app.use(ElementPlus, params);
  for (const [key, component] of Object.entries(ElementPlusIconsVue)) {
    app.component(key, component);
  }
  Modal._context = app._context;
  return app;
}
```
- 按需导入
```js
import type { App } from "vue";

/**
 * 按需导入 element-plus 组件
 * @param app {App}
 */
import "element-plus/dist/index.css";
import { ElIcon, ElButton, ElInput, ElCheckbox } from "element-plus";
import "@/style/element-plus.css";

/**
 * 按需导入 element-plus 图标
 * @param app {App}
 */
import { Edit, Tools, Location, Setting } from "@element-plus/icons-vue";

export default function loadOnDemandEl(app: App) {
  [ElButton, ElIcon, ElInput, ElCheckbox].forEach((v) => {
    app.use(v);
  });
  [Edit, Tools, Location, Setting].forEach((v) => {
    app.component(v.name, v);
  });
  return app;
}
```

### 加载
```js
import { createApp } from "vue";
import App from "./App.vue";
import fullLoadEl from "@/packages/element-plus/fullLoadEl ";

const app = createApp(App);

// 全局引入
fullLoadEl(app);

app.mount("#app");
```
## 集成axios
### 安装axios
```js
npm install axios
```
### 二次封装axios
参考vben-admin对axios的封装
调用封装的axios类VAxios，传自设参数和项目预设参数的合并对象为参数，生成VAxios实例对象
```js
import type { AxiosResponse } from "axios";
import type { RequestOptions, Result } from "./types/axios";
/*
  export type ErrorMessageMode = "none" | "modal" | "message" | undefined;
  export type SuccessMessageMode = "none" | "message";

  export interface RequestOptions {
    // Format request parameter time
    formatDate?: boolean;
    // Whether to process the request result
    isTransformResponse?: boolean;
    // Whether to return native response headers
    // For example: use this attribute when you need to get the response headers
    isReturnNativeResponse?: boolean;
    // Error message prompt type
    errorMessageMode?: ErrorMessageMode;
    // Successful request message prompt
    successMessageMode?: SuccessMessageMode;
    // Whether to add a timestamp
    joinTime?: boolean;
    ignoreCancelToken?: boolean;
    // Whether to send token in header
    withToken?: boolean;
    // 请求重试机制
    retryRequest?: RetryRequest;
  }

  export interface Result<T = any> {
    code: number | string;
    type: "success" | "error" | "warning";
    msg: string;
    data: T;
  }

  // multipart/form-data: upload file
  export interface UploadFileParams {
    // Other parameters
    data?: Recordable;
    // File parameter interface field name
    name?: string;
    // file name
    file: File | Blob;
    // file name
    filename?: string;
    [key: string]: any;
  }
*/
import type { AxiosTransform, CreateAxiosOptions } from "@/packages/http/axios/axiosTransform";
import axios from "axios";
import { clone } from "lodash-es";
import { VAxios } from "@/packages/http/axios/Axios";
import { checkStatus } from "@/packages/http/axios/checkStatus";
import { AxiosRetry } from "@/packages/http/axios/axiosRetry";
import { joinTimestamp, formatRequestDate } from "@/packages/http/axios/helper";
import { RequestEnum, ResultEnum, ContentTypeEnum } from "@/enums/httpEnum";
/*

  // @description: Request result set
  export enum ResultEnum {
    SUCCESS = 200,
    ERROR = -1,
    TIMEOUT = 401,
    TYPE = "success",
  }

  // @description: request method
  export enum RequestEnum {
    GET = "GET",
    POST = "POST",
    PUT = "PUT",
    DELETE = "DELETE",
  }

  // @description:  contentType
  export enum ContentTypeEnum {
    // json
    JSON = "application/json;charset=UTF-8",
    // form-data qs
    FORM_URLENCODED = "application/x-www-form-urlencoded;charset=UTF-8",
    // form-data  upload
    FORM_DATA = "multipart/form-data;charset=UTF-8",
  }
*/
import { deepMerge } from "@/utils";
/*
  export function deepMerge<T = any>(src: any = {}, target: any = {}): T {
    let key: string;
    for (key in target) {
      src[key] = isObject(src[key]) ? deepMerge(src[key], target[key]) : (src[key] = target[key]);
    }
    return src;
  }
*/
import { getAppEnvConfig } from "@/utils/env";
/* 
  // build/getConfigFileName.ts
  export const getConfigFileName = (env: Record<string, any>) => {
    return `__PRODUCTION__${env.VITE_GLOB_APP_SHORT_NAME || "__APP"}__CONF__`
      .toUpperCase()
      .replace(/\s/g, "");
  };

  export function getAppEnvConfig() {
    const ENV_NAME = getConfigFileName(import.meta.env);
      
    const ENV = (import.meta.env.DEV
      ? // Get the global configuration (the configuration will be extracted independently when packaging)
        (import.meta.env as unknown as GlobEnvConfig)
      : window[ENV_NAME as any]) as unknown as GlobEnvConfig;

    const { VITE_GLOB_API_URL } = ENV;

    return {
      VITE_GLOB_API_URL,
    };
 }
*/
import { removeToken，getToken } from "@/utils/token";
/*
  import Cookies from "js-cookie";

  const TokenKey = "Token";

  export function getToken() {
    return Cookies.get(TokenKey);
  }

  export function setToken(token: string) {
    return Cookies.set(TokenKey, token);
  }

  export function removeToken() {
    return Cookies.remove(TokenKey);
  }
*/
import { isString } from "@/utils/is";
/*
  const toString = Object.prototype.toString;

  export function is(val: unknown, type: string) {
    return toString.call(val) === `[object ${type}]`;
  }
  
  export function isString(val: unknown): val is string {
    return is(val, "String");
  }
*/
import { useUserStoreWithOut } from "$store/user";
import { ElMessage, ElMessageBox } from "element-plus";
import { s } from "$locale";

const { VITE_GLOB_API_URL } = getAppEnvConfig();

const transform: AxiosTransform = {
  /**
   * @description: 处理响应数据。如果数据不是预期格式，可直接抛出错误
   */
  transformResponseHook: (res: AxiosResponse<Result>, options: RequestOptions) => {
    const { isTransformResponse, isReturnNativeResponse, successMessageMode } = options;
    // 是否返回原生响应头 比如：需要获取响应头时使用该属性
    if (isReturnNativeResponse) {
      return res;
    }
    // 不进行任何处理，直接返回
    // 用于页面代码可能需要直接获取code，data，message这些信息时开启
    if (!isTransformResponse) {
      return res.data;
    }
    // 错误的时候返回
    const result = res.data;
    if (!result) {
      // return '[HTTP] Request has no return value';
      throw new Error(s("请求出错，请稍后重试"));
    }
    //  这里 code，result，message为 后台统一的字段，需要在 types.ts内修改为项目自己的接口返回格式
    const { code, data, msg } = result;

    // 这里逻辑可以根据项目进行修改
    const hasSuccess = result && Reflect.has(result, "code") && code === ResultEnum.SUCCESS;
    if (hasSuccess) {
      if (successMessageMode === "message") {
        ElMessage({
          message: result.msg,
          type: "success",
        });
      }
      return data;
    }

    // 在此处根据自己项目的实际情况对不同的code执行不同的操作
    // 如果不希望中断当前请求，请return数据，否则直接抛出异常即可
    let timeoutMsg = "";
    switch (code) {
      case ResultEnum.TIMEOUT:
        timeoutMsg = s("登录超时，请重新登录！");
        const userStore = useUserStoreWithOut();
        removeToken();
        userStore.Logout(true);
        break;
      default:
        if (msg) {
          timeoutMsg = msg;
        }
    }

    // token过期的操作
    if (timeoutMsg === "登录过期，请重新登录!") {
      ElMessageBox({
        type: "error",
        title: s("错误提示"),
        message: timeoutMsg,
        confirmButtonText: s("确认"),
        callback: function (action: string) {
          if (action === "confirm") {
            removeToken();
            const userStore = useUserStoreWithOut();
            userStore.Logout(true);
          }
        },
      });
      return;
    }

    // errorMessageMode=‘modal’的时候会显示modal错误弹窗，而不是消息提示，用于一些比较重要的错误
    // errorMessageMode='none' 一般是调用时明确表示不希望自动弹出错误提示
    if (options.errorMessageMode === "modal") {
      if (timeoutMsg === "登录过期，请重新登录!") {
        ElMessageBox({
          type: "error",
          title: s("错误提示"),
          message: timeoutMsg,
          confirmButtonText: s("确认"),
          callback: function (action: string) {
            if (action === "confirm") {
              removeToken();
              const userStore = useUserStoreWithOut();
              userStore.Logout(true);
            }
          },
        });
      } else {
        ElMessageBox({
          type: "error",
          title: s("错误提示"),
          message: timeoutMsg,
          confirmButtonText: s("确认"),
        });
      }
    } else if (options.errorMessageMode === "message") {
      if (timeoutMsg === "登录过期，请重新登录!") {
        ElMessageBox({
          type: "error",
          title: s("错误提示"),
          message: timeoutMsg,
          confirmButtonText: s("确认"),
          callback: function (action: string) {
            if (action === "confirm") {
              removeToken();
              const userStore = useUserStoreWithOut();
              userStore.Logout(true);
            }
          },
        });
      } else {
        ElMessage.error(timeoutMsg);
      }
    }

    throw new Error(timeoutMsg || s("请求出错，请稍后重试"));
  },

  // 请求之前处理config
  beforeRequestHook: (config, options) => {
    const { formatDate, joinTime = true } = options;

    const params = config.params || {};
    const data = config.data || false;
    formatDate && data && !isString(data) && formatRequestDate(data);
    if (config.method?.toUpperCase() === RequestEnum.GET) {
      if (!isString(params)) {
        // 给 get 请求加上时间戳参数，避免从缓存中拿数据。
        config.params = Object.assign(params || {}, joinTimestamp(joinTime, false));
      } else {
        // 兼容restful风格
        config.url = config.url + params + `${joinTimestamp(joinTime, true)}`;
        config.params = undefined;
      }
    } else {
      if (!isString(params)) {
        formatDate && formatRequestDate(params);
        if (
          Reflect.has(config, "data") &&
          config.data &&
          (Object.keys(config.data).length > 0 || config.data instanceof FormData)
        ) {
          config.data = data;
          config.params = params;
        } else {
          // 非GET请求如果没有提供data，则将params视为data
          config.data = params;
          config.params = undefined;
        }
      } else {
        // 兼容restful风格
        config.url = config.url + params;
        config.params = undefined;
      }
    }
    return config;
  },

  /**
   * @description: 请求拦截器处理
   */
  requestInterceptors: (config, options) => {
    // 请求之前处理config
    const token = getToken();
    if (token && (config as Recordable)?.requestOptions?.withToken !== false) {
      // jwt token
      (config as Recordable).headers.Authorization = options.authenticationScheme
        ? `${options.authenticationScheme} ${token}`
        : token;
    }
    return config;
  },

  /**
   * @description: 响应拦截器处理
   */
  responseInterceptors: (res: AxiosResponse<any>) => {
    return res;
  },

  /**
   * @description: 响应错误处理
   */
  responseInterceptorsCatch: (axiosInstance: AxiosResponse, error: any) => {
    const { response, code, message, config } = error || {};
    const errorMessageMode = config?.requestOptions?.errorMessageMode || "none";
    const msg: string = response?.data?.error?.message ?? "";
    const err: string = error?.toString?.() ?? "";
    let errMessage = "";

    if (axios.isCancel(error)) {
      return Promise.reject(error);
    }

    try {
      if (code === "ECONNABORTED" && message.indexOf("timeout") !== -1) {
        errMessage = s("接口请求超时，请刷新页面重试！");
      }
      if (err?.includes("Network Error")) {
        errMessage = s("网络异常，请检查您的网络连接是否正常！");
      }

      if (errMessage) {
        if (errorMessageMode === "modal") {
          ElMessageBox({
            type: "error",
            title: s("错误提示"),
            message: errMessage,
            confirmButtonText: s("确认"),
          });
        } else if (errorMessageMode === "message") {
          ElMessage.error(errMessage);
        }
        return Promise.reject(error);
      }
    } catch (error) {
      throw new Error(error as unknown as string);
    }

    checkStatus(error?.response?.status, msg, errorMessageMode);

    // 添加自动重试机制 保险起见 只针对GET请求
    const retryRequest = new AxiosRetry();
    const { isOpenRetry } = config.requestOptions.retryRequest;
    config.method?.toUpperCase() === RequestEnum.GET &&
      isOpenRetry &&
      // @ts-ignore
      retryRequest.retry(axiosInstance, error);
    return Promise.reject(error);
  },
};

function createAxios(opt?: Partial<CreateAxiosOptions>) {
  return new VAxios(
    deepMerge(
      {
        authenticationScheme: "Bearer",
        timeout: 10 * 1000,
        // 基础接口地址
        baseURL: "basic-api",
        headers: { "Content-Type": ContentTypeEnum.JSON },
        // 数据处理方式 clone——克隆，对象的深拷贝
        transform: clone(transform),

        requestOptions: {
          // 是否返回原生响应头 比如：需要获取响应头时使用该属性
          isReturnNativeResponse: false,
          // 需要对返回数据进行处理
          isTransformResponse: true,
          // 格式化提交参数时间
          formatDate: true,
          // 消息提示类型
          errorMessageMode: "message",
          // 成功请求消息提示
          successMessageMode: "message",
          //  是否加入时间戳
          joinTime: true,
          // 忽略重复请求
          ignoreCancelToken: true,
          // 是否携带token
          withToken: true,
          retryRequest: {
            isOpenRetry: true,
            count: 5,
            waitTime: 100,
          },
        },
      },
      opt || {},
    ),
  );
}
export const defHttp = createAxios();
```
### 将axios封装成一个类
### 依据请求返回状态码显示提醒信息