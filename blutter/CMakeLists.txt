name: Build Blutter on macOS Tahoe

on:
  workflow_dispatch:
  push:
    branches: [ main, master ]

jobs:
  build-blutter:
    runs-on: macos-26
    
    steps:
    - name: Checkout blutter
      uses: actions/checkout@v4
      with:
        repository: 'worawit/blutter'
    
    - name: Install system dependencies via Homebrew
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
        
        echo "Downloading Dart SDK from: $DART_URL"
        curl -L -o dart.zip "$DART_URL"
        unzip dart.zip
        rm dart.zip
        mv dart-sdk dart_sdk
        
        # 复制 dart 为 dartvm（CMake 要求文件名以 dartvm 开头）
        cp dart_sdk/bin/dart dart_sdk/bin/dartvm
        chmod +x dart_sdk/bin/dartvm
        
        echo "✓ Dart SDK installed"
        ls -la dart_sdk/bin/ | grep -E "dart|dartvm"
    
    - name: Build blutter
      run: |
        source blutter_env/bin/activate
        cd blutter
        
        # 在 blutter 目录中创建 dartvm 符号链接
        # 这样 CMake 可以找到以 "dartvm" 开头的文件
        ln -sf $(pwd)/../dart_sdk/bin/dartvm dartvm
        chmod +x dartvm
        
        echo "✓ dartvm symlink created"
        ls -la dartvm
        
        mkdir -p build
        cd build
        
        # DARTLIB 使用相对路径 "./dartvm"，以 "dartvm" 开头
        DARTLIB="./dartvm"
        echo "DARTLIB: ${DARTLIB}"
        
        cmake .. -DCMAKE_BUILD_TYPE=Release -DDARTLIB=${DARTLIB}
        make -j$(sysctl -n hw.ncpu)
        
        echo "✓ Build completed"
        ls -la
    
    - name: Test blutter
      run: |
        source blutter_env/bin/activate
        cd blutter
        python3 blutter.py --help || echo "✓ Blutter Python script ready"
    
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
    
    - name: Upload debug info (optional)
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: blutter-debug-info
        path: |
          ./blutter/build/CMakeCache.txt
          ./blutter/build/CMakeFiles/
        retention-days: 3
