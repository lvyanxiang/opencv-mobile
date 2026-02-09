# 构建无 OpenMP 版本的 OpenCV Mobile

本文档说明如何构建不依赖 OpenMP 的 OpenCV Mobile 版本。

## 为什么需要无 OpenMP 版本？

OpenMP (Open Multi-Processing) 是一个支持多平台共享内存并行编程的 API。但在某些情况下，你可能不需要 OpenMP：

- 某些嵌入式平台不支持 OpenMP
- 需要减少库的依赖复杂度
- 避免多线程带来的潜在问题
- 减小库的体积

## 文件说明

已创建以下文件来支持无 OpenMP 构建：

1. **[opencv4_cmake_options_no_openmp.txt](opencv4_cmake_options_no_openmp.txt)** - CMake 配置文件
   - 将 `-DWITH_OPENMP=ON` 改为 `-DWITH_OPENMP=OFF`
   - 所有其他配置与标准版本相同

2. **[.github/workflows/release-no-openmp.yml](.github/workflows/release-no-openmp.yml)** - GitHub Actions 工作流
   - 自动构建无 OpenMP 的 Android 版本
   - 跳过 `link-openmp.patch` 补丁的应用

## 主要区别

### 与标准版本的区别：

| 项目 | 标准版本 | 无 OpenMP 版本 |
|------|----------|----------------|
| OpenMP 支持 | ✓ | ✗ |
| CMake 选项 | `-DWITH_OPENMP=ON` | `-DWITH_OPENMP=OFF` |
| 应用补丁 | 包含 `link-openmp.patch` | 不包含 |
| 性能 | 多线程并行加速 | 单线程执行 |
| 依赖 | 需要	libomp (Android) | 无额外依赖 |

## 在 GitHub 上构建

### 方法 1：手动触发工作流

1. 推送代码到你的 GitHub 仓库
2. 进入 GitHub Actions 页面
3. 选择 `release-no-openmp` 工作流
4. 点击 `Run workflow` 按钮

### 方法 2：通过标签触发

推送一个标签来触发构建：

```bash
git tag v4.13.0-no-openmp-test
git push origin v4.13.0-no-openmp-test
```

### 下载构建产物

构建完成后，可以从 GitHub Actions 的 Artifacts 中下载：

- `opencv-mobile-4.13.0-no-openmp-source` - 源码包
- `opencv-mobile-4.13.0-no-openmp-android` - Android 构建产物

## 本地构建

如果你想在本地构建无 OpenMP 版本，可以按照以下步骤操作：

### 1. 准备源码

```bash
# 下载 OpenCV 源码
wget https://github.com/opencv/opencv/archive/4.13.0.zip
unzip 4.13.0.zip
cd opencv-4.13.0

# 应用补丁（跳过 link-openmp.patch）
patch -p1 -i ../patches/opencv-4.13.0-no-gpu.patch
patch -p1 -i ../patches/opencv-4.13.0-no-rtti.patch
patch -p1 -i ../patches/opencv-4.13.0-no-zlib.patch
patch -p1 -i ../patches/opencv-4.13.0-fix-windows-arm-arch.patch
patch -p1 -i ../patches/opencv-4.13.0-minimal-install.patch
patch -p1 -i ../patches/opencv-4.13.0-no-atomic.patch
patch -p1 -i ../patches/opencv-4.13.0-unsafe-xadd.patch
# 注意：不要应用 opencv-4.13.0-link-openmp.patch
```

### 2. 使用 CMake 构建

```bash
mkdir build && cd build

# 使用无 OpenMP 的配置文件
cmake -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=install \
  -DBUILD_opencv_world=OFF \
  $(cat ../opencv4_cmake_options_no_openmp.txt) \
  ..

make -j4
make install
```

### 3. Android 构建

```bash
# 设置 Android NDK 路径
export ANDROID_NDK=/path/to/your/ndk

# 构建 arm64-v8a
mkdir build-arm64-v8a && cd build-arm64-v8a
cmake -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-21 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=install \
  $(cat ../opencv4_cmake_options_no_openmp.txt) \
  ..
make -j4
make install
```

## 使用构建的库

构建完成后，使用方式与标准版本完全相同：

### Android

```cmake
set(OpenCV_DIR ${CMAKE_SOURCE_DIR}/opencv-mobile-4.13.0-no-openmp-android/sdk/native/jni)
find_package(OpenCV REQUIRED)

target_link_libraries(your_jni_target ${OpenCV_LIBS})
```

### 其他平台

```cmake
set(OpenCV_DIR ${CMAKE_SOURCE_DIR}/opencv-mobile-4.13.0-no-openmp/lib/cmake/opencv4)
find_package(OpenCV REQUIRED)

target_link_libraries(your_target ${OpenCV_LIBS})
```

## 注意事项

1. **性能差异**：无 OpenMP 版本在多核设备上可能会有性能下降，因为无法利用多线程并行

2. **兼容性**：如果你的代码依赖于 OpenMP 的功能（如并行 for 循环），需要修改代码

3. **库大小**：无 OpenMP 版本不会链接 libomp，可以略微减小最终应用的大小

4. **测试**：建议在目标平台上充分测试，确保性能满足需求

## 扩展其他平台

当前 workflow 仅包含 Android 构建。如需添加其他平台（iOS、Linux、Windows 等），可以参考 [.github/workflows/release.yml](.github/workflows/release.yml) 中的对应部分，并确保：

1. 不应用 `link-openmp.patch` 补丁
2. 使用 `opencv4_cmake_options_no_openmp.txt` 配置文件
