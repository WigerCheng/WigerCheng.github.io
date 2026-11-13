---
title: AbilityStage组件管理器
draft: true
tags:
  - HarmonyOS/App
---
AbilityStage是一个Module级别的组件管理器，用于进行Module级别的资源预加载、线程创建等初始化操作，以及维护Module下的应用状态。

**AbilityStage与Module一一对应，即一个Module拥有一个AbilityStage**。应用的HAP/HSP在首次加载时会创建一个AbilityStage实例。当一个Module中存在AbilityStage和其他组件（UIAbility/ExtensionAbility组件），AbilityStage实例会早于其他组件实例创建。

## 生命周期回调

AbilityStage拥有onCreate()、onDestroy()生命周期回调。

### onCreate

`onCreate(): void`

- 在加载Module的第一个Ability实例前，系统会先创建对应的AbilityStage实例，并在AbilityStage创建完成后，自动触发该回调。
- 开发者可以在该回调中**执行Module的初始化操作**（如资源预加载、线程创建等）。
- 同步接口，不支持异步回调。

### onDestroy

`onDestroy(): void`

- 在对应Module的最后一个Ability实例退出后会触发该回调。
- 此方法将在正常的调度生命周期中调用，当应用程序异常退出或被终止时，将不会调用此方法
- 同步接口，不支持异步回调。

## 事件回调

### onAcceptWant
