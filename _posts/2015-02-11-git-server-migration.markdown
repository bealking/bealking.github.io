---
layout: post
title: "Git库迁移手记"
date: 2015-02-11 15:03:31
category: server
---

公司的 Git 库原本托在兄弟公司的服务器上，琢磨着年后要搬办公地点，不好维护，外加我即将离职，于是就将把 Git 库迁回我们自己服务器的任务火速派给了我，呵呵……
于是在此简单记录了一下迁移过程，以便后面接手的人（好吧，如果有的话）有问题能够查阅。

由于没有人家服务器的管理权限，所以本来简单的移库变得麻烦了一点。

首先从 Git 上依次将所有 repository 都检出一份纯版本库。

```bash
  git clone --bare git@127.0.0.1:/home/git/project.git
```

然后创建新 Git 服务器的 git 账户和目录：

```bash
  adduser -m git  #同步创建 /home/git 目录
  mkdir -p /home/git/.ssh
  touch authorized_keys
```

把刚导出的所有 Git 库打包后通过sftp上传到服务器 /home/git 目录。上传完成后，ssh连上去，解压库文件，同时调整目录所有权。

```bash
  unzip gits.zip
  chown -R git:git /home/git/*.git  #递归地将 /home/git 目录下所有含有 .git 名字的目录划归用户 git 所有
```

注：如果不调整目录权限，很可能你push数据上来的时候会遇到下面错误

```bash
  Git Push Error: insufficient permission for adding an object to repository database
```

因为原来的 Git 公匙都保存在兄弟公司的服务器上拿不回来，所以要重新收集每个人的 ssh key，并添加到 authorized_keys 里，每行一个，注意不要断行。

Linux下生成key使用如下方式：

```bash
  sh-keygen -t rsa -C "test@gmail.com"  #大C参数嫌麻烦可省略。输入后按回车，再输入两次密码即可
```

运行后，会在当前用户的家目录下的.ssh目录下生成两个文件。一个名为 id\_rsa 的为私匙，无需理会。另一个名为 id\_rsa.pub 的为公匙，需要收集上来填入 authorized_keys。

使用 TortoiseGit 的用户如何生成key？

>TortoiseGit 使用扩展名为 ppk 的密钥，而不是 ssh-keygen 生成的 rsa 密钥。所以需要使用 TortoiseGit 自带的 putty key generator 来生成 ppk 密匙。启动 putty key generator 后，点击 Generate 按钮，之后按提示在窗体中划动鼠标以生成随机要素，进度条走完后即会生成公匙。按需修改公匙注释，填写密码后，将文本框里的公匙添加进 /home/git/.ssh/authorized_keys 中。之后点 Save private key 保存私匙发给相应的维护人员以供拉去和提交修改。

以上都搞定之后，即可通过 git 正常 pull/push 数据了。

不过还需要继续做一些改动才能正常使用 mina 进行部署。

1. 如果项目存在子模块，需要修改项目目录下的 .gitmodules 文件，将其中子模块的源路径改为新的地址。
2. 在生产服务器上执行 sh-keygen -t rsa 生成 ssh key，并追加到 Git 服务器的 /home/git/.ssh/authorized_keys 文件中，这样就不需要在每次部署的时候都进行 Git 的身份校验了（少输入一次密码，减少点麻烦）。
3. 修改项目的 config/deploy.rb 文件：

```ruby
  # 将下面这行源地址的设置改为新库的地址
  set :repository, "ssh://git@127.0.0.1/home/git/#{App}"
```

以上修改做完后，即可使用mina正常部署了，迁移顺利完成：）
