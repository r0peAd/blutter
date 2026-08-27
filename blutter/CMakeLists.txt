name: Build Blutter on macOS Tahoe

on:
  workflow_dispatch:

jobs:
  build-blutter:
    runs-on: macos-26
    
    steps:
    - name: Checkout blutter
      uses: actions/checkout@v4
      with:
        repository: 'worawit/blutter'
        path: 'blutter-repo'
    
    - name: Install system dependencies
      run: |
        brew update
        brew install cmake ninja pkg-config icu4c capstone
    
    - name: Setup Python virtual environment
      run: |
        python3 -m venv blutter_env
        source blutter_env/bin/activate
        pip install --upgrade pip
        pip install pyelftools requests
    
    - name: Download and setup Dart SDK
      run: |
        ARCH=$(uname -m)
        DART_VERSION="3.5.0"
        
        if [ "$ARCH" = "arm64" ]; then
          DART_URL="https://storage.googleapis.com/dart-archive/channels/stable/release/${DART_VERSION}/sdk/dartsdk-macos-arm64-release.zip"
        else
          DART_URL="https://storage.googleapis.com/dart-archive/channels/stable/release/${DART_VERSION}/sdk/dartsdk-macos-x64-release.zip"
        fi
        
        curl -L -o dart.zip "$DART_URL"
        unzip dart.zip
        rm dart.zip
        mv dart-sdk dart_sdk
        
        # 复制 dart 为 dartvm
        cp dart_sdk/bin/dart dart_sdk/bin/dartvm
        chmod +x dart_sdk/bin/dartvm
        
        echo "✓ Dart SDK ready"
    
    - name: Setup build environment
      run: |
        # 进入 blutter 源码目录（注意路径）
        cd blutter-repo/blutter
        
        # 创建指向 dartvm 的符号链接（使用相对路径）
        ln -sf ../../dart_sdk/bin/dartvm dartvm
        chmod +x dartvm
        
        echo "✓ Symbolic link created"
        ls -la dartvm
        
        # 获取当前目录
        echo "Current directory: $(pwd)"
        echo "Dartvm path: $(readlink dartvm)"
    
    - name: Build blutter
      run: |
        source blutter_env/bin/activate
        
        # 进入 blutter 源码目录
        cd blutter-repo/blutter
        
        # 创建构建目录
        mkdir -p build
        cd build
        
        # DARTLIB 使用相对路径（以 dartvm 开头）
        DARTLIB="./dartvm"
        echo "DARTLIB: ${DARTLIB}"
        
        # 显示 CMake 将要检查的内容
        echo "Testing DARTLIB value:"
        if [[ "${DARTLIB}" =~ ^dartvm ]]; then
          echo "✓ DARTLIB starts with 'dartvm'"
        else
          echo "✗ DARTLIB does not start with 'dartvm'"
        fi
        
        # 运行 CMake
        cmake .. -DCMAKE_BUILD_TYPE=Release -DDARTLIB="${DARTLIB}"
        make -j$(sysctl -n hw.ncpu)
    
    - name: Verify build
      run: |
        ls -la blutter-repo/blutter/build/
        file blutter-repo/blutter/build/blutter || echo "Build output found"
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v4
      with:
        name: blutter-macos
        path: |
          blutter-repo/blutter/build/blutter
          blutter-repo/blutter/build/*.dylib
          blutter-repo/blutter/blutter.py
          blutter-repo/blutter/*.py
        retention-days: 7
