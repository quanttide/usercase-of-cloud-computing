# AWS 使用

## 项目背景  
数据清洗项目面临多源异构专利数据整合挑战，需处理两个核心数据集：  
1. 包含 1985-2022 年 3600 余万条专利的 CSV 文件，  
2. 以 DTA 格式存储的 2011-2017 年补充数据。  

目标是将前者没有但后者有的数据提取出来并插入前者。由于数据格式割裂（CSV/DTA）、时间范围交叉且存在重复缺失问题，传统单机处理效率低下——百万级数据匹配耗时过高，严重制约项目进度。为此，我们计划借助 AWS 云平台构建自动化数据处理流水线，实现高效数据清洗、精准匹配与标准化交付。  

## 项目概述  
首先，我们将 DTA 文件转换为 CSV 格式，并将所有数据拆分成小份，防止 EC2 崩坏。  

通过 AWS 云服务实现以下核心能力：  
- **加速匹配计算**：利用 EC2 计算优化实例运行任务，将专利匹配耗时大幅缩短，并利用 RDS MySQL 存储合并后的专利主数据集，对专利申请号（`application_no`）建立唯一索引，加速缺失数据的比对查询；  
- **可审计交付**：在 S3 中建立原始数据区、清洗中间层及标准化交付区，通过版本控制与生命周期策略满足数据溯源与长期归档需求。

## AWS 具体使用步骤  

### 1. 需求分析与架构设计

#### 1.1 业务需求  
- 确定需要存储和处理的数据类型，预估数据量，决定数据库的存储规格和计算资源。  
- 选择 AWS 资源（EC2 计算，RDS 数据库，S3 存储）。

#### 1.2 主要 AWS 组件

| 组件            | 作用                                                        |  
|-----------------|-------------------------------------------------------------|  
| **Amazon S3**   | 存储原始数据、备份、长期归档（作用类似网盘）               |  
| **Amazon EC2**  | 运行数据处理任务                                            |  
| **Amazon RDS**  | 结构化数据存储（支持 MySQL、PostgreSQL、MariaDB）           |  
| **IAM**         | 权限管理，控制 EC2 访问 S3、RDS                            |  

### 2. AWS 资源创建与配置  

#### 2.1 创建 Amazon S3 存储桶  
S3 用于存储原始数据以及数据库的备份。  

**步骤：**  
1. 登录 AWS 控制台，转到 S3 服务。  
2. 创建存储桶（Bucket）：  
   - 进入 S3 > **Create bucket**  
   - 设定 Bucket 名称  
   - 配置一般默认配置即可  
   - 点击 **创建**  
3. 配置权限：  
   - 进入 S3 > 存储桶 > 权限  
   - 用 IAM Role 绑定 S3 访问权限（IAM Role 的创建请参考 3.1，可以稍后返回执行）

#### 2.2 创建 Amazon EC2 并配置  
EC2 用于运行数据处理任务（ETL、分析、模型训练等）。  

**步骤：**  
1. 登录 AWS 控制台，进入 EC2 > **Launch Instance**  
2. 启动新实例  
3. 选择 AMI（操作系统）  
   - 选择 **Amazon Linux 2**  
4. 选择实例类型  
   - 专利清洗项目使用的是最基础的 **t2.micro** 实例，足以应对大部分任务。若计算需求增长，可以随时更改实例类型。  
     - 一般用途：`t3.medium`（适合小型任务）  
     - 计算优化：`c5.large`（适合数据处理）  
     - 内存优化：`r5.large`（适合大数据计算）  
5. 选择密钥对  
   - 如果没有密钥对，创建一个并下载 `.pem` 文件，记住位置，后续需要使用。  
6. 配置安全组  
   - 开启 **SSH (22)**，仅允许特定 IP（如 My IP）。  
   - 允许 RDS 端口（MySQL `3306` / PostgreSQL `5432`）。创建好 EC2 后，进入实例设置并修改安全组入站规则。

#### 2.3 创建 Amazon RDS 并配置  
RDS 用于存储结构化数据。  

**步骤：**  
1. 登录 AWS 控制台，进入 RDS > **Create database**  
2. 选择 **轻松创建**  
3. 选择数据库类型  
   - MySQL 8.0 / Aurora (PostgreSQL Compatible)（根据业务需求选择）  
   - 免费套餐  
4. 设置主用户名和密码  
5. 设置 EC2 连接  
   - 选择已启动的 EC2 实例  
6. 创建数据库  

### 3. 配置 EC2 访问 S3 和 RDS

#### 3.1 配置 IAM 角色（允许 EC2 访问 S3）  
1. 进入 **IAM > Roles > Create Role**  
2. 选择 EC2 作为使用案例  
3. 附加权限策略：`AmazonRDSFullAccess`、`AmazonS3FullAccess`  
4. 绑定 IAM 角色到 EC2 实例：  
   - 进入 EC2 实例 > **操作** > **安全** > **修改 IAM 角色**  

#### 3.2 配置 EC2 访问 RDS  
1. **EC2 安全组**  
   - 确保允许 `3306`（MySQL）或 `5432`（PostgreSQL）端口访问。  
2. **在 EC2 上安装数据库客户端**  `
   ```
   sudo yum install -y mysql
   ```

### 4. 测试

#### 4.1 测试 EC2 是否能连接 RDS  
你需要检查：  
- EC2 是否能访问 RDS（网络连通性）  
- RDS 账号和权限是否正确  

**步骤 1：安装数据库客户端**  
启用 CRB 仓库  
```
sudo dnf config-manager --set-enabled crb sudo dnf makecache
```
安装 MariaDB（兼容 MySQL）  
```
sudo dnf install -y mariadb105
```
测试 MySQL 是否安装  
```
mysql --version
```

**步骤 2：测试网络连通性**  
安装 `nc`  
```
sudo dnf install -y nmap-ncat
```
测试网络连接  
```
nc -zv mydatabase.abcdefg1234.us-east-1.rds.amazonaws.com 3306
```
若无报错说明 EC2 可以访问 RDS。  

**步骤 3：用数据库账号测试连接**  
MySQL 连接命令：  
```
mysql -h mydatabase.abcdefg1234.us-east-1.rds.amazonaws.com -P 3306 -u myuser -p
```
输入密码，如果能进入数据库，说明连接正常。  

#### 4.2 测试 EC2 是否能访问 S3  
你需要检查：  
- IAM Role 是否有 S3 访问权限  
- EC2 是否能连接 S3（网络连通性）  

**步骤 1：检查 IAM Role 权限**  
在 EC2 执行：  
```
aws sts get-caller-identity
```
若返回 IAM 角色信息，说明 EC2 绑定的 IAM Role 生效了。否则，检查 EC2 是否正确绑定 IAM Role。  

**步骤 2：列出 S3 存储桶**  
运行  
```
aws s3 ls
```
如果返回你的 S3 存储桶列表，说明 EC2 访问 S3 正常。  

**步骤 3：上传和下载测试**  
测试上传：  

```
aws s3 cp testfile.txt s3://your-bucket-name/
```
测试下载：  
```
aws s3 cp s3://your-bucket-name/testfile.txt ./testfile.txt
```
如果都能成功，说明 EC2 和 S3 连接正常。  

### 5. 同步本地文件至 EC2  
在本地电脑执行：  
```
scp -r -i "C:\Users\86135\Downloads\ec2_1.pem" "D:\pythoncode\patent-cleaner" ec2-52-80-175-60.cn-north-1.compute.amazonaws.com.cn:/home/ec2-user/
```
分别替换成自己的密钥对地址、本地文件地址、EC2 的 IP 地址（每次重启后都会变）。

### 常见错误  
1. 连接 EC2 时可能会出现以下报错，通过在 EC2 安全组中添加 `ssh-0.0.0.0` 解决。  
   **错误信息**：  
```
Failed to connect to your instance EC2 Instance Connect is unable to connect to your instance. Ensure your instance network settings are configured correctly for EC2 Instance Connect.
```
