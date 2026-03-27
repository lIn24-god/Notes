把本地仓库上传到github一般步骤：

## git命令行方法

1. 在本地初始化git仓库
   `cd /path/to/your/project   # 进入项目目录
   `git init                   # 初始化一个空的 Git 仓库`
2. 添加所有文件到暂存区
   `git add .                  # 添加当前目录下所有文件到暂存区`
3. 提交到本地仓库
   `git commit -m "first commit"`
4. 在 GitHub 上创建一个新仓库
- 登录 GitHub，点击右上角“+” → “New repository”。
- 填写仓库名称，**不要勾选**“Initialize this repository with a README”。
- 点击“Create repository”。
5. 添加远程仓库地址
   `git remote add origin https://github.com/你的用户名/仓库名.git`
6. 推送到GitHub
   `git push -u origin main   # 将本地 main 分支推送到远程 origin`

## GitHubDeskTop图形化操作

### 1. 安装并登录 GitHub Desktop

- 从 [desktop.github.com](https://desktop.github.com/) 下载安装。
    
- 打开后用 GitHub 账号登录，它会自动关联你 GitHub 上的仓库。
    

### 2. 添加本地仓库到 GitHub Desktop

- 打开 GitHub Desktop，点击菜单 **File** → **Add Local Repository**。
    
- 在弹出的窗口中，点击 **Choose…** 选择你的项目文件夹。
    
- 如果该文件夹还不是 Git 仓库，GitHub Desktop 会提示“是否创建仓库”，点击 **Create a Repository**。
    
- 填写仓库信息（名称、描述），注意 **Local path** 就是你的项目路径，可以勾选 **Initialize this repository with a README**（不过如果你已有文件，不勾选也行）。
    
- 点击 **Create Repository**。
    

现在你的本地项目已经被 GitHub Desktop 管理了，界面上会显示未提交的文件。

### 3. 提交本地更改

- 在左侧 **Changes** 列表中，勾选你要提交的文件（一般全选）。
    
- 在下方的 **Summary** 输入提交说明，例如 “first commit”。
    
- 点击 **Commit to main**。
    

### 4. 发布到 GitHub

- 点击顶部工具栏的 **Publish repository** 按钮（如果按钮显示为 **Push origin**，说明仓库已关联过，直接推送即可）。
    
- 在弹出的窗口中，确认 **Name** 和 **Description**，并确保 **Keep this code private** 的勾选状态符合你的要求（不勾选即为公开仓库）。
    
- 点击 **Publish Repository**。
    

等待几秒，你的代码就会推送到 GitHub，并在 GitHub Desktop 中显示 “Successfully published”。
