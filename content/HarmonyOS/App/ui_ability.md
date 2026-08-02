---
title: UIAbility
draft: true
---

UIAbility组件是一种包含UI的应用组件，主要用于和用户交互。它继承自Ability，提供UIAbility组件创建、销毁、前后台切换等生命周期回调，同时也具备后台通信能力。(有点类似Android的Activity)

## UIAbility的声明配置

## UIAbility的生命周期

当用户在执行应用启动、应用前后台切换、应用退出等操作时，系统会触发相关应用组件的生命周期回调。

UIAbility组件的核心生命周期回调包括onCreate、onForeground、onBackground、onDestroy。

![[ui_ability_lifecycle.png]]

### onCreate

`onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void`

当UIAbility实例创建完成时，系统会触发该回调，开发者可在该回调中执行初始化逻辑（如定义变量、加载资源等）。

**该回调仅会在UIAbility==冷启动==时触发**。

| 参数名         | 类型                                                                                                                                               | 说明                         |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------- |
| want        | [Want](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-app-ability-want)                                               | 调用方拉起该UIAbility时传递的数据。     |
| launchParam | [AbilityConstant.LaunchParam](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-app-ability-abilityconstant#launchparam) | 应用启动参数，包含应用启动原因、应用上次退出原因等。 |

```ts
//如在UIAbility创建完后设置颜色模式
onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {  
  try {  
    this.context.getApplicationContext()
    .setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_NOT_SET);  
  } catch (err) {  
  }  
  hilog.info(DOMAIN, 'testTag', '%{public}s', 'Ability onCreate');  
}
```
### onWindowStageCreate

`onWindowStageCreate(windowStage: window.WindowStage): void`

当WindowStage实例创建完成后，系统会触发该回调。开发者可以在该回调中通过WindowStage加载页面。

| 参数名         | 类型                                                                                                                    | 说明               |
| ----------- | --------------------------------------------------------------------------------------------------------------------- | ---------------- |
| windowStage | [window.WindowStage](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-window-windowstage) | WindowStage实例对象。 |


## UIAbility使用

### 指定启动页面

应用中的UIAbility在启动过程中，需要指定启动页面，否则应用启动后会因为没有默认加载页面而导致白屏。

在[[#onWindowStageCreate]]生命周期回调中，**通过WindowStage对象的loadContent()方法设置启动页面**。

```ts
  onWindowStageCreate(windowStage: window.WindowStage): void {
    // Main window is created, set main page for this ability
    windowStage.loadContent('pages/Index', (err) => {
      // ···
    });
  }
```
