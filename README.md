# students-management
[在线测评系统技术框架.docx](https://github.com/user-attachments/files/34914712/default.docx)
[学生管理系统.docx](https://github.com/user-attachments/files/34960339/default.docx)
在线测评系统技术框架
1 系统概述
在线测评系统（Online Judge System，简称OJ）是一套基于Web的自动化代码评测平台，旨在为程序设计课程教学和算法竞赛训练提供高效、稳定、安全的自动化评测服务。用户可以在线提交源代码，系统自动完成编译、运行和结果比对，并将评测结果实时反馈给用户。
系统采用前后端分离的架构设计，前端基于Vue.js框架构建用户界面和管理后台，后端使用PHP Laravel框架提供RESTful API服务，数据库采用MySQL存储业务数据，Redis作为缓存和任务队列中间件。判题服务采用独立的Python worker进程架构，通过Redis队列异步接收提交任务，在沙箱环境中隔离编译运行用户代码。
2 总体架构设计
系统采用前后端分离的四层架构设计，自上而下分别为表现层、业务逻辑层、数据存储层和判题服务层。各层职责清晰、耦合度低，支持独立部署和水平扩展。
2.1 表现层
表现层即前端应用，基于Vue.js 2.x构建，分为用户端和管理后台两个独立的单页应用（SPA）。前端通过HTTP协议与后端API交互，使用JSON格式传输数据。前端负责页面渲染、用户交互和数据展示，不包含业务逻辑。
前端技术栈包括：Vue.js 2.x（核心框架）、Vue Router（路由管理）、Vuex（状态管理）、Axios（HTTP请求库）、CodeMirror（代码编辑器）、ECharts（数据可视化）、Bootstrap 4（管理后台UI框架）。
2.2 业务逻辑层
业务逻辑层即Web服务端，基于PHP Laravel 9.x框架开发，运行在Nginx 1.20和PHP-FPM 8.1之上。该层负责处理HTTP请求、身份认证、权限校验、业务逻辑处理和数据持久化。Web服务端不直接执行用户代码，而是将判题任务异步推送到消息队列。
后端核心特性包括：Eloquent ORM（对象关系映射）、RESTful路由设计、中间件机制（身份认证、权限校验、跨域处理）、队列系统（异步任务处理）、数据库事务支持。
2.3 数据存储层
数据存储层包括MySQL 8.0关系型数据库和Redis 7.0内存数据库。MySQL存储用户、题目、提交、竞赛等核心业务数据；Redis承担缓存和消息队列双重角色，缓存热点题目数据、用户会话信息和排行榜数据，同时作为Web服务层与判题服务层之间的通信桥梁。
2.4 判题服务层
判题服务层是系统的特色模块，由独立的Python 3.10 worker进程组成。worker从Redis队列中消费判题任务，在沙箱环境中编译运行用户代码，比对测试用例输出，最后将结果写回数据库。判题服务与Web服务完全解耦，独立部署、独立扩展。
系统共部署9个判题worker进程，其中4个default-queue负责常规任务处理，5个judge-queue专门负责代码判题。通过多进程并行处理提高判题吞吐量，单个进程崩溃不影响整体服务稳定性。
3 技术栈详解
3.1 后端技术栈
技术组件	版本	用途说明
PHP	8.1	后端开发语言
Laravel	9.x	Web应用框架，MVC架构
Eloquent ORM	—	对象关系映射，数据库操作
Nginx	1.20	Web服务器，反向代理
PHP-FPM	8.1	PHP FastCGI进程管理器
MySQL	8.0	关系型数据库
Redis	7.0	缓存与消息队列
3.2 前端技术栈
技术组件	版本	用途说明
Vue.js	2.x	前端核心框架，响应式数据绑定
Vue Router	—	前端路由管理
Vuex	—	全局状态管理
Axios	—	HTTP请求库
CodeMirror	—	代码编辑器，语法高亮
ECharts	—	数据可视化图表
Bootstrap	4	管理后台UI框架
3.3 判题服务技术栈
技术组件	版本	用途说明
Python	3.10	判题服务开发语言
Redis客户端	—	队列任务消费
subprocess模块	—	子进程管理，编译运行
resource模块	—	系统资源限制（CPU/内存/进程数）
GCC/G++	—	C/C++编译器
Python解释器	3.10	Python代码执行
JDK	—	Java编译运行环境
4 数据库设计
系统数据库采用MySQL 8.0，主要包含以下核心数据表：
表名	中文名称	主要字段	说明
users	用户表	id, username, password, school, class, name, email, role	存储用户基本信息，密码使用bcrypt加密
problems	题目表	id, title, description, input, output, sample, time_limit, memory_limit, tags	存储题目信息，支持标签分类
solutions	提交记录表	id, user_id, problem_id, code, language, result, time, memory, created_at	存储用户提交记录和评测结果
contests	竞赛表	id, name, description, start_time, end_time, freeze_ratio	存储竞赛信息，支持封榜比例配置
contest_problems	竞赛题目表	contest_id, problem_id, order_num	竞赛与题目的多对多关联
announcements	公告表	id, title, content, is_top, created_at	存储系统公告
tags	标签表	id, name	题目分类标签
5 判题机制设计
判题机制是在线评测系统的核心，其设计直接影响系统的正确性、安全性和性能。系统判题流程分为五个步骤：
5.1 任务入队
用户提交代码后，Web服务将提交记录写入数据库（状态为"等待判题"），同时将任务信息（提交ID、题目ID、编程语言、源代码）序列化为JSON推入Redis的judge队列。
5.2 任务消费
判题worker进程使用阻塞式BRPOP命令从队列中获取任务，没有任务时进程休眠等待，不消耗CPU资源。获取任务后，worker将提交状态更新为"正在判题"。
5.3 编译
worker根据编程语言选择对应的编译器：C语言使用gcc，C++使用g++，Python直接解释执行无需编译，Java使用javac。编译过程设置超时限制，防止恶意代码导致编译挂起。编译失败则直接返回"编译错误"。
5.4 运行与比对
编译成功后，worker读取该题目的所有测试数据，逐组运行用户程序。每组测试用例的运行都在独立的沙箱进程中进行，设置严格的CPU时间限制和内存限制。运行完成后，将程序输出与标准输出进行逐字节比对（忽略行尾空格差异）。全部通过则结果为Accepted，任一组失败则根据失败类型返回对应错误。
5.5 结果回写
worker将最终评测结果、最大运行时间、最大内存占用更新到数据库，并清理临时文件。用户可以在提交记录页看到最终结果。
6 沙箱安全机制
沙箱隔离是判题安全的关键。系统通过以下多维度资源限制防止恶意代码攻击：
（1）CPU时间限制：使用resource模块的RLIMIT_CPU限制进程CPU执行时间，超时后系统自动发送SIGKILL信号终止进程。
（2）内存限制：使用RLIMIT_AS限制进程虚拟内存大小，防止大量内存分配导致服务器内存耗尽。
（3）进程数限制：限制进程可创建的子进程数量，防止fork炸弹攻击。
（4）文件大小限制：使用RLIMIT_FSIZE限制进程输出文件大小，防止恶意代码生成超大文件占满磁盘。
（5）编译超时控制：编译过程设置独立超时，防止恶意代码导致编译器挂起。
7 部署架构
系统采用经典的LNMP（Linux + Nginx + MySQL + PHP）部署架构，配合Redis缓存队列和Python判题服务。部署环境为Ubuntu 22.04 LTS服务器，配置Intel Xeon 4核CPU、8GB DDR4内存。
Nginx作为Web服务器，负责接收HTTP请求、处理静态资源、将动态请求转发给PHP-FPM处理。PHP-FPM管理PHP进程池，处理Laravel应用请求。MySQL存储业务数据，Redis提供缓存和队列服务。9个Python判题worker进程作为独立服务运行，通过Supervisor进行进程守护和自动重启。
8 安装运行指南
系统采用前后端分离的LNMP架构，结合Redis与Python独立判题服务。整体部署需按顺序分为五个阶段。

第一阶段：服务器环境准备
需准备运行Ubuntu 22.04 LTS系统的服务器。首先更新系统软件源，并安装Git、Curl、解压工具等基础组件，确保后续软件下载正常。

### 第二阶段：Web与数据核心组件搭建
1. 安装Nginx：作为Web服务器接收HTTP请求，静态资源直接返回，动态请求转发至PHP。
2. 安装PHP 8.1及扩展：安装PHP本体、PHP-FPM及连接数据库、处理XML等必需的扩展。安装Composer用于管理PHP依赖库。
3. 安装MySQL 8.0：存储用户、题目、提交记录等核心业务数据，安装后需创建数据库及专用账号。
4. 安装Redis 7.0：承担缓存与消息队列双重角色，既缓存热点数据，又作为Web后端与Python判题服务之间的通信桥梁。

第三阶段：Laravel后端部署
1. 拉取代码与安装依赖：将Laravel代码部署至服务器，使用Composer安装依赖库。
2. 配置环境变量：复制`.env`文件，填入MySQL数据库凭证、Redis地址，并将队列驱动设置为Redis。
3. 初始化数据库：生成应用密钥，执行数据库迁移命令，自动创建所需的数据表结构。
4. 设置目录权限：确保Web服务器用户（通常为`www-data`）拥有`storage`和`bootstrap/cache`目录的写入权限，避免出现500错误。
5. 配置Nginx站点：修改Nginx配置，将网站根目录指向Laravel项目下的`public`目录，并将PHP请求转发至PHP-FPM处理。

第四：Python判题服务部署
1. 安装编译环境：安装`gcc`、`g++`（用于C/C++）、`default-jdk`（用于Java）以及Redis的Python客户端库。
2. 配置沙箱机制：判题worker通过Python的`resource`模块限制用户代码的CPU时间、内存大小、进程数及文件大小，防止恶意代码耗尽服务器资源。
3. 配置Supervisor进程守护：安装Supervisor工具，编写配置文件启动9个判题进程（4个处理默认任务，5个处理代码判题）。设置崩溃自动重启，确保判题服务持续运行。
第五阶段：Vue前端部署
1.构建前端静态文件：在前端项目目录执行依赖安装，随后运行构建命令，生成`dist`静态资源文件夹。
2. 配置Nginx托管与反向代理：将`dist`内容放置于服务器静态目录，修改Nginx配置托管前端文件，并配置反向代理将API请求转发至Laravel后端。
