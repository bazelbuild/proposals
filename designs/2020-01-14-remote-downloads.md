---
created: 2019-04-18
last updated: 2020-01-20
status: Approved
reviewers:
  - aehlig
  - buchgr
title: Remote Downloads
authors:
  - EricBurnett
  - jmillikin
  - sstriker
discussion threads:
  - https://groups.google.com/forum/#!topic/remote-execution-apis/Gyzvz7WkQPE
  - https://groups.google.com/forum/#!topic/remote-execution-apis/kTcjXrijWpU
---

# Abstract

The [Remote Execution APIs](https://github.com/bazelbuild/remote-apis) are a
set of common gRPC APIs covering remote execution and content-addressed storage,
primarily intended for use by build tools. Currently these APIs require all
execution inputs to be uploaded by the client, which has undesirable performance
and security characteristics in some cases. This proposal expands the remote
execution APIs to support downloading inputs from external sources, allowing the
build tool to offload expensive or privileged network requests.

# Background

When a build tool is calculating which actions to execute as part of a build, it
may need to download input files from external sources -- common external
dependencies include compiler toolchains, third-party libraries, test data
snapshots, and so on. There are three important reasons why the build tool
should not be the thing doing the actual downloading:

* The external dependencies might be hosted on a third-party service, and build
  reliability will be degraded if that service has an outage or temporarily
  throttles downloads. Downloading via an intermediate service allows the
  injection of a caching proxy layer.

* Some organizations would like to enforce policies on use of third-party code,
  and a build tool that directly downloads external dependencies will bypass
  technical controls that implement these policies.

* The build tool might have a low-quality network connection, such that
  downloading and re-uploading dependencies is a significant fraction of the
  build time. If the remote execution service is hosted in a datacenter, then
  it can fetch dependencies much faster than the client can.

This proposal is derived from two previous documents:

* [Remote Repository Cache](https://docs.google.com/document/d/1aagKnM8kRmMvLqSXasShsI-NYOBZYtgNrbYWLPwBoBg/view)
  ([discussion](https://groups.google.com/forum/#!topic/remote-execution-apis/Gyzvz7WkQPE))
  is a Bazel-specific gRPC API for downloading external dependencies. It lays
  out the basic design goals and implementation strategy for a remote downloader.

* [Remote 'Fetch' API](https://docs.google.com/document/d/10ari9WtTTSv9bqB_UU-oe2gBtaAA7HyQgkpP-RFP80c/view)
  ([discussion](https://groups.google.com/forum/#!topic/remote-execution-apis/kTcjXrijWpU))
  generalized the above API to be non-Bazel-specific so that it could be accpted
  into the Remote Execution APIs, and expanded it to cover use cases of
  the BuildStream project.

# Protocol

The new API is built on top of the existing `ContentAddressableStorage` service,
but has its own gRPC service names so that traffic can be routed independently.

Resources are identified by URI(s) and expected qualifiers. Qualifiers are
arbitrary key-value pairs, with certain "well-known" keys documented in the
[**Qualifiers**](#qualifiers)] section of this document.

A minimal compliant Fetcher implementation might support only pushed content and
return `NOT_FOUND` for any resource that was not pushed first. Alternatively,
a compliant implementation may choose to not support Push and only return
resources that can be retrieved from their origin.

## Resource Identifiers

Clients **MUST** supply one or more URIs in requests; servers **MAY** match any
of the supplied URIs. All URIs in a request are expected to identify the same
content, and thus share the same Qualifiers.

Support for specific URI schemes is implementation-dependent. Servers may choose
to support only URLs, only URNs, only other URIs for which the server and client
agree upon semantics thereof, or any mixture of the above.

Multiple URIs may be used to refer to the same content. For example, the same
tarball may exist at multiple mirrors and thus be retrievable from multiple
URLs. When URLs are used, these should refer to actual content as Fetch API
service implementations may choose to fetch the content directly from the
origin. For example, the HEAD of a git repository's active branch can be
referred to as:

```
uri: "https://github.com/bazelbuild/remote-apis.git"
```

URNs may be used as opaque content identifiers, for instance by using the `uuid`
namespace: `urn:uuid:f81d4fae-7dec-11d0-a765-00a0c91e6bf6`. This is most
applicable to content that is injected into the CAS via the Push API, where the
URN serves as an agreed-upon key, but carries no other inherent meaning.

## The 'Fetcher' API

The `Fetcher` gRPC service injects entries into the content-addressable storage
by downloading data from external sources. If successful, the server returns the
`Digest` of the fetched content so that the client can retrieve it from the CAS.

```proto
service Fetcher {
  rpc FetchBlob(FetchBlobRequest) returns (FetchBlobResponse);
  rpc FetchDirectory(FetchDirectoryRequest) returns (FetchDirectoryResponse);
}
```

If a request is successful then servers **MUST** ensure the referenced assets
are present in the CAS at the time of response and (if supported) for a
reasonable period of time thereafter.

The client **MAY** request a resource be re-fetched from its origin by
setting the request field `oldest_content_accepted` to a timestamp more recent
than the previous attempt.

Servers **MAY** cache fetched content and reuse it for subsequent requests,
subject to `oldest_content_accepted`. Servers **MAY** fetch content that they
do not already have cached, for any URIs they support.

Servers **MUST** reject requests containing unknown Qualifiers. Servers **MAY**
validate the fetched content against the requested qualifiers. Servers **MAY**
have different validation policies for assets obtained from different sources,
for example by special-casing assets injected into the CAS via the `Pusher` API.

Servers **MAY** transform assets as part of the fetch. For example a tarball
fetched by `FetchDirectory` might be unpacked, or a Git repository fetched by
`FetchBlob` might be passed through `git-archive`.

Errors encountered processing the request will be returned as gRPC errors with
the following codes:

* `INVALID_ARGUMENT`: One or more arguments to the RPC were invalid.

* `RESOURCE_EXHAUSTED`: There is insufficient quota of some resource to
  perform the requested operation. The client may retry after a delay.

* `UNAVAILABLE`: Due to a transient condition the operation could not be
  completed. The client should retry.

* `INTERNAL`: An internal error occurred while performing the operation. The
  client should retry.

* `DEADLINE_EXCEEDED`: The fetch could not be completed within the given
  RPC deadline. The client should retry for at least as long as the value
  provided in `request.timeout`.

Errors encountered while fetching assets will be returned in the responses'
`status` field. See the relevant response types for details.

### FetchBlobRequest

```proto
message FetchBlobRequest {
  string instance_name = 1;

  // The timeout for the underlying fetch, if content needs to be retrieved from
  // origin.
  //
  // If unset, the server *MAY* apply an implementation-defined timeout.
  //
  // If set, and the user-provided timeout exceeds the RPC deadline, the server
  // *SHOULD* keep the fetch going after the RPC completes, to be made
  // available for future Fetch calls. The server may also enforce (via clamping
  // and/or an INVALID_ARGUMENT error) implementation-defined minimum and
  // maximum timeout values.
  //
  // If this timeout is exceeded on an attempt to retrieve content from origin
  // the client will receive DEADLINE_EXCEEDED in [FetchBlobResponse.status].
  google.protobuf.Duration timeout = 2;

  // The oldest content the client is willing to accept, as measured from the
  // time it was Push'd or when the underlying retrieval from origin was started.
  // Upon retries of Fetch requests that cannot be completed within a single RPC,
  // clients *SHOULD* provide the same value for subsequent requests as the
  // original, to simplify combining the request with the previous attempt.
  //
  // If unset, the client *SHOULD* accept content of any age.
  google.protobuf.Timestamp oldest_content_accepted = 3;

  // The URI(s) of the content to fetch. These may be resources that the server can
  // directly fetch from origin, in which case multiple URIs *SHOULD* represent
  // the same content available at different locations (such as an origin and
  // secondary mirrors). These may also be URIs for content known to the server
  // through other mechanisms, e.g. pushed via the [Pusher] API.
  repeated string uris = 4;

  // Qualifiers sub-specifying the content to fetch - see comments on [Qualifier].
  // The same qualifiers apply to all URIs.
  repeated Qualifier qualifiers = 5;
}
```

### FetchBlobResponse

```proto
message FetchBlobResponse {
  // If the status has a code other than `OK`, it indicates that the operation was
  // unable to be completed for reasons outside the servers' control.
  // The possible fetch errors include:
  // * `DEADLINE_EXCEEDED`: The operation could not be completed within the
  //   specified timeout.
  // * `NOT_FOUND`: The requested asset was not found at the specified location.
  // * `PERMISSION_DENIED`: The request was rejected by a remote server, or
  //   requested an asset from a disallowed origin.
  // * `ABORTED`: The operation could not be completed, typically due to a failed
       consistency check.
  google.rpc.Status status = 1;

  // The uri from the request that resulted in a successful retrieval, or from
  // which the error indicated in `status` was obtained.
  string uri = 2;

  // Any qualifiers known to the server and of interest to clients.
  repeated Qualifier qualifiers = 3;

  // A minimum timestamp the content is expected to be available through.
  // Servers *MAY* omit this field, if not known with confidence.
  google.protobuf.Timestamp expires_at = 4;

  // The result of the fetch, if the status had code `OK`.
  // The digest of the file's contents, available for download through the CAS.
  Digest blob_digest = 5;
}
```

### FetchDirectoryRequest

```proto
message FetchDirectoryRequest {
  string instance_name = 1;

  // The timeout for the underlying fetch, if content needs to be retrieved from
  // origin. This value is allowed to exceed the RPC deadline, in which case the
  // server *SHOULD* keep the fetch going after the RPC completes, to be made
  // available for future Fetch calls.
  // If this timeout is exceeded on an attempt to retrieve content from origin
  // the client will receive DEADLINE_EXCEEDED in [FetchResponse.status].
  google.protobuf.Duration timeout = 2;

  // The oldest content the client is willing to accept, as measured from the
  // time it was Push'd or when the underlying retrieval from origin was started.
  // Upon retries of Fetch requests that cannot be completed within a single RPC,
  // clients *SHOULD* provide the same value for subsequent requests as the
  // original, to simplify combining the request with the previous attempt.
  google.protobuf.Timestamp oldest_content_accepted = 3;

  // The URI(s) of the content to fetch. These may be resources that the server can
  // directly fetch from origin, in which case multiple URIs *SHOULD* represent
  // the same content available at different locations (such as an origin and
  // secondary mirrors). These may also be URIs for content known to the server
  // through other mechanisms, e.g. pushed via the [Pusher] API.
  repeated string uris = 4;

  // Qualifiers sub-specifying the content to fetch - see comments on [Qualifier].
  // The same qualifiers apply to all URIs.
  repeated Qualifier qualifiers = 5;
}
```

### FetchDirectoryResponse

```proto
message FetchDirectoryResponse {
  // If the status has a code other than `OK`, it indicates that the operation was
  // unable to be completed for reasons outside the servers' control.
  // The possible fetch errors include:
  // * `DEADLINE_EXCEEDED`: The operation could not be completed within the
  //    specified timeout.
  // * `NOT_FOUND`: The requested asset was not found at the specified location.
  // * `PERMISSION_DENIED`: The request was rejected by a remote server, or
  //     requested an asset from a disallowed origin.
  // * `ABORTED`: The operation could not be completed, typically due to a failed
         consistency check.
  google.rpc.Status status = 1;

  // The uri from the request that resulted in a successful retrieval, or from
  // which the error indicated in `status` was obtained.
  string uri = 2;

  // Any qualifiers known to the server and of interest to clients.
  repeated Qualifier qualifiers = 3;

  // A minimum timestamp the content is expected to be available through.
  // Servers *MAY* omit this field, if not known with confidence.
  google.protobuf.Timestamp expires_at = 4;

  // The result of the fetch, if the status had code `OK`.
  // the root digest of a directory tree, suitable for fetching via
  // [ContentAddressableStorage.GetTree].
  Digest root_directory_digest = 5;
}
```

## The 'Pusher' API

The `Pusher` service is complementary to the Fetcher, and allows for
associating contents of URIs to be returned in future Fetcher API calls.

These APIs associate the identifying information of a resource, as indicated
by URI and optionally Qualifiers, with content available in the CAS. For
example, a Git repository URL and a commit ID can be associated with a tree
digest.

Servers **SHOULD** only allow trusted clients to associate content, and **MAY**
only allow certain URIs to be pushed.

Clients **MUST** ensure associated content is available in CAS prior to pushing.
Clients **MUST** ensure the Qualifiers listed correctly match the contents, and
Servers **MAY** trust these values without validation.

Fetch servers **MAY** require exact match of all qualifiers when returning
content previously pushed, or allow fetching content with only a subset of
the qualifiers specified on Push.

Clients can specify expiration information that the server **SHOULD**
respect. Subsequent requests can be used to alter the expiration time.

Errors will be returned as gRPC Status errors. The possible RPC errors include:

* `INVALID_ARGUMENT`: One or more arguments to the RPC were invalid.

* `RESOURCE_EXHAUSTED`: There is insufficient quota of some resource to
  perform the requested operation. The client **MAY** retry after a delay.

* `UNAVAILABLE`: Due to a transient condition the operation could not be
  completed. The client **SHOULD** retry.

* `INTERNAL`: An internal error occurred while performing the operation. The
  client **SHOULD** retry.

```proto
service Pusher {
  rpc PushBlob(PushBlobRequest) returns (PushBlobResponse)
  rpc PushDirectory(PushDirectoryRequest) returns (PushDirectoryResponse)
}
```

### PushBlobRequest

```proto
message PushBlobRequest {
  string instance_name = 1;

  // The URI(s) of the content to associate. If multiple URIs are specified, the
  // pushed content will be available to fetch by specifying any of them.
  repeated string uris = 2;

  // Qualifiers sub-specifying the content that is being pushed - see comments on
  // [Qualifier]. The same qualifiers apply to all URIs.
  repeated Qualifier qualifiers = 3;

  // A time after which this content should stop being returned via Fetch.
  // Servers *MAY* expire content early, e.g. due to storage pressure.
  google.protobuf.Timestamp expire_at = 4;

  // The blob to associate.
  Digest blob_digest = 5;

  // Referenced blobs or directories that need to not expire before expiration
  // of this association, in addition to `blob_digest` itself.
  // These fields are hints - clients *MAY* omit them, and servers *SHOULD* respect
  // them, at the risk of increased incidents of Fetch responses indirectly
  // referencing unavailable blobs.
  repeated Digest references_blobs = 6;
  repeated Digest references_directories = 7;
}
```

### PushBlobResponse

```proto
message PushBlobResponse { /* empty */ }
```

### PushDirectoryRequest

```proto
message PushDirectoryRequest {
  string instance_name = 1;

  // The URI(s) of the content to associate. If multiple URIs are specified, the
  // pushed content will be available to fetch by specifying any of them.
  repeated string uris = 2;

  // Qualifiers sub-specifying the content that is being pushed - see comments on
  // [Qualifier]. The same qualifiers apply to all URIs.
  repeated Qualifier qualifiers = 3;

  // A time after which this content should stop being returned via Fetch.
  // Servers *MAY* expire content early, e.g. due to storage pressure.
  google.protobuf.Timestamp expire_at = 4;

  // Directory to associate
  Digest root_directory_digest = 5;

  // Referenced blobs or directories that need to not expire before expiration
  // of this association, in addition to `root_directory_digest` itself.
  // These fields are hints - clients *MAY* omit them, and servers *SHOULD* respect
  // them, at the risk of increased incidents of Fetch responses indirectly
  // referencing unavailable blobs.
  repeated Digest references_blobs = 6;
  repeated Digest references_directories = 7;
}
```

### PushDirectoryResponse

```proto
message PushDirectoryResponse { /* empty */ }
```

## Qualifiers

Qualifiers are used to disambiguate, sub-select, or validate content that shares
a URI. This may include specifying a particular commit or branch, in the case of
URIs referencing a repository; they could also be used to specify a particular
subdirectory of a repository or tarball.

Qualifiers may also be used to ensure content matches what the client expects,
even when there is no ambiguity to be had -- for example, a qualifier specifying
a checksum value.

For example, a particular subdirectory of a particular branch
of a git repository might be identified as:

```
uris: "https://github.com/bazelbuild/remote-apis.git"
qualifiers <
  key: "directory"
  value: "build/bazel/remote/execution/v2"
>
qualifiers <
  key: "scm.branch"
  value: "master"
>
```

Qualifiers are optionally used to identify unstable content at a URL. For
example, a git repository at a certain ref of e.g. e77c4eb could be referenced
as:

```
uri: "https://github.com/bazelbuild/remote-apis.git"
uri: "https://git-mirror.example.com/bazelbuild/remote-apis.git"
qualifiers <
  key: "git.ref"
  value: "e77c4eb"
>
```

Another example would be a tarball, where we qualify the content by a
Subresource Integrity checksum.

```
uri: "https://github.com/bazelbuild/bazel/archive/1.2.0.tar.gz"
qualifiers <
  key: "checksum.sri"
  value: "sha384-6NmGPKYyMNRmjZe6r+7kpAy06QYOEbHg31zhc51IxeeJ8qsqww9w/99Vd6n9W3xj"
>
```

In all cases, the server **MUST** return contents that match the given
qualifiers. So for example, it would be a spec violation to accept a Fetch
request with Subresource Integrity checksum provided and then to return content
that does not match said checksum. Note that it is not required that clients
make fully-specified requests, only that servers respond with a matching
response.

A few additional clarifications:
* Qualifier keys are expected to be unique. Semantics of repeated keys are
  unspecified at this time.
* Qualifiers may be present in any order.
* No attempt is made to distinguish between "Standard" and "Nonstandard"
  qualifiers. In this we follow the guidance of
  https://tools.ietf.org/html/rfc6648. It is anticipated that a standard set of
  keys and associated value semantics will be developed as this proposal lands,
  and added to the documentation as appropriate.
* In cases where the semantics of the request are not immediately clear from the
  URL and/or qualifiers - e.g. dictated by URL scheme - it is recommended to use
  an additional qualifier to remove the ambiguity. The key `resource_type` is
  recommended for this purpose.
* For Push'd assets with qualifiers, the push originator **MUST** ensure the
  qualifiers match the content they're pushing.
* Transitively, for Fetch's of content that were previously Push'd, servers may
  support either exact-match or subset-match of Qualifiers. This allows servers
  to return content including Qualifiers that the server itself doesn't
  understand and cannot validate.

```proto
message Qualifier {
  // The "key" of the qualifier, for example "resource_type" or "scm.branch". No
  // separation is made between 'standard' and 'nonstandard' keys, in accordance
  // with https://tools.ietf.org/html/rfc6648, however implementers *SHOULD* take
  // care to avoid ambiguity.
  string key = 1;

  // The "value" of the qualifier. Semantics will be dictated by the key.
  string value = 2;
}
```

# Bazel

## Implementation plan

1. Refactor the existing `HttpDownloader` path to support non-HTTP
   implementations. While the default implementation will match existing
   behavior, it will be possible for modules to inject a new `Downloader` with
   custom logic.

2. Add a new `--experimental_remote_downloader=ADDRESS` flag to `RemoteModule`,
   which injects a `GrpcRemoteDownloader` backed by a remote endpoint. The
   implementation should leverage existing support code for mTLS credentials
   and UNIX socket proxying.

3. After the implementation has received enough use to be considered stable,
   rename the flag to `--remote_downloader`.

## Backward-compatibility

This feature is intended to be fully backwards compatible with existing workspaces.

# Alternatives Considered

## HTTP Proxy
Bazel supports HTTP proxies when downloading external dependencies. It would be
straightforward to implement a caching intermediate proxy for downloads with the
`http://` scheme, but important functionality would not be possible:

* Proxying of `https://` URLs uses the `CONNECT` method instead of `GET`, and
  the proxy is expected to act as a dumb pipe to upstream. A caching `CONNECT`
  proxy must therefore MITM the TLS handshake, and Bazel would need to have the
  proxy's certificate injected into its CA pool.

* Users want to implement custom policies, for example to prevent resources from
  being downloaded unless their checksum is known (cf.
  [bazel#2442](https://github.com/bazelbuild/bazel/issues/2442)). For security
  reasons, content integrity checksums should not be sent to untrusted
  upstreams. It would be difficult to make Bazel send integrity headers in some
  proxy environments, and not in others.

* This proposal does not prevent a simplified HTTP-based cache from being
  implemented at a later date. If demand for that capability exists, it should
  be proposed separately.

## Direct downloads

The first version of the Remote Repository Cache proposal had the server stream
downloads directly, as in `ByteStream`:

```proto
service RepositoryCache {
  rpc Download(DownloadRequest) returns (stream google.bytestream.ReadResponse);
}
```

This was replaced with the "resolve to digest" mechanism to enable better reuse
of existing storage components, and to make the API more extensible.

* The cache implements policy while allowing Bazel's existing download code to
reason about concurrency and timeouts.

* The cache's protocol specification would no longer have a direct build-time
dependency on `ByteStream`.

* The API structure more closely resembles a traditional load balancer, where
the client asks the LB for lightweight metadata instead of using it for data
transport.

The primary advantage of direct downloads is avoiding an extra round-trip,
because Bazel must query the remote cache potentially up to once per download
in addition to the download itself.

# References

Open issues that could be resolved with this feature (non-exhaustive):
* [bazel#2442](https://github.com/bazelbuild/bazel/issues/2442): Flag to verify
  all WORKSPACE rules have a sha1/sha256
* [bazel#2557](https://github.com/bazelbuild/bazel/issues/2557): Feature
  request: external dependency caching using remote gRPC protocol
* [bazel#4155](https://github.com/bazelbuild/bazel/issues/4155): Allow
  specifying http(s)_proxy per http_archive
* [bazel#4499](https://github.com/bazelbuild/bazel/issues/4499): Generate a
  download list and design tooling to add mirror entries
* [bazel#5144](https://github.com/bazelbuild/bazel/issues/5144): Downloading
  succeeds with a valid (cached?) hash but invalid URL
* [bazel#6342](https://github.com/bazelbuild/bazel/issues/6342): Want a way to
  re-host dependencies easily
* [bazel#6359](https://github.com/bazelbuild/bazel/issues/6359): Allow using
  remote cache for repository cache

