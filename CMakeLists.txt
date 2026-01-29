# =========== Modern CMake Start ===================
cmake_minimum_required(VERSION 3.18) # 3.18+ 开始支持 CMAKE_CUDA_ARCHITECTURES

# 声明项目语言包含 CUDA，这会自动触发编译器的智能探测
project(cuda_hgemm LANGUAGES CXX CUDA)

# 1. 设置 C++ 和 CUDA 标准 (对应原脚本的 -std=c++11)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CUDA_STANDARD 11)
set(CMAKE_CUDA_STANDARD_REQUIRED ON)

# 2. 接收命令行参数
# 对应脚本里的 -DHGEMM_VERBOSE_MAKEFILE
set(CMAKE_VERBOSE_MAKEFILE ${HGEMM_VERBOSE_MAKEFILE})

# 设置 CUDA 架构 (默认为 86，也可以通过 cmake -DCMAKE_CUDA_ARCHITECTURES=80 覆盖)
if(NOT CMAKE_CUDA_ARCHITECTURES)
    set(CMAKE_CUDA_ARCHITECTURES 86)
endif()

# 3. 查找依赖包
# 现代 CMake 使用 CUDAToolkit 来找 cuBLAS 等库，而不是 find_package(CUDA)
find_package(CUDAToolkit REQUIRED) 
find_package(gflags REQUIRED)
find_package(OpenMP REQUIRED)

# 4. 收集源文件 (保持原有逻辑)
# 注意：Modern CMake 推荐显式列出文件，但为了方便你的现有结构，GLOB 依然可用
file(GLOB HGEMM_SRCS 
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cu"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/wmma/*.cu"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/mma/*.cu"
)

# 5. 定义可执行文件 (关键变化！)
# 不再使用 cuda_add_executable，直接用标准的 add_executable
add_executable(hgemm ${HGEMM_SRCS})

# 6. 设置编译选项 (关键变化！)
# 这一步替换了原来复杂的 CUDA_NVCC_FLAGS 操作

# 6.1 通用选项
target_compile_options(hgemm PRIVATE
    $<$<COMPILE_LANGUAGE:CXX>: -Wall -Werror -Wextra > # C++ 编译器的警告
    $<$<COMPILE_LANGUAGE:CUDA>: --expt-relaxed-constexpr -Xcompiler -fopenmp > # NVCC 的选项
    
    # --- 核心修复：在这里优雅地解决 GCC 版本不兼容问题 ---
    $<$<COMPILE_LANGUAGE:CUDA>: -allow-unsupported-compiler >
)

# 6.2 Debug / Release 特定选项
# 使用生成器表达式自动判断 Build Type，无需 if-else
target_compile_options(hgemm PRIVATE
    $<$<CONFIG:Debug>: -g -lineinfo -Xptxas=-v -O0 >
    $<$<CONFIG:Release>: --use_fast_math -O3 >
)

# 7. 设置包含目录
target_include_directories(hgemm PRIVATE
    "${CMAKE_CURRENT_SOURCE_DIR}/src/common"
    ${GFLAGS_INCLUDE_DIR}
    # CUDA Include 目录会自动处理，不需要手动加 /usr/local/cuda/include
)

# 8. 链接库 (关键变化！)
target_link_libraries(hgemm PRIVATE
    CUDA::cublas           # 现代写法：链接 cuBLAS
    CUDA::cudart           # 链接 CUDA Runtime
    OpenMP::OpenMP_CXX     # 链接 OpenMP
    ${GFLAGS_LIBRARIES}    # 链接 gflags
)

# 9. 安装
install(TARGETS hgemm RUNTIME DESTINATION bin)