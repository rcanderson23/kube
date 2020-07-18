# kube-rs
[![CircleCI](https://circleci.com/gh/clux/kube-rs.svg?style=shield)](https://circleci.com/gh/clux/kube-rs)
[![Client Capabilities](https://img.shields.io/badge/Kubernetes%20client-Silver-blue.svg?style=plastic&colorB=C0C0C0&colorA=306CE8)](http://bit.ly/kubernetes-client-capabilities-badge)
[![Client Support Level](https://img.shields.io/badge/kubernetes%20client-alpha-green.svg?style=plastic&colorA=306CE8)](http://bit.ly/kubernetes-client-support-badge)
[![Crates.io](https://img.shields.io/crates/v/kube.svg)](https://crates.io/crates/kube)
[![Discord chat](https://img.shields.io/discord/500028886025895936.svg?logo=discord&style=plastic)](https://discord.gg/tokio)

Rust client for [Kubernetes](http://kubernetes.io) in the style of a more generic [client-go](https://github.com/kubernetes/client-go) plus a runtime abstraction inspired by [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime).

These crate makes certain assumptions about the kubernetes api to allow writing generic abstractions, and as such contains rust reinterpretations of reflectors, informers, and controller so that you can writing kubernetes controllers/watchers/operators more easily.

NB: This library is currently undergoing a lot of changes with async/await stabilizing. Please check the [CHANGELOG](./CHANGELOG.md) when upgrading.

## Installation
Select a version of `kube` along with the [generated k8s api types](https://github.com/Arnavion/k8s-openapi) that corresponds to your cluster version:

```toml
[dependencies]
kube = "0.35.1"
kube-derive = "0.35.1"
k8s-openapi = { version = "0.8.0", default-features = false, features = ["v1_17"] }
```

Note that turning off `default-features` for `k8s-openapi` is recommended to speed up your compilation (and we provide an api anyway).

## Usage
See the [examples directory](./kube/examples) for how to watch over resources in a simplistic way.

**[API Docs](https://docs.rs/kube/)**

Some real world examples:

- [version-rs](https://github.com/clux/version-rs): super lightweight reflector deployment with actix 2 and prometheus metrics

- [krustlet](https://github.com/deislabs/krustlet): a complete `WASM` running `kubelet`

## Api
The direct `Api` type takes a client, and is constructed with either the `::global` or `::namespaced` functions:

```rust
use k8s_openapi::api::core::v1::Pod;
let pods: Api<Pod> = Api::namespaced(client, "default");

let p = pods.get("blog").await?;
println!("Got blog pod with containers: {:?}", p.spec.unwrap().containers);

let patch = json!({"spec": {
    "activeDeadlineSeconds": 5
}});
let patched = pods.patch("blog", &pp, serde_json::to_vec(&patch)?).await?;
assert_eq!(patched.spec.active_deadline_seconds, Some(5));

pods.delete("blog", &DeleteParams::default()).await?;
```

See the examples ending in `_api` examples for more detail.

## Custom Resource Definitions
Working with custom resources uses automatic code-generation via [proc_macros in kube-derive](./kube-derive).

You need to `#[derive(CustomResource)]` and some `#[kube(attrs..)]` on a spec struct:

```rust
#[derive(CustomResource, Serialize, Deserialize, Default, Clone)]
#[kube(group = "clux.dev", version = "v1", namespaced)]
pub struct FooSpec {
    name: String,
    info: String,
}
```

Then you can use a lot of generated code as:

```rust
println!("kind = {}", Foo::KIND); // impl k8s_openapi::Resource
let foos: Api<Foo> = Api::namespaced(client, "default");
let f = Foo::new("my-foo");
println!("foo: {:?}", f)
println!("crd: {}", serde_yaml::to_string(Foo::crd());
```

There are a ton of kubebuilder like instructions that you can annotate with here. See the `crd_` prefixed [examples](./kube/examples) for more.

## Runtime
The `kube_runtime` create contains sets of higher level abstractions on top of the `Api` and `Resource` types so that you don't have to do all the `watch`/`resourceVersion`/storage book-keeping yourself.

### watcher
A low level streaming interface (similar to informers) that presents `Applied`, `Deleted` or `Restarted` events.


```rust
let api = Api::<Pod>::namespaced(client, "default");
let watcher = watcher(api, ListParams::default());
```

This now gives a continual stream of events and you do not need to care about the watch having to restart, or connections dropping.

```rust
let apply_events = try_flatten_applied(watcher).boxed_local()
while let Some(event) = watcher.try_next().await? {
    println!("Applied: {}", Meta::name(&event));
}
```

NB: the plain stream items a `watcher` returns are different from `WatchEvent`. If you are following along to "see what changed", you should flatten it with one of the utilities like `try_flatten_applied` or `try_flatten_touched`.

## reflector
A `reflector` is a `watcher` with `Store` on `K`. It uses all the `Event<K>` exposed by `watcher` to ensure that the state therein is as accurate as possible.


```rust
let nodes: Api<Node> = Api::namespaced(client, &namespace);
let lp = ListParams::default()
    .labels("beta.kubernetes.io/instance-type=m4.2xlarge");
let store = reflector::store::Writer::<Node>::default();
let rf = reflector(store, watcher(nodes, lp));
```

At this point you can listen to the `reflector` as if it was a `watcher`, but you can also query the store at any point.

### controller
A `reflector` with an arbitrary number of watchers that schedule events internally to send events through a reconciler:

```rust
Controller::new(root_kind_api, ListParams::default())
    .owns(child_kind_api, ListParams::default())
    .run(reconcile, error_policy, context)
    .for_each(|res| async move {
        match res {
            Ok(o) => info!("reconciled {:?}", o),
            Err(e) => warn!("reconcile failed: {}", Report::from(e)),
        }
    })
    .await;
```

Here `reconcile` and `error_policy` refer to functions you define. The first will be called when the root or child elements change, and the second when the `reconciler` returns an `Err`.

## Examples
Examples that show a little common flows. These all have logging of this library set up to `debug`, and where possible pick up on the `NAMSEPACE` evar.

```sh
# watch pod events
cargo run --example pod_informer
# watch event events
cargo run --example event_informer
# watch for broken nodes
cargo run --example node_informer
```

or for the reflectors:

```sh
cargo run --example pod_reflector
cargo run --example node_reflector
cargo run --example deployment_reflector
cargo run --example secret_reflector
cargo run --example configmap_reflector
```

for one based on a CRD, you need to create the CRD first:

```sh
kubectl apply -f examples/foo.yaml
cargo run --example crd_reflector
```

then you can `kubectl apply -f crd-baz.yaml -n default`, or `kubectl delete -f crd-baz.yaml -n default`, or `kubectl edit foos baz -n default` to verify that the events are being picked up.

For straight API use examples, try:

```sh
cargo run --example crd_api
cargo run --example job_api
cargo run --example log_stream
cargo run --example pod_api
NAMESPACE=dev cargo run --example log_stream -- kafka-manager-7d4f4bd8dc-f6c44
```

## Rustls
Kube has basic support ([with caveats](https://github.com/clux/kube-rs/issues?q=is%3Aissue+is%3Aopen+rustls)) for [rustls](https://github.com/ctz/rustls) as a replacement for the `openssl` dependency. To use this, turn off default features, and enable `rustls-tls`:

```sh
cargo run --example pod_informer --no-default-features --features=rustls-tls
```

or in `Cargo.toml`:

```toml
[dependencies]
kube = { version = "0.35.1", default-features = false, features = ["rustls-tls"] }
k8s-openapi = { version = "0.8.0", default-features = false, features = ["v1_17"] }
```

This will pull in the variant of `reqwest` that also uses its `rustls-tls` feature.

## License
Apache 2.0 licensed. See LICENSE for details.
