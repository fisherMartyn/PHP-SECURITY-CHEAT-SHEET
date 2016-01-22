# 介绍

这篇博客是翻译自OWSAP的<a href ="https://www.owasp.org/index.php/PHP_Security_Cheat_Sheet">PHP Security Cheat Sheet</a>，你可以查看原文。

本文章的目的是提供基本的PHP安全建议，主要针对开发者和系统管理员。需要注意的是这里提到的安全建议可能并不足已支撑安全的网络应用，而只是一个组成部分。

## PHP介绍

根据W3 Techs的统计，PHP是最流行的Server端编程语言，大约81.8%的Web Server用PHP部署(任何语言都要为自己吹一把的)。

与其它语言不同，PHP即是一门语言，也是一个web框架，在PHP语言中内嵌了典型的web框架的特性。跟其它语言一样，PHP背后也有庞大的社区软件库，贡献了PHP安全和其它各个方面的特性。所有的这三个方面（语言、框架和开源库）都是在构建安全的web应用中需要考虑的因素。

PHP一直在发展中，而不是固定的设计，因此你很可能写出了不安全的PHP程序。为了让应用更安全，你需要了解所有的陷阱。

### 语言相关

#### 弱类型

PHP是弱类型的，意味着不正确的数据类型会被自动转换。这个特性经常会隐藏开发中的错误信息或者不规范数据的注入，导致系统风险性增加。(例子在下面的输入处理部分)。

尽量使用非类型转换的函数和操作符(例如用===替换==)。但并非所有的操作符都有严格类型检查的版本（例如<=和>=），另外很多内置的函数(例如in_array)默认进行类型转换，给编写安全的代码带来了挑战。

#### 异常和错误处理

几乎所有的PHP内置函数很多PHP库，不支持异常，而是用其它的报警机制(例如notice)，允许出错的代码继续运行。这也会隐藏很多bug。很多其它的语言、大部分高级语言，相对于PHP，开发者代码错误或者开发者未处理的运行时错误，都会导致程序停止执行，从而保护程序。

考虑下面的程序，程序的目的是检查一个username是不是在数据库的黑名单中，以限制访问。

    $db_link = mysqli_connect('localhost', 'dbuser', 'dbpassword', 'dbname');

    function can_access_feature($current_user) {
       global $db_link;
       $username = mysqli_real_escape_string($db_link, $current_user->username);
       $res = mysqli_query($db_link, "SELECT COUNT(id) FROM blacklisted_users WHERE username = '$username';");
       $row = mysqli_fetch_array($res);
       if ((int)$row[0] > 0) {
           return false;
       } else {
           return true;
       }
    }

    if (!can_access_feature($current_user)) {
       exit();
    }
    // Code for feature here

上面的代码可能产生各种各样的运行时错误，例如由于用户名和密码错误或者数据库server宕机可能导致的数据库连接失败、或者连接可能被server断开。在上述情况下，`mysqli`函数会抛出notice或者warnings，但不会抛出异常或者fatal error，代码会继续执行。变量`$row`会成为`NULL`，基于若类型转换`(int)$row[0]`会成为0，最终`can_access_feature`会返回`true`，用户会获得访问权限，不管用户是否在黑名单中。

如果使用native的数据库API，需要在各个地方添加错误检查。然而由于这需要额外的工作、又如此容易被忽略，所以它是'默认不安全的'。这就是为什么访问数据库通常要采用<a href="https://secure.php.net/manual/en/intro.pdo.php">PHP Data Objects (PDO)</a>和其错误处理<a href="https://secure.php.net/manual/en/pdo.error-handling.php"> ERRMODE_WARNING or ERRMODE_EXCEPTION flags </a>，除非有特殊的原因，不会使用native API和错误检查工作。

使用<a href="http://www.php.net/manual/en/function.error-reporting.php">error_reporting</a>函数将错误优先级设置越高越好，修复warnings，让系统越鲁棒越好。

#### php.ini

PHP代码的行为经常强依赖于很多配置文件的设置，例如基本的错误处理配置等。这使得很难写出在所有场景下正常运行的代码。不同的库依赖不同的设置，使得很难使用第三方的代码，这在后面的'配置'部分也会提到。

#### 无用的buildin函数

PHP提供了大量的内嵌函数，例如`addslashes`、`mysql_escape_string`、`mysql_real_escape_string`等。这些试图提供安全的特性的函数经常是有bug的，并不能解决安全性的问题。许多内嵌的函数已经被弃用或者迁移，但由于向下兼容的规则，这些函数会保留很长时间。

PHP会提供`array`的数据结构，在各种代码中随意使用，但由于混合了数组和字典，经常导致混淆。这种混淆经常导致甚至是有经验的程序员引入重大的安全隐患，例如<a href="https://www.drupal.org/SA-CORE-2014-005">Drupal SA-CORE-2014-005</a> 和 <a href="http://cgit.drupalcode.org/drupal/commit/?id=26a7752c34321fd9cb889308f507ca6bdb777f08">the patch</a>。

### 框架相关

#### URL路由

PHP默认的路由机制是使用文件目录中`.php`的后缀，这带来很多风险

* 文件上传没有进行文件名过滤导致的远程执行风险（换句话说，服务化器执行了外部的代码）。在接收远程文件上传时，请确保文件名和内容经过过滤。

* 源代码和配置文件存储在公共可访问的目录中，可以被下载。误配置或者缺少配置可能会导致源代码或者包含密码等信息的配置文件被攻击者随意下载（换句话说，服务器暴露了私密的信息）。你可以使用`.htaccess`来限制访问，但这并不是最完美的方案，因为它是'默认不安全的'，也没有可替代的方案。

* URL路由机制只是模块系统。这意味着攻击者可能可以使用非法的文件名作为进入点，带来认证机制可能被完全绕过的风险。这在PHP中极其容易，因为有全局可访问的请求信息（例如`$_GET`），所以文件层面的代码即可操作整个请求、而不是在固定的函数中进行请求处理。

* URL路由机制的缺失导致开发者开发各自的路由方法。这往往是不安全的，容易导致请求处理函数未进行合适的请求限制。

#### 输入处理

PHP将HTTP的输入转换成了数组、而不是简单字符串。这会导致对于数据的混淆，容易导致安全风险。例如下面的代码是一个密码重置的一次性机制判断

    $supplied_nonce = $_GET['nonce'];
    $correct_nonce = get_correct_value_somehow();

    if (strcmp($supplied_nonce, $correct_nonce) == 0) {
        // Go ahead and reset the password
    } else {
       echo 'Sorry, incorrect link';
    }

如果攻击者使用如下的查询字符串

     http://example.com/?nonce[]=a

会导致`$supplied_nonce`成为数组、`strcmp()`函数会返回NULL（连异常都不会抛出），由于若类型转换和没有使用`===`操作符，校验成功，攻击者可以在不提供原密码的情况下进行密码修改。

相似的case和`array`结构导致的混淆事故，可以在<a href="https://www.drupal.org/SA-CORE-2014-005"> Drupal SA-CORE-2014-005 </a>，参见<a href ="http://www.zoubi.me/blog/drupageddon-sa-core-2014-005-drupal-7-sql-injection-exploit-demo"> example exploit</a>。

#### 模板语言

PHP本质上是一门模板语言，但它默认并没有提供HTML转义，在web应用中带来极大使用困难，可以参见下面的XSS部分。

#### 其它不足

另外有很多web框架应该支持的重要特性，例如默认开启CSRF的保护机制。因为PHP只是提供了一个基本的、功能足够的特性来创建web站点，很多不了解CSRF保护机制的人就仅使用了PHP基本的功能。

### 第三方PHP代码



# 配置

# 不可靠的数据

# 数据库

# 其它注入

# XSS

# CSRF

# 认证和SESSION管理

# 配置和部署

# 来源相关
