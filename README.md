## 安装 Git

    Windows：下载并安装 git.
    Mac：使用 Homebrew, MacPorts 或下载 安装程序 安装。
    Linux (Ubuntu, Debian)：sudo apt-get install git-core
    Linux (Fedora, Red Hat, CentOS)：sudo yum install git-core

## 安装 Node.js

安装 Node.js 的最佳方式是使用 nvm。

cURL:

$ curl https://raw.github.com/creationix/nvm/master/install.sh | sh

Wget:

$ wget -qO- https://raw.github.com/creationix/nvm/master/install.sh | sh

安装完成后，重启终端并执行下列命令即可安装 Node.js。

$ nvm install stable

## 安装Hexo

`npm install -g hexo-cli`

### 安装Plugin:

`npm install hexo-deployer-git --save`

### 安装hexo-server

`npm install hexo-server --save`

### 初始化博客目录

- `cd D:/blog`
- `hexo init`

好啦，至此，全部安装工作已经完成！blog就是你的博客根目录，所有的操作都在里面进行。

生成静态页面

`hexo generate（hexo g也可以）`

本地启动，命令：

`hexo s -g`

浏览器输入http://localhost:4000

## 配置Github

建立Repository

建立与你用户名对应的仓库，仓库名必须为【your_user_name.github.io】，固定写法

## **安装hexo-admin** 

`npm install --save hexo-admin`

将可以可视化编辑markdown文章

`运行 hexo s -d`

`浏览器输入 http://localhost:4000/admin`





