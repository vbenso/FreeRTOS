cmake_minimum_required(VERSION 3.15)

project( posix_demo )

add_compile_options( -D_WINDOWS_ )

add_compile_options( -Wall -Wextra -Wpedantic )

set( NO_TRACING 1 )
set( COVERAGE_TEST 0 )
add_compile_options( -DprojCOVERAGE_TEST=0 )
add_compile_options( -DprojENABLE_TRACING=0 )

if( PROFILE )
    set( CMAKE_BUILD_TYPE "Release" )
else()
    set( CMAKE_BUILD_TYPE "Debug" )
endif()

set(CMAKE_CXX_FLAGS_RELEASE "-O3")
set(CMAKE_CXX_FLAGS_DEBUG "-O0 -g3")

set( SANITIZE_ADDRESS 0 )
if( SANITIZE_ADDRESS )
    add_compile_options( -fsanitize=address -fsanitize=alignment )
endif()

set( SANITIZE_LEAK 0 )
if( SANITIZE_LEAK )
    add_compile_options( -fsanitize=leak )
endif()

set( FREERTOS_KERNEL_PATH "../../Source" )
set( FREERTOS_PLUS_TRACE_PATH "../../../FreeRTOS-Plus/Source/FreeRTOS-Plus-Trace" )

# Add the freertos_config for FreeRTOS-Kernel
add_library( freertos_config INTERFACE )

target_include_directories( freertos_config
    INTERFACE
        ./
        ./Trace_Recorder_Configuration
        ${FREERTOS_PLUS_TRACE_PATH}/include
        ${FREERTOS_PLUS_TRACE_PATH}/kernelports/FreeRTOS/include
)

# Select the heap port
set( FREERTOS_HEAP "3" CACHE STRING "" FORCE)

# Select the native compile PORT
set( FREERTOS_PORT "GCC_POSIX" CACHE STRING "" FORCE)

# Adding the FreeRTOS-Kernel subdirectory
add_subdirectory( ${FREERTOS_KERNEL_PATH} ${CMAKE_CURRENT_BINARY_DIR}/FreeRTOS-Kernel )

target_compile_options( freertos_kernel
    PRIVATE
        # Trace macro cast pointer to int to store memory management event
        $<IF:${NO_TRACING},,-Wno-pointer-to-int-cast>
)

file( GLOB FREERTOS_PLUS_TRACE_SOURCES ${FREERTOS_PLUS_TRACE_PATH}/*.c ${FREERTOS_PLUS_TRACE_PATH}/kernelports/FreeRTOS/*.c )

find_package(SDL2 REQUIRED SDL2)

add_compile_definitions($<$<BOOL:${LV_USE_DRAW_SDL}>:LV_USE_DRAW_SDL=1>)
add_compile_definitions($<$<BOOL:${LV_USE_LIBPNG}>:LV_USE_LIBPNG=1>)
add_compile_definitions($<$<BOOL:${LV_USE_LIBJPEG_TURBO}>:LV_USE_LIBJPEG_TURBO=1>)
add_compile_definitions($<$<BOOL:${LV_USE_FFMPEG}>:LV_USE_FFMPEG=1>)

set( LVGL_PATH "lvgl" )
add_subdirectory(${LVGL_PATH})
target_include_directories(lvgl PUBLIC ${PROJECT_SOURCE_DIR} ${SDL2_INCLUDE_DIRS})

set( SIMULATOR_PATH "simulator" )
file( GLOB SIMULATOR_SOURCES ${SIMULATOR_PATH}/*.c )

add_executable( posix_demo
                code_coverage_additions.c
                console.c
                main.c
                main_blinky.c
                main_full.c
                run-time-stats-utils.c
                $<$<NOT:${NO_TRACING}>:${FREERTOS_PLUS_TRACE_SOURCES}>
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/AbortDelay.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/BlockQ.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/blocktim.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/countsem.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/death.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/dynamic.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/EventGroupsDemo.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/flop.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/GenQTest.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/integer.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/IntSemTest.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/MessageBufferAMP.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/MessageBufferDemo.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/PollQ.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/QPeek.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/QueueOverwrite.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/QueueSet.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/QueueSetPolling.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/recmutex.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/semtest.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/StaticAllocation.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/StreamBufferDemo.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/StreamBufferInterrupt.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/TaskNotify.c
                ${CMAKE_CURRENT_LIST_DIR}/../Common/Minimal/TimerDemo.c
                ${SIMULATOR_SOURCES}
              )

target_include_directories( posix_demo
    PRIVATE
        ${CMAKE_CURRENT_LIST_DIR}
        ${CMAKE_CURRENT_LIST_DIR}/../Common/include
        ${FREERTOS_PLUS_TRACE_PATH}/Include
        ${FREERTOS_PLUS_TRACE_PATH}/streamports/File/include
        ${FREERTOS_PLUS_TRACE_PATH}/streamports/File/config
        ${SIMULATOR_PATH}/include
)

target_compile_definitions( posix_demo
    PRIVATE
        $<IF:$<STREQUAL:${USER_DEMO},BLINKY_DEMO>,USER_DEMO=0,>
        $<IF:$<STREQUAL:${USER_DEMO},FULL_DEMO>,USER_DEMO=1,>
)

target_link_libraries( posix_demo freertos_kernel freertos_config lvgl lvgl::examples lvgl::demos lvgl::thorvg ${SDL2_LIBRARIES} m pthread)

if(LV_USE_DRAW_SDL)
    set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")
    # Need to install libsdl2-image-dev
    # `sudo apt install libsdl2-image-dev`
    # `brew install sdl2_image`
    find_package(SDL2_image REQUIRED)
    target_include_directories(lvgl PUBLIC ${SDL2_IMAGE_INCLUDE_DIRS})
    target_link_libraries(posix_demo ${SDL2_IMAGE_LIBRARIES})
endif(LV_USE_DRAW_SDL)

if(LV_USE_LIBPNG)
    find_package(PNG REQUIRED)
    target_include_directories(lvgl PUBLIC ${PNG_INCLUDE_DIR})
    target_link_libraries(posix_demo ${PNG_LIBRARY})
endif(LV_USE_LIBPNG)

if(LV_USE_LIBJPEG_TURBO)
    # Need to install libjpeg-turbo8-dev
    # `sudo apt install libjpeg-turbo8-dev`
    # `brew install libjpeg-turbo`
    find_package(JPEG REQUIRED)
    target_include_directories(lvgl PUBLIC ${JPEG_INCLUDE_DIRS})
    target_link_libraries(posix_demo ${JPEG_LIBRARIES})
endif(LV_USE_LIBJPEG_TURBO)

if(LV_USE_FFMPEG)
    target_link_libraries(posix_demo avformat avcodec avutil swscale z)
endif(LV_USE_FFMPEG)

if(LV_USE_FREETYPE)
    find_package(Freetype REQUIRED)
    target_link_libraries(main ${FREETYPE_LIBRARIES})
    target_include_directories(posix_demo PUBLIC ${FREETYPE_INCLUDE_DIRS})
endif(LV_USE_FREETYPE)