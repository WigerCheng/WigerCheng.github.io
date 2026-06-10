[jEnv - Manage your Java environment](https://www.jenv.be/)
一个command line tool工具帮助你设置JAVA_HOME变量
通过brew安装jenv
`brew install jenv`
启用jenv，需将jenv加入zshvc
```
echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(jenv init -)"' >> ~/.zshrc
```
添加jdk
`jenv add /Library/Java/JavaVirtualMachines/microsoft-11.jdk/Contents/Home`
![[jenv_add_1.png]]
`jenv add /Library/Java/JavaVirtualMachines/microsoft-17.jdk/Contents/Home`
![[jenv_add_2.png]]
显示已安装的jdk列表
`jenv versions`
![[jenv_versions.png]]
配置jdk
- 对某个（项目）文件夹配置jdk
	- `jenv local openjdk64-17.0.10`
	- ![[jenv_local.png]]
- 全局配置jdk
	- `jenv global 11.0`
	- ![[jenv_global.png]]
——————
额外：
1. Mac 安装 JDK 的默认路径是  **/Library/Java/JavaVirtualMachines**