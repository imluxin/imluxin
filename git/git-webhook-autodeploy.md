# 利用 WebHook 实现 Git 代码自动部署

## 前提：
1. 本文基于LNMP PHP代码自动部署；
2. 必须使用www用户；
3. 本文Git平台为 腾讯工蜂；

## 操作步骤：
1. 目标服务器切www用户：
```su www```
切换用户（www）时：```this account is currently not available```
``` cat /etc/passwd | grep www # 查看是否为 /sbin/nolgin ```
解决办法：
```vim /etc/passwd```
修改 ```/sbin/nolgin``` 为 ```/bin/bash```

2. 在目标服务器上生成 ssh 公钥，生成公钥在 /home/www/.ssh 文件夹下：
```ssh-keygen -t rsa -C "your_name"```
![公钥密钥.png](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/26/171b626ed219568c~tplv-t2oaga2asx-image.image)

3. 部署公钥至Git平台：
公钥位置：```/home/www/.ssh/id_rsa.pub```
在工蜂平台新增SSH公钥：工蜂平台 -> 个人设置 -> SSH密钥 -> 添加SSH密钥， 复制粘贴 ```id_rsa.pub``` 中内容；
![部署SSH公钥](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/26/171b626ed2905070~tplv-t2oaga2asx-image.image)

4. 使用 www 用户 git clone 代码。
需要使用 SSH 地址 ```git@github.com:someaccount/someproject.git```，若之前是使用 https 地址更新，则使用以下命令切换至 SSH 地址：
```git remote set-url origin git@github.com:someaccount/someproject.git```
针对非www用户已部署代码，可修改其用户权限至 www：
```chown -R www:www code_folder```

5. 安装 githook.php 文件至外网可以访问的位置，如 http://test.com/githook.php
```
<?php
/* security */
$token   = 'token12345';
$project = '/home/wwwroot/default/test/test';
//ip地址为gitlab服务器请求地址
$access_ip = [];

/* get user token and ip address */
$client_token = isset($_SERVER['HTTP_X_TOKEN']) ? $_SERVER['HTTP_X_TOKEN'] : '';
$client_ip    = $_SERVER['REMOTE_ADDR'];
//查询服务器运行的php-fpm用户和文件所属权限是否一致
//print_r($_SERVER);

//文件记录日志
/* create open log */
$fs = fopen($project . '/storage/webhook.log', 'a');
fwrite($fs, '===================================start====================================' . PHP_EOL);
fwrite($fs, 'Request on [' . date("Y-m-d H:i:s") . '] from [' . $client_ip . ']' . PHP_EOL);

/* test token */
if ($client_token !== $token) {
    echo "error 403";
    fwrite($fs, "Invalid token [{$client_token}]" . PHP_EOL);
    exit(0);
}

/* test ip*/
// if ( ! in_array($client_ip, $access_ip))
// {
// echo "error 503";
// fwrite($fs, "Invalid ip [{$client_ip}]".PHP_EOL);
// exit(0);
// }


/* get json data */
$json = file_get_contents('php://input');
$data = json_decode($json, true);

/* get branch */
$branch              = $data["ref"];
$total_commits_count = $data["total_commits_count"];
fwrite($fs, '=======================================================================' . PHP_EOL);
/* if you need get full json input */
//fwrite($fs, 'DATA: '.print_r($data, true).PHP_EOL);

/* branch filter */
if ($branch === 'refs/heads/develop' && $total_commits_count > 0) {
    /* if master branch*/
    fwrite($fs, 'BRANCH: ' . print_r($branch, true) . PHP_EOL);
    fwrite($fs, '=======================================================================' . PHP_EOL);

    /* then pull master */
    // $result = shell_exec("cd {$project} && git pull 2>&1");
    $result = shell_exec("cd {$project} && git pull 2>&1");

    fwrite($fs, 'RESULT: ' . print_r($result, true) . PHP_EOL);
	fwrite($fs, '===================================end====================================' . PHP_EOL);
    $fs and fclose($fs);
}
```
6. 在工蜂平台添加 hook，注意秘密令牌需要与 githook.php 文件中的 token 一至
位置：项目 -> 设置 -> 高级设置 -> 网络回调勾子
![工蜂平台添加回调hook](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/26/171b626ed27f7846~tplv-t2oaga2asx-image.image)

7. 部署完成，当 git 有 push 更新时，目标服务器端将自动拉取；
