# Theia Electron 扩展开发指南

## 概述

本文档总结了 Theia Electron 主进程扩展的开发流程，详细介绍了如何在 Theia 桌面应用中添加自定义启动项。

---

## 目录

1. [Electron 主进程架构](#1-electron-主进程架构)
2. [启动进程详解](#2-启动进程详解)
3. [多窗口进程模型](#3-多窗口进程模型)
4. [如何添加自定义启动项](#4-如何添加自定义启动项)
5. [构建系统自动流程](#5-构建系统自动流程)
6. [完整流程架构图](#6-完整流程架构图)
7. [完整示例](#7-完整示例)

---

## 1. Electron 主进程架构

### 1.1 进程架构

Theia 桌面应用由多个进程组成：

| 进程类型 | 单实例 | 说明 |
|---------|-------|------|
| **Electron Main Process** | 是 | 应用主进程，只有一个 |
| **Backend Process** | 是 | Node.js 后端进程，多窗口共享 |
| **Renderer Process** | 否 | 每个窗口独立 |
| **Plugin Host Process** | 否 | 每个窗口独立的插件进程 |

### 1.2 核心类

核心类位于 [electron-main-application.ts](file:///home/jerry/go/src/github.com/JerryZhou343/theia/packages/core/src/electron-main/electron-main-application.ts#L157)：

```typescript
export class ElectronMainApplication {
    // ...
}
```

---

## 2. 启动进程详解

### 2.1 启动时序

启动过程的启动顺序（来自 [electron-main-application.ts#L221-250](file:///home/jerry/go/src/github.com/JerryZhou343/theia/packages/core/src/electron-main/electron-main-application.ts#L221-L250)）：

1. Electron Main Process 启动
2. 启动 Backend Process（通过 startBackend()）
3. 附加安全令牌
4. 启动贡献者（startContributions()）
5. 创建初始窗口（包含 Renderer Process）
6. 根据需要启动 Plugin Host Processes

### 2.2 后端启动

Backend Process 的启动方式（来自 [electron-main-application.ts#L732-785](file:///home/jerry/go/src/github.com/JerryZhou343/theia/packages/core/src/electron-main/electron-main-application.ts#L732-L785)）：

- 默认 fork 出独立进程
- 使用 `--no-cluster` 参数可禁用 fork

---

## 3. 多窗口进程模型

### 3.1 进程关系

多个窗口时的进程关系：

- **Backend Process**：所有窗口共享
- **Renderer Process**：每个窗口独立
- **Plugin Host Process**：每个窗口独立

### 3.2 贡献者收集

每个窗口的贡献者通过 [ContributionProvider](file:///home/jerry/go/src/github.com/JerryZhou343/theia/packages/core/src/electron-main/electron-main-application.ts#L158-L161) 收集：

```typescript
@inject(ContributionProvider)
@named(ElectronMainApplicationContribution)
protected readonly contributions: ContributionProvider<ElectronMainApplicationContribution>;
```

---

## 4. 如何添加自定义启动项

### 4.1 步骤 1：创建贡献者类

创建实现 `ElectronMainApplicationContribution` 接口的类：

```typescript
// my-electron-contribution.ts
import { injectable } from '@theia/core/shared/inversify';
import { ElectronMainApplication, ElectronMainApplicationContribution } from '@theia/core/lib/electron-main/electron-main-application';
import { MaybePromise } from '@theia/core';

@injectable()
export class MyElectronContribution implements ElectronMainApplicationContribution {
    
    onStart(application: ElectronMainApplication): MaybePromise<void> {
        // 在这里执行启动时的初始化逻辑
        console.log('My contribution starting...');
    }
    
    onStop(application: ElectronMainApplication): void {
        // 在这里执行应用停止时的清理逻辑
        console.log('My contribution stopping...');
    }
}
```

接口定义（来自 [electron-main-application.ts#L93-105](file:///home/jerry/go/src/github.com/JerryZhou343/theia/packages/core/src/electron-main/electron-main-application.ts#L93-L105)）：

```typescript
export interface ElectronMainApplicationContribution {
    onStart?(application: ElectronMainApplication): MaybePromise<void>;
    onStop?(application: ElectronMainApplication): void;
}
```

### 4.2 步骤 2：创建模块文件

创建 ContainerModule：

```typescript
// my-electron-main-module.ts
import { ContainerModule } from 'inversify';
import { ElectronMainApplicationContribution } from '@theia/core/lib/electron-main/electron-main-application';
import { MyElectronContribution } from './my-electron-contribution';

export default new ContainerModule(bind => {
    bind(MyElectronContribution).toSelf().inSingletonScope();
    bind(ElectronMainApplicationContribution).toService(MyElectronContribution);
});
```

参考示例：[filesystem/electron-main-module.ts](file:///home/jerry/go/src/github.com/JerryZhou343/theia/packages/filesystem/src/electron-main/electron-main-module.ts#L20-23)。

### 4.3 步骤 3：在 package.json 中声明

```json
{
  "theiaExtensions": [
    {
      "electronMain": "lib/electron-main/my-electron-module"
    }
  ]
}
```

---

## 5. 构建系统自动流程

### 5.1 完整流程

1. **扩展包声明** → **收集扩展包** → **模块解析** → **生成入口文件** → **运行时加载** → **启动贡献**

### 5.2 详细步骤

#### 阶段 1：构建时

**1. 扩展包声明**

在 `package.json` 中声明 `electronMain` 字段

**2. 收集扩展包**

[ExtensionPackageCollector](file:///home/jerry/go/src/github.com/JerryZhou343/theia/dev-packages/application-package/src/extension-package-collector.ts#L31-36) 扫描：

```typescript
collect(packagePath: string, pck: NodePackage): ReadonlyArray<ExtensionPackage> {
    this.root = pck;
    this.collectPackages(packagePath, pck);
    return this.sorted;
}
```

**3. 模块解析**

[computeModules](file:///home/jerry/go/src/github.com/JerryZhou343/theia/dev-packages/application-package/src/application-package.ts#L185-202) 收集模块：

```typescript
get electronMainModules(): Map<string, string> {
    return this._electronMainModules ??= this.computeModules('electronMain');
}
```

**4. 生成入口文件**

[BackendGenerator](file:///home/jerry/go/src/github.com/JerryZhou343/theia/dev-packages/application-manager/src/generator/backend-generator.ts#L35-113) 生成 `electron-main.js`：

```typescript
protected compileElectronMain(electronMainModules?: Map<string, string>): string {
    return `
        // ...
        ${Array.from(electronMainModules?.values() ?? [], jsModulePath => `
            await load(require('${jsModulePath}'));
        `).join('')}
        // ...
    `;
}
```

生成的文件包含：

```javascript
await load(require('@theia/filesystem/lib/electron-main/electron-main-module'));
await load(require('@theia/core/lib/electron-main/electron-main-application-module'));
await load(require('my-extension/lib/electron-main/my-electron-module'));
```

#### 阶段 2：运行时

**5. 加载到容器**

```javascript
const container = new Container();
container.load(electronMainApplicationModule);
container.bind(ElectronMainApplicationGlobals).toConstantValue({...});

function load(raw) {
    return Promise.resolve(raw.default).then(module =>
        container.load(module)
    );
}
```

**6. 启动应用**

```typescript
const application = container.get(ElectronMainApplication);
await application.start(config);
```

**7. 调用贡献者**

```typescript
protected async startContributions(): Promise<void> {
    const promises: Promise<void>[] = [];
    for (const contribution of this.contributions.getContributions()) {
        if (contribution.onStart) {
            promises.push(contribution.onStart!(this));
        }
    }
    await Promise.all(promises);
}
```

---

## 6. 完整流程架构图

```
┌─────────────────────────────────────────────────────────────────┐
│  构建时                                                          │
│  1. @theia/filesystem/package.json                             │
│     "theiaExtensions": { "electronMain": "..." }                 │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. ExtensionPackageCollector.collect()                          │
│     - 扫描 dependencies/peerDependencies                       │
│     - 检查是否有 theiaExtensions                               │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. computeModules('electronMain')                              │
│     - 生成 Map: { "electronMain_1": "@theia/filesystem/..." }   │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. compileElectronMain()                                      │
│     - 生成 electron-main.js                                    │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  运行时                                                          │
│  5. electron-main.js 执行                                      │
│     - 创建 Inversify 容器                                       │
│     - 加载所有模块到容器                                          │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. application.start()                                        │
│     - 调用每个贡献者的 onStart()                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. 完整示例

### 7.1 文件结构

```
my-extension/
├── package.json
└── src/
    └── electron-main/
        ├── my-electron-contribution.ts
        └── my-electron-main-module.ts
```

### 7.2 package.json

```json
{
  "name": "my-extension",
  "version": "1.0.0",
  "theiaExtensions": [
    {
      "electronMain": "lib/electron-main/my-electron-main-module"
    }
  ]
}
```

### 7.3 my-electron-contribution.ts

```typescript
import { injectable } from '@theia/core/shared/inversify';
import { ElectronMainApplication, ElectronMainApplicationContribution } from '@theia/core/lib/electron-main/electron-main-application';
import { MaybePromise } from '@theia/core';

@injectable()
export class MyElectronContribution implements ElectronMainApplicationContribution {
    
    onStart(application: ElectronMainApplication): MaybePromise<void> {
        console.log('My custom extension starting...');
        // 初始化自定义逻辑
    }
    
    onStop(application: ElectronMainApplication): void {
        console.log('My custom extension stopping...');
        // 清理逻辑
    }
}
```

### 7.4 my-electron-main-module.ts

```typescript
import { ContainerModule } from 'inversify';
import { ElectronMainApplicationContribution } from '@theia/core/lib/electron-main/electron-main-application';
import { MyElectronContribution } from './my-electron-contribution';

export default new ContainerModule(bind => {
    bind(MyElectronContribution).toSelf().inSingletonScope();
    bind(ElectronMainApplicationContribution).toService(MyElectronContribution);
});
```

---

## 总结

要添加自定义启动项只需要三个步骤：

1. **创建贡献者类，实现 ElectronMainApplicationContribution 接口
2. **创建模块文件，注册绑定
3. **在 package.json 中声明 electronMain 字段

**剩下的都是 Theia 构建系统自动完成的！**
