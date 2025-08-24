# Mysql-test
一、
  MySQL-Test 是 MySQL 官方维护的 自动化测试框架，核心用于验证 MySQL 服务器的功能正确性、兼容性及稳定性。通过 测试用例（.test） 定义执行逻辑，对比 预期结果（.result） 输出判定测试 pass/fail，支撑以下场景：
  新特性开发的 回归测试（如事务、复制功能）；
  补丁 / 版本升级的 兼容性验证；
  社区贡献代码的 准入测试。
二、架构设计：目录与代码模块
  MySQL-Test 以 目录分层 + 脚本驱动 为核心架构，关键组成如下：
  2.1 核心目录结构
    目录	作用	关键文件示例
    t/	                存储测试用例（.test），按功能模块分类（如 t/transaction/ 存事务测试）	t/select.test、t/replication/
    r/	                存储预期结果（.result），与 t/ 用例一一对应                        	r/select.result
    suite/	            测试套件定义（聚合关联用例，如 suite/main 包含核心功能测试）	        suite/main.test（套件清单）
    lib/	              公共逻辑库（Perl 脚本，封装通用操作如环境清理、连接数据库）           	lib/mysql-test-lib.pl
    mysql-test-run.pl  	测试执行入口（Perl 脚本，解析参数、调度用例、对比结果）	              核心执行逻辑入口
  2.2 代码模块分工
    1.执行层：mysql-test-run.pl（Perl 实现）
    解析命令行参数（如 --suite=main 指定套件、--user=root 配置连接）；
    启动 临时 MySQL 实例（独立端口，避免干扰生产环境）；
    遍历用例，调用 mysqltest 工具执行测试。
    2.执行引擎：mysqltest（C 实现的测试客户端）
    解析 .test 文件，区分 SQL 语句（如 SELECT * FROM t1;）和 控制指令（如 --source lib/mysql-test-lib.pl 引入公共逻辑）；
    与测试实例建立连接，执行 SQL 并捕获输出；
    将实际输出写入 out/ 目录（如 out/select.out）。
   3. 结果对比层：diff 命令 + 自定义逻辑
    对比 out/*.out 与 r/*.result，输出差异（如行内容不一致、顺序错误）；
    标记测试结果（pass/fail），汇总到最终报告。
三、运行机制：从代码到执行的全流程
  以 mysql-test-run.pl select 执行单条用例为例，拆解核心流程：
  3.1 步骤 1：参数解析与环境初始化
    参数解析：mysql-test-run.pl 读取 --user、--password 等参数，初始化数据库连接配置。
    启动测试实例：调用 mysqld 启动 临时服务器（默认端口 13306，数据存储于临时目录），执行 mysql_install_db 初始化系统表。
  3.2 步骤 2：用例执行（mysqltest 工作流）
    1.对 t/select.test 文件：
    sql
    --disable_query_log  # 控制指令：关闭查询日志输出
    --source lib/mysql-test-lib.pl  # 引入公共库（如环境清理函数）
    CREATE TABLE t1 (id INT);
    INSERT INTO t1 VALUES (1);
    SELECT * FROM t1;
    --enable_query_log   # 控制指令：恢复查询日志输出
    2.指令解析：mysqltest 逐行处理：
    以 -- 开头的为 控制指令（如 --disable_query_log 关闭日志记录）；
    无 -- 开头的为 SQL 语句（如 CREATE TABLE、SELECT）。
    3.执行逻辑：
    SQL 语句通过客户端连接发送到测试实例执行；
    控制指令调用内置逻辑（如 --source 加载 Perl 库函数）。
  3.3 步骤 3：结果对比与清理
  输出捕获：mysqltest 将 SQL 执行结果写入 out/select.out，内容如下：
  plaintext
  id
  1
  差异对比：调用系统 diff 命令，对比 out/select.out 与 r/select.result：
  若完全一致，标记 select.test 为 pass；
  若存在差异（如 SQL 输出多一行），输出差异详情（如 Line 2: Expected '1', Got '2'），标记为 fail。
  环境清理：关闭测试实例，删除临时数据目录。
四、测试用例开发：从编写到验证
  4.1 用例分类与场景
    用例类型	目标场景	示例用例
    功能测试	验证 SQL 语法、存储引擎特性	事务提交 / 回滚、索引查询
    兼容性测试	验证版本升级后行为一致性（如 SQL 语法兼容）	旧版本语法在新版本的执行结果
    压力测试	验证高并发下稳定性（需结合 mtr 并发参数）	多线程插入数据
  4.2 用例编写规范
    （1）.test 文件：SQL + 控制指令
    示例：事务回滚测试（t/transaction_rollback.test）
    sql
    --source lib/mysql-test-lib.pl  # 引入公共库（含环境清理逻辑）
    --disable_query_log
    CREATE DAUNT(*) FROM t1;  # 验证数据未插入
    （2）.result 文件：预期输出
    对应 r/transaction_rollback.result：
    plaintext
    COUNT(*)
    0TABASE test_db;
    USE test_db;
    CREATE TABLE t1 (id INT);
    --enable_query_log
    
    BEGIN;  # 启动事务
    INSERT INTO t1 VALUES (1);
    ROLLBACK;  # 回滚事务
    
    SELECT CO
  4.3 高级技巧：控制指令扩展
    控制指令	作用
    --echo "Message"	打印调试信息（如 --echo "Start testing transaction"）
    --error 1062	预期 SQL 执行报错（如唯一键冲突，错误码 1062）
    --replace_result	替换输出中的动态值（如时间戳 → [TIMESTAMP]，避免结果波动）
五、框架扩展：定制化与集成
  5.1 代码级扩展：修改 mysql-test-run.pl
    场景：添加自定义参数 --custom-tag 标记测试用例。
    修改参数解析逻辑：在 mysql-test-run.pl 中添加：
    perl
    GetOptions(
        ...,
        "custom-tag=s" => \$custom_tag,  # 新增参数
    );
    
    扩展用例筛选逻辑：根据 $custom_tag 过滤 t/ 目录下的用例（如仅执行含 @custom 标记的用例）。
  5.2 集成 CI/CD 流水线（以 Jenkins 为例）
    核心脚本（Jenkins Pipeline）：
    groovy
    pipeline {
        agent any
        stages {
            stage('Test') {
                steps {
                    sh './mysql-test-run.pl --suite=main --user=root --password=xxx'  # 执行测试
                    junit '**/test-results.xml'  # 解析测试报告（需提前配置报告输出格式）
                }
            }
        }
    }
  5.3 自定义逻辑库：扩展 lib/ 目录
    场景：封装通用的 “数据清理” 逻辑。
    在 lib/custom-lib.pl 中添加：
    perl
    sub clean_up {
        my ($dbh) = @_;  # $dbh 是数据库连接句柄
        $dbh->do("DROP DATABASE IF EXISTS test_db");
    }
    
    在 .test 中引入：
    sql
    --source lib/custom-lib.pl
    --perl use custom-lib; clean_up($dbh);  # 调用 Perl 函数

七、总结
  MySQL-Test 以 “用例定义 → 执行引擎 → 结果对比” 为核心流程，通过灵活的控制指令和可扩展的 Perl 脚本，支撑 MySQL 从功能到性能的全链路测试。掌握其 代码级原理（如 mysql-test-run.pl 的调度逻辑、mysqltest 的指令解析），结合 场景化用例开发 与 CI/CD 集成，可高效保障数据库质量。
