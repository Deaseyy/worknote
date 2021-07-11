##### 题外linux操作

文本编辑器中显示行号： `:set nu`

多屏显示控制：空格向下翻页，b 向上翻页，q退出



# 创建项目操作

1.创建本地仓库

2.创建远程仓库

3.推送本地仓库到远端

3.其他成员需加入团队

4.其他成员克隆仓库到本地，克隆会自动将工作目录初始化为仓库



# Git命令行操作

工作区 --> 暂存区 --> 本地库

![image-20210710133126913](https://user-images.githubusercontent.com/49275906/125193353-8d575680-e27e-11eb-843c-d6550f9c90bf.png)


​		当 working tree  clean 时，可以理解为：三个区域都分别保存了相同的完整数据；每次工作区更新，都需要同步到暂存区和本地库，并保持一致。

## 基础操作

#### 本地库初始化

```
git init
```

会自动生成 `.git` 目录，存放的是本地库相关的子目录和文件

#### 设置签名

```cmd
# 项目级别/仓库级别：仅在当前本地库范围内有效
git config user.name yyds
git config user.email 123456@qq.com
# 系统用户级别：登录当前操作系统的用户范围
git config --global user.name yyds
git config -- global 123456.qq.com
```

主要用于区分不同开发人员身份。和登陆远程仓库的账号密码无关

#### 查看状态

```
git status
```

#### 添加'新建/修改'到暂存区

追踪新建的文件，如果文件已经追踪，则也可以直接使用 `git commit` 提交

```
git add <file>   # 指定文件
git add .        # .代表所有文件
```

- 撤销：将文件从暂存区撤出，恢复到之前状态

  ```
  git rm --cached <file>
  ```

#### 提交暂存区内容到本地仓库

```
git commit -m "修改bug" [<file>]
```

#### 查看历史提交记录

每次提交都有一个hash生成的版本号

```cmd
# 查看完整提交日志
git log
# 每条日志只显示一行
git log --pretty=oneline 
# 显示部分hash值，仅显示过去到当前的版本
git log --oneline
# 显示所有版本, HEAD@{n},回退到指定版本需要移动多少步
git reflog  # 版本回退时常用的命令
```

#### 版本前进后退

HEAD：指针默认指向当前的版本；HEAD^：指向过去一个版本。

```cmd
# 基于索引值
git reset --hard <索引值>  # 索引值如:9A9ebe0

# 基于 ^ 符号
git reset --hard  HEAD^^^
- ^只能向后退，几个^表示退几个版本

# 使用 ~ 符号
git reset --hard HEAD~3
- ~只能向后退，~n表示向后退n个版本
```

> 其他技巧：
>
> 1. 还未提交到本地库，想要丢弃修改时：使用 `git reset --hard HEAD` 指向当前版本，可以刷新工作区和暂存区，使其和当前指向的本地库版本保持一致。
>
> 2. 删除文件并找回：`git reset --hard <文件曾经存在过的版本位置>`。

reset 命令的三个参数区别：

- --soft
  - 仅仅在本地库移动 HEAD 指针
- --mixed
  - 在本地库移动 HEAD 指针
  - 重置暂存区
- --hard
  - 在本地库移动 HEAD 指针
  - 重置暂存区
  - 重置工作区

> 注意：soft 和 mixed 都不会改变工作区文件
>
> 使用--soft 回退，那么当前暂存区相对于回退后的版本，就会有新修改的文件（因为之前数据是同步的，本地库回退了，那么暂存区相对就多出来了）；
>
> 使用--mixed回退，那么和上面类似，本地库和暂存区都后退了，那么工作区相对就多出来了。

#### 比较文件差异

```cmd
# 将工作区中的文件和暂存区进行比较
git diff <file>
# 将工作区中的文件和本地库历史版本比较
git diff <历史版本> <文件名> # 如 git diff HEAD~3 aa.txt

- 不带文件名将比较多个文件
```



## 分支管理

#### 基本命令

```cmd
# 查看分支
git branch # 查看本地分支
git branch -r # 查看远程分支
git branch -v # 查看当前所有分支
git branch -vv # 查看本地分支对于远程分支的追踪关系

# 创建分支
git branch <新分支名>

# 切换分支
git checkout <分支名>

# 切换并创建分支
git checkout -b <分支名>

# 删除分支
git branch -d <分支名>
git branch -D <分支名>  # 强制切换

# 合并分支 到 当前分支上
git merge <分支名>
- 需先切换到接受合并的分支上，再merge有修改过的分支。

```



#### 解决冲突

合并分支时，若修改了同一个文件，可能需要手动解决冲突；

如将master分支合并到当前分支 `git merge master`，有冲突表现如下：

<img src="C:\Users\12395\AppData\Roaming\Typora\typora-user-images\image-20210710181438056.png" alt="image-20210710181438056" style="zoom:67%;" />

解释：HEAD 表示当前分支的内容，另一个就是合并的分支内容。

步骤：

1. 可能需和同事商量后，再去手动保留或删除某些内容，解决并退出文件，查看状态：

<img src="C:\Users\12395\AppData\Roaming\Typora\typora-user-images\image-20210710182006697.png" alt="image-20210710182006697" style="zoom:67%;" />

2. 执行 `git add 文件名` 后，查看状态：

![image-20210710182159040](C:\Users\12395\AppData\Roaming\Typora\typora-user-images\image-20210710182159040.png)

3. 最后执行 `git commit -m "注释"`， merging中不能带文件名。



## 其他命令

#### 仓库地址别名

```cmd
# 查看远程仓库地址的别名
git remote -v

# 给远程仓库地址添加别名（一般用origin）
git remote add origin <远程仓库地址>
- 后面就可以用 origin 代替该地址来使用


```

#### 推送本地分支到远端

```cmd
git push origin 本地分支[:远程分支]
- 只写本地分支，则推送该分支到其追踪的远端分支
```

- 如果不是基于GitHub远程库的最新版本所作的修改，不能推送，必须先拉取；

- 拉取下来后进入冲突解决状态，按分支冲突解决即可。

#### 克隆远程仓库到本地

```cmd
git clone <远程仓库地址>
git clone -b <指定分支> <远程仓库地址>
```

git clone包含三个操作：

- 完整的把远程库下载到本地
- 创建 origin 远程库地址别名
- 初始化本地库

#### 拉取远程分支到本地

```cmd
git pull origin <远程分支>
```

包含两个操作：
- git fetch  # 下载远程分支的内容

- git merge  # 合并到当前分支

有时候文件内容修改比较多且复杂，可以先抓取下来，自行查看确认后，再进行合并。

> git pull 还有个效果，刷新远程库的分支：
>
> 有时候新建了分支，但是通过命令去查看却找不到，使用 `git pull` 后面不接任何参数，可以刷新远程库的分支。

#### 下载远程分支的内容

```
git fetch origin <远程分支名>
```

如何查看下载下来的内容，可以通过切换到远程分支上：

1. git checkout origin/分支名
2. cat 文件名



# Git工作流

### gitflow

团队项目开发常用的模式

<img src="C:\Users\12395\AppData\Roaming\Typora\typora-user-images\image-20210711183118133.png" alt="image-20210711183118133" style="zoom: 33%;" />



### 分支实战

一般我们不会直接到master分支上去开发，都会创建自己的开发分支。

**步骤：**

1.员工克隆远程仓库到本地

2.本地创建新分支featureA，并在上面开发功能

3.本地测试后，推送featureA分支到远程(origin/featureA)

4.PM拉取远程分支featureA到本地测试

5.PM测试featureA没问题后，在本地合到master分支上

6.最后推送master到远程master分支

一般来说，代码测试，检视，合并都是在本地完成，最后推送到远程；

但有时为了加快进度，开发者将自己的分支推送到远端后，直接在远程仓库上创建合并请求，PM进行代码检视后直接合入目标分支（一般都是测试分支，发布新版本时才合入主分支）。



# Git原理

## 哈希

HASH的加密算法：

- MD5：加密结果为 32位16进制数
- SHA1
- CRC32

几个共同点：

- 不管输入数据的数据量有多大，输入同一个哈希算法，得到的加密结果长度固定。
- 哈希算法确定，输入数据确定，输出数据保证不变。
- 哈希算法确定，输入数据有变化，输出数据一定有变化，而且变化很大。
- 哈希算法不可逆

哈希算法可以被用来验证文件，原理如下图：

<img src="C:\Users\12395\AppData\Roaming\Typora\typora-user-images\image-20210710184423389.png" alt="image-20210710184423389" style="zoom: 50%;" />

Git底层采用的是 `SHA-1` 算法。就是靠这种机制从根本上保证数据完整性。

具体原理暂时省略。。。。。。。。。。。。。。。。。。。。。

