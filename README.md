# Docker Registry Client

An API client for the [V2 Docker Registry
API](http://docs.docker.com/registry/spec/api/), for Go applications.

## Imports

```go
import (
    "github.com/heroku/docker-registry-client/registry"
    "github.com/docker/distribution/digest"
    "github.com/docker/distribution/manifest"
    "github.com/docker/libtrust"
)
```

## Creating A Client

```go
url      := "https://registry-1.docker.io/"
username := "" // anonymous
password := "" // anonymous
hub, err := registry.New(url, username, password)
```

Creating a registry will also ping it to verify that it supports the registry
API, which may fail. Failures return non-`nil` err values.

Authentication supports both HTTP Basic authentication and OAuth2 token
negotiation.

## Listing Tags

Each Docker repository has a set of tags -- named images that can be downloaded.

```go
tags, err := hub.Tags("heroku/cedar")
```

The tags will be returned as a slice of `string`s.

## Downloading Manifests

Each tag has a corresponding manifest, which lists the layers and image
configuration for that tag.

```go
manifest, err := hub.Manifest("heroku/cedar", "14")
```

The returned manifest will be a `manifest.SignedManifest` pointer. For details,
see the `github.com/docker/distribution/manifest` library.

## Downloading Layers

Each manifest contains a list of layers, filesystem images that Docker will
compose to create containers.

```go
// or obtain the digest from an existing manifest's FSLayer list
digest := digest.NewDigestFromHex(
    "sha256",
    "a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4",
)
reader, err := hub.DownloadLayer("heroku/cedar", digest)
)
if reader != nil {
    defer reader.Close()
}
if err != nil {
    return err
}
```

## Uploading Layers

This library can also publish new layers:

```go
digest := digest.NewDigestFromHex(
    "sha256",
    "a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4",
)
exists, err := hub.HasLayer("example/repo", digest)
)
if err != nil {
    // …
}
if !exists {
    stream := …;
    hub.UploadLayer("example/repo", digest, stream)
}
```

## Uploading Manifests

First, create a signed manifest:

```go
manifest := &manifest.Manifest{
    Versioned: manifest.Versioned{
        SchemaVersion: 1,
    },
    Tag: "latest",
    // …
}

key, err := libtrust.GenerateECP256PrivateKey()
if err != nil {
    // …
}

signedManifest := manifest.Sign(manifest, key)
if err != nil {
    // …
}
```

Production applications should probably reuse keys, rather than generating
ephemeral keys. See the libtrust documentation for details.

Then, upload the signed manifest:

```go
err := hub.PutManifest("example/repo", "latest", signedManifest)
if err != nil {
    // …
}
```

This will also create or update tags, as necessary.
