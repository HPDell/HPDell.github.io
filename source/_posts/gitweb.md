---
title: GitWeb 的安装和配置
date: 2017-12-20 16:27:00
tags: Git
categories:
    - Git
---
> 本文转载并整合自
>
> - [梦觉知晓的博客](http://blog.cs-tao.cc/2017/10/19/gitweb安装和配置/)
> - [树莓派上搭建 gitweb](http://gengcheng.cn/archives/5.html)
> 
> 有修改。

Git提供了一个CGI脚本，来搭建一个网页端Git版本库可视化工具，这个工具就是GitWeb。本文提供基于Ubuntu Mate的搭建过程，并在树莓派上试验成功。
<!-- more -->
# 基本搭建
基本的搭建GitWeb的步骤如下：

1. 安装GitWeb和Apache
    ``` bash
    sudo apt-get install gitweb lighttpd highlight markdown
    ```
1. 修改文件：`/etc/gitweb.conf`，该文件功能是设置项目集根目录、临时文件目录、布局文件位置及资源文件位置等：
    ``` bash
    $projectroot = "/home/git/repositories";            # Git仓库的根目录
    $git_temp = "/tmp";                                 # 临时文件鲁姆
    $projects_list = $projectroot;
    @stylesheets = ("../gitweb/static/gitweb.css");     # 页面的样式表
    $javascript = "../gitweb/static/gitweb.js";         # 页面的脚本文件
    $logo = "../gitweb/static/git-logo.png";            # 页面Logo
    $favicon = "../gitweb/static/git-favicon.png";
    @diff_opts = ();
    $feature{'highlight'}{'default'} = [1];             # 开启代码高亮支持
    ```
1. 修改`/usr/share/gitweb/gitweb.cgi`：
    ``` bash
    our $projectroot = "/home/git"; # 修改为Git仓库的根目录
    ```
1. 修改`/etc/lighttpd/lighttpd.conf`，添加CGI支持：
    ``` bash
    url.redirect += ( "^/gitweb$" => "/gitweb/" )
    alias.url += ("/gitweb/" => "/usr/share/gitweb/")
    $HTTP["url"] =~ "^/gitweb/" {
        cgi.assign = (".cgi" => "/usr/bin/perl")
        server.indexfiles = ("index.cgi")
    }
    ```
1. 创建认证文件：
    ``` bash
    htpasswd -c 认证文件位置 用户名
    ```
1. 配置Apache，使我们可以访问到GitWeb。修改`/etc/apache2/conf-available/gitweb.conf`文件，该文件的功能是指定CGI文件位置、认证文件位置等。修改后的文件如下（原始文件会有几行和这个重复，直接查找即可：
    ```
    <IfDefine ENABLE_GITWEB>
        Alias /gitweb /usr/share/gitweb

        <Directory /usr/share/gitweb>
            Options +FollowSymLinks +ExecCGI
            AddHandler cgi-script .cgi
            AuthType Basic
            AuthName "Restricted Content"
            AuthUserFile /home/git/.htpasswd # 这里是认证文件的位置
            Require  valid-user
        </Directory>
    </IfDefine>
    ```
1. 启动CGI支持：
    ``` bash
    sudo lighty-enable-mod cgi
    sudo service apache2 restart
    ```

# 附加功能

## 更改GitWeb主题

克隆别人写好的GitWeb主题即可，位置位于GitHub的[kogakure/gitweb-theme](https://github.com/kogakure/gitweb-theme)仓库中。
``` bash
cd /usr/share/gitweb
git clone https://github.com/kogakure/gitweb-theme.git
mv static static.bak
mv gitweb-theme static
```

## 增加Mardown格式的README.md文件的支持
编辑/usr/share/gitweb/gitweb.cgi，在sub git_sumary（大约在第6400行，可使用编辑器的搜索功能）中加入以下代码：
``` bash
if (!$prevent_xss) {
    $file_name = "README.md";
    my $proj_head_hash = git_get_head_hash($project);
    my $readme_blob_hash = git_get_hash_by_path($proj_head_hash, "README.md", "blob");
if ($readme_blob_hash) { # if README.md exists                                                                                                                                                      
    print "<div class=\"header\">README.md</div>\n";
    print "<div class=\"readme page_body\">";                                                                                                  
    my $cmd_markdownify = $GIT . " " . git_cmd() . " cat-file blob " . $readme_blob_hash . " | markdown |";
    open FOO, $cmd_markdownify or die_error(500, "Open git-cat-file blob '$hash' failed");
    while (<FOO>) {
        print to_utf8($_);
    }
    close(FOO);

    print "</div>";
    }
}
```

## 第二种认证模块添加方式

1. 添加认证模块
    ``` bash
    apt-get install apache2-utils
    sudo lighty-enable-mod auth
    sudo /etc/init.d/lighttpd restart
    ```
1. 创建用户密码文件
    ``` bash
    cd /etc/lighttpd
    htdigest -c lighttpd.user  'Enter Password' your_name
    ```
1. 修改`/etc/lighttpd/lighttpd.conf`配置文件，添加以下内容
    ``` bash 
    auth.backend = "htdigest"
    auth.backend.htdigest.userfile = "/etc/lighttpd/lighttpd.user"
    auth.require = ( "/gitweb" =>
    (
        "method" => "digest",
        "realm" => "Enter Password",
        "require" => "user=your_name"
    ))
    ```

# 注意事项

## GitWeb文件权限问题
如果提示如下错误：

{% asset_img interval-server-error.png%}

参考下面的解决方式：

> 需要将`/usr/share/gitweb`文件夹下的文件和文件夹设置正确的权限，<其他用户>必须有读取文件权限和执行文件权限。缺少读文件的权限服务器会返回”Internal Server Error(500)”错误，缺少执行文件的权限服务器会返回”Forbidden(403)”错误。读取文件权限为4，执行文件权限为1，也就是说<其他用户>的权限至少为’5’。如下，笔者设置的’755’权限的最后一个’5’对应<其他用户>的权限。

即将`/usr/share/gitweb`文件夹如下设置权限：
``` bash
cd /usr/share/gitweb/
sudo chmod -R 755 .
```

## 旧版本Apache配置文件
旧版本Apache的/etc/apache2/conf.d/gitweb和新版本的/etc/apache2/conf-available/gitweb的是同一个目录。按照同样的方法设置即可。