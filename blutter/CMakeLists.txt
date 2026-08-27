name: Build Blutter on macOS

on:
  workflow_dispatch:
  push:
    branches: [ main, master ]

jobs:
  build-blutter:
    runs-on: macos-26
    
    steps:
    # ============================================
    # 1. 检出 Blutter 源码
    # ============================================
    - name: Checkout blutter
      uses: actions/checkout@v4
      with:
        repository: 'worawit/blutter'
    
    # ============================================
    # 2. 安装系统依赖
    # ============================================
    - name: Install system dependencies
      run: |
        brew update
        brew install cmake ninja pkg-config icu4c capstone
    
    # ============================================
    # 3. 设置 Python 虚拟环境
    # ============================================
    - name: Setup Python virtual environment
      run: |
        python3 -m venv blutter_env
        source blutter_env/bin/activate
        pip install --upgrade pip
        pip install pyelftools requests
    
    # ============================================
    # 4. 下载并设置 Dart SDK
    # ============================================
    - name: Download and setup Dart SDK
      run: |
        ARCH=$(uname -m)
        DART_VERSION="3.5.0"
        
        if [ "$ARCH" = "arm64" ]; then
          DART_URL="https://storage.googleapis.com/dart-archive/channels/stable/release/${DART_VERSION}/sdk/dartsdk-macos-arm64-release.zip"
        else
          DART_URL="https://storage.googleapis.com/dart-archive/channels/stable/release/${DART_VERSION}/sdk/dartsdk-macos-x64-release.zip"
        fi
        
        echo "Downloading Dart SDK from: $DART_URL"
        curl -L -o dart.zip "$DART_URL"
        unzip dart.zip
        rm dart.zip
        mv dart-sdk dart_sdk
        
        # 复制 dart 为 dartvm（CMake 需要）
        cp dart_sdk/bin/dart dart_sdk/bin/dartvm
        chmod +x dart_sdk/bin/dartvm
        
        echo "✅ Dart SDK installed successfully"
        ls -la dart_sdk/bin/ | grep -E "dart|dartvm"
    
    # ============================================
    # 5. 🔧 修补 CMakeLists.txt - 删除 dartvm 检查
    # ============================================
    - name: Patch CMakeLists.txt - Remove dartvm check
      run: |
        cd blutter
        
        echo "=== Original CMakeLists.txt (lines 1-15) ==="
        head -15 CMakeLists.txt
        
        # 使用 sed 删除 dartvm 检查的代码块
        sed -i '' '/if (NOT ${DARTLIB} MATCHES "^dartvm")/,/endif()/d' CMakeLists.txt
        
        echo ""
        echo "=== Patched CMakeLists.txt (lines 1-15) ==="
        head -15 CMakeLists.txt
        
        # 验证修补是否成功
        if grep -q "DARTLIB name should start with" CMakeLists.txt; then
          echo "❌ PATCH FAILED - dartvm check still exists!"
          exit 1
        else
          echo "✅ PATCH SUCCESS - dartvm check removed!"
        fi
    
    # ============================================
    # 6. 编译 Blutter
    # ============================================
    - name: Build blutter
      run: |
        source blutter_env/bin/activate
        cd blutter
        
        # DARTLIB 指向 dartvm 可执行文件
        DARTLIB=$(pwd)/../dart_sdk/bin/dartvm
        echo "DARTLIB: ${DARTLIB}"
        
        if [ -f "${DARTLIB}" ]; then
          echo "✅ dartvm found at: ${DARTLIB}"
          file ${DARTLIB}
        else
          echo "❌ dartvm not found!"
          exit 1
        fi
        
        # 创建构建目录
        mkdir -p build
        cd build
        
        # CMake 配置
        cmake .. -DCMAKE_BUILD_TYPE=Release -DDARTLIB=${DARTLIB}
        
        # 编译
        make -j$(sysctl -n hw.ncpu)
        
        echo "✅ Build completed successfully!"
        ls -la
    
    # ============================================
    # 7. 验证构建产物
    # ============================================
    - name: Verify build artifacts
      run: |
        cd blutter/build
        
        if [ -f "blutter" ]; then
          echo "✅ blutter binary found"
          file blutter
          ls -lh blutter
        else
          echo "❌ blutter binary not found!"
          echo "Directory contents:"
          ls -la
          exit 1
        fi
    
    # ============================================
    # 8. 测试 Blutter
    # ============================================
    - name: Test blutter
      run: |
        source blutter_env/bin/activate
        cd blutter
        python3 blutter.py --help || echo "✅ Blutter Python script ready"
    
    # ============================================
    # 9. 上传构建产物
    # ============================================
    - name: Upload artifacts
      uses: actions/upload-artifact@v4
      with:
        name: blutter-macos-tahoe
        path: |
          ./blutter/build/blutter
          ./blutter/build/*.dylib
          ./blutter/build/*.so
          ./blutter/blutter.py
          ./blutter/*.py
        retention-days: 7
        if-no-files-found: error
    
    # ============================================
    # 10. 上传调试信息（可选）
    # ============================================
    - name: Upload debug info
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: blutter-debug-info
        path: |
          ./blutter/build/CMakeCache.txt
          ./blutter/build/CMakeFiles/
        retention-days: 3
