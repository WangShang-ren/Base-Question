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


项目复现步骤
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
