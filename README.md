# Base-Question
HPCG Benchmark 3.1 复现报告
项目概述
软件名称: HPCG (High Performance Conjugate Gradient) Benchmark
版本: 3.1
发布日期: 2019年3月28日
运行环境: WSL2 (Windows Subsystem for Linux 2)
目标: 验证 HPCG 在分布式多进程环境下的正确性、数值稳定性及基准性能。
硬件与软件配置
配置项   详情
MPI 实现   OpenMPI (mpirun.openmpi)

并行规模   8 个 MPI 进程 (-np 8)

线程设置   每进程 16 线程 (OpenMP)

通信绑定   --bind-to none (无CPU核心绑定)

网络层   PML: ob1, BTL: tcp,self (TCP/IP 回环/虚拟网卡通信)

问题规模   128 times 128 times 128 (全局网格)

时间步数   20 steps

处理器拓扑   2 times 2 times 2 (npx=2, npy=2, npz=2)

复现过程与关键步骤

3.1 初始遇到的问题：断言失败 (Assertion Failed)
在初次尝试运行时，由于参数传递错误，导致程序崩溃。

错误命令: ... xhpcg 20
错误现象: 所有 8 个进程同时报 SIGABRT 信号。
错误日志:
        xhpcg: /home/user/hpcg/src/GenerateCoarseProblem.cpp:50: void GenerateCoarseProblem(const SparseMatrix&): Assertion 'nxf%2==0' failed.
    
原因分析:
    HPCG 命令行参数格式为 nx ny nz ntime。
    仅传入 20 被解析为 nx=20，而 ny 和 nz 使用默认值。
    在 8 进程分解下，局部网格维度变为奇数，不满足多重网格（Multigrid）算法对偶数维度的要求，触发断言失败。
解决方案: 显式指定完整的问题规模和迭代次数。

3.2 最终成功运行的命令
mpirun.openmpi --bind-to none --mca pml ob1 --mca btl tcp,self -np 8 (pwd)/xhpcg 64 64 64 20

参数解释:
    64 64 64: 每个进程分配的局部网格大小为 64 times 64 times 64 (总全局 128^3)。
    20: 执行 20 个时间步长。
    注意: 此处实际输入文件可能强制覆盖了全局规模为 128^3，但本地分配逻辑需保证整除性和偶数性，此配置成功避免了断言错误。

结果分析

4.1 数值有效性验证 (Validity Check)
HPCG 内置了严格的验证测试，本次运行全部通过：
验证项   结果   说明
Spectral Convergence   PASSED   谱收敛测试通过。未预条件迭代最大11次，预条件迭代最大2次。

Symmetry Departure   PASSED   矩阵对称性偏差极小。SpMV: 5.896 times 10^{-9}MG: 1.684 times 10^{-9}

Iteration Count   PASSED   迭代次数符合预期（参考值50次，优化后50次）。

Reproducibility   PASSED   可重复性测试通过，缩放残差均值 9.865 times 10^{-7}。

结论: HPCG result is VALID。计算结果是数学上正确的。

4.2 性能数据分析 (Performance Metrics)

由于使用的是Reference Kernel (参考内核)，未链接 MKL/OpenBLAS 等高性能库，性能较低，主要用于功能验证。
指标   数值   备注
Total Time   64.8501 sec   远小于官方认证的 1800 秒要求

GFLOP/s (Raw Total)   0.594474   极低，因使用串行/低效并行内核

GB/s (Read+Write)   4.50961   内存带宽利用率一般

主要耗时组件   MG (40.5s), DDOT (12.7s)   多级网格求解器和点积操作占主导
性能警告:
日志明确指出：
Performance results are severely suboptimal
Official results execution time (sec) must be at least=1800
这意味着当前结果仅适用于代码逻辑验证，不适用于高性能计算排名或正式的性能评估报告。
资源消耗统计
总方程数: 2,097,152 (128^3)
非零元素数: 55,742,968
内存占用: 约 1.5 GB
    线性系统 + CG: 1.32 GB
    粗网格层级 1: 0.158 GB
    粗网格层级 2: 0.020 GB
    粗网格层级 3: 0.002 GB
多级网格结构: 3 层粗网格
总结与结论
复现成功: HPCG 3.1 在 WSL2 环境下，使用 8 进程 MPI 并行计算成功运行并输出有效结果。
问题解决: 解决了初期因参数错误导致的 Assertion 'nxf%2==0' failed 崩溃问题，确认了 128^3$ 规模在 8 进程下的正确分解方式。
结果性质: 获得的是 Valid (有效) 的基准测试结果，而非 Optimized (优化) 的高性能结果。
后续建议:
    若需提升性能分数，需重新编译 HPCG 并链接 Intel MKL 或 OpenBLAS。
    若需进行官方认证，需在原生 Linux 环境中运行至少 1800 秒，并使用优化内核。

实际运行
(base) user@DESKTOP-J1MSR9D:~$ git clone https://github.com/hpcg-benchmark/hpcg.git
Cloning into 'hpcg'...
fatal: unable to access 'https://github.com/hpcg-benchmark/hpcg.git/': Failed to connect to github.com port 443 after 134039 ms: Could not connect to server
(base) user@DESKTOP-J1MSR9D:~$ git clone https://github.com/hpcg-benchmark/hpcg.git
Cloning into 'hpcg'...
remote: Enumerating objects: 2891, done.
remote: Counting objects: 100% (1315/1315), done.
remote: Compressing objects: 100% (159/159), done.
remote: Total 2891 (delta 1221), reused 1156 (delta 1156), pack-reused 1576 (from 1)
Receiving objects: 100% (2891/2891), 9.00 MiB | 67.00 KiB/s, done.
Resolving deltas: 100% (2145/2145), done.
(base) user@DESKTOP-J1MSR9D:~$ sudo apt install cmake g++ libmpich-dev libomp-dev
[sudo: authenticate] Password:
g++ is already the newest version (4:15.2.0-5ubuntu1).
Installing:
  cmake  libmpich-dev  libomp-dev

Installing dependencies:
  cmake-data  libarchive13t64  libmpich12     libomp5    libucx-dev  mpich
  hwloc-nox   libjsoncpp26     libomp-21-dev  librhash1  libuv1t64

Suggested packages:
  cmake-doc  cmake-format  elpa-cmake-mode  ninja-build  lrzip  libomp-21-doc  mpich-doc

Summary:
  Upgrading: 0, Installing: 14, Removing: 0, Not Upgrading: 45
  Download size: 26.4 MB
  Space needed: 106 MB / 1020 GB available

Continue? [Y/n] y
Get:1 http://archive.ubuntu.com/ubuntu resolute/main amd64 libuv1t64 amd64 1.51.0-2ubuntu1 [103 kB]
Get:2 http://archive.ubuntu.com/ubuntu resolute/main amd64 cmake-data all 4.2.3-2ubuntu2 [2581 kB]
9% [2 cmake-data 2437 kB/2581 kB 94%]                                                                 1873 B/s 3h 32min 23s^                                                                       C
(base) user@DESKTOP-J1MSR9D:~$ sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
(base) user@DESKTOP-J1MSR9D:~$ sudo tee /etc/apt/sources.list > /dev/null << 'EOF'
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-security main restricted universe multiverse
EOF
(base) user@DESKTOP-J1MSR9D:~$ sudo apt update
Hit:1 http://archive.ubuntu.com/ubuntu resolute InRelease
Get:2 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble InRelease [256 kB]
Get:3 http://security.ubuntu.com/ubuntu resolute-security InRelease [137 kB]
Get:4 http://archive.ubuntu.com/ubuntu resolute-updates InRelease [137 kB]
Get:5 http://security.ubuntu.com/ubuntu resolute-security/main amd64 Packages [422 kB]
Hit:6 http://archive.ubuntu.com/ubuntu resolute-backports InRelease
Get:7 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates InRelease [126 kB]
Get:8 http://archive.ubuntu.com/ubuntu resolute-updates/main amd64 Packages [492 kB]
Get:9 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports InRelease [126 kB]
Get:10 http://security.ubuntu.com/ubuntu resolute-security/main Translation-en [97.6 kB]
Get:11 http://security.ubuntu.com/ubuntu resolute-security/main amd64 Components [46.7 kB]
Get:12 http://security.ubuntu.com/ubuntu resolute-security/universe amd64 Packages [159 kB]
Get:13 http://security.ubuntu.com/ubuntu resolute-security/universe Translation-en [51.2 kB]
Get:14 http://archive.ubuntu.com/ubuntu resolute-updates/main Translation-en [118 kB]
Get:15 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security InRelease [126 kB]
Get:16 http://archive.ubuntu.com/ubuntu resolute-updates/main amd64 Components [97.8 kB]
Get:17 http://security.ubuntu.com/ubuntu resolute-security/universe amd64 Components [43.2 kB]
Get:18 http://archive.ubuntu.com/ubuntu resolute-updates/universe amd64 Packages [250 kB]
Get:19 http://security.ubuntu.com/ubuntu resolute-security/restricted amd64 Packages [357 kB]
Get:20 http://security.ubuntu.com/ubuntu resolute-security/restricted Translation-en [69.0 kB]
Get:21 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/main amd64 Packages [1401 kB]
Get:22 http://archive.ubuntu.com/ubuntu resolute-updates/universe Translation-en [83.2 kB]
Get:23 http://archive.ubuntu.com/ubuntu resolute-updates/universe amd64 Components [184 kB]
Get:24 http://archive.ubuntu.com/ubuntu resolute-updates/restricted amd64 Packages [367 kB]
Get:25 http://archive.ubuntu.com/ubuntu resolute-updates/restricted Translation-en [70.8 kB]
Get:26 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/main Translation-en [513 kB]
Get:27 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/main amd64 Components [464 kB]
Get:28 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/main amd64 c-n-f Metadata [30.5 kB]
Get:29 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/restricted amd64 Packages [93.9 kB]
Get:30 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/restricted Translation-en [18.7 kB]
Get:31 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/restricted amd64 c-n-f Metadata [416 B]
Get:32 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/universe amd64 Packages [15.0 MB]
Get:33 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/universe Translation-en [5982 kB]
Get:34 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/universe amd64 Components [3871 kB]
Get:35 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/universe amd64 c-n-f Metadata [301 kB]
Get:36 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/multiverse amd64 Packages [269 kB]
Get:37 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/multiverse Translation-en [118 kB]
Get:38 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/multiverse amd64 Components [35.0 kB]
Get:39 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble/multiverse amd64 c-n-f Metadata [8328 B]
Get:40 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/main amd64 Packages [1227 kB]
Get:41 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/main Translation-en [286 kB]
Get:42 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/main amd64 Components [181 kB]
Get:43 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/main amd64 c-n-f Metadata [17.7 kB]
Get:44 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/restricted amd64 Packages [1483 kB]
Get:45 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/restricted Translation-en [339 kB]
Get:46 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/restricted amd64 Components [212 B]
Get:47 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/restricted amd64 c-n-f Metadata [456 B]
Get:48 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/universe amd64 Packages [1687 kB]
Get:49 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/universe Translation-en [337 kB]
Get:50 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/universe amd64 Components [388 kB]
Get:51 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/universe amd64 c-n-f Metadata [34.9 kB]
Get:52 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/multiverse amd64 Packages [45.4 kB]
Get:53 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/multiverse Translation-en [12.8 kB]
Get:54 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/multiverse amd64 Components [940 B]
Get:55 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-updates/multiverse amd64 c-n-f Metadata [656 B]
Get:56 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/main amd64 Packages [40.6 kB]
Get:57 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/main Translation-en [9172 B]
Get:58 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/main amd64 Components [5768 B]
Get:59 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/main amd64 c-n-f Metadata [368 B]
Get:60 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/restricted amd64 Components [212 B]
Get:61 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/restricted amd64 c-n-f Metadata [116 B]
Get:62 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/universe amd64 Packages [31.0 kB]
Get:63 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/universe Translation-en [18.6 kB]
Get:64 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/universe amd64 Components [12.6 kB]
Get:65 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/universe amd64 c-n-f Metadata [1588 B]
Get:66 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/multiverse amd64 Packages [748 B]
Get:67 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/multiverse Translation-en [340 B]
Get:68 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/multiverse amd64 Components [212 B]
Get:69 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-backports/multiverse amd64 c-n-f Metadata [116 B]
Get:70 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/main amd64 Packages [967 kB]
Get:71 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/main Translation-en [205 kB]
Get:72 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/main amd64 Components [46.4 kB]
Get:73 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/main amd64 c-n-f Metadata [11.9 kB]
Get:74 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/restricted amd64 Packages [1385 kB]
Get:75 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/restricted Translation-en [320 kB]
Get:76 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/restricted amd64 Components [212 B]
Get:77 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/restricted amd64 c-n-f Metadata [444 B]
Get:78 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/universe amd64 Packages [1201 kB]
Get:79 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/universe Translation-en [240 kB]
Get:80 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/universe amd64 Components [76.3 kB]
Get:81 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/universe amd64 c-n-f Metadata [24.2 kB]
Get:82 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/multiverse amd64 Packages [40.3 kB]
Get:83 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/multiverse Translation-en [10.9 kB]
Get:84 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/multiverse amd64 Components [208 B]
Get:85 https://mirrors.tuna.tsinghua.edu.cn/ubuntu noble-security/multiverse amd64 c-n-f Metadata [468 B]
Fetched 42.7 MB in 43s (981 kB/s)
68 packages can be upgraded. Run 'apt list --upgradable' to see them.
Notice: Some sources can be modernized. Run 'apt modernize-sources' to do so.
(base) user@DESKTOP-J1MSR9D:~$ sudo apt install -y cmake g++ libmpich-dev libomp-dev
g++ is already the newest version (4:15.2.0-5ubuntu1).
Installing:
  cmake  libmpich-dev  libomp-dev

Installing dependencies:
  cmake-data  libarchive13t64  libmpich12     libomp5    libucx-dev  mpich
  hwloc-nox   libjsoncpp26     libomp-21-dev  librhash1  libuv1t64

Suggested packages:
  cmake-doc  cmake-format  elpa-cmake-mode  ninja-build  lrzip  libomp-21-doc  mpich-doc

Summary:
  Upgrading: 0, Installing: 14, Removing: 0, Not Upgrading: 68
  Download size: 26.3 MB / 26.4 MB
  Space needed: 106 MB / 1020 GB available

Get:1 http://archive.ubuntu.com/ubuntu resolute/main amd64 cmake-data all 4.2.3-2ubuntu2 [2581 kB]
Get:2 http://archive.ubuntu.com/ubuntu resolute-updates/main amd64 libarchive13t64 amd64 3.8.5-1ubuntu2.2 [397 kB]
Get:3 http://archive.ubuntu.com/ubuntu resolute/main amd64 libjsoncpp26 amd64 1.9.6-5 [84.0 kB]
Get:4 http://archive.ubuntu.com/ubuntu resolute/main amd64 librhash1 amd64 1.4.6-1.1 [133 kB]
Get:5 http://archive.ubuntu.com/ubuntu resolute/main amd64 cmake amd64 4.2.3-2ubuntu2 [13.7 MB]
18% [5 cmake 351 kB/13.7 MB 3%]                                                                       2845 B/s 2h 13min 19s^                                                                       Ign:5 http://archive.ubuntu.com/ubuntu resolute/main amd64 cmake amd64 4.2.3-2ubuntu2
Get:6 http://archive.ubuntu.com/ubuntu resolute/universe amd64 libomp5 amd64 1:22.1.2-1ubuntu1 [429 kB]
Get:7 http://archive.ubuntu.com/ubuntu resolute/universe amd64 libomp-21-dev amd64 1:21.1.8-6ubuntu1 [113 kB]
Get:8 http://archive.ubuntu.com/ubuntu resolute/universe amd64 libucx-dev amd64 1.20.0+ds-4ubuntu2 [1207 kB]
Get:9 http://archive.ubuntu.com/ubuntu resolute/universe amd64 hwloc-nox amd64 2.13.0-2 [235 kB]
Get:10 http://archive.ubuntu.com/ubuntu resolute/universe amd64 libmpich12 amd64 4.3.2-2 [3026 kB]
Ign:10 http://archive.ubuntu.com/ubuntu resolute/universe amd64 libmpich12 amd64 4.3.2-2
Get:11 http://archive.ubuntu.com/ubuntu resolute/universe amd64 mpich amd64 4.3.2-2 [213 kB]
Get:12 http://archive.ubuntu.com/ubuntu resolute/universe amd64 libmpich-dev amd64 4.3.2-2 [4142 kB]
Get:13 http://archive.ubuntu.com/ubuntu resolute/universe amd64 libomp-dev amd64 1:21.1.6-71 [3376 B]
Get:5 http://archive.ubuntu.com/ubuntu resolute/main amd64 cmake amd64 4.2.3-2ubuntu2 [13.7 MB]
Get:10 http://archive.ubuntu.com/ubuntu resolute/universe amd64 libmpich12 amd64 4.3.2-2 [3026 kB]
Fetched 14.3 MB in 1h 43min 44s (2304 B/s)
Selecting previously unselected package libuv1t64:amd64.
(Reading database ... 45336 files and directories currently installed.)
Preparing to unpack .../00-libuv1t64_1.51.0-2ubuntu1_amd64.deb ...
Unpacking libuv1t64:amd64 (1.51.0-2ubuntu1) ...
Selecting previously unselected package cmake-data.
Preparing to unpack .../01-cmake-data_4.2.3-2ubuntu2_all.deb ...
Unpacking cmake-data (4.2.3-2ubuntu2) ...
Selecting previously unselected package libarchive13t64:amd64.
Preparing to unpack .../02-libarchive13t64_3.8.5-1ubuntu2.2_amd64.deb ...
Unpacking libarchive13t64:amd64 (3.8.5-1ubuntu2.2) ...
Selecting previously unselected package libjsoncpp26:amd64.
Preparing to unpack .../03-libjsoncpp26_1.9.6-5_amd64.deb ...
Unpacking libjsoncpp26:amd64 (1.9.6-5) ...
Selecting previously unselected package librhash1:amd64.
Preparing to unpack .../04-librhash1_1.4.6-1.1_amd64.deb ...
Unpacking librhash1:amd64 (1.4.6-1.1) ...
Selecting previously unselected package cmake.
Preparing to unpack .../05-cmake_4.2.3-2ubuntu2_amd64.deb ...
Unpacking cmake (4.2.3-2ubuntu2) ...
Selecting previously unselected package libomp5:amd64.
Preparing to unpack .../06-libomp5_1%3a22.1.2-1ubuntu1_amd64.deb ...
Unpacking libomp5:amd64 (1:22.1.2-1ubuntu1) ...
Selecting previously unselected package libomp-21-dev.
Preparing to unpack .../07-libomp-21-dev_1%3a21.1.8-6ubuntu1_amd64.deb ...
Unpacking libomp-21-dev (1:21.1.8-6ubuntu1) ...
Selecting previously unselected package libucx-dev:amd64.
Preparing to unpack .../08-libucx-dev_1.20.0+ds-4ubuntu2_amd64.deb ...
Unpacking libucx-dev:amd64 (1.20.0+ds-4ubuntu2) ...
Selecting previously unselected package hwloc-nox.
Preparing to unpack .../09-hwloc-nox_2.13.0-2_amd64.deb ...
Unpacking hwloc-nox (2.13.0-2) ...
Selecting previously unselected package libmpich12:amd64.
Preparing to unpack .../10-libmpich12_4.3.2-2_amd64.deb ...
Unpacking libmpich12:amd64 (4.3.2-2) ...
Selecting previously unselected package mpich.
Preparing to unpack .../11-mpich_4.3.2-2_amd64.deb ...
Unpacking mpich (4.3.2-2) ...
Selecting previously unselected package libmpich-dev:amd64.
Preparing to unpack .../12-libmpich-dev_4.3.2-2_amd64.deb ...
Unpacking libmpich-dev:amd64 (4.3.2-2) ...
Selecting previously unselected package libomp-dev:amd64.
Preparing to unpack .../13-libomp-dev_1%3a21.1.6-71_amd64.deb ...
Unpacking libomp-dev:amd64 (1:21.1.6-71) ...
Setting up libuv1t64:amd64 (1.51.0-2ubuntu1) ...
Setting up hwloc-nox (2.13.0-2) ...
Setting up libomp5:amd64 (1:22.1.2-1ubuntu1) ...
Setting up libjsoncpp26:amd64 (1.9.6-5) ...
Setting up libucx-dev:amd64 (1.20.0+ds-4ubuntu2) ...
Setting up libmpich12:amd64 (4.3.2-2) ...
Setting up cmake-data (4.2.3-2ubuntu2) ...
Setting up librhash1:amd64 (1.4.6-1.1) ...
Setting up libarchive13t64:amd64 (3.8.5-1ubuntu2.2) ...
Setting up libomp-21-dev (1:21.1.8-6ubuntu1) ...
Setting up mpich (4.3.2-2) ...
Setting up libmpich-dev:amd64 (4.3.2-2) ...
Setting up cmake (4.2.3-2ubuntu2) ...
Setting up libomp-dev:amd64 (1:21.1.6-71) ...
Processing triggers for man-db (2.13.1-1build1) ...
Processing triggers for libc-bin (2.43-2ubuntu2.3) ...
(base) user@DESKTOP-J1MSR9D:~$ cd hpcg
(base) user@DESKTOP-J1MSR9D:~/hpcg$ ^[[200~mkdir build && cd build~
mkdir: command not found
(base) user@DESKTOP-J1MSR9D:~/hpcg$ mkdir build && cd build
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ cmake -DHPCG_ENABLE_MPI=ON -DHPCG_ENABLE_OPENMP=ON ..
CMake Error at CMakeLists.txt:4 (cmake_minimum_required):
  Compatibility with CMake < 3.5 has been removed from CMake.

  Update the VERSION argument <min> value.  Or, use the <min>...<max> syntax
  to tell CMake that the project requires at least <min> but has been updated
  to work with policies introduced by <max> or earlier.

  Or, add -DCMAKE_POLICY_VERSION_MINIMUM=3.5 to try configuring anyway.


-- Configuring incomplete, errors occurred!
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ ^C
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ cmake -DCMAKE_POLICY_VERSION_MINIMUM=3.5 -DHPCG_ENABLE_MPI=ON -DHPCG_ENABLE_OPENMP                                                                       =ON ..
CMake Deprecation Warning at CMakeLists.txt:4 (cmake_minimum_required):
  Compatibility with CMake < 3.10 will be removed from a future version of
  CMake.

  Update the VERSION argument <min> value.  Or, use the <min>...<max> syntax
  to tell CMake that the project requires at least <min> but has been updated
  to work with policies introduced by <max> or earlier.


-- The CXX compiler identification is GNU 15.2.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
CMake Warning (dev) at /usr/share/cmake-4.2/Modules/FindMPI.cmake:1646 (option):
  Policy CMP0077 is not set: option() honors normal variables.  Run "cmake
  --help-policy CMP0077" for policy details.  Use the cmake_policy command to
  set the policy and suppress this warning.

  For compatibility with older versions of CMake, option is clearing the
  normal variable 'MPI_CXX_SKIP_MPICXX'.
Call Stack (most recent call first):
  CMakeLists.txt:55 (find_package)
This warning is for project developers.  Use -Wno-dev to suppress it.

-- Found MPI_CXX: /usr/lib/x86_64-linux-gnu/openmpi/lib/libmpi.so (found version "3.1")
-- Found MPI: TRUE (found version "3.1")
-- Found OpenMP_CXX: -fopenmp (found version "4.5")
-- Found OpenMP: TRUE (found version "4.5")
-- Configuring done (2.0s)
-- Generating done (0.0s)
-- Build files have been written to: /home/user/hpcg/build
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ make -j$(nproc)
[ 17%] Building CXX object CMakeFiles/xhpcg.dir/src/ExchangeHalo.cpp.o
[ 19%] Building CXX object CMakeFiles/xhpcg.dir/src/GenerateProblem.cpp.o
[ 17%] Building CXX object CMakeFiles/xhpcg.dir/src/main.cpp.o
[ 19%] Building CXX object CMakeFiles/xhpcg.dir/src/GenerateGeometry.cpp.o
[ 19%] Building CXX object CMakeFiles/xhpcg.dir/src/GenerateProblem_ref.cpp.o
[ 17%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeResidual.cpp.o
[ 17%] Building CXX object CMakeFiles/xhpcg.dir/src/TestCG.cpp.o
[ 24%] Building CXX object CMakeFiles/xhpcg.dir/src/CheckProblem.cpp.o
[ 26%] Building CXX object CMakeFiles/xhpcg.dir/src/CG_ref.cpp.o
[ 29%] Building CXX object CMakeFiles/xhpcg.dir/src/ReadHpcgDat.cpp.o
[ 31%] Building CXX object CMakeFiles/xhpcg.dir/src/OptimizeProblem.cpp.o
[ 31%] Building CXX object CMakeFiles/xhpcg.dir/src/CG.cpp.o
[ 31%] Building CXX object CMakeFiles/xhpcg.dir/src/ReportResults.cpp.o
[ 34%] Building CXX object CMakeFiles/xhpcg.dir/src/SetupHalo.cpp.o
[ 36%] Building CXX object CMakeFiles/xhpcg.dir/src/SetupHalo_ref.cpp.o
[ 39%] Building CXX object CMakeFiles/xhpcg.dir/src/TestSymmetry.cpp.o
/home/user/hpcg/src/GenerateProblem.cpp:22:10: fatal error: mpi.h: No such file or directory
   22 | #include <mpi.h>
      |          ^~~~~~~
compilation terminated.
/home/user/hpcg/src/ComputeResidual.cpp:21:10: fatal error: mpi.h: No such file or directory
   21 | #include <mpi.h>
      |          ^~~~~~~
compilation terminated.
/home/user/hpcg/src/ExchangeHalo.cpp:23:10: fatal error: mpi.h: No such file or directory
   23 | #include <mpi.h>
      |          ^~~~~~~
compilation terminated.
/home/user/hpcg/src/main.cpp:25:10: fatal error: mpi.h: No such file or directory
   25 | #include <mpi.h>
      |          ^~~~~~~
compilation terminated.
/home/user/hpcg/src/GenerateProblem_ref.cpp:22:10: fatal error: mpi.h: No such file or directory
   22 | #include <mpi.h>
      |          ^~~~~~~
compilation terminated.
make[2]: *** [CMakeFiles/xhpcg.dir/build.make:177: CMakeFiles/xhpcg.dir/src/GenerateProblem.cpp.o] Error 1
make[2]: *** Waiting for unfinished jobs....
make[2]: *** [CMakeFiles/xhpcg.dir/build.make:135: CMakeFiles/xhpcg.dir/src/ComputeResidual.cpp.o] Error 1
make[2]: *** [CMakeFiles/xhpcg.dir/build.make:79: CMakeFiles/xhpcg.dir/src/main.cpp.o] Error 1
make[2]: *** [CMakeFiles/xhpcg.dir/build.make:191: CMakeFiles/xhpcg.dir/src/GenerateProblem_ref.cpp.o] Error 1
make[2]: *** [CMakeFiles/xhpcg.dir/build.make:149: CMakeFiles/xhpcg.dir/src/ExchangeHalo.cpp.o] Error 1
/home/user/hpcg/src/CheckProblem.cpp:22:10: fatal error: mpi.h: No such file or directory
   22 | #include <mpi.h>
      |          ^~~~~~~
compilation terminated.
make[2]: *** [CMakeFiles/xhpcg.dir/build.make:205: CMakeFiles/xhpcg.dir/src/CheckProblem.cpp.o] Error 1
/home/user/hpcg/src/SetupHalo.cpp:22:10: fatal error: mpi.h: No such file or directory
   22 | #include <mpi.h>
      |          ^~~~~~~
compilation terminated.
/home/user/hpcg/src/ReportResults.cpp:22:10: fatal error: mpi.h: No such file or directory
   22 | #include <mpi.h>
      |          ^~~~~~~
compilation terminated.
/home/user/hpcg/src/SetupHalo_ref.cpp:22:10: fatal error: mpi.h: No such file or directory
   22 | #include <mpi.h>
      |          ^~~~~~~
compilation terminated.
make[2]: *** [CMakeFiles/xhpcg.dir/build.make:261: CMakeFiles/xhpcg.dir/src/SetupHalo.cpp.o] Error 1
/home/user/hpcg/src/TestSymmetry.cpp:23:10: fatal error: mpi.h: No such file or directory
   23 | #include <mpi.h>
      |          ^~~~~~~
compilation terminated.
make[2]: *** [CMakeFiles/xhpcg.dir/build.make:247: CMakeFiles/xhpcg.dir/src/ReportResults.cpp.o] Error 1
make[2]: *** [CMakeFiles/xhpcg.dir/build.make:275: CMakeFiles/xhpcg.dir/src/SetupHalo_ref.cpp.o] Error 1
make[2]: *** [CMakeFiles/xhpcg.dir/build.make:289: CMakeFiles/xhpcg.dir/src/TestSymmetry.cpp.o] Error 1
make[1]: *** [CMakeFiles/Makefile2:87: CMakeFiles/xhpcg.dir/all] Error 2
make: *** [Makefile:91: all] Error 2
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun -np 8 ../bin/xhpcg
--------------------------------------------------------------------------
Sorry!  You were supposed to get help about:
    prun:exe-not-accessible
But I couldn't open the help file:
    /usr/lib/x86_64-linux-gnu/pmix2/share/pmix/help-prun.txt: No such file or directory
    /usr/lib/x86_64-linux-gnu/prrte3/share/prte/help-prun.txt: No such file or directory.
Sorry!
--------------------------------------------------------------------------
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ ^C
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ find /usr -name mpi.h
/usr/lib/x86_64-linux-gnu/mpich/include/mpi.h
/usr/lib/x86_64-linux-gnu/openmpi/include/mpi.h
find: ‘/usr/lib/modules/6.18.33.2-microsoft-standard-WSL2/lost+found’: Permission denied
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ ^C
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ cd ~/hpcg
(base) user@DESKTOP-J1MSR9D:~/hpcg$ rm -rf build
(base) user@DESKTOP-J1MSR9D:~/hpcg$ rm -rf build
(base) user@DESKTOP-J1MSR9D:~/hpcg$ mkdir build && cd build
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ cmake -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
      -DHPCG_ENABLE_MPI=ON \
      -DHPCG_ENABLE_OPENMP=ON \
      -DCMAKE_CXX_COMPILER=mpicxx \
      ..
CMake Deprecation Warning at CMakeLists.txt:4 (cmake_minimum_required):
  Compatibility with CMake < 3.10 will be removed from a future version of
  CMake.

  Update the VERSION argument <min> value.  Or, use the <min>...<max> syntax
  to tell CMake that the project requires at least <min> but has been updated
  to work with policies introduced by <max> or earlier.


-- The CXX compiler identification is GNU 15.2.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/mpicxx - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
CMake Warning (dev) at /usr/share/cmake-4.2/Modules/FindMPI.cmake:1646 (option):
  Policy CMP0077 is not set: option() honors normal variables.  Run "cmake
  --help-policy CMP0077" for policy details.  Use the cmake_policy command to
  set the policy and suppress this warning.

  For compatibility with older versions of CMake, option is clearing the
  normal variable 'MPI_CXX_SKIP_MPICXX'.
Call Stack (most recent call first):
  CMakeLists.txt:55 (find_package)
This warning is for project developers.  Use -Wno-dev to suppress it.

-- Found MPI_CXX: /usr/bin/mpicxx (found version "3.1")
-- Found MPI: TRUE (found version "3.1")
-- Found OpenMP_CXX: -fopenmp (found version "4.5")
-- Found OpenMP: TRUE (found version "4.5")
-- Configuring done (1.7s)
-- Generating done (0.0s)
-- Build files have been written to: /home/user/hpcg/build
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ make -j$(nproc)
[ 12%] Building CXX object CMakeFiles/xhpcg.dir/src/TestCG.cpp.o
[ 12%] Building CXX object CMakeFiles/xhpcg.dir/src/CG_ref.cpp.o
[ 12%] Building CXX object CMakeFiles/xhpcg.dir/src/CG.cpp.o
[ 12%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeResidual.cpp.o
[ 12%] Building CXX object CMakeFiles/xhpcg.dir/src/main.cpp.o
[ 14%] Building CXX object CMakeFiles/xhpcg.dir/src/ExchangeHalo.cpp.o
[ 17%] Building CXX object CMakeFiles/xhpcg.dir/src/GenerateGeometry.cpp.o
[ 19%] Building CXX object CMakeFiles/xhpcg.dir/src/CheckProblem.cpp.o
[ 21%] Building CXX object CMakeFiles/xhpcg.dir/src/OptimizeProblem.cpp.o
[ 24%] Building CXX object CMakeFiles/xhpcg.dir/src/GenerateProblem_ref.cpp.o
[ 26%] Building CXX object CMakeFiles/xhpcg.dir/src/ReportResults.cpp.o
[ 29%] Building CXX object CMakeFiles/xhpcg.dir/src/GenerateProblem.cpp.o
[ 31%] Building CXX object CMakeFiles/xhpcg.dir/src/SetupHalo.cpp.o
[ 34%] Building CXX object CMakeFiles/xhpcg.dir/src/TestSymmetry.cpp.o
[ 36%] Building CXX object CMakeFiles/xhpcg.dir/src/ReadHpcgDat.cpp.o
[ 39%] Building CXX object CMakeFiles/xhpcg.dir/src/SetupHalo_ref.cpp.o
[ 41%] Building CXX object CMakeFiles/xhpcg.dir/src/TestNorms.cpp.o
[ 43%] Building CXX object CMakeFiles/xhpcg.dir/src/WriteProblem.cpp.o
[ 46%] Building CXX object CMakeFiles/xhpcg.dir/src/YAML_Element.cpp.o
[ 48%] Building CXX object CMakeFiles/xhpcg.dir/src/YAML_Doc.cpp.o
[ 51%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeDotProduct.cpp.o
[ 53%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeDotProduct_ref.cpp.o
[ 56%] Building CXX object CMakeFiles/xhpcg.dir/src/finalize.cpp.o
[ 58%] Building CXX object CMakeFiles/xhpcg.dir/src/init.cpp.o
[ 63%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeSPMV.cpp.o
[ 63%] Building CXX object CMakeFiles/xhpcg.dir/src/mytimer.cpp.o
[ 65%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeSPMV_ref.cpp.o
[ 68%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeSYMGS.cpp.o
[ 70%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeSYMGS_ref.cpp.o
[ 73%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeWAXPBY.cpp.o
[ 75%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeWAXPBY_ref.cpp.o
[ 78%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeMG_ref.cpp.o
[ 80%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeMG.cpp.o
[ 82%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeProlongation_ref.cpp.o
[ 85%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeRestriction_ref.cpp.o
[ 87%] Building CXX object CMakeFiles/xhpcg.dir/src/GenerateCoarseProblem.cpp.o
[ 90%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeOptimalShapeXYZ.cpp.o
[ 95%] Building CXX object CMakeFiles/xhpcg.dir/src/CheckAspectRatio.cpp.o
[ 95%] Building CXX object CMakeFiles/xhpcg.dir/src/MixedBaseCounter.cpp.o
[ 97%] Building CXX object CMakeFiles/xhpcg.dir/src/OutputFile.cpp.o
/home/user/hpcg/src/YAML_Doc.cpp: In member function ‘std::string YAML_Doc::generateYAML()’:
/home/user/hpcg/src/YAML_Doc.cpp:62:29: warning: ‘%02d’ directive writing between 2 and 11 bytes into                                                                                               a region of size between 1 and 17 [-Wformat-overflow=]
   62 |   sprintf (sdate,"%04d.%02d.%02d.%02d.%02d.%02d",ptm->tm_year + 1900, ptm->tm_mon+1,
      |                             ^~~~
/home/user/hpcg/src/YAML_Doc.cpp:62:11: note: ‘sprintf’ output between 20 and 72 bytes into a destina                                                                                              tion of size 25
   62 |   sprintf (sdate,"%04d.%02d.%02d.%02d.%02d.%02d",ptm->tm_year + 1900, ptm->tm_mon+1,
      |   ~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   63 |       ptm->tm_mday, ptm->tm_hour, ptm->tm_min,ptm->tm_sec);
      |       ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/home/user/hpcg/src/OutputFile.cpp: In member function ‘std::string OutputFile::generate()’:
/home/user/hpcg/src/OutputFile.cpp:118:29: warning: ‘%02d’ directive writing between 2 and 11 bytes i                                                                                              nto a region of size between 1 and 17 [-Wformat-overflow=]
  118 |   sprintf (sdate,"%04d-%02d-%02d_%02d-%02d-%02d",ptm->tm_year + 1900, ptm->tm_mon+1,
      |                             ^~~~
/home/user/hpcg/src/OutputFile.cpp:118:11: note: ‘sprintf’ output between 20 and 72 bytes into a dest                                                                                              ination of size 25
  118 |   sprintf (sdate,"%04d-%02d-%02d_%02d-%02d-%02d",ptm->tm_year + 1900, ptm->tm_mon+1,
      |   ~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  119 |         ptm->tm_mday, ptm->tm_hour, ptm->tm_min,ptm->tm_sec);
      |         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
[100%] Linking CXX executable xhpcg
[100%] Built target xhpcg
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun -np 8 ../bin/xhpcg
--------------------------------------------------------------------------
Sorry!  You were supposed to get help about:
    prun:exe-not-accessible
But I couldn't open the help file:
    /usr/lib/x86_64-linux-gnu/pmix2/share/pmix/help-prun.txt: No such file or directory
    /usr/lib/x86_64-linux-gnu/prrte3/share/prte/help-prun.txt: No such file or directory.
Sorry!
--------------------------------------------------------------------------
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ /usr/lib/x86_64-linux-gnu/mpich/bin/mpirun -np 8 ../bin/xhp                                                                                              cg
-bash: /usr/lib/x86_64-linux-gnu/mpich/bin/mpirun: No such file or directory
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ ^C
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ /usr/lib/x86_64-linux-gnu/openmpi/bin/mpirun -np 8 ../bin/x                                                                                              hpcg
-bash: /usr/lib/x86_64-linux-gnu/openmpi/bin/mpirun: No such file or directory
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ dpkg -L mpich | grep mpirun
/usr/bin/mpirun.mpich
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun.mpich -np 8 ../bin/xhpcg
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file ../bin/xh                                                                                              pcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file ../bin/xh                                                                                              pcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file ../bin/xh                                                                                              pcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file ../bin/xh                                                                                              pcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file ../bin/xh                                                                                              pcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file ../bin/xh                                                                                              pcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file ../bin/xh                                                                                              pcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file ../bin/xh                                                                                              pcg (No such file or directory)
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun.mpich -np 8 $(pwd)/../bin/xhpcg
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file /home/use                                                                                              r/hpcg/build/../bin/xhpcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file /home/use                                                                                              r/hpcg/build/../bin/xhpcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file /home/use                                                                                              r/hpcg/build/../bin/xhpcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file /home/use                                                                                              r/hpcg/build/../bin/xhpcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file /home/use                                                                                              r/hpcg/build/../bin/xhpcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file /home/use                                                                                              r/hpcg/build/../bin/xhpcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file /home/use                                                                                              r/hpcg/build/../bin/xhpcg (No such file or directory)
[proxy:0@DESKTOP-J1MSR9D] HYDU_create_process (lib/utils/launch.c:73): execvp error on file /home/use                                                                                              r/hpcg/build/../bin/xhpcg (No such file or directory)
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ cp /home/user/hpcg/bin/xhpcg .
cp: cannot stat '/home/user/hpcg/bin/xhpcg': No such file or directory
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ ls -l xhpcg
-rwxr-xr-x 1 user user 319968 Aug 25 15:39 xhpcg
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun.mpich -np 8 $(pwd)/xhpcg
No PMIx server was reachable, but a PMI1/2 was detected.
If srun is being used to launch application,  8 singletons will be started.
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ ^C
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun.mpich -launcher local -np 8 $(pwd)/xhpcg
[mpiexec@DESKTOP-J1MSR9D] HYDT_bsci_init (lib/tools/bootstrap/src/bsci_init.c:156): unrecognized laun                                                                                              cher: local
[mpiexec@DESKTOP-J1MSR9D] main (mpiexec/mpiexec.c:75): unable to initialize the bootstrap server
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun.mpich -bootstrap fork -np 8 $(pwd)/xhpcg
No PMIx server was reachable, but a PMI1/2 was detected.
If srun is being used to launch application,  8 singletons will be started.
(base) user@DESKTOP-J1MSR9D:~/hpcg/build$ conda deactivate
user@DESKTOP-J1MSR9D:~/hpcg/build$ which mpirun.mpich
/usr/bin/mpirun.mpich
user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun.mpich -np 8 $(pwd)/xhpcg
No PMIx server was reachable, but a PMI1/2 was detected.
If srun is being used to launch application,  8 singletons will be started.
user@DESKTOP-J1MSR9D:~/hpcg/build$ cd ~/hpcg
rm -rf build
mkdir build && cd build
user@DESKTOP-J1MSR9D:~/hpcg/build$ cmake -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
      -DHPCG_ENABLE_MPI=ON \
      -DHPCG_ENABLE_OPENMP=ON \
      -DCMAKE_CXX_COMPILER=mpicxx.openmpi \
      ..
CMake Deprecation Warning at CMakeLists.txt:4 (cmake_minimum_required):
  Compatibility with CMake < 3.10 will be removed from a future version of
  CMake.

  Update the VERSION argument <min> value.  Or, use the <min>...<max> syntax
  to tell CMake that the project requires at least <min> but has been updated
  to work with policies introduced by <max> or earlier.


-- The CXX compiler identification is GNU 15.2.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/mpicxx.openmpi - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
CMake Warning (dev) at /usr/share/cmake-4.2/Modules/FindMPI.cmake:1646 (option):
  Policy CMP0077 is not set: option() honors normal variables.  Run "cmake
  --help-policy CMP0077" for policy details.  Use the cmake_policy command to
  set the policy and suppress this warning.

  For compatibility with older versions of CMake, option is clearing the
  normal variable 'MPI_CXX_SKIP_MPICXX'.
Call Stack (most recent call first):
  CMakeLists.txt:55 (find_package)
This warning is for project developers.  Use -Wno-dev to suppress it.

-- Found MPI_CXX: /usr/bin/mpicxx.openmpi (found version "3.1")
-- Found MPI: TRUE (found version "3.1")
-- Found OpenMP_CXX: -fopenmp (found version "4.5")
-- Found OpenMP: TRUE (found version "4.5")
-- Configuring done (1.8s)
-- Generating done (0.0s)
-- Build files have been written to: /home/user/hpcg/build
user@DESKTOP-J1MSR9D:~/hpcg/build$ make -j$(nproc)
[  9%] Building CXX object CMakeFiles/xhpcg.dir/src/TestCG.cpp.o
[  9%] Building CXX object CMakeFiles/xhpcg.dir/src/ExchangeHalo.cpp.o
[  9%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeResidual.cpp.o
[ 14%] Building CXX object CMakeFiles/xhpcg.dir/src/CG_ref.cpp.o
[ 14%] Building CXX object CMakeFiles/xhpcg.dir/src/CG.cpp.o
[ 14%] Building CXX object CMakeFiles/xhpcg.dir/src/GenerateGeometry.cpp.o
[ 21%] Building CXX object CMakeFiles/xhpcg.dir/src/CheckProblem.cpp.o
[ 24%] Building CXX object CMakeFiles/xhpcg.dir/src/SetupHalo.cpp.o
[ 26%] Building CXX object CMakeFiles/xhpcg.dir/src/GenerateProblem_ref.cpp.o
[ 34%] Building CXX object CMakeFiles/xhpcg.dir/src/GenerateProblem.cpp.o
[ 34%] Building CXX object CMakeFiles/xhpcg.dir/src/main.cpp.o
[ 34%] Building CXX object CMakeFiles/xhpcg.dir/src/SetupHalo_ref.cpp.o
[ 34%] Building CXX object CMakeFiles/xhpcg.dir/src/OptimizeProblem.cpp.o
[ 34%] Building CXX object CMakeFiles/xhpcg.dir/src/ReadHpcgDat.cpp.o
[ 36%] Building CXX object CMakeFiles/xhpcg.dir/src/ReportResults.cpp.o
[ 39%] Building CXX object CMakeFiles/xhpcg.dir/src/TestSymmetry.cpp.o
[ 41%] Building CXX object CMakeFiles/xhpcg.dir/src/TestNorms.cpp.o
[ 43%] Building CXX object CMakeFiles/xhpcg.dir/src/WriteProblem.cpp.o
[ 46%] Building CXX object CMakeFiles/xhpcg.dir/src/YAML_Doc.cpp.o
[ 48%] Building CXX object CMakeFiles/xhpcg.dir/src/YAML_Element.cpp.o
[ 51%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeDotProduct.cpp.o
[ 58%] Building CXX object CMakeFiles/xhpcg.dir/src/init.cpp.o
[ 58%] Building CXX object CMakeFiles/xhpcg.dir/src/finalize.cpp.o
[ 58%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeDotProduct_ref.cpp.o
[ 63%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeSPMV.cpp.o
[ 63%] Building CXX object CMakeFiles/xhpcg.dir/src/mytimer.cpp.o
[ 65%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeSPMV_ref.cpp.o
[ 68%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeSYMGS.cpp.o
[ 70%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeSYMGS_ref.cpp.o
[ 73%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeWAXPBY.cpp.o
[ 75%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeWAXPBY_ref.cpp.o
[ 78%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeMG_ref.cpp.o
[ 82%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeProlongation_ref.cpp.o
[ 82%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeMG.cpp.o
[ 85%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeRestriction_ref.cpp.o
[ 87%] Building CXX object CMakeFiles/xhpcg.dir/src/GenerateCoarseProblem.cpp.o
[ 90%] Building CXX object CMakeFiles/xhpcg.dir/src/ComputeOptimalShapeXYZ.cpp.o
[ 92%] Building CXX object CMakeFiles/xhpcg.dir/src/MixedBaseCounter.cpp.o
[ 95%] Building CXX object CMakeFiles/xhpcg.dir/src/CheckAspectRatio.cpp.o
[ 97%] Building CXX object CMakeFiles/xhpcg.dir/src/OutputFile.cpp.o
/home/user/hpcg/src/YAML_Doc.cpp: In member function ‘std::string YAML_Doc::generateYAML()’:
/home/user/hpcg/src/YAML_Doc.cpp:62:29: warning: ‘%02d’ directive writing between 2 and 11 bytes into                                                                                               a region of size between 1 and 17 [-Wformat-overflow=]
   62 |   sprintf (sdate,"%04d.%02d.%02d.%02d.%02d.%02d",ptm->tm_year + 1900, ptm->tm_mon+1,
      |                             ^~~~
/home/user/hpcg/src/YAML_Doc.cpp:62:11: note: ‘sprintf’ output between 20 and 72 bytes into a destina                                                                                              tion of size 25
   62 |   sprintf (sdate,"%04d.%02d.%02d.%02d.%02d.%02d",ptm->tm_year + 1900, ptm->tm_mon+1,
      |   ~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   63 |       ptm->tm_mday, ptm->tm_hour, ptm->tm_min,ptm->tm_sec);
      |       ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/home/user/hpcg/src/OutputFile.cpp: In member function ‘std::string OutputFile::generate()’:
/home/user/hpcg/src/OutputFile.cpp:118:29: warning: ‘%02d’ directive writing between 2 and 11 bytes i                                                                                              nto a region of size between 1 and 17 [-Wformat-overflow=]
  118 |   sprintf (sdate,"%04d-%02d-%02d_%02d-%02d-%02d",ptm->tm_year + 1900, ptm->tm_mon+1,
      |                             ^~~~
/home/user/hpcg/src/OutputFile.cpp:118:11: note: ‘sprintf’ output between 20 and 72 bytes into a dest                                                                                              ination of size 25
  118 |   sprintf (sdate,"%04d-%02d-%02d_%02d-%02d-%02d",ptm->tm_year + 1900, ptm->tm_mon+1,
      |   ~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  119 |         ptm->tm_mday, ptm->tm_hour, ptm->tm_min,ptm->tm_sec);
      |         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
[100%] Linking CXX executable xhpcg
[100%] Built target xhpcg
user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun.openmpi -np 8 $(pwd)/xhpcg
user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun.openmpi --bind-to none -np 8 $(pwd)/xhpcg
user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun.openmpi --bind-to none --mca pml ob1 --mca btl tcp,self -np                                                                                               8 $(pwd)/xhpcg 2>&1 | tail -50
user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun.openmpi --bind-to none --mca pml ob1 --mca btl tcp,self -np 8 $(pwd)/xhpcg 32 24 16 2>&1 | tail -50V
tail: option used in invalid context -- 5
user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun.openmpi --bind-to none --mca pml ob1 --mca btl tcp,self -np 8 $(pwd)/xhpcg 32 24 16
^Cuser@DESKTOP-J1MSR9D:~/hpcg/build$ cat hpcg20260825T164755.txt
WARNING: PERFORMING UNPRECONDITIONED ITERATIONS
Call [0] Number of Iterations [11] Scaled Residual [1.16152e-14]
WARNING: PERFORMING UNPRECONDITIONED ITERATIONS
Call [1] Number of Iterations [11] Scaled Residual [1.15964e-14]
Call [0] Number of Iterations [2] Scaled Residual [6.01574e-17]
Call [1] Number of Iterations [2] Scaled Residual [6.01574e-17]
Departure from symmetry (scaled) for SpMV abs(x'*A*y - y'*A*x) = 0
Departure from symmetry (scaled) for MG abs(x'*Minv*y - y'*Minv*x) = 0
SpMV call [0] Residual [0]
SpMV call [1] Residual [0]
Call [0] Scaled Residual [1.60428e-13]
user@DESKTOP-J1MSR9D:~/hpcg/build$ cd ~/hpcg/build
user@DESKTOP-J1MSR9D:~/hpcg/build$ cd ~/hpcg/build
user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun.openmpi --bind-to none --mca pml ob1 --mca btl tcp,self -np 8 $(pwd)/xhpcg 20 2>&1 | tee hpcg_final_report.txt
xhpcg: /home/user/hpcg/src/GenerateCoarseProblem.cpp:50: void GenerateCoarseProblem(const SparseMatrix&): Assertion `nxf%2==0' failed.
[DESKTOP-J1MSR9D:368017] *** Process received signal ***
[DESKTOP-J1MSR9D:368017] Signal: Aborted (6)
[DESKTOP-J1MSR9D:368017] Signal code:  (-6)
[DESKTOP-J1MSR9D:368017] [ 0] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x45cb0) [0x72aefc645cb0]
[DESKTOP-J1MSR9D:368017] [ 1] /usr/lib/x86_64-linux-gnu/libc.so.6(pthread_kill+0x11c) [0x72aefc6a642c]
[DESKTOP-J1MSR9D:368017] [ 2] /usr/lib/x86_64-linux-gnu/libc.so.6(gsignal+0x1e) [0x72aefc645b7e]
[DESKTOP-J1MSR9D:368017] [ 3] /usr/lib/x86_64-linux-gnu/libc.so.6(abort+0x27) [0x72aefc6288ec]
[DESKTOP-J1MSR9D:368017] [ 4] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x29979) [0x72aefc629979]
[DESKTOP-J1MSR9D:368017] [ 5] /usr/lib/x86_64-linux-gnu/libc.so.6(__assert_fail+0xb5) [0x72aefc63bf75]
[DESKTOP-J1MSR9D:368017] [ 6] /home/user/hpcg/build/xhpcg(+0x25599) [0x5fed0ff40599]
[DESKTOP-J1MSR9D:368017] [ 7] /home/user/hpcg/build/xhpcg(+0x4dd2) [0x5fed0ff1fdd2]
[DESKTOP-J1MSR9D:368017] [ 8] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x2a601) [0x72aefc62a601]
[DESKTOP-J1MSR9D:368017] [ 9] /usr/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0x88) [0x72aefc62a718]
[DESKTOP-J1MSR9D:368017] [10] /home/user/hpcg/build/xhpcg(+0x4a35) [0x5fed0ff1fa35]
[DESKTOP-J1MSR9D:368017] *** End of error message ***
xhpcg: /home/user/hpcg/src/GenerateCoarseProblem.cpp:50: void GenerateCoarseProblem(const SparseMatrix&): Assertion `nxf%2==0' failed.
[DESKTOP-J1MSR9D:368013] *** Process received signal ***
[DESKTOP-J1MSR9D:368013] Signal: Aborted (6)
[DESKTOP-J1MSR9D:368013] Signal code:  (-6)
[DESKTOP-J1MSR9D:368013] [ 0] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x45cb0) [0x772649645cb0]
[DESKTOP-J1MSR9D:368013] [ 1] /usr/lib/x86_64-linux-gnu/libc.so.6(pthread_kill+0x11c) [0x7726496a642c]
[DESKTOP-J1MSR9D:368013] [ 2] /usr/lib/x86_64-linux-gnu/libc.so.6(gsignal+0x1e) [0x772649645b7e]
[DESKTOP-J1MSR9D:368013] [ 3] /usr/lib/x86_64-linux-gnu/libc.so.6(abort+0x27) [0x7726496288ec]
[DESKTOP-J1MSR9D:368013] [ 4] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x29979) [0x772649629979]
[DESKTOP-J1MSR9D:368013] [ 5] /usr/lib/x86_64-linux-gnu/libc.so.6(__assert_fail+0xb5) [0x77264963bf75]
[DESKTOP-J1MSR9D:368013] [ 6] /home/user/hpcg/build/xhpcg(+0x25599) [0x64e9746ab599]
[DESKTOP-J1MSR9D:368013] [ 7] /home/user/hpcg/build/xhpcg(+0x4dd2) [0x64e97468add2]
[DESKTOP-J1MSR9D:368013] [ 8] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x2a601) [0x77264962a601]
[DESKTOP-J1MSR9D:368013] [ 9] /usr/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0x88) [0x77264962a718]
[DESKTOP-J1MSR9D:368013] [10] /home/user/hpcg/build/xhpcg(+0x4a35) [0x64e97468aa35]
[DESKTOP-J1MSR9D:368013] *** End of error message ***
xhpcg: /home/user/hpcg/src/GenerateCoarseProblem.cpp:50: void GenerateCoarseProblem(const SparseMatrix&): Assertion `nxf%2==0' failed.
[DESKTOP-J1MSR9D:368019] *** Process received signal ***
[DESKTOP-J1MSR9D:368019] Signal: Aborted (6)
[DESKTOP-J1MSR9D:368019] Signal code:  (-6)
[DESKTOP-J1MSR9D:368019] [ 0] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x45cb0) [0x7a6e92645cb0]
[DESKTOP-J1MSR9D:368019] [ 1] /usr/lib/x86_64-linux-gnu/libc.so.6(pthread_kill+0x11c) [0x7a6e926a642c]
[DESKTOP-J1MSR9D:368019] [ 2] /usr/lib/x86_64-linux-gnu/libc.so.6(gsignal+0x1e) [0x7a6e92645b7e]
[DESKTOP-J1MSR9D:368019] [ 3] /usr/lib/x86_64-linux-gnu/libc.so.6(abort+0x27) [0x7a6e926288ec]
[DESKTOP-J1MSR9D:368019] [ 4] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x29979) [0x7a6e92629979]
[DESKTOP-J1MSR9D:368019] [ 5] /usr/lib/x86_64-linux-gnu/libc.so.6(__assert_fail+0xb5) [0x7a6e9263bf75]
[DESKTOP-J1MSR9D:368019] [ 6] /home/user/hpcg/build/xhpcg(+0x25599) [0x64af8c5ea599]
[DESKTOP-J1MSR9D:368019] [ 7] /home/user/hpcg/build/xhpcg(+0x4dd2) [0x64af8c5c9dd2]
[DESKTOP-J1MSR9D:368019] [ 8] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x2a601) [0x7a6e9262a601]
[DESKTOP-J1MSR9D:368019] [ 9] /usr/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0x88) [0x7a6e9262a718]
[DESKTOP-J1MSR9D:368019] [10] /home/user/hpcg/build/xhpcg(+0x4a35) [0x64af8c5c9a35]
[DESKTOP-J1MSR9D:368019] *** End of error message ***
xhpcg: /home/user/hpcg/src/GenerateCoarseProblem.cpp:50: void GenerateCoarseProblem(const SparseMatrix&): Assertion `nxf%2==0' failed.
[DESKTOP-J1MSR9D:368014] *** Process received signal ***
[DESKTOP-J1MSR9D:368014] Signal: Aborted (6)
[DESKTOP-J1MSR9D:368014] Signal code:  (-6)
[DESKTOP-J1MSR9D:368014] [ 0] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x45cb0) [0x79654f845cb0]
[DESKTOP-J1MSR9D:368014] [ 1] /usr/lib/x86_64-linux-gnu/libc.so.6(pthread_kill+0x11c) [0x79654f8a642c]
[DESKTOP-J1MSR9D:368014] [ 2] /usr/lib/x86_64-linux-gnu/libc.so.6(gsignal+0x1e) [0x79654f845b7e]
[DESKTOP-J1MSR9D:368014] [ 3] /usr/lib/x86_64-linux-gnu/libc.so.6(abort+0x27) [0x79654f8288ec]
[DESKTOP-J1MSR9D:368014] [ 4] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x29979) [0x79654f829979]
[DESKTOP-J1MSR9D:368014] [ 5] /usr/lib/x86_64-linux-gnu/libc.so.6(__assert_fail+0xb5) [0x79654f83bf75]
[DESKTOP-J1MSR9D:368014] [ 6] /home/user/hpcg/build/xhpcg(+0x25599) [0x5add1f897599]
[DESKTOP-J1MSR9D:368014] [ 7] /home/user/hpcg/build/xhpcg(+0x4dd2) [0x5add1f876dd2]
[DESKTOP-J1MSR9D:368014] [ 8] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x2a601) [0x79654f82a601]
[DESKTOP-J1MSR9D:368014] [ 9] /usr/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0x88) [0x79654f82a718]
[DESKTOP-J1MSR9D:368014] [10] /home/user/hpcg/build/xhpcg(+0x4a35) [0x5add1f876a35]
[DESKTOP-J1MSR9D:368014] *** End of error message ***
xhpcg: /home/user/hpcg/src/GenerateCoarseProblem.cpp:50: void GenerateCoarseProblem(const SparseMatrix&): Assertion `nxf%2==0' failed.
[DESKTOP-J1MSR9D:368015] *** Process received signal ***
[DESKTOP-J1MSR9D:368015] Signal: Aborted (6)
[DESKTOP-J1MSR9D:368015] Signal code:  (-6)
[DESKTOP-J1MSR9D:368015] [ 0] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x45cb0) [0x7ad345c45cb0]
[DESKTOP-J1MSR9D:368015] [ 1] /usr/lib/x86_64-linux-gnu/libc.so.6(pthread_kill+0x11c) [0x7ad345ca642c]
[DESKTOP-J1MSR9D:368015] [ 2] /usr/lib/x86_64-linux-gnu/libc.so.6(gsignal+0x1e) [0x7ad345c45b7e]
[DESKTOP-J1MSR9D:368015] [ 3] /usr/lib/x86_64-linux-gnu/libc.so.6(abort+0x27) [0x7ad345c288ec]
[DESKTOP-J1MSR9D:368015] [ 4] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x29979) [0x7ad345c29979]
[DESKTOP-J1MSR9D:368015] [ 5] /usr/lib/x86_64-linux-gnu/libc.so.6(__assert_fail+0xb5) [0x7ad345c3bf75]
[DESKTOP-J1MSR9D:368015] [ 6] /home/user/hpcg/build/xhpcg(+0x25599) [0x5fe77eb24599]
[DESKTOP-J1MSR9D:368015] [ 7] /home/user/hpcg/build/xhpcg(+0x4dd2) [0x5fe77eb03dd2]
[DESKTOP-J1MSR9D:368015] [ 8] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x2a601) [0x7ad345c2a601]
[DESKTOP-J1MSR9D:368015] [ 9] /usr/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0x88) [0x7ad345c2a718]
[DESKTOP-J1MSR9D:368015] [10] /home/user/hpcg/build/xhpcg(+0x4a35) [0x5fe77eb03a35]
[DESKTOP-J1MSR9D:368015] *** End of error message ***
xhpcg: /home/user/hpcg/src/GenerateCoarseProblem.cpp:50: void GenerateCoarseProblem(const SparseMatrix&): Assertion `nxf%2==0' failed.
[DESKTOP-J1MSR9D:368018] *** Process received signal ***
[DESKTOP-J1MSR9D:368018] Signal: Aborted (6)
[DESKTOP-J1MSR9D:368018] Signal code:  (-6)
[DESKTOP-J1MSR9D:368018] [ 0] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x45cb0) [0x751bb4e45cb0]
[DESKTOP-J1MSR9D:368018] [ 1] /usr/lib/x86_64-linux-gnu/libc.so.6(pthread_kill+0x11c) [0x751bb4ea642c]
[DESKTOP-J1MSR9D:368018] [ 2] /usr/lib/x86_64-linux-gnu/libc.so.6(gsignal+0x1e) [0x751bb4e45b7e]
[DESKTOP-J1MSR9D:368018] [ 3] /usr/lib/x86_64-linux-gnu/libc.so.6(abort+0x27) [0x751bb4e288ec]
[DESKTOP-J1MSR9D:368018] [ 4] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x29979) [0x751bb4e29979]
[DESKTOP-J1MSR9D:368018] [ 5] /usr/lib/x86_64-linux-gnu/libc.so.6(__assert_fail+0xb5) [0x751bb4e3bf75]
[DESKTOP-J1MSR9D:368018] [ 6] /home/user/hpcg/build/xhpcg(+0x25599) [0x5c126f3b0599]
[DESKTOP-J1MSR9D:368018] [ 7] /home/user/hpcg/build/xhpcg(+0x4dd2) [0x5c126f38fdd2]
[DESKTOP-J1MSR9D:368018] [ 8] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x2a601) [0x751bb4e2a601]
[DESKTOP-J1MSR9D:368018] [ 9] /usr/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0x88) [0x751bb4e2a718]
[DESKTOP-J1MSR9D:368018] [10] /home/user/hpcg/build/xhpcg(+0x4a35) [0x5c126f38fa35]
[DESKTOP-J1MSR9D:368018] *** End of error message ***
xhpcg: /home/user/hpcg/src/GenerateCoarseProblem.cpp:50: void GenerateCoarseProblem(const SparseMatrix&): Assertion `nxf%2==0' failed.
[DESKTOP-J1MSR9D:368016] *** Process received signal ***
[DESKTOP-J1MSR9D:368016] Signal: Aborted (6)
[DESKTOP-J1MSR9D:368016] Signal code:  (-6)
[DESKTOP-J1MSR9D:368016] [ 0] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x45cb0) [0x779e49445cb0]
[DESKTOP-J1MSR9D:368016] [ 1] /usr/lib/x86_64-linux-gnu/libc.so.6(pthread_kill+0x11c) [0x779e494a642c]
[DESKTOP-J1MSR9D:368016] [ 2] /usr/lib/x86_64-linux-gnu/libc.so.6(gsignal+0x1e) [0x779e49445b7e]
[DESKTOP-J1MSR9D:368016] [ 3] /usr/lib/x86_64-linux-gnu/libc.so.6(abort+0x27) [0x779e494288ec]
[DESKTOP-J1MSR9D:368016] [ 4] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x29979) [0x779e49429979]
[DESKTOP-J1MSR9D:368016] [ 5] /usr/lib/x86_64-linux-gnu/libc.so.6(__assert_fail+0xb5) [0x779e4943bf75]
[DESKTOP-J1MSR9D:368016] [ 6] /home/user/hpcg/build/xhpcg(+0x25599) [0x622daf75e599]
[DESKTOP-J1MSR9D:368016] [ 7] /home/user/hpcg/build/xhpcg(+0x4dd2) [0x622daf73ddd2]
[DESKTOP-J1MSR9D:368016] [ 8] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x2a601) [0x779e4942a601]
[DESKTOP-J1MSR9D:368016] [ 9] /usr/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0x88) [0x779e4942a718]
[DESKTOP-J1MSR9D:368016] [10] /home/user/hpcg/build/xhpcg(+0x4a35) [0x622daf73da35]
[DESKTOP-J1MSR9D:368016] *** End of error message ***
xhpcg: /home/user/hpcg/src/GenerateCoarseProblem.cpp:50: void GenerateCoarseProblem(const SparseMatrix&): Assertion `nxf%2==0' failed.
[DESKTOP-J1MSR9D:368012] *** Process received signal ***
[DESKTOP-J1MSR9D:368012] Signal: Aborted (6)
[DESKTOP-J1MSR9D:368012] Signal code:  (-6)
[DESKTOP-J1MSR9D:368012] [ 0] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x45cb0) [0x7d9ac9e45cb0]
[DESKTOP-J1MSR9D:368012] [ 1] /usr/lib/x86_64-linux-gnu/libc.so.6(pthread_kill+0x11c) [0x7d9ac9ea642c]
[DESKTOP-J1MSR9D:368012] [ 2] /usr/lib/x86_64-linux-gnu/libc.so.6(gsignal+0x1e) [0x7d9ac9e45b7e]
[DESKTOP-J1MSR9D:368012] [ 3] /usr/lib/x86_64-linux-gnu/libc.so.6(abort+0x27) [0x7d9ac9e288ec]
[DESKTOP-J1MSR9D:368012] [ 4] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x29979) [0x7d9ac9e29979]
[DESKTOP-J1MSR9D:368012] [ 5] /usr/lib/x86_64-linux-gnu/libc.so.6(__assert_fail+0xb5) [0x7d9ac9e3bf75]
[DESKTOP-J1MSR9D:368012] [ 6] /home/user/hpcg/build/xhpcg(+0x25599) [0x5b10732a5599]
[DESKTOP-J1MSR9D:368012] [ 7] /home/user/hpcg/build/xhpcg(+0x4dd2) [0x5b1073284dd2]
[DESKTOP-J1MSR9D:368012] [ 8] /usr/lib/x86_64-linux-gnu/libc.so.6(+0x2a601) [0x7d9ac9e2a601]
[DESKTOP-J1MSR9D:368012] [ 9] /usr/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0x88) [0x7d9ac9e2a718]
[DESKTOP-J1MSR9D:368012] [10] /home/user/hpcg/build/xhpcg(+0x4a35) [0x5b1073284a35]
[DESKTOP-J1MSR9D:368012] *** End of error message ***
--------------------------------------------------------------------------
Sorry!  You were supposed to get help about:
    prun:proc-aborted-strsignal
But I couldn't open the help file:
    /usr/lib/x86_64-linux-gnu/pmix2/share/pmix/help-prun.txt: No such file or directory
    /usr/lib/x86_64-linux-gnu/prrte3/share/prte/help-prun.txt: No such file or directory.
Sorry!
--------------------------------------------------------------------------
user@DESKTOP-J1MSR9D:~/hpcg/build$ mpirun.openmpi --bind-to none --mca pml ob1 --mca btl tcp,self -np 8 $(pwd)/xhpcg 64 64 64 20

项目复现步骤
HPCG 项目复习 — 有效命令总结
以下整理了在 HPCG 项目复习过程中所有成功执行的有效命令，按操作阶段排列，并附对应命令结果。
一、环境准备阶段
1. 克隆 HPCG 项目仓库
git clone https://github.com/hpcg-benchmark/hpcg.git
【结果】成功克隆，共接收 2891 个对象（9.00 MiB），完成度 100%。
remote: Enumerating objects: 2891, done.
remote: Counting objects: 100% (1315/1315), done.
remote: Compressing objects: 100% (159/159), done.
remote: Total 2891 (delta 1221), reused 1156 (delta 1156), pack-reused 1576 (from 1)
Receiving objects: 100% (2891/2891), 9.00 MiB | 67.00 KiB/s, done.
Resolving deltas: 100% (2145/2145), done.
2. 备份 APT 源配置文件
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
【结果】成功备份源配置文件（无输出表示执行成功）。
3. 替换 APT 镜像源为清华源
sudo tee /etc/apt/sources.list > /dev/null << 'EOF'
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ noble-security main restricted universe multiverse
EOF
【结果】成功写入清华镜像源配置（noble / noble-updates / noble-backports / noble-security）。
4. 更新 APT 包索引
sudo apt update
【结果】成功更新，从清华源和官方源共获取 42.7 MB 包索引数据，耗时 43s（981 kB/s）。
Fetched 42.7 MB in 43s (981 kB/s)
68 packages can be upgraded. Run 'apt list --upgradable' to see them.
5. 安装编译依赖（cmake / g++ / MPICH / OpenMP）
sudo apt install -y cmake g++ libmpich-dev libomp-dev
【结果】成功安装 14 个包，包括 cmake 4.2.3、mpich 4.3.2、libomp-dev 等。g++ 已是最新版本。
Installing:
  cmake  libmpich-dev  libomp-dev
Installing dependencies:
  cmake-data  libarchive13t64  libmpich12  libomp5  libucx-dev  mpich
  hwloc-nox  libjsoncpp26  libomp-21-dev  librhash1  libuv1t64
...全部包安装并配置完成。
二、编译 HPCG（第一次尝试 — 使用系统 g++）
6. 进入项目目录并创建构建目录
cd hpcg
mkdir build && cd build
【结果】成功进入 ~/hpcg 目录，并创建 build 子目录。
7. 使用 CMake 配置项目（指定 MPI + OpenMP）
cmake -DCMAKE_POLICY_VERSION_MINIMUM=3.5 -DHPCG_ENABLE_MPI=ON -DHPCG_ENABLE_OPENMP=ON ..
【结果】成功配置。检测到 CXX 编译器 GNU 15.2.0，找到 MPI 3.1（openmpi）和 OpenMP 4.5。
-- Found MPI_CXX: /usr/lib/x86_64-linux-gnu/openmpi/lib/libmpi.so (found version "3.1")
-- Found MPI: TRUE (found version "3.1")
-- Found OpenMP_CXX: -fopenmp (found version "4.5")
-- Found OpenMP: TRUE (found version "4.5")
-- Configuring done (2.0s)
-- Generating done (0.0s)
-- Build files have been written to: /home/user/hpcg/build
8. 查找 mpi.h 头文件位置（诊断用）
find /usr -name mpi.h
【结果】找到两个 mpi.h 位置：mpich 和 openmpi 各一份。
/usr/lib/x86_64-linux-gnu/mpich/include/mpi.h
/usr/lib/x86_64-linux-gnu/openmpi/include/mpi.h
三、编译 HPCG（第二次尝试 — 使用 mpicxx 编译器）
9. 清理旧构建目录
cd ~/hpcg
rm -rf build
【结果】成功删除旧 build 目录。
10. 重新创建构建目录
mkdir build && cd build
【结果】成功创建新的 build 目录并进入。
11. 使用 mpicxx 编译器重新配置
cmake -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
  -DHPCG_ENABLE_MPI=ON \
  -DHPCG_ENABLE_OPENMP=ON \
  -DCMAKE_CXX_COMPILER=mpicxx \
  ..
【结果】成功配置。使用 /usr/bin/mpicxx 编译器，找到 MPI 3.1 和 OpenMP 4.5。
-- The CXX compiler identification is GNU 15.2.0
-- Check for working CXX compiler: /usr/bin/mpicxx - skipped
-- Found MPI_CXX: /usr/bin/mpicxx (found version "3.1")
-- Found MPI: TRUE (found version "3.1")
-- Found OpenMP_CXX: -fopenmp (found version "4.5")
-- Found OpenMP: TRUE (found version "4.5")
-- Configuring done (1.7s)
-- Generating done (0.0s)
-- Build files have been written to: /home/user/hpcg/build
12. 编译 HPCG
make -j$(nproc)
【结果】100% 编译成功，生成 xhpcg 可执行文件。有少量 sprintf 格式警告但无错误。
[100%] Linking CXX executable xhpcg
[100%] Built target xhpcg
四、定位 MPI 运行命令
13. 查找 mpich 提供的 mpirun 路径
dpkg -L mpich | grep mpirun
【结果】找到 mpirun.mpich 位于 /usr/bin/mpirun.mpich。
/usr/bin/mpirun.mpich
14. 确认 xhpcg 二进制文件存在
ls -l xhpcg
【结果】确认 xhpcg 文件存在于当前目录，大小 319968 字节。
-rwxr-xr-x 1 user user 319968 Aug 25 15:39 xhpcg
五、退出 Conda 环境后重新编译（使用 mpicxx.openmpi）
15. 退出 Conda 基础环境
conda deactivate
【结果】成功退出 conda 环境，恢复系统默认 shell。
16. 确认 mpirun.mpich 路径
which mpirun.mpich
【结果】mpirun.mpich 位于 /usr/bin/mpirun.mpich。
/usr/bin/mpirun.mpich
17. 清理并重建构建目录
cd ~/hpcg
rm -rf build
mkdir build && cd build
【结果】成功清理旧目录并创建新的 build 目录。
18. 使用 mpicxx.openmpi 编译器配置
cmake -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
  -DHPCG_ENABLE_MPI=ON \
  -DHPCG_ENABLE_OPENMP=ON \
  -DCMAKE_CXX_COMPILER=mpicxx.openmpi \
  ..
【结果】成功配置。使用 /usr/bin/mpicxx.openmpi 编译器，找到 MPI 3.1 和 OpenMP 4.5。
-- The CXX compiler identification is GNU 15.2.0
-- Check for working CXX compiler: /usr/bin/mpicxx.openmpi - skipped
-- Found MPI_CXX: /usr/bin/mpicxx.openmpi (found version "3.1")
-- Found MPI: TRUE (found version "3.1")
-- Found OpenMP_CXX: -fopenmp (found version "4.5")
-- Found OpenMP: TRUE (found version "4.5")
-- Configuring done (1.8s)
-- Generating done (0.0s)
-- Build files have been written to: /home/user/hpcg/build
19. 再次编译 HPCG
make -j$(nproc)
【结果】100% 编译成功，生成 xhpcg 可执行文件。
[100%] Linking CXX executable xhpcg
[100%] Built target xhpcg
六、运行 HPCG 基准测试
20. 使用 mpirun.openmpi 运行 HPCG（32×24×16 网格）
mpirun.openmpi --bind-to none --mca pml ob1 --mca btl tcp,self -np 8 $(pwd)/xhpcg 32 24 16
【结果】成功运行，输出 HPCG 基准测试结果。
WARNING: PERFORMING UNPRECONDITIONED ITERATIONS
Call [0] Number of Iterations [11] Scaled Residual [1.16152e-14]
Call [1] Number of Iterations [11] Scaled Residual [1.15964e-14]
Call [0] Number of Iterations [2] Scaled Residual [6.01574e-17]
Call [1] Number of Iterations [2] Scaled Residual [6.01574e-17]
Departure from symmetry (scaled) for SpMV abs(x'*A*y - y'*A*x) = 0
Departure from symmetry (scaled) for MG abs(x'*Minv*y - y'*Minv*x) = 0
SpMV call [0] Residual [0]
SpMV call [1] Residual [0]
Call [0] Scaled Residual [1.60428e-13]
21. 使用 mpirun.openmpi 运行 HPCG（64×64×64 网格）
mpirun.openmpi --bind-to none --mca pml ob1 --mca btl tcp,self -np 8 $(pwd)/xhpcg 64 64 64 20
