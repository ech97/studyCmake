# Cmake Study Note

> Started at 21/09/29 16:26



### Ref

> ```CC0 Public Domain License```
>
>  https://gist.github.com/luncliff/6e2d4eb7ca29a0afd5b592f72b80cb5c



## Configure / Generate

- configure: ```CMakeLists.txt``` 파일 해석 (Cmake)
- Generation: 해석 결과를 바탕으로 Project 파일 생성 (W/ Tool Chain)



## 1. Cmake Basic

### 1-1. Program Type

#### - add_executable

> 실행파일로 만들 목록들

```cmake
# project-example/src/CMakeLists.txt 
# case 1

add_executable(my_exe   # 이후에 나오는 .cpp 파일을 사용해 .exe를 생성한다
    main.cpp
                        # 개행을 여러번 하여도 문제되지 않습니다.
    feature1.cpp
    feature2.cpp
    algorithm3.cpp      # 상대 경로로 소스 코드를 찾아냅니다. 
                        # 현재 사용중인 CMakeList의 위치를 기준으로
                        # 경로를 지시해야 합니다
    data_structure4.cpp
    # ... 
)
```

#### - add_library

```cmake
# project-example/src/CMakeLists.txt 
# case 2.1, 2.2

add_library(my_archive  STATIC  # 정적 링킹 라이브러리(.a, .lib)
    main.cpp
    feature1.cpp
    # ...
)

add_library(my_shared_object  SHARED  # 동적 링킹 라이브러리(.so, .dll)
    main.cpp
    feature1.cpp
    # ...
)
```

#### - add_subdirectory

> 하위 폴더에 작성해놓은 CMakeLists.txt 파일 실행

```cmake
$ tree ./project-example/
./project-example
├── CMakeLists.txt          # <---- project
├── include
└── src
    ├── CMakeLists.txt      # <---- add_executable/add_library
    └── ... 
```

  

### 1-2. Dependency

>  빌드 순서를 제어하기 위해 의존성 관계 설정

```cmake
add_subdirectory(module1)
add_subdirectory(module2)
add_subdirectory(test)

# dependency: from -> { to }
add_dependencies(module1 module2)            # `module1` requires `module2`
add_dependencies(test    module1 module2)    # `test` requires `module1` & `module2
```

#### - ALIAS

```cmake
# 별명 붙이기
add_library(my_custom_logger_lib 
    # ...
)

add_library(module::logger ALIAS my_custom_logger_lib)
```

  

### 1-3. Linking

> 빌드 순서를 정했다면 이젠 프로그램 생성 중 Linking에 필요한 정보들이 공유되도록 하기 위해서는 ```target_link_libraries``` 필요

#### - target_link_libraries

```cmake
add_library(my_custom_logger_lib 
    # ...
)

target_link_libraries(my_custom_logger_lib # 
PUBLIC
    spdlog fmt
PRIVATE
    utf8proc
)
```

---

## 2. Cross Platform

### 2-1. Cmake Variables

```cmake
# *현재* CMake가 실행되는 시스템을 알려진 변수들로 확인하는 방법

# wrapper::system 같은 별명을 붙이면 상대적으로 편해진다
if(WIN32)      
    add_subdirectory(external/winrt)
    add_subdirectory(impl/win32)
elseif(APPLE)
    add_subdirectory(impl/posix)
    # additional implementation for MacOS
    add_subdirectory(impl/macos)
elseif(UNIX)
    add_subdirectory(impl/posix)
    # additional implementation with Linux API
    if(${CMAKE_SYSTEM} MATCHES Linux)
        add_subdirectory(impl/linux)
    endif()
else()
    # 지원하지 않음.
    # android.toolchain.cmake 혹은 ios.toolchain.cmake 에서 지정하는 변수들
    if(ANDROID OR IOS)
        message(FATAL_ERROR "No implementation for the platform")
    endif()
    # ...
endif()
```

  

### 2-2. include

#### - target_include_directories

> 구현이 다르더라도 공통된 헤더파일들이 있다면, 이들을 특정한 (일반적으로 include)폴더에 모아놓는다.
>
> 이 폴더에 있는 header file들은 <> 로 include하는 것이 가능하다.

```cmake
# some-huge-project/impl/win32/CMakeLists.txt

add_library(my_win32_wrapper
    src/i-love-win32.cpp
)
add_library(wrapper::system ALIAS my_win32_wrapper)

target_include_directories(my_win32_wrapper
PUBLIC
    ${CMAKE_SOURCE_DIR}/include
                # CMAKE_SOURCE_DIR 는 최상위 CMakeLists.txt가 위치한 폴더를 의미한다.
                # 이 프로젝트에서는 some-huge-project/
PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/include     
                # include 폴더를 절대경로를 사용해 접근
                # CMAKE_CURRENT_SOURCE_DIR 는 현재 해석 중인 CMakeLists.txt가 위치한 폴더를 의미한다.
                # 즉, 경로는 some-huge-project/impl/win32
)
```

```cmake
# some-huge-project/src/CMakeLists.txt

add_executable(my_system_utility
    some.cpp
    source.cpp
    files.cpp
)

target_link_libraries(my_system_utility
PRIVATE
    wrapper::system # wrapper 라이브러리들의 ALIAS
                    # target_include_directories에 PUBLIC으로 명시된
                    # ${CMAKE_SOURCE_DIR}/include 폴더를 자동으로 접근할 수 있게 된다
)
```

#### - target_sources

> ```add_executable```과 ```add_library```의 한계는 소스 파일 목록을 한번에 결정해서 전달해야 한다는 점이다. 이는 Target들이 CMakeList 파일의 끝부분에 나타나게 만들며, 소스파일 목록을 만들어야하는 불편함이 존재. 이를 해결하기 위해 CMake 3.x에서는 Target에 소스파일을 '추가'할 수 있도록 ```target_sources```를 사용

```cmake
# preview 가 구현된 소스파일을 추가할지 결정하는 변수
set(USE_PREVIEW true)

# target: my_program
add_executable(my_program
    main.cpp
    feature1.cpp
)

# ...
# add_dependencies				// 의존성 설정
# set_target_properties		
# target_include_directories	// include 폴더 설정
# target_link_libraries			// linking할 애들 엮기
# ...

target_sources(my_program
PRIVATE
    feature2.cpp
)
if(USE_PREVIEW)
    target_sources(my_program
    PRIVATE
        feature3_preview.cpp
    )
else()
```

---

## 3. Compiler 대응

> 컴파일 옵션 설정

### 3-1. 컴파일러 검사

#### - 관련 Cmake 변수

```cmake
message(STATUS "Compiler")
message(STATUS " - ID       \t: ${CMAKE_CXX_COMPILER_ID}")		# 컴파일러 이름
message(STATUS " - Version  \t: ${CMAKE_CXX_COMPILER_VERSION}")	# 컴파일러 버전
message(STATUS " - Path     \t: ${CMAKE_CXX_COMPILER}")			# 컴파일러 실행파일 경로
```

#### - compiler에 따라 다르게 처리

```cmake
if(MSVC)        # Microsoft Visual C++ Compiler
    # ...
elseif(${CMAKE_CXX_COMPILER_ID} MATCHES Clang)  # Clang + AppleClang
    # ...
elseif(${CMAKE_CXX_COMPILER_ID} MATCHES GNU)    # GNU C Compiler
    # ...
endif()
```

  

### 3-2. 컴파일 옵션 사용

#### - include(CheckCXXCompilerFlag)

> CMake Module(미리 작성된 Cmake file)들이 옵션을 지원하는지 검사할 필요가 있음

```tree```

```cmake
tree -L 2 . 
.
├── CMakeLists.txt
├── cmake             # <---- 이 프로젝트에서 필요한 cmake 파일들을 모아둔다
│   ├── display-compiler-info.cmake
│   └── test-cxx-flags.cmake
├── include
│   └── ...
├── modules
│   ├── ...
│   └── ...
├── scripts
│   └── ...
├── src
└── test
```

```test-cxx-flag.cmake```

```cmake
# test-cxx-flags.cmake
#
# `include(cmake/check-compiler-flags.cmake)` from the root CMakeList
#
include(CheckCXXCompilerFlag)

# Test latest C++ Standard and High warning level to prevent mistakes
if(MSVC)
    check_cxx_compiler_flag(/std:c++latest  cxx_latest          )
    check_cxx_compiler_flag(/W4             high_warning_level  )
elseif(${CMAKE_CXX_COMPILER_ID} MATCHES Clang)
    check_cxx_compiler_flag(-std=c++2a      cxx_latest          )	
    check_cxx_compiler_flag(-Wall           high_warning_level  )
elseif(${CMAKE_CXX_COMPILER_ID} MATCHES GNU)
    check_cxx_compiler_flag(-std=gnu++2a    cxx_latest          )	# 가능한지 체크
    check_cxx_compiler_flag(-Wextra         high_warning_level  )	# 가능한지 체크
endif()
```

```Result```

```shell
# ...
-- Compiler
--  - ID        : Clang
--  - Version   : 6.0.0
--  - Path      : /usr/bin/clang-6.0
# ...
-- Performing Test cxx_latest
-- Performing Test cxx_latest - Success
-- Performing Test high_warning_level
-- Performing Test high_warning_level - Success
# ...
```

#### - target_compile_options

> 이제 컴파일 옵션을 사용할 준비가 되었으므로, 간단히 Target에서 사용할 컴파일 옵션으로 지정하는 예시

```cmake
if(MSVC)
    target_compile_options(my_modern_cpp_lib 
    PUBLIC
        /std:c++latest /W4  # MSVC 가 식별 가능한 옵션을 지정
    )
else() # Clang + GCC
    target_compile_options(my_modern_cpp_lib
    PUBLIC
        -std=c++2a -Wall    # GCC/Clang이 식별 가능한 옵션을 지정
    PRIVATE
        -fPIC 
        -fno-rtti 
    )
endif()
```

> my_modern_cpp_lib을 ```target_link_libraries```로 사용하는 모든 Target들은 c++ lastest, Warn ALL 옵션으로 빌드가 수행된다.
>
> **옵션의 중복은 걱정할 필요 없다. CMake에서 자동으로 하나로 합쳐 적용한다**

​    

### 3-3. 매크로 선언

#### - Target_compile_definitions

> 최신 C++에서는 Macro를 대체할 방법으로 ```enum class```, ```constemxpr``` 등이 있지만, 여전히 Macro에 의존하고 있는 코드가 많은 것 또한 사실이다. 하지만 수십, 혹은 수백개의 소스파일에 Macro를 선언하려면 시간이 많이 들뿐만 아니라 이후에 수정하기도 번거로울 것이다.

```cmake
if(MSVC)
    # 묵시적으로 #define을 추가합니다 (컴파일 시간에 적용)
    target_compile_definitions(my_modern_cpp_lib
    PRIVATE
        WIN32_LEAN_AND_MEAN
        NOMINMAX    # numeric_limits를 사용할때 방해가 되는
                    # max(), min() Macro를 제거합니다
        _CRT_SECURE_NO_WARNINGS 
                    # Visual Studio로 C++에 입문했다면 한번쯤 만나본 녀석일 겁니다
                    # 오래된 코드를 위한 프로젝트라면 선택의 여지가 없을 수도 있겠죠...
    )
endif()
```

 ❗ **Macro는 소스코드에 보이지 않기 때문에 프로젝트를 가져다 쓰는 사람이 알아채기 어려우니 조심하여 사용** 

---

## 4. Cmake 파일 작성

### 4-1 Cmake Module 폴더 만들기

> 최초의 Root CMakeList 호출이나, ```add_subdirectory```는 CMakeLists.txt를 사용하지만 일반적인 경우 ```.cmake```파일을 사용한다. ToolChain 파일들이 이런 확장자를 가지고있다.

#### - include

> include함수를 이용하면, ```.cmake``` 파일을 그대로 복사-붙여넣기 한 것처럼 동작한다.
>
> ```.cmake```의 변수 사용가능

```cmake
# Root/CMakeLists.txt
project(my_new_project)

# .cmake의 내용을 복사-붙여넣기 한 것처럼 동작한다
include(cmake/check-compiler-flags.cmake) 
#
# include(CheckCXXCompilerFlag) # 또다른 CMake 기본 모듈을 가져온다
#
# if(MSVC)
#    check_cxx_compiler_flag(/std:c++latest  cxx_latest          )
#    check_cxx_compiler_flag(/W4             high_warning_level  )
# elseif(${CMAKE_CXX_COMPILER_ID} MATCHES Clang)
#    check_cxx_compiler_flag(-std=c++2a      cxx_latest          )
#    check_cxx_compiler_flag(-Wall           high_warning_level  )
# elseif(${CMAKE_CXX_COMPILER_ID} MATCHES GNU)
#    check_cxx_compiler_flag(-std=gnu++2a    cxx_latest          )
#    check_cxx_compiler_flag(-Wextra         high_warning_level  )
# endif()
#

if(cxx_latest)  # include 파일 내에서 설정한 변수를 사용 가능하다
    target_compile_options(...)
endif()

message(STATUS ${CMAKE_SOURCE_DIR})         # -- Root
message(STATUS ${CMAKE_CURRENT_SOURCE_DIR}) # -- Root

add_subdirectory(src)
    # src/CMakeLists.txt를 실행하기 전에 일부 변수들이 새로 설정된다
    #
    #   message(STATUS ${CMAKE_SOURCE_DIR})         # -- Root
    #   message(STATUS ${CMAKE_CURRENT_SOURCE_DIR}) # -- Root/src
    #

add_subdirectory(test)
    #
    #   message(STATUS ${CMAKE_SOURCE_DIR})         # -- Root
    #   message(STATUS ${CMAKE_CURRENT_SOURCE_DIR}) # -- Root/test
    #

# ...
```



