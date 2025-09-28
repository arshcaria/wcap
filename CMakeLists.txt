# CMake 最低版本要求
cmake_minimum_required(VERSION 3.15)

# 项目名称和语言
project(wcap C)

# 设置 C 语言标准为 C11
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

# 设置 MSVC 运行时库为静态链接，以匹配原始脚本
if(MSVC)
  set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

# --- 步骤 1: 编译 HLSL 着色器 ---
find_program(FXC_EXECUTABLE fxc.exe)
if(NOT FXC_EXECUTABLE)
  message(FATAL_ERROR "fxc.exe not found. Please add the Windows SDK bin directory to your PATH.")
endif()

file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/shaders)

set(SHADER_SOURCES
        ResizePassH
        ResizePassV
        ResizeLinearPassH
        ResizeLinearPassV
        ConvertSinglePass
        ConvertPass1
        ConvertPass2
)

set(GENERATED_SHADER_HEADERS "")

foreach(SHADER_NAME ${SHADER_SOURCES})
  set(HLSL_FILE ${CMAKE_CURRENT_SOURCE_DIR}/wcap_shaders.hlsl)
  set(DXBC_FILE ${CMAKE_CURRENT_BINARY_DIR}/shaders/${SHADER_NAME}.dxbc)
  set(HEADER_FILE ${CMAKE_CURRENT_BINARY_DIR}/shaders/${SHADER_NAME}.h)

  add_custom_command(
          OUTPUT ${HEADER_FILE}
          COMMAND ${FXC_EXECUTABLE} /nologo /Ges /T cs_5_0 /E ${SHADER_NAME} /Fo ${DXBC_FILE} ${HLSL_FILE}
          COMMAND ${FXC_EXECUTABLE} /nologo /compress /Vn ${SHADER_NAME}ShaderBytes /Fo ${CMAKE_CURRENT_BINARY_DIR}/shaders/${SHADER_NAME}.dcs /Fh ${HEADER_FILE} ${DXBC_FILE}
          DEPENDS ${HLSL_FILE}
          COMMENT "Compiling shader ${SHADER_NAME}"
  )
  list(APPEND GENERATED_SHADER_HEADERS ${HEADER_FILE})
endforeach()

add_custom_target(
        Shaders
        DEPENDS ${GENERATED_SHADER_HEADERS}
)

# --- 步骤 2: 定义可执行文件 ---
enable_language(RC)

add_executable(wcap
        wcap.c
        wcap.rc
        # 注意：这里不再将 manifest 作为源文件，因为我们通过链接器选项处理它
)

add_dependencies(wcap Shaders)

# --- 步骤 3: 配置编译和链接选项 ---
target_include_directories(wcap PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

execute_process(
        COMMAND git describe --always --dirty
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        OUTPUT_VARIABLE GIT_INFO
        OUTPUT_STRIP_TRAILING_WHITESPACE
)
if(GIT_INFO)
  target_compile_definitions(wcap PRIVATE WCAP_GIT_INFO="\\\"${GIT_INFO}\\\"")
endif()

target_compile_options(wcap PRIVATE
        /experimental:c11atomics
        $<$<CONFIG:Debug>:/Od /Z7 /RTC1 /W3 /WX>
        $<$<CONFIG:Release>:/GL /O1 /Oi /GS- /W3 /WX>
)
target_compile_definitions(wcap PRIVATE
        $<$<CONFIG:Debug>:_DEBUG>
        $<$<CONFIG:Release>:NDEBUG>
)

# --- 修正点: 强制 MSVC 以 UTF-8 编译 ---
if(MSVC)
  target_compile_options(wcap PRIVATE /utf-8)
endif()

# --- 修正点: 调整链接选项以避免重复的清单 ---
target_link_options(wcap PRIVATE
        /SUBSYSTEM:WINDOWS
        # 告诉链接器嵌入清单
        /MANIFEST:EMBED
        # 告诉链接器不要自动生成清单
        /MANIFEST:NO
        # 指定我们要使用的清单文件
        /MANIFESTINPUT:${CMAKE_CURRENT_SOURCE_DIR}/wcap.manifest

        $<$<CONFIG:Debug>:/DEBUG>
        $<$<CONFIG:Release>:/LTCG /OPT:REF /OPT:ICF>
)

target_link_libraries(wcap PRIVATE
        ntdll.lib
        kernel32.lib
        user32.lib
        gdi32.lib
        msimg32.lib
        dxgi.lib
        d3dcompiler.lib
        d3d11.lib
        dwmapi.lib
        shell32.lib
        shlwapi.lib
        mfplat.lib
        mfuuid.lib
        mfreadwrite.lib
        evr.lib
        strmiids.lib
        ksuser.lib
        mmdevapi.lib
        ole32.lib
        wmcodecdspuuid.lib
        avrt.lib
        uxtheme.lib
        OneCore.lib
        CoreMessaging.lib
)