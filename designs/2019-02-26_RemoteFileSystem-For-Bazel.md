---
created: 2019-02-26
last updated: 2019-02-26
status: Draft
reviewers: 
  - ulfjack
title: Remote File System for Bazel
authors:
  - royrah
discussion thread: 
---

# Overview
Bazel as a build tool natively expects files of the build action requested to be present on Disk. For a monorepo of a significant size, dumping out the entire repository for every single build , even with bazel's caching is going to be a significant overhead.

Statistically, a repo which is ~2 GBs of source needs around 1-2 mins to dump out to disk. This proposal intends to address this particular issue by extending bazel to support a Remote File system , which can talk to any repository which supports Content addressable storage of source files to be able to provide on demand , requested files/contents hashes/file stats via HTTP/GRPC endpoints.

# Existing Approaches
Currently , the bazel server does not support OOTB any way to get around having files on the local disk where bazel server is running. Even with the remote [ BuildFarm ] setup , the local files are uploaded to the remote server for being actioned on.

# Requirements
The extended RemoteFileSystem needs to support at minimum the following requests which bazel makes to build it's action graph by reading files :

### Directory listing of source Globs
Every build target has a glob of input files which bazel resolves during the loading phase to build an action graph of a target. The Remote endpoint needs to resolve the requested glob from the underlying filesystem. This does not get the actual content , but only the list of files 
in the directory.

### Content Hashes of glob paths and dependency paths
Bazel is fast because it caches the content hashes of every path it has built previously and just provides them when an Action requesting the particular path is requested. The remote endpoint needs to be able to provide a content hash[ Digest ] for a requested path. The digest is the identifier for a any file in the CAS when executing in remote mode.

### Actual Files for a requested path
When an action is actually executed the worker needs the actual file. The remote endpoint needs to resolve the path and send back the byte content of the file.

### Resolve a particular path as onDisk/remote
There are cases when a particular implementation will be a hybrid of a remote and local file system, such as cases where third part dependencies are actually present on disk and source files are not. The remote filesystem needs to delegate requests for file paths to the appropriate handling filesystem.

This design currently needs the BUILD, WORKSPACE and .bzl files to be present on disk to be actually able to resolve the targets  to be built. A true-remote FS should have the ability to fetch these on demand from the remote endpoints and then proceed.

Additionally, there should be a set of options to customize the remote behaviour based on the underlying store [ FUSE ] as well as the protocol to be supported [ HTTP/GRPC ].

# Suggested Implementation
We propose to have RemoteFileSystem which can talk to a HTTP/GRPC file server to satisfy the above listed requirements. The FileSystem at minimum needs to extend AbstractFileSystemWithCustomStat to intercept all Filesystem requests, and then appropiately delegate based on a list of either custom prefixes/any other delegation implementation.

The BazelFileSystemModule needs to be extended to support the new Filesystem, and a set of Options need to be provided to provide the endpoint [ hosts / port ] as well as authentication options.

RemoteFileSystem options should at the minimum be the following :

### Startup Options

1. **--enable_remote_filesystem** : flag to enable the intercation with remote file system, off by default
2. **--remote_filesystem_type** : one of HTTP/GRPC...and FUSE ?
3. **--remote_filesystem_host** : host:port for the backend server for the filesystem requests in case of HTTP/GRPC
4. **--remote_filesystem_prefixes** : enables the delegating file system to decide what paths should be delegated to the RemoteFileSystem
5. **--remote_filesystem_version** : optional parameter to point to version of the underlying repository to be used , defaults to master

```Java
public static DelegatingFileSystem make(
        FileSystem writeDelegate, FileSystem remoteDelegate, PathFragment ... customPrefixes ) throws DigestHashFunction.DefaultHashFunctionNotSetException {
   //Basic checks to ensure read and write delegates are compliant, then call a private constructor

 return new DelegatingFileSystem(writeDelegate, readDelegate, customPrefixes);
}
private FileSystem getDelegate(Path path) {
    //based of custom prefixes/other implementation detail figure out the correct handler for operations
}
```

An example :
```Java
@Override
protected long getFileSize(Path path, boolean followSymlinks) throws IOException {
    return getDelegate(path).getFileSize(path, followSymlinks);
}
```
The RemoteFileSystem consists of the client which makes the HTTP/GRPC calls to the backend server .Minimally looks as follows :

```Java
private RemoteFileSystemClient remoteClient;

public RemoteFileSystem() throws DigestHashFunction.DefaultHashFunctionNotSetException {}

public RemoteFileSystem(DigestHashFunction hashFunction) {
    super(hashFunction);
}

public RemoteFileSystem(RemoteFileSystemOptions remoteOptions, PathFragment workspaceDir, BlazeServerStartupOptions startupOptions)
        throws DigestHashFunction.DefaultHashFunctionNotSetException {
 remoteClient = RemoteFileSystemClientFactory.create(remoteOptions, workspaceDir, startupOptions);
}
```

Any server attempting to support a remote file system implementation , will need to support the following requests :

Operation | Endpoint | Returns
----------|----------|--------
Read Directory Entries |/bazel/dirents/:version/* | A List of directory entries
Git File Input Stream | /bazel/data/:version/* | A stream of file bytes
Get file Stat [ size, time_modified ]	| /bazel/filestat/:version/* | pair of size and time modified
Get fast Digest	| /bazel/digest/:version/* | byte content of the file hash

*Delegate provider implementation can be custom to all implementers , but at the very least they need to be a list of Path prefixes which can be resolved against a path to figure out the delegate.

Alternative GRPC implementation proto is as follows. : remote_file_system.proto
```Protobuf
syntax = "proto3";

package remote_file_system;

option java_package = "com.google.devtools.build.lib.vfs.rfs";
option java_outer_classname = "RemoteFileSystemProtos";
option java_multiple_files = true;

import "google/api/annotations.proto";
import "build/bazel/remote/execution/v2/remote_execution.proto";

//RPC endpoints for interfacing with a remotefile system server which is responsible
//for resolving all file system interaction requested by bazel when running with remote
//file system enabled.
service RemoteFileSystem {

    rpc FetchData(FileSystemRequest) returns (FileSystemResponse) {
        option (google.api.http) = { get: "/remotefs/{type}/{repo_version}/{path}" };
    }
}

//a request to the remote file system is almost exclusively always comprised of 3 components
//the type of the request,  a path to resolve , and the version of the repository against which to find the path
//The type parameter helps the server respond appropriately to different queries
message FileSystemRequest {
    //the relative path for this request, note that this relative to the WORKSPACE
    string path = 1;

    //version/commit of the repository against which the above path needs to be resolve.
    //Remote filesystems will be usually backed by a repository like Git, which rely on a version
    //to fetch a file/content from.
    //default value for this be master , and can be overridden with an empty string to if the underlying
    //FileSystem of the remote server doesn't rely on versions.
    string repo_version = 2;

    enum RequestType {
        //request for hash of the file denoted by the path
        DIGEST = 0;

        //request for file state returning {@FileStat}
        FILE_STAT = 1;

        //request for listing directory entries at a particular path
        DIRENTS = 2;

        //request for the actual byte content of a file
        FILE_CONTENT = 3;
    }

    RequestType type = 3;
}

message FileSystemResponse {
    oneof response {
        build.bazel.remote.execution.v2.Digest digest = 1;
        DirectoryEntries entries = 2;
        FileStat filestat = 3;
        OutputFile filecontent = 4;
    }
}

//collection containing the names of all entities within the directory denoted by the path
message DirectoryEntries {
    //the relative path of the directory whose entries are being requested
    string path = 1;

    //the list of entries in this particulr directory path
    repeated string entries = 2;

}

//a 'FileStat' represents the status entries for a file : File status: mode, mtime, size, etc.
//the remote implementer of the FileSystem service needs to provide the fields to accurately define
//the state of a file in the FileSystem
message FileStat {
    //The relative path of the file for whom the state is requested, including
    //the filename. The path separator is a forward slash '/'
    string path = 1;

    //The last modified time of the file requested. analogous to mtime on unix
    //like file sysytems
    int64 last_modified_time = 2;

    //The size of the file/directory
    int64 size_bytes = 3;

    //The last updated time of the file requested. analogous to utime on unix
    //like file sysytems
    int64 last_change_time = 4;

    //Is the requested path a regular file ?
    bool isFile = 5;
}

message OutputFile {
    // The full path of the file relative to the input root, including the
    // filename. The path separator is a forward slash `/`.
    string path = 1;

    // The digest of the file's content.
    build.bazel.remote.execution.v2.Digest digest = 2;

    // The raw content of the file.
    //
    // This field may be used by the server to provide the content of a file
    // inline in an
    // [ActionResult][google.devtools.remoteexecution.v1test.ActionResult] and
    // avoid requiring that the client make a separate call to
    // [ContentAddressableStorage.GetBlob] to retrieve it.
    //
    // The client SHOULD NOT assume that it will get raw content with any request,
    // and always be prepared to retrieve it via `digest`.
    bytes content = 3;
}

```

# Caveats
1. This implementation currently, causes a server restart on change of the source tree ( branch in git terminology ). 
2. There is presently a need to have BUILD/WORKSPACE files locally dumped to disk before kicking off the build, Need to figure out if atleast the BUILD files can be remoted too.

