<!-- SPDX-License-Identifier: Apache-2.0 -->

# 4. Build, load, and roll back safely

Provider experiments can make every RDMA application on a host fail if the
system library is replaced incorrectly. The tutorial therefore uses an
in-place, side-by-side build and never installs over `/usr`.

## 4.1 Pick an appropriate lab environment

Use one of the following:

- a dedicated RDMA development node;
- a VM with a suitable virtual or passed-through device;
- a container used only as a build environment, with deliberate device access
  for hardware tests;
- an upstream software-RDMA setup for generic libibverbs tests, while accepting
  that mlx5-specific paths still require mlx5 hardware.

Do not begin on a host whose management, storage, or cluster control path
depends on the RDMA stack being modified. Preserve a second login or console
path before hardware testing.

## 4.2 Record the installed stack before choosing source

On the target Linux host, collect facts without changing the system:

```sh
ofed_info -s                  # when MLNX_OFED supplies this command
ibv_devices
ibv_devinfo
ldconfig -p | grep -E 'libibverbs|libmlx5'
uname -a
```

Also record distribution package versions, NIC model, firmware, kernel driver,
and the exact application binary used for validation. Package commands differ
between Debian-derived and RPM-derived distributions; store the command and raw
version string in the experiment record.

If the host uses MLNX_OFED, obtain the matching public source package from the
vendor rather than assuming that an upstream tag has the same layout. NVIDIA's
public package documentation identifies the source-code content and layout. Do
not publish a vendor source archive under this tutorial's Apache license.

## 4.3 Use a pinned public baseline for learning

The examples in this guide are checked against public rdma-core v64.0:

```sh
git clone https://github.com/linux-rdma/rdma-core.git
cd rdma-core
git checkout --detach f272237493ef309984036f7b85655e11104c61c8
test "$(git rev-parse HEAD)" = f272237493ef309984036f7b85655e11104c61c8
git switch -c tutorial/provider-overlay
```

For hardware work, keep a second pristine worktree/build at the same commit.
This gives the native control and rollback test an independent artifact instead
of relying only on a runtime mode inside the modified provider.

The expected peeled commit is recorded in [REFERENCES.md](REFERENCES.md). If it
does not match, stop and update the version record before relying on source
locations.

Install build dependencies using the pinned release's own README. Dependency
names evolve; duplicating a package list in this tutorial would age poorly.

## 4.4 Identify integration points before adding files

In the selected provider, locate:

1. its provider CMake target and source list;
2. the provider registration descriptor;
3. context allocation/import functions;
4. every base and variant call to `verbs_set_ops()`;
5. the provider context structure and its cleanup path;
6. implementation functions used as the native underlay;
7. tests or examples that exercise the intended operation family.

For an in-tree experiment, keep the change localized. A typical source layout
would add one provider-private implementation file and one private declaration
file, then list the implementation file in the provider's CMake target. This
guide intentionally provides no ready-to-apply diff.

Before coding, create a design record containing:

- source tag and commit;
- target provider and private ABI version;
- callback fields to replace;
- base/variant/custom overlay order;
- per-family representation/callback profile: native-object wrapper, strict
  custom family, or registry mixed family;
- unsupported constructors and attributes;
- rollback procedure.

## 4.5 Build in place

The upstream convenience script configures an in-place build:

```sh
./build.sh
```

For an explicit debug build:

```sh
cmake -S . -B build -GNinja \
  -DIN_PLACE=1 \
  -DCMAKE_BUILD_TYPE=Debug \
  -DMLX5_DEBUG=ON \
  -DNO_MAN_PAGES=1 \
  -DNO_PYVERBS=1 \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=1

cmake --build build --target ibverbs mlx5 ibv_devices ibv_devinfo ibv_rc_pingpong
```

Target names can differ in a vendor fork. Inspect its CMake definitions rather
than renaming targets until the command happens to work.

An in-place rdma-core build generates provider configuration under the build
tree and provider libraries with a suffix tied to the private ABI version. Keep
the build's libibverbs, headers, provider, configuration, and tools together.

## 4.6 Verify the baseline before enabling customization

Run the in-place tools with no custom mode enabled:

```sh
build/bin/ibv_devices
build/bin/ibv_devinfo
```

Enable libibverbs diagnostics when needed:

```sh
env VERBS_LOG_LEVEL=4 build/bin/ibv_devinfo
```

On a suitable two-endpoint or loopback lab setup, run the upstream RC ping-pong
example before changing callbacks. Save the command, exit status, device/port,
and a short result summary. This becomes the native control.

```sh
# Server
build/bin/ibv_rc_pingpong -d <device> -i <port> -n 1000 -c

# Client
build/bin/ibv_rc_pingpong -d <device> -i <port> -n 1000 -c <server-host>
```

After the classic path passes, repeat with the upstream options relevant to the
declared profile: `-e` exercises CQ events, `-N` uses the new/extended send WR
API, and `-t` requests an extended CQ path with timestamps. These are valuable
negative tests when the custom overlay intentionally supports only classic
post/poll.

## 4.7 Prove which provider is loaded

“The program ran” does not prove it used the custom build. Verify shared-object
selection with at least one loader-level method:

```sh
env LD_DEBUG=libs build/bin/ibv_devinfo 2>/tmp/verbs-loader.log
strace -f -e trace=openat build/bin/ibv_devinfo 2>/tmp/verbs-openat.log
```

Inspect only the library paths and loader errors; do not publish logs containing
machine-specific paths or device identifiers.

libibverbs also recognizes `RDMAV_DRIVERS`/the legacy `IBV_DRIVERS` while it is
not running setuid. At the pinned revision, an absolute value is treated as a
provider path prefix and the private-ABI suffix is appended. This can help a
controlled lab invocation:

```sh
env RDMAV_DRIVERS="$PWD/build/lib/libmlx5" \
    VERBS_LOG_LEVEL=4 \
    "$PWD/build/bin/ibv_devinfo"
```

Use this only with the matching in-place libibverbs. It is not a supported way
to load a provider built for one private ABI into an arbitrary system library.
The generated in-place configuration is usually simpler.

## 4.8 Enable custom mode only after the control passes

The provider should parse its opt-in configuration during context construction
and emit one concise diagnostic indicating:

- source/build identity;
- provider-private ABI version;
- device/context identity in a non-sensitive form;
- selected mode (`NATIVE`, `CUSTOM`, or failure);
- supported operation profile.

Do not log payload, virtual addresses, lkeys/rkeys, GIDs, credentials, or
application secrets. Do not reread configuration on the datapath.

Test one operation family at a time. Start with create/query/destroy closure
before adding post/poll traffic. Keep the customization off by default until
negative tests pass.

## 4.9 Rollback

Because nothing was installed system-wide, rollback is procedural:

1. stop the in-place test process;
2. unset custom provider and mode variables in the test shell;
3. run the pristine same-commit control build, or the distribution-provided
   `ibv_devinfo`, from a clean shell;
4. verify the loaded libibverbs and provider paths again;
5. preserve the failed build and diagnostics privately for analysis.

Never make `sudo make install`, package replacement, or copying a shared object
into `/usr` a normal tutorial step. Packaging and fleet deployment require a
separate operational and license review.

Next: [Validate behavior and compatibility](05-validation-matrix.md).
