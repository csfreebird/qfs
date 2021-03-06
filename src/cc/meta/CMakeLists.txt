#
# $Id$
#
# Created 2006
# Author: Sriram Rao (Kosmix Corp)
#
# Copyright 2006 Kosmix Corp.
#
# This file is part of Kosmos File System (KFS).
#
# Licensed under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License. You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
# implied. See the License for the specific language governing
# permissions and limitations under the License.
#
#

#
# For the library take everything except the *_main.cc files
#
set (lib_srcs
    AuditLog.cc
    Checkpoint.cc
    ChunkServer.cc
    ChildProcessTracker.cc
    ClientSM.cc
    DiskEntry.cc
    kfsops.cc
    kfstree.cc
    LayoutManager.cc
    meta.cc
    MetaRequest.cc
    NetDispatch.cc
    Replay.cc
    Restorer.cc
    util.cc
    AuthContext.cc
    UserAndGroup.cc
    MetaRequestHandler.cc
    IdempotentRequestTracker.cc
    LogWriter.cc
    LogReceiver.cc
    LogTransmitter.cc
    MetaDataStore.cc
    MetaDataSync.cc
    MetaVrSM.cc
    globals.cc
)

add_library (kfsMeta STATIC ${lib_srcs})
set_target_properties (kfsMeta PROPERTIES OUTPUT_NAME "qfs_meta")
set_target_properties (kfsMeta PROPERTIES CLEAN_DIRECT_OUTPUT 1)
target_link_libraries(kfsMeta
    kfsClient
)

if (NOT USE_STATIC_LIB_LINKAGE)
    add_library (kfsMeta-shared SHARED ${lib_srcs} layoutmanager_instance.cc)
    set_target_properties (kfsMeta-shared PROPERTIES OUTPUT_NAME "qfs_meta")
    set_target_properties (kfsMeta-shared PROPERTIES CLEAN_DIRECT_OUTPUT 1)
    target_link_libraries(kfsMeta-shared
        kfsClient-shared
    )
    install (TARGETS kfsMeta-shared
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib)
endif (NOT USE_STATIC_LIB_LINKAGE)

set (exe_files metaserver logcompactor filelister qfsfsck qfsobjstorefsck)
foreach (exe_file ${exe_files})
    if (USE_STATIC_LIB_LINKAGE)
        add_executable (${exe_file}
            ${exe_file}_main.cc
            layoutmanager_instance.cc
        )
        target_link_libraries (${exe_file}
            kfsMeta
        )
        add_dependencies (${exe_file}
            kfsMeta
        )
    else (USE_STATIC_LIB_LINKAGE)
        add_executable (${exe_file}
            ${exe_file}_main.cc
        )
        target_link_libraries (${exe_file}
            kfsMeta-shared
        )
        add_dependencies (${exe_file}
            kfsCommon-shared
        )
    endif (USE_STATIC_LIB_LINKAGE)
endforeach (exe_file)

if (USE_STATIC_LIB_LINKAGE)
    target_link_libraries (qfsfsck tools)
    add_dependencies      (qfsfsck tools)
else (USE_STATIC_LIB_LINKAGE)
    target_link_libraries (qfsfsck tools-shared)
    add_dependencies      (qfsfsck tools-shared)
endif (USE_STATIC_LIB_LINKAGE)

if (CMAKE_SYSTEM_NAME STREQUAL "SunOS")
   # mtmalloc seemed to worsen metaserver startup time; it took
   # 4 mins for fsck to load checkpoint from WORM, where as 30 for metaserver.
   # So, switch to umem
   target_link_libraries(kfsMeta umem)
endif (CMAKE_SYSTEM_NAME STREQUAL "SunOS")

if (CYGWIN)
    # Workaround for "too many sections" asm failure by turning on optimization
    # for compiler to inline for all build types.
    set_source_files_properties(
        MetaRequestHandler.cc PROPERTIES COMPILE_FLAGS -O2)
endif (CYGWIN)

#
# Install them
#
install (TARGETS ${exe_files} kfsMeta
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib/static)

