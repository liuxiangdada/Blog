# Git

## 定义

Git是一款软件配置管理工具，或者说是一个版本控制系统，目的是管理代码的版本
- 其他的版本控制系统诸如CVS、Subversion、Perforce、Bazaar 等等都是保存文件的变更，即它们保存基本文件和每个文件随着时间逐步积累的差异
- Git是创建每次提交后全部文件的快照并保存这个快照的索引

## 特点

- 直接记录快照，而非差异比较
- 操作几乎都在本地执行
- git保证完整性：每次存储前都会计算所有数据的校验和，生成一个40位的字符串并根据校验和确保数据不被无意修改
- git一般只添加数据：我们几乎只对文件进行增加或者修改操作，而不是执行删除这种不可逆操作

## 工作流程

1. 在本地工作区修改文件
2. 将修改保存到暂存区
3. 提交修改，将暂存区的文件以快照的形式永久的存储到本地仓库目录
4. 将提交推送到远端仓库

![git工作流](../img/git工作流.png)


## 常用命令

## 基于gitFlow的工作流