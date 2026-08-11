---
title: jenv的使用
tags:
  - Other
---

## 什么是jenv？

jenv是一款能帮助你给不同的项目设置不同的jdk版本的命令行工具。

## 使用jenv

### 1.安装jenv

在mac OS系统中通过brew来安装jenv`brew install jenv`。

### 2.配置jenv

执行下面的语句，将jenv的配置写入.zshrc文件中。

```bash
echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(jenv init -)"' >> ~/.zshrc
jenv enable-plugin export
```

执行`jenv doctor`来验证jenv是否安装成功。

![[jenv_doctor.png]]

### 3. 添加jdk

> [!tip]
> Mac 安装 JDK 的默认路径是  **/Library/Java/JavaVirtualMachines**

通过`jenv add`指令将本地的jdk添加到jenv中。

`jenv add /Library/Java/JavaVirtualMachines/microsoft-11.jdk/Contents/Home`
![[jenv_add_1.png]]

`jenv add /Library/Java/JavaVirtualMachines/microsoft-17.jdk/Contents/Home`
![[jenv_add_2.png]]

### 4.其他命令

#### 显示已安装的jdk列表

通过执行`jenv versions`可以看到已经添加到jenv的jdk列表。

![[jenv_versions.png]]

#### 对某个（项目）文件夹配置jdk

通过执行`jenv local`来指定jdk版本。

`jenv local openjdk64-17.0.10`
![[jenv_local.png]]

#### 全局配置jdk

通过执行`jenv global`来指定jdk版本。

`jenv global 11.0`
![[jenv_global.png]]

---

- [jenv官网链接](https://www.jenv.be/)
- [jenvGithub链接](https://github.com/jenv/jenv)
