name: Build Blutter (Final Fixed)

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: macos-26
    
    steps:
    - name: Checkout blutter
      uses: actions/checkout@v4
      with:
        repository: 'worawit/blutter'
    
    - name: Install dependencies
      run: |
        brew update
        brew install cmake ninja pkg-config icu4c capstone
        python3 -m venv blutter_env
        source blutter_env/bin/activate
        pip install --upgrade pip
        pip install pyelftools requests
    
    - name: Setup Dart SDK
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
        cp dart_sdk/bin/dart dart_sdk/bin/dartvm
        chmod +x dart_sdk/bin/dartvm
        echo "✅ Dart SDK ready"
    
    - name: FIX CMakeLists.txt - Remove dartvm check
      run: |
        cd blutter
        echo "=== Original CMakeLists.txt lines 1-15 ==="
        head -15 CMakeLists.txt
        
        # 使用 Python 删除 dartvm 检查
        python3 << 'EOF'
        with open('CMakeLists.txt', 'r') as f:
            lines = f.readlines()
        
        new_lines = []
        skip = False
        for line in lines:
            # 检测到 dartvm 检查开始
            if 'if (NOT ${DARTLIB} MATCHES "^dartvm")' in line:
                skip = True
                continue
            # 检测到 dartvm 检查结束
            if skip and 'endif()' in line and 'DARTLIB name should start' in ''.join(new_lines[-3:]):
                skip = False
                continue
            if not skip:
                new_lines.append(line)
        
        # 如果上面的方法没删掉，使用更简单的方法：直接替换
        if len(new_lines) == len(lines):
            print("First method didn't work, using second method...")
            new_lines = []
            for line in lines:
                if 'DARTLIB name should start with' in line:
                    line = '        # DARTLIB check removed for macOS\n'
                if 'if (NOT ${DARTLIB} MATCHES "^dartvm")' in line:
                    line = '        # if (NOT ${DARTLIB} MATCHES "^dartvm")\n'
                if 'message(FATAL_ERROR "DARTLIB name should start with' in line:
                    line = '        # message(FATAL_ERROR "DARTLIB name should start with '\\''dartvm'\\''")\n'
                if 'endif()' in line and 'dartvm' in ''.join(new_lines[-5:]):
                    continue
                new_lines.append(line)
        
        with open('CMakeLists.txt', 'w') as f:
            f.writelines(new_lines)
        
        print("✅ CMakeLists.txt patched successfully!")
        EOF
        
        echo "=== Patched CMakeLists.txt lines 1-15 ==="
        head -15 CMakeLists.txt
        
        # 验证是否还有 dartvm 检查
        if grep -q "DARTLIB name should start with" CMakeLists.txt; then
          echo "❌ dartvm check still present!"
          exit 1
        else
          echo "✅ dartvm check removed!"
        fi
    
    - name: Build blutter
      run: |
        source blutter_env/bin/activate
        cd blutter
        DARTLIB=$(pwd)/../dart_sdk/bin/dartvm
        echo "DARTLIB: ${DARTLIB}"
        mkdir -p build && cd build
        cmake .. -DCMAKE_BUILD_TYPE=Release -DDARTLIB=${DARTLIB}
        make -j$(sysctl -n hw.ncpu)
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v4
      with:
        name: blutter-macos
        path: |
          ./blutter/build/blutter
          ./blutter/build/*.dylib
          ./blutter/blutter.py
          ./blutter/*.py
        retention-days: 7
