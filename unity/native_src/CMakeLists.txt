# Tencent is pleased to support the open source community by making xLua available.
# Copyright (C) 2016 Tencent. All rights reserved.
# Licensed under the MIT License (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at
# http://opensource.org/licenses/MIT
# Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

cmake_minimum_required(VERSION 3.15)
cmake_policy(SET CMP0091 NEW)

project(PuertsPlugin LANGUAGES C CXX)

include(CheckCCompilerFlag)

option ( USING_V8 "using v8" ON )
option ( USING_QJS "using quickjs" ON )
option ( USING_QJS_SUFFIX "add _qjs to namespace of simule v8 api " ON )
option ( WITH_SYMBOLS "generate debug symbols" OFF )

if("${JS_ENGINE}" MATCHES "^v8_10.6.194")
    set(CMAKE_CXX_STANDARD 17)
else ()
    set(CMAKE_CXX_STANDARD 14)
endif ()

if ( IOS )
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fembed-bitcode")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fembed-bitcode")
endif ()

find_path(PUERTS_PROJECT_DIR NAMES SConstruct
    PATHS 
    ${PROJECT_SOURCE_DIR}
    NO_DEFAULT_PATH
)

MARK_AS_ADVANCED(PUERTS_PROJECT_DIR)

if ( NOT DEFINED JS_ENGINE )
    set(JS_ENGINE v8)
endif()

set(USING_MULT_BACKEND OFF)
if("${JS_ENGINE}" MATCHES "^mult")
    set(USING_MULT_BACKEND ON)
endif()

set(USING_QUICKJS OFF)
if("${JS_ENGINE}" MATCHES "^quickjs")
    set(USING_QUICKJS ON)
endif()

set(BACKEND_ROOT ${PROJECT_SOURCE_DIR}/../native_src/.backends/${JS_ENGINE})

set(ThirdParty ${PROJECT_SOURCE_DIR}/../../unreal/Puerts/ThirdParty)

if(NOT ("${JS_ENGINE}" STREQUAL "quickjs"))
    string (REPLACE ";" "$<SEMICOLON>${BACKEND_ROOT}" BACKEND_INC_NAMES "${BACKEND_ROOT}${BACKEND_INC_NAMES}")
    string (REPLACE ";" "$<SEMICOLON>${BACKEND_ROOT}" BACKEND_LIB_NAMES "${BACKEND_ROOT}${BACKEND_LIB_NAMES}")
endif()
string (REPLACE ";" "$<SEMICOLON>" BACKEND_DEFINITIONS "${BACKEND_DEFINITIONS}")

include_directories(
    ${PROJECT_SOURCE_DIR}
    ${ThirdParty}/Include/websocketpp
    ${ThirdParty}/Include/asio
    Inc
    ${PROJECT_SOURCE_DIR}/../../unreal/Puerts/Source/JsEnv/Private
    ${BACKEND_INC_NAMES}
)

set ( PUERTS_INC
    Inc/Common.h
    Inc/Log.h
    Inc/BackendEnv.h
    Inc/JSEngine.h
    Inc/V8Utils.h
    Inc/JSFunction.h
    Inc/IPuertsPlugin.h
    ${PROJECT_SOURCE_DIR}/../../unreal/Puerts/Source/JsEnv/Private/V8InspectorImpl.h
    ${PROJECT_SOURCE_DIR}/../../unreal/Puerts/Source/JsEnv/Private/PromiseRejectCallback.hpp
)


macro(source_group_by_dir proj_dir source_files)
    if(MSVC OR APPLE)
        get_filename_component(sgbd_cur_dir ${proj_dir} ABSOLUTE)
        foreach(sgbd_file ${${source_files}})
			get_filename_component(sgbd_abs_file ${sgbd_file} ABSOLUTE)
            file(RELATIVE_PATH sgbd_fpath ${sgbd_cur_dir} ${sgbd_abs_file})
            string(REGEX REPLACE "\(.*\)/.*" \\1 sgbd_group_name ${sgbd_fpath})
            string(COMPARE EQUAL ${sgbd_fpath} ${sgbd_group_name} sgbd_nogroup)
            string(REPLACE "/" "\\" sgbd_group_name ${sgbd_group_name})
            if(sgbd_nogroup)
                set(sgbd_group_name "\\")
            endif(sgbd_nogroup)
            source_group(${sgbd_group_name} FILES ${sgbd_file})
        endforeach(sgbd_file)
    endif(MSVC OR APPLE)
endmacro(source_group_by_dir)

source_group_by_dir(${CMAKE_CURRENT_SOURCE_DIR} PUERTS_INC)
source_group_by_dir(${CMAKE_CURRENT_SOURCE_DIR} PUERTS_SRC)

if( USING_MULT_BACKEND )
    message("mult backend>>>>>>>>>>>>>>>>>>>>>>>>")
    set ( PUERTS_SRC
        Src/Log.cpp
        Src/PuertsMultBackend.cpp
    )
else ()
    set ( PUERTS_SRC
        Src/Puerts.cpp
        Src/Log.cpp
        Src/BackendEnv.cpp
        Src/JSEngine.cpp
        Src/JSFunction.cpp
        ${PROJECT_SOURCE_DIR}/../../unreal/Puerts/Source/JsEnv/Private/V8InspectorImpl.cpp
        Src/pesapi_wrap.c
    )
endif()

set(PUERTS_COMPILE_DEFINITIONS)
set(PUERTS_BACKEND_SRC)
set(BEACKEND_QUICKJS_INC)
set(BEACKEND_QUICKJS_SRC)


if(USING_QUICKJS OR USING_MULT_BACKEND)
    set(CMAKE_C_STANDARD_REQUIRED ON)
    set(CMAKE_C_EXTENSIONS ON)
    set(CMAKE_C_STANDARD 11)
    
    macro(xcheck_add_c_compiler_flag FLAG)
        string(REPLACE "-" "" FLAG_NO_HYPHEN ${FLAG})
        check_c_compiler_flag(${FLAG} COMPILER_SUPPORTS_${FLAG_NO_HYPHEN})
        if(COMPILER_SUPPORTS_${FLAG_NO_HYPHEN})
            add_compile_options(${FLAG})
        endif()
    endmacro()
    
    xcheck_add_c_compiler_flag(-Wno-implicit-fallthrough)
    xcheck_add_c_compiler_flag(-Wno-sign-compare)
    xcheck_add_c_compiler_flag(-Wno-missing-field-initializers)
    xcheck_add_c_compiler_flag(-Wno-unused-parameter)
    xcheck_add_c_compiler_flag(-Wno-unused-but-set-variable)
    xcheck_add_c_compiler_flag(-Wno-array-bounds)
    xcheck_add_c_compiler_flag(-Wno-format-truncation)
    xcheck_add_c_compiler_flag(-funsigned-char)

    # ClangCL is command line compatible with MSVC, so 'MSVC' is set.
    if(MSVC)
        xcheck_add_c_compiler_flag(-Wno-unsafe-buffer-usage)
        xcheck_add_c_compiler_flag(-Wno-sign-conversion)
        xcheck_add_c_compiler_flag(-Wno-nonportable-system-include-path)
        xcheck_add_c_compiler_flag(-Wno-implicit-int-conversion)
        xcheck_add_c_compiler_flag(-Wno-shorten-64-to-32)
        xcheck_add_c_compiler_flag(-Wno-reserved-macro-identifier)
        xcheck_add_c_compiler_flag(-Wno-reserved-identifier)
        xcheck_add_c_compiler_flag(-Wdeprecated-declarations)
        xcheck_add_c_compiler_flag(/experimental:c11atomics)
        xcheck_add_c_compiler_flag(/wd4018) # -Wno-sign-conversion
        xcheck_add_c_compiler_flag(/wd4061) # -Wno-implicit-fallthrough
        xcheck_add_c_compiler_flag(/wd4100) # -Wno-unused-parameter
        xcheck_add_c_compiler_flag(/wd4200) # -Wno-zero-length-array
        xcheck_add_c_compiler_flag(/wd4242) # -Wno-shorten-64-to-32
        xcheck_add_c_compiler_flag(/wd4244) # -Wno-shorten-64-to-32
        xcheck_add_c_compiler_flag(/wd4245) # -Wno-sign-compare
        xcheck_add_c_compiler_flag(/wd4267) # -Wno-shorten-64-to-32
        xcheck_add_c_compiler_flag(/wd4388) # -Wno-sign-compare
        xcheck_add_c_compiler_flag(/wd4389) # -Wno-sign-compare
        xcheck_add_c_compiler_flag(/wd4710) # Function not inlined
        xcheck_add_c_compiler_flag(/wd4711) # Function was inlined
        xcheck_add_c_compiler_flag(/wd4820) # Padding added after construct
        xcheck_add_c_compiler_flag(/wd4996) # -Wdeprecated-declarations
        xcheck_add_c_compiler_flag(/wd5045) # Compiler will insert Spectre mitigation for memory load if /Qspectre switch specified
        xcheck_add_c_compiler_flag(/wd4514)
        xcheck_add_c_compiler_flag(/wd4146)
        xcheck_add_c_compiler_flag(/wd4334)
        xcheck_add_c_compiler_flag(-fchar8_t)
        xcheck_add_c_compiler_flag(/Zc:char8_t)
    endif()

    xcheck_add_c_compiler_flag(-fno-strict-aliasing)

    if ( APPLE )
        xcheck_add_c_compiler_flag(-Wno-shorten-64-to-32)
    endif()

    xcheck_add_c_compiler_flag(-Wno-unused-variable)

    if(APPLE AND CMAKE_C_COMPILER_ID STREQUAL "GNU" AND CMAKE_C_COMPILER_VERSION VERSION_GREATER_EQUAL 11)
        xcheck_add_c_compiler_flag(-Wno-maybe-uninitialized)
    endif()

    if(CMAKE_SYSTEM_NAME STREQUAL "WASI")
        add_compile_definitions(
            _WASI_EMULATED_PROCESS_CLOCKS
            _WASI_EMULATED_SIGNAL
        )
        add_link_options(
            -lwasi-emulated-process-clocks
            -lwasi-emulated-signal
        )
    endif()

    if(CMAKE_BUILD_TYPE MATCHES "Debug")
        add_compile_options(-O0)
        xcheck_add_c_compiler_flag(-ggdb)
        xcheck_add_c_compiler_flag(-fno-omit-frame-pointer)
    endif()
    
    if(WIN32)
        list(APPEND PUERTS_COMPILE_DEFINITIONS WIN32_LEAN_AND_MEAN _WIN32_WINNT=0x0602)
    endif()

    list(APPEND BEACKEND_QUICKJS_INC ${PROJECT_SOURCE_DIR}/backend-quickjs/include
                        ${PROJECT_SOURCE_DIR}/backend-quickjs/quickjs
    )
    
    list(APPEND BEACKEND_QUICKJS_SRC
                        ${PROJECT_SOURCE_DIR}/backend-quickjs/src/v8-impl.cc
                        ${PROJECT_SOURCE_DIR}/backend-quickjs/quickjs/cutils.c
                        ${PROJECT_SOURCE_DIR}/backend-quickjs/quickjs/libbf.c
                        ${PROJECT_SOURCE_DIR}/backend-quickjs/quickjs/libregexp.c
                        ${PROJECT_SOURCE_DIR}/backend-quickjs/quickjs/libunicode.c
                        ${PROJECT_SOURCE_DIR}/backend-quickjs/quickjs/quickjs.c
    )
endif()

if (NOT WITHOUT_IL2CPP_OPTIMIZATION)
    if (USING_MULT_BACKEND)
        #if (USING_V8) #放这里会受到CUSTOMV8NAMESPACE=v8_qjs的影响
        #    list(APPEND PUERTS_SRC
        #       Src/CppObjectMapper.cpp
        #       Src/DataTransfer.cpp
        #       Src/PesapiAddonLoad.cpp
        #       Src/PesapiV8Impl.cpp
        #    )
        #endif()
    else ()
        if (NOT USING_QUICKJS)
            list(APPEND PUERTS_SRC
               Src/CppObjectMapper.cpp
               Src/DataTransfer.cpp
               Src/PesapiAddonLoad.cpp
               Src/PesapiV8Impl.cpp
            )
        endif()
        
        if (USING_QUICKJS)
            list(APPEND BEACKEND_QUICKJS_INC ${PROJECT_SOURCE_DIR}/papi-quickjs/include)
            list(APPEND PUERTS_SRC
               papi-quickjs/source/CppObjectMapperQuickjs.cpp
               papi-quickjs/source/PapiQuickjsImpl.cpp
               Src/PesapiV8Impl.cpp
            )
        endif()
   endif()
endif()

if (NOT WITHOUT_IL2CPP_OPTIMIZATION)
    list(APPEND PUERTS_SRC
       Src/JSClassRegister.cpp
    )
    list(APPEND BEACKEND_QUICKJS_INC ${PROJECT_SOURCE_DIR}/papi-quickjs/include)
endif()

if(USING_QUICKJS)
    list(APPEND PUERTS_COMPILE_DEFINITIONS WITH_QUICKJS)
elseif (NOT USING_MULT_BACKEND)
    list(APPEND PUERTS_COMPILE_DEFINITIONS WITH_V8)
endif()

if ( WITH_WEBSOCKET )
    list(APPEND PUERTS_COMPILE_DEFINITIONS WITH_WEBSOCKET)
    if (USING_MULT_BACKEND)
        list(APPEND PUERTS_BACKEND_SRC ${PROJECT_SOURCE_DIR}/../../unreal/Puerts/Source/JsEnv/Private/WebSocketImpl.cpp)
    else ()
        list(APPEND PUERTS_SRC ${PROJECT_SOURCE_DIR}/../../unreal/Puerts/Source/JsEnv/Private/WebSocketImpl.cpp)
    endif()
endif()

if(DEFINED PUERTS_EXTRA_SRC)
    list(APPEND PUERTS_SRC ${PUERTS_EXTRA_SRC})
endif()

if(DEFINED PUERTS_EXTRA_INC)
    include_directories(${PUERTS_EXTRA_INC})
endif()

if ( APPLE )
    if ( IOS )
        set(CMAKE_OSX_ARCHITECTURES "$(ARCHS_STANDARD)")
        add_library(puerts STATIC
           ${PUERTS_SRC} ${PUERTS_INC} ${BEACKEND_QUICKJS_SRC}
        )
		set_xcode_property (puerts IPHONEOS_DEPLOYMENT_TARGET "7.0" "all")
    else ()
        if ( NOT DEFINED FOR_SILICON )
            set(CMAKE_OSX_ARCHITECTURES x86_64)
        else ()
            set(CMAKE_OSX_ARCHITECTURES arm64)
        endif ()
        
        add_library(puerts SHARED
            ${PUERTS_SRC} ${PUERTS_INC} ${BEACKEND_QUICKJS_SRC}
        )
        target_link_options(
            puerts 
            PRIVATE "-Wl,-rpath,@loader_path/${CMAKE_OSX_ARCHITECTURES}/"
            PRIVATE "-Wl,-rpath,@loader_path/"
        )
    endif ()
elseif ( SWITCH_PLATFORM )
    add_library(puerts STATIC
        ${PUERTS_SRC} ${PUERTS_INC} ${BEACKEND_QUICKJS_SRC}
    )
else ()
    if (CMAKE_SYSTEM_NAME MATCHES "Emscripten") 
        add_library(puerts STATIC
            ${PUERTS_SRC} ${PUERTS_INC} ${BEACKEND_QUICKJS_SRC}
        )
    else ()
        add_library(puerts SHARED
            ${PUERTS_SRC} ${PUERTS_INC} ${BEACKEND_QUICKJS_SRC}
        )
   endif ()
endif ()

target_include_directories(puerts PRIVATE ${BEACKEND_QUICKJS_INC})

if ( WITH_WEBSOCKET EQUAL 3 )
    set(OPENSSL_SOURCE_DIR ${CMAKE_CURRENT_BINARY_DIR}/openssl-src) # default path by CMake
    set(OPENSSL_INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/openssl)
    set(OPENSSL_INCLUDE_DIR ${OPENSSL_INSTALL_DIR}/include)
    set(OPENSSL_BUILD_COMMAND make)
    set(OPENSSL_LIBRARY_SUFFIX a)
    set(OPENSSL_CONFIGURE_COMMAND ${OPENSSL_SOURCE_DIR}/config)
    if(WIN32)
        if(CMAKE_SIZEOF_VOID_P EQUAL 4)
            # 32位 Windows
            set(OPENSSL_PLATFORM_CONF_ARG "VC-WIN32")
        elseif(CMAKE_SIZEOF_VOID_P EQUAL 8)
            # 64位 Windows
            set(OPENSSL_PLATFORM_CONF_ARG "VC-WIN64A")
        endif()
        set(OPENSSL_BUILD_COMMAND nmake)
        set(OPENSSL_LIBRARY_SUFFIX lib)
        set(OPENSSL_CONFIGURE_COMMAND perl ${OPENSSL_SOURCE_DIR}/Configure ${OPENSSL_PLATFORM_CONF_ARG})
    endif()

    include(ExternalProject)

    ExternalProject_Add(
      OpenSSL
      SOURCE_DIR ${OPENSSL_SOURCE_DIR}
      GIT_REPOSITORY https://github.com/openssl/openssl.git
      GIT_TAG OpenSSL_1_1_1t
      #GIT_TAG openssl-3.0.5
      USES_TERMINAL_DOWNLOAD TRUE
      CONFIGURE_COMMAND
        ${OPENSSL_CONFIGURE_COMMAND}
        --prefix=${OPENSSL_INSTALL_DIR}
        --openssldir=${OPENSSL_INSTALL_DIR}
      BUILD_COMMAND ${OPENSSL_BUILD_COMMAND}
      TEST_COMMAND ""
      INSTALL_COMMAND ${OPENSSL_BUILD_COMMAND} install
      INSTALL_DIR ${OPENSSL_INSTALL_DIR}
    )

    file(MAKE_DIRECTORY ${OPENSSL_INCLUDE_DIR})

    add_library(OpenSSL::SSL STATIC IMPORTED GLOBAL)
    set_property(TARGET OpenSSL::SSL PROPERTY IMPORTED_LOCATION ${OPENSSL_INSTALL_DIR}/lib/libssl.${OPENSSL_LIBRARY_SUFFIX})
    set_property(TARGET OpenSSL::SSL PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${OPENSSL_INCLUDE_DIR})
    add_dependencies(OpenSSL::SSL OpenSSL)

    add_library(OpenSSL::Crypto STATIC IMPORTED GLOBAL)
    set_property(TARGET OpenSSL::Crypto PROPERTY IMPORTED_LOCATION ${OPENSSL_INSTALL_DIR}/lib/libcrypto.${OPENSSL_LIBRARY_SUFFIX})
    set_property(TARGET OpenSSL::Crypto PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${OPENSSL_INCLUDE_DIR})
    add_dependencies(OpenSSL::Crypto OpenSSL)

    add_dependencies(puerts OpenSSL)
    target_link_libraries(puerts OpenSSL::SSL OpenSSL::Crypto)
    list(APPEND PUERTS_COMPILE_DEFINITIONS WITH_WEBSOCKET_SSL)
endif ()

if ( WITH_WEBSOCKET EQUAL 2 )
    include(FetchContent)

    FetchContent_Declare(
        wolfssl
        GIT_REPOSITORY https://github.com/wolfSSL/wolfssl.git
        GIT_TAG        v5.7.2-stable 
    )
    set(WOLFSSL_EXAMPLES  OFF)
    set(WOLFSSL_CRYPT_TESTS OFF)
    set(BUILD_SHARED_LIBS OFF)
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    
    if ( APPLE )
        if ( IOS )
            set(WOLFSSL_SYS_CA_CERTS OFF)
        else ()
            if ( NOT DEFINED FOR_SILICON )
                set(CMAKE_SYSTEM_PROCESSOR x86_64)
            else ()
                set(CMAKE_SYSTEM_PROCESSOR arm64)
            endif ()
        endif ()
    elseif ( OHOS )
        set(WARNING_C_FLAGS "-Wall -Wextra -Wno-unused")
    endif ()

    FetchContent_MakeAvailable(wolfssl)
    
    target_compile_definitions(wolfssl PUBLIC WOLFSSL_OPENSSLEXTRA WOLFSSL_OPENSSLALL OPENSSL_EXTRA OPENSSL_ALL HAVE_EX_DATA WOLFSSL_OPENSSH)
    
    target_link_libraries(puerts wolfssl)
    
    target_include_directories(puerts PRIVATE ${wolfssl_SOURCE_DIR}/wolfssl)

    list(APPEND PUERTS_COMPILE_DEFINITIONS WITH_WEBSOCKET_SSL OPENSSL_EXTRA OPENSSL_ALL WOLFSSL_OPENSSH PUERTS_USE_WOLFSSL)
    
    if ( WIN32 AND NOT CYGWIN AND NOT ( CMAKE_SYSTEM_NAME STREQUAL "WindowsStore" ) AND NOT ANDROID AND NOT MSYS)
	    set_property(TARGET wolfssl PROPERTY
             MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    endif ()

    get_target_property(target_type puerts TYPE)
    if (target_type STREQUAL "STATIC_LIBRARY" AND NOT BUILD_SHARED_LIBS)
        add_custom_command(TARGET puerts POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "$<TARGET_FILE:wolfssl>"
            "$<TARGET_FILE_DIR:puerts>/$<TARGET_FILE_NAME:wolfssl>"
            COMMENT "copy wolfssl to output directory")
    endif()
endif ()

if ( USING_MULT_BACKEND )
    list(APPEND PUERTS_COMPILE_DEFINITIONS PUERTS_REG_NAMESPACE=puerts)
    # -------------------begin v8 backend-------------------------
    if ( USING_V8)
        set(v8backend_src 
            Src/BackendEnv.cpp
            Src/JSEngine.cpp
            Src/JSFunction.cpp
            ${PROJECT_SOURCE_DIR}/../../unreal/Puerts/Source/JsEnv/Private/V8InspectorImpl.cpp
            Src/PluginImpl.cpp
            ${PUERTS_BACKEND_SRC}
        )
        if (NOT WITHOUT_IL2CPP_OPTIMIZATION)
            list(APPEND v8backend_src
               Src/CppObjectMapper.cpp
               Src/DataTransfer.cpp
               Src/PesapiAddonLoad.cpp
               Src/PesapiV8Impl.cpp
            )
        endif()
        add_library(v8backend STATIC ${v8backend_src})
        target_include_directories (v8backend PRIVATE "${BACKEND_ROOT}/Inc/v8")
        target_compile_definitions (v8backend PRIVATE PUERTS_NAMESPACE=puerts)
        target_compile_definitions (v8backend PRIVATE MULT_BACKENDS)
        target_compile_definitions (v8backend PRIVATE V8_BACKEND)
        target_compile_definitions (v8backend PRIVATE WITH_V8)
        set_target_properties(v8backend PROPERTIES POSITION_INDEPENDENT_CODE ON)
        target_link_libraries(puerts v8backend )
        target_compile_definitions (puerts PRIVATE MULT_BACKEND_V8)
        if (NOT WITHOUT_IL2CPP_OPTIMIZATION)
            target_compile_definitions (v8backend PUBLIC WITH_IL2CPP_OPTIMIZATION)
        endif()
        if ( WITH_WEBSOCKET EQUAL 2 )
            target_include_directories(v8backend PRIVATE ${wolfssl_SOURCE_DIR}/wolfssl)
            target_link_libraries(v8backend PRIVATE wolfssl)
        endif()
    endif ()
    # -------------------end v8 backend-------------------------
    
    # -------------------begin qjs backend-------------------------
    if ( USING_QJS)
        set(qjsbackend_src 
            Src/BackendEnv.cpp
            Src/JSEngine.cpp
            Src/JSFunction.cpp
            ${PROJECT_SOURCE_DIR}/../../unreal/Puerts/Source/JsEnv/Private/V8InspectorImpl.cpp
            Src/PluginImpl.cpp
            ${PUERTS_BACKEND_SRC}
        )
        if (NOT WITHOUT_IL2CPP_OPTIMIZATION)
            list(APPEND BEACKEND_QUICKJS_INC ${PROJECT_SOURCE_DIR}/papi-quickjs/include)
            list(APPEND qjsbackend_src
               papi-quickjs/source/CppObjectMapperQuickjs.cpp
               papi-quickjs/source/PapiQuickjsImpl.cpp
            )
        endif()
        
        add_library(qjsbackend STATIC ${qjsbackend_src})
        target_include_directories (qjsbackend PRIVATE ${BEACKEND_QUICKJS_INC})
        target_compile_definitions (qjsbackend PRIVATE PUERTS_NAMESPACE=puerts_qjs)
        target_compile_definitions (qjsbackend PRIVATE MULT_BACKENDS)
        target_compile_definitions (qjsbackend PRIVATE QJS_BACKEND)
        target_compile_definitions (qjsbackend PRIVATE WITH_QUICKJS)
        target_compile_definitions (qjsbackend PRIVATE WITHOUT_INSPECTOR)
        if (USING_QJS_SUFFIX)
            target_compile_definitions (qjsbackend PRIVATE USING_QJS_SUFFIX)
            target_compile_definitions (qjsbackend PRIVATE CUSTOMV8NAMESPACE=v8_qjs)
        endif ()
        set_target_properties(qjsbackend PROPERTIES POSITION_INDEPENDENT_CODE ON)
        target_link_libraries(puerts qjsbackend )
        target_compile_definitions (puerts PRIVATE MULT_BACKEND_QJS)
        if (NOT WITHOUT_IL2CPP_OPTIMIZATION)
            target_compile_definitions (qjsbackend PUBLIC WITH_IL2CPP_OPTIMIZATION)
        endif()
        if ( WITH_WEBSOCKET EQUAL 2 )
            target_include_directories(qjsbackend PRIVATE ${wolfssl_SOURCE_DIR}/wolfssl)
            target_link_libraries(qjsbackend PRIVATE wolfssl)
        endif()
    endif ()
    # -------------------end qjs backend-------------------------
endif ()

# list(APPEND PUERTS_COMPILE_DEFINITIONS THREAD_SAFE)

if ( WIN32 AND NOT CYGWIN )
    list(APPEND PUERTS_COMPILE_DEFINITIONS BUILDING_V8_SHARED)
endif ()

if ( NOT CMAKE_BUILD_TYPE MATCHES "Release" )
    list(APPEND PUERTS_COMPILE_DEFINITIONS PUERTS_DEBUG)
endif ()

if ( MSYS OR WIN32 )
    if ( WIN32 ) 
        target_link_libraries(puerts
            winmm.lib
            dbghelp.lib
            shlwapi.lib
        )
    endif ()

    # definition
    list(APPEND PUERTS_COMPILE_DEFINITIONS PLATFORM_WINDOWS)
elseif ( OHOS )
    set(BACKEND_LIB_NAMES "-Wl,--whole-archive$<SEMICOLON>${BACKEND_LIB_NAMES}$<SEMICOLON>-Wl,--no-whole-archive")
    list(APPEND PUERTS_COMPILE_DEFINITIONS PLATFORM_OHOS)
    if ( OHOS_ARCH STREQUAL "armeabi-v7a")
        # definition
        list(APPEND PUERTS_COMPILE_DEFINITIONS PLATFORM_OHOS_ARM)

    elseif ( OHOS_ARCH STREQUAL "arm64-v8a")
        # definition
        list(APPEND PUERTS_COMPILE_DEFINITIONS PLATFORM_OHOS_ARM64)
    else ()
        # definition
        list(APPEND PUERTS_COMPILE_DEFINITIONS PLATFORM_OHOS_x64)
    endif ()

    if (WITH_SYMBOLS)
        add_custom_command(TARGET puerts POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E copy_if_different "${CMAKE_BINARY_DIR}/libpuerts.so" "${CMAKE_BINARY_DIR}/libpuerts.symbol.so~"
                COMMAND "${TOOLCHAIN_BIN_PATH}/llvm-strip" -x "${CMAKE_BINARY_DIR}/libpuerts.so"
                COMMENT "Strip debug symbols done on final binary.")
    endif()
elseif ( ANDROID )

    find_library(log-lib log)
    target_link_libraries(puerts
        ${log-lib}
    )

    set(BACKEND_LIB_NAMES "-Wl,--whole-archive$<SEMICOLON>${BACKEND_LIB_NAMES}$<SEMICOLON>-Wl,--no-whole-archive")
    if ( ANDROID_ABI STREQUAL "armeabi-v7a")
        # definition
        list(APPEND PUERTS_COMPILE_DEFINITIONS PLATFORM_ANDROID_ARM)

    elseif ( ANDROID_ABI STREQUAL "arm64-v8a")
        # link
        target_link_libraries(puerts
            ${log-lib}
        )
        
        # definition
        list(APPEND PUERTS_COMPILE_DEFINITIONS PLATFORM_ANDROID_ARM64)
    else ()

        list(APPEND PUERTS_COMPILE_DEFINITIONS PLATFORM_ANDROID_x64)
    endif ()
    
    #set_target_properties( puerts PROPERTIES LINK_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/android_version.script)
    #set (CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--gc-sections -Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/android_version.script")
    #set (CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--gc-sections")
    if (WITH_SYMBOLS)
        add_custom_command(TARGET puerts POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E copy_if_different "${CMAKE_BINARY_DIR}/libpuerts.so" "${CMAKE_BINARY_DIR}/libpuerts.symbol.so~"
                COMMAND "${ANDROID_TOOLCHAIN_PREFIX}strip" -g -S -d --strip-debug --verbose "${CMAKE_BINARY_DIR}/libpuerts.so"
                COMMENT "Strip debug symbols done on final binary.")
    else()
        target_link_options(puerts PRIVATE -s)
    endif()
elseif ( APPLE )

    if ( IOS )
        #definition
        if(PLATFORM STREQUAL "SIMULATOR64")
            list(APPEND PUERTS_COMPILE_DEFINITIONS PLATFORM_IOS_SIMULATOR)
            message(STATUS "SIMULATOR64 BUILD...")
        else ()
            list(APPEND PUERTS_COMPILE_DEFINITIONS PLATFORM_IOS)
        endif ()

    else ()
    
        #definition
        if ( DEFINED FOR_SILICON )
            list(APPEND PUERTS_COMPILE_DEFINITIONS PLATFORM_MAC_ARM64)
        endif()
        list(APPEND PUERTS_COMPILE_DEFINITIONS PLATFORM_MAC)
    endif ()

elseif (UNIX)
    # link
    target_link_libraries(puerts pthread)
    
    # definition
    list(APPEND PUERTS_COMPILE_DEFINITIONS PLATFORM_LINUX)
    if (NOT ("${JS_ENGINE}" STREQUAL "nodejs_16") AND NOT ("${JS_ENGINE}" STREQUAL "v8_10.6.194") AND NOT (CMAKE_SYSTEM_NAME MATCHES "Emscripten"))
        set(CMAKE_CXX_COMPILER "clang++")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -stdlib=libc++ -lc++abi")
    endif ()
endif ()

# ref https://github.com/Tencent/puerts/issues/1796 Author:NiceTry12138 
if (SWITCH_PLATFORM) 
    set(NINTENDO_SDK_ROOT $ENV{NINTENDO_SDK_ROOT})
    set(NS_INCLUDE "${NINTENDO_SDK_ROOT}/Include")
    set(NS_SPECIFIC_INCLUDE "${NINTENDO_SDK_ROOT}/TargetSpecificInclude/NX-Win32-v142")
    set(NS_AARCH64 "${NINTENDO_SDK_ROOT}/Compilers/NX/nx/aarch64")
    
    set(NS_INCLUDE_LIST ${NS_INCLUDE})                                      # 定义搜索路径
    list(APPEND NS_INCLUDE_LIST ${NS_SPECIFIC_INCLUDE})                     # 添加特殊文件搜索路径
    #list(APPEND NS_INCLUDE_LIST "${NS_AARCH64}/include")                    # 添加 AARCH64 头文件
    list(APPEND NS_INCLUDE_LIST "${NINTENDO_SDK_ROOT}/Include/nn/socket")   # 网络库相关
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")

    #set(NS_LIB_LIST "$${NS_AARCH64}/lib")                                   # 添加 AARCH64 静态库
   
    if(NS_US_ELF)
        #list(APPEND NS_INCLUDE_LIST "${NS_AARCH64}/include/aarch64-nintendo-nx-elf")
        #list(APPEND NS_LIB_LIST "${NS_AARCH64}/lib/aarch64-nintendo-nx-elf")
    else ()
        #list(APPEND NS_INCLUDE_LIST "${NS_AARCH64}/include/aarch64-nintendo-nx-nncfi")
        #list(APPEND NS_LIB_LIST "${NS_AARCH64}/lib/aarch64-nintendo-nx-nncfi")
    endif()
    
    add_compile_options(-fdeclspec)
    add_definitions(-DNN_SDK_BUILD_RELEASE)
    #add_definitions(-DNN_SDK_BUILD_DEBUG)

    
    include_directories(${NS_INCLUDE_LIST})
    link_directories(${NS_LIB_LIST})
endif()

# link
target_link_libraries(puerts
    ${BACKEND_LIB_NAMES}
)
list(APPEND PUERTS_COMPILE_DEFINITIONS ${BACKEND_DEFINITIONS})

target_compile_definitions (puerts PUBLIC ${PUERTS_COMPILE_DEFINITIONS})

if (NOT WITHOUT_IL2CPP_OPTIMIZATION)
    target_compile_definitions (puerts PUBLIC WITH_IL2CPP_OPTIMIZATION)
    
    if (USING_MULT_BACKEND AND USING_V8)
        target_compile_definitions (puerts PUBLIC WITH_V8)
    endif()
endif()
if (USING_MULT_BACKEND)
    target_compile_definitions (puerts PUBLIC CUSTOMV8NAMESPACE=v8_qjs)
endif()

if( USING_MULT_BACKEND )
    if ( USING_V8)
        target_compile_definitions (v8backend PUBLIC ${PUERTS_COMPILE_DEFINITIONS})
        if ( WIN32 AND NOT CYGWIN AND NOT ( CMAKE_SYSTEM_NAME STREQUAL "WindowsStore" ) AND NOT ANDROID AND NOT MSYS)
            set_property(TARGET v8backend PROPERTY
                     MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
        endif ()
    endif ()
    
    if ( USING_QJS)
        target_compile_definitions (qjsbackend PUBLIC ${PUERTS_COMPILE_DEFINITIONS})
        if ( WIN32 AND NOT CYGWIN AND NOT ( CMAKE_SYSTEM_NAME STREQUAL "WindowsStore" ) AND NOT ANDROID AND NOT MSYS)
            set_property(TARGET qjsbackend PROPERTY
                     MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
        endif ()
    endif ()
endif ()

if ( WIN32 AND NOT CYGWIN AND NOT ( CMAKE_SYSTEM_NAME STREQUAL "WindowsStore" ) AND NOT ANDROID AND NOT MSYS)
	set_property(TARGET puerts PROPERTY
             MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif ()

install(TARGETS puerts DESTINATION bin)