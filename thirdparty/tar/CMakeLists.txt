PROJECT(tar)
cmake_minimum_required(VERSION 2.8.3)

SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_LIST_DIR}/../cmake_modules")
include("koreader_thirdparty_common")
#include("koreader_thirdparty_git")
enable_language(C)

assert_var_defined(CC)
assert_var_defined(CFLAGS)
assert_var_defined(LDFLAGS)
assert_var_defined(LIBS)

ep_get_source_dir(SOURCE_DIR)
ep_get_binary_dir(BINARY_DIR)

# The wordsplit changes introduced in 1.31 require glob.h, which is not present on Androdi before ABI 28 (c.f., #787)
if(DEFINED ENV{ANDROID})
    set(PATCH_CMD1 sh -c "patch -N -p1 < ${CMAKE_CURRENT_SOURCE_DIR}/tar-1.31-revert-wordsplit-for-android-glob.patch || true")
else()
    set(PATCH_CMD1 sh -c "true")
endif()
#set(PATCH_CMD2 sh -c "touch ABOUT-NLS po/Makevars.template")
#set(PATCH_CMD3 sh -c "env AUTOPOINT=true ./bootstrap --skip-po")

set(CFG_ENV_VAR "CC=\"${CC} -DHAVE_MKFIFO=1\" CFLAGS=\"${CFLAGS}\" LDFLAGS=\"${LDFLAGS}\"")
set(CFG_OPTS "-q LIBS=\"${LIBS}\" --host=\"${CHOST}\"")

if(${DISABLE_LARGEFILE})
    set(CFG_OPTS "${CFG_OPTS} --disable-largefile")
endif(${DISABLE_LARGEFILE})
set(CFG_CMD "${CFG_ENV_VAR} ${SOURCE_DIR}/configure --disable-nls --disable-gcc-warnings --disable-acl --without-xattrs --without-posix-acls ${CFG_OPTS}")

if(${DISABLE_FORTIFY})
    # Forcibly disable FORTIFY on legacy devices...
    set(CFG_CMD "${CFG_CMD} && ${ISED} 's/# define _FORTIFY_SOURCE 2/#undef _FORTIFY_SOURCE/' config.h")
endif(${DISABLE_FORTIFY})
set(CFG_CMD sh -c "${CFG_CMD}")

# We build in-tree...
set(INSTALL_CMD1 ${CMAKE_COMMAND} -E rename ${BINARY_DIR}/src/tar ${BINARY_DIR}/tar)
#set(INSTALL_CMD1 ${CMAKE_COMMAND} -E make_directory ${BINARY_DIR})
#set(INSTALL_CMD2 ${CMAKE_COMMAND} -E rename ${SOURCE_DIR}/src/tar ${BINARY_DIR}/tar)

#ko_write_gitclone_script(
#    GIT_CLONE_SCRIPT_FILENAME
#    https://git.savannah.gnu.org/git/tar.git
#    release_1_31
#    ${SOURCE_DIR}
#)

include(ExternalProject)
set(TAR_VER "1.31")
ExternalProject_Add(
    ${PROJECT_NAME}
    URL http://ftp.gnu.org/gnu/tar/tar-${TAR_VER}.tar.gz
    URL_MD5 f2d3b2c7130390f2fd204527c4c22aa5
    DOWNLOAD_DIR ${KO_DOWNLOAD_DIR}
    PATCH_COMMAND COMMAND ${PATCH_CMD1}
    #DOWNLOAD_COMMAND ${CMAKE_COMMAND} -P ${GIT_CLONE_SCRIPT_FILENAME}
    #BUILD_IN_SOURCE 1
    #PATCH_COMMAND COMMAND ${PATCH_CMD1} COMMAND ${PATCH_CMD2} COMMAND ${PATCH_CMD3}
    CONFIGURE_COMMAND ${CFG_CMD}
    BUILD_COMMAND $(MAKE) -j${PARALLEL_JOBS} --silent
    # The buildsystem outputs binaries in a subdirectory...
    INSTALL_COMMAND COMMAND ${INSTALL_CMD1}
    #INSTALL_COMMAND COMMAND ${INSTALL_CMD1} COMMAND ${INSTALL_CMD2}
)
