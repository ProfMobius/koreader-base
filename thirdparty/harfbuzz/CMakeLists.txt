PROJECT(harfbuzz)
cmake_minimum_required(VERSION 2.8.3)

SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_LIST_DIR}/../cmake_modules")
include("koreader_thirdparty_common")
include("koreader_thirdparty_git")

enable_language(C CXX)

assert_var_defined(CC)
assert_var_defined(CFLAGS)
assert_var_defined(LDFLAGS)
assert_var_defined(FREETYPE_DIR)

ep_get_source_dir(SOURCE_DIR)
ep_get_binary_dir(BINARY_DIR)

# HB is modern C++, old compilers beware!
if(DEFINED ENV{POCKETBOOK})
    # c++14 is a no-go
    set(CXX_STDFLAG "-std=c++11")
else()
    # Fire and forget! Should generally be sane on GCC >= 6
    set(CXX_STDFLAG "")
endif(DEFINED ENV{POCKETBOOK})

set(CFG_ENV_VAR "CC=\"${CC}\" CXX=\"${CXX}\" CFLAGS=\"${CFLAGS}\" CXXFLAGS=\"${CXXFLAGS} ${CXX_STDFLAG}\" LDFLAGS=\"${LDFLAGS}\"")
set(CFG_ENV_VAR "${CFG_ENV_VAR} FREETYPE_CFLAGS=\"-I${FREETYPE_DIR}/include/freetype2\"")
set(CFG_ENV_VAR "${CFG_ENV_VAR} FREETYPE_LIBS=\"-L${FREETYPE_DIR} -lfreetype\"")
set(CFG_OPTS "--prefix=\"${BINARY_DIR}\" --enable-shared --disable-static --with-freetype --with-ucdn --without-glib --without-gobject --without-cairo --without-fontconfig --without-icu --without-graphite2 --without-uniscribe --without-directwrite --without-coretext --host=\"${CHOST}\"")
set(CFG_CMD sh -c "${CFG_ENV_VAR} ${SOURCE_DIR}/configure ${CFG_OPTS}")

if($ENV{ANDROID})
    set(CFG_CMD "${CFG_CMD} && ${ISED} 's|version_type=none|version_type=linux|' libtool")
    set(CFG_CMD "${CFG_CMD} && ${ISED} 's|need_lib_prefix=no|need_lib_prefix=yes|' libtool")
    set(CFG_CMD "${CFG_CMD} && ${ISED} 's|need_version=no|need_version=yes|' libtool")
    set(CFG_CMD "${CFG_CMD} && ${ISED} 's|library_names_spec=.*|library_names_spec=\"\\\\\$libname\\\\\$release\\\\\$shared_ext\\\\\$versuffix \\\\\$libname\\\\\$release\\\\\$shared_ext\\\\\$major \\\\\$libname\\\\\$shared_ext\"|' libtool")
    set(CFG_CMD "${CFG_CMD} && ${ISED} 's|soname_spec=.*|soname_spec=\"\\\\\$libname\\\\\$release\\\\\$shared_ext\\\\\$major\"|' libtool")
endif()

ko_write_gitclone_script(
    GIT_CLONE_SCRIPT_FILENAME
    https://github.com/harfbuzz/harfbuzz.git
    2.3.0
    ${SOURCE_DIR}
)

include(ExternalProject)
ExternalProject_Add(
    ${PROJECT_NAME}
    DOWNLOAD_COMMAND ${CMAKE_COMMAND} -P ${GIT_CLONE_SCRIPT_FILENAME}
    BUILD_IN_SOURCE 1
    PATCH_COMMAND NOCONFIGURE=1 ./autogen.sh
    CONFIGURE_COMMAND ${CFG_CMD}
    BUILD_COMMAND $(MAKE) -j${PARALLEL_JOBS} --silent
    INSTALL_COMMAND $(MAKE) -j${PARALLEL_JOBS} install
)
