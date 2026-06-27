# Yocto Project Interview Questions

**Focus: Recipes, Layers, BitBake, Configuration, Image Customization, Debugging Build Failures**

---

## ⚡ Quick-Fire (Yocto Fluency Gauge)

| Question | Expected Answer |
|----------|----------------|
| What is a recipe in Yocto? | A `.bb` file describing how to fetch, configure, compile, and install a single software component |
| What is a layer? | A collection of recipes, classes, and config organized by purpose (e.g., `meta-raspberrypi`). Layered override system. |
| What is BitBake? | The task execution engine  parses recipes, resolves dependencies, executes tasks in parallel |
| What is `local.conf`? | Machine-local build configuration (MACHINE, DISTRO, build tuning). Not version-controlled. |
| What is `bblayers.conf`? | Declares which layers are active in the build. Order determines priority. |
| What is a `.bbappend` file? | Extends/overrides an existing recipe from another layer without forking it |
| What is `MACHINE`? | Defines the target hardware (e.g., `imx8mm-lpddr4-evk`, `raspberrypi4-64`) |
| What is `DISTRO`? | Distribution policy  init system, C library, package format, distro features |
| What is the difference between `do_compile` and `do_install`? | `do_compile`: builds artifacts. `do_install`: places them in `${D}` (staging filesystem). |
| What is `WORKDIR`? | Per-recipe build directory containing sources, patches, temp logs |

---

## 🔬 Q1: Writing a Custom Recipe

**Question:** Write a BitBake recipe to cross-compile a C++ sensor daemon from a private Git repo. It:
- Depends on `libgpiod` and `protobuf`
- Installs to `/usr/bin/sensor-daemon`
- Installs a systemd service file
- Has a version based on Git tag

**Expected answer:**

```bitbake
SUMMARY = "Sensor data acquisition daemon"
DESCRIPTION = "Reads IMU/temp sensors and publishes via protobuf over Unix socket"
LICENSE = "CLOSED"
LIC_FILES_CHKSUM = ""

SRC_URI = "git://git.company.com/firmware/sensor-daemon.git;protocol=ssh;branch=main"
SRCREV = "${AUTOREV}"
PV = "1.0+git${SRCPV}"

S = "${WORKDIR}/git"

inherit cmake systemd

DEPENDS = "libgpiod protobuf protobuf-native"

EXTRA_OECMAKE = "-DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=OFF"

SYSTEMD_SERVICE:${PN} = "sensor-daemon.service"
SYSTEMD_AUTO_ENABLE = "enable"

do_install:append() {
    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${S}/deploy/sensor-daemon.service ${D}${systemd_system_unitdir}/
}

FILES:${PN} += "${systemd_system_unitdir}/sensor-daemon.service"
```

**Follow-up probes:**
- What does `AUTOREV` do? (Always fetches latest commit  use explicit SRCREV for reproducibility)
- Why `protobuf-native`? (Protoc compiler runs on host during build, needs native build)
- How do you pin to a specific tag? (`SRCREV = "v2.1.0"` with `git://...;tag=v2.1.0`)
- What if cmake isn't available? Use `inherit autotools` or write custom `do_compile()`

---

## 🔬 Q2: Layer Structure and Priority

**Question:** Your project has these layers:

```
meta-poky          (priority 5)
meta-oe            (priority 6)
meta-freescale     (priority 8)
meta-company       (priority 10)
meta-product-xyz   (priority 12)
```

1. A recipe exists in both `meta-oe` and `meta-company`. Which one wins?
2. How do you override a single variable from `meta-freescale` recipe without forking it?
3. How do you add a patch to a recipe from `meta-poky`?

**Expected answers:**

1. **`meta-company` wins**  higher `BBFILE_PRIORITY` layer takes precedence for same-named recipes.

2. **Use a `.bbappend` in `meta-company` or `meta-product-xyz`:**
   ```bitbake
   # meta-company/recipes-bsp/u-boot/u-boot-imx_%.bbappend
   EXTRA_OEMAKE += "CONFIG_CUSTOM_BOARD=y"
   ```

3. **Add a `.bbappend` with `SRC_URI:append` and place patch in the layer:**
   ```bitbake
   # meta-product-xyz/recipes-core/busybox/busybox_%.bbappend
   FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
   SRC_URI:append = " file://fix-segfault.patch"
   ```

**Follow-up:** What is `FILESEXTRAPATHS` and why `:prepend`? (Tells BitBake where to search for `file://` URIs. Prepend ensures your layer's `files/` directory is searched first.)

---

## 🔬 Q3: Image Customization

**Question:** You need to build a minimal Linux image for a gateway device that includes:
- Your custom daemon
- NetworkManager (not ConnMan)
- SSH server (dropbear, not openssh)
- Read-only rootfs with overlay for `/var`
- No package manager in final image

Show the image recipe and relevant `local.conf` / distro settings.

**Expected answer:**

```bitbake
# meta-product/recipes-core/images/gateway-image.bb
SUMMARY = "Minimal gateway image"

inherit core-image

IMAGE_INSTALL = " \
    packagegroup-core-boot \
    sensor-daemon \
    networkmanager \
    dropbear \
    ca-certificates \
    tzdata \
"

IMAGE_FEATURES = ""
# Explicitly no: debug-tweaks, package-management, ssh-server-openssh

# Read-only rootfs
IMAGE_FEATURES += "read-only-rootfs"
EXTRA_IMAGE_FEATURES = ""

# Overlay for writable /var
IMAGE_INSTALL:append = " overlayfs-tools"
```

```bitbake
# local.conf or distro.conf
DISTRO_FEATURES:remove = "x11 wayland vulkan"
DISTRO_FEATURES:append = " systemd"
VIRTUAL-RUNTIME_init_manager = "systemd"
VIRTUAL-RUNTIME_initscripts = "systemd-compat-units"

# Use dropbear instead of openssh
IMAGE_FEATURES:remove = "ssh-server-openssh"
EXTRA_IMAGE_FEATURES += "ssh-server-dropbear"

# No package manager in image
PACKAGE_CLASSES = "package_ipk"
IMAGE_FEATURES:remove = "package-management"
```

**Follow-up probes:**
- How do you make the image smaller? (`IMAGE_ROOTFS_SIZE`, `IMAGE_OVERHEAD_FACTOR`, strip debug symbols, `INHIBIT_PACKAGE_STRIP = "0"`)
- How do you add users/passwords at build time? (`inherit extrausers`, `EXTRA_USERS_PARAMS`)
- What is the difference between `IMAGE_INSTALL` and `RDEPENDS`? (`IMAGE_INSTALL`: image-level package list. `RDEPENDS`: per-recipe runtime dependency.)

---

## 🔬 Q4: Debugging Build Failures

**Question:** You run `bitbake gateway-image` and get:

```
ERROR: sensor-daemon-1.0+gitAUTOINC+abc123-r0 do_compile: ExecutionError('...', 2)
ERROR: Logfile: /home/build/poky/build/tmp/work/cortexa53-poky-linux/sensor-daemon/1.0+gitAUTOINC+abc123-r0/temp/log.do_compile
```

Walk through your debugging process.

**Expected answer:**

```bash
# 1. Read the log file
cat tmp/work/cortexa53-poky-linux/sensor-daemon/1.0+git*/temp/log.do_compile

# 2. Enter the devshell for interactive debugging
bitbake sensor-daemon -c devshell
# Now you have a shell in the build directory with cross-compiler in PATH

# 3. Try building manually
cd /home/build/poky/build/tmp/work/.../sensor-daemon/1.0+git*/git
cmake -B build -DCMAKE_TOOLCHAIN_FILE=...
cmake --build build

# 4. Common failure causes:
# - Missing DEPENDS (header not found)
# - Host contamination (picking up /usr/include instead of sysroot)
# - Cross-compilation issue (linking against host .so)
# - Patch doesn't apply (fuzz/offset)

# 5. Clean and rebuild
bitbake sensor-daemon -c cleansstate
bitbake sensor-daemon

# 6. Check dependency graph
bitbake -g sensor-daemon
cat recipe-depends.dot | grep sensor-daemon
```

**Follow-up probes:**
- What's the difference between `cleanall`, `cleansstate`, and `clean`? (`clean`: removes build dir. `cleansstate`: removes sstate cache. `cleanall`: removes downloads too.)
- How do you add a compile flag just for debugging? (`bitbake sensor-daemon -c compile -f` after modifying recipe, or use `EXTRA_OECMAKE:append`)
- What is sstate and why does it matter? (Shared State cache  prebuilt task outputs. Speeds up rebuilds. Invalidation is hash-based.)

---

## 🔬 Q5: SDK Generation and Application Development Workflow

**Question:** Describe the workflow for a C++ developer who doesn't use Yocto daily but needs to cross-compile against your custom image's sysroot.

**Expected answer:**

```bash
# 1. Generate extensible SDK
bitbake gateway-image -c populate_sdk_ext
# Or standard SDK:
bitbake gateway-image -c populate_sdk

# 2. Install SDK on developer machine
./tmp/deploy/sdk/poky-glibc-x86_64-gateway-image-cortexa53-toolchain-4.0.sh

# 3. Developer sources the environment
source /opt/poky/4.0/environment-setup-cortexa53-poky-linux

# 4. Build with cmake (cross-compilation is transparent)
cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build

# 5. Deploy to target
scp build/sensor-daemon root@192.168.1.100:/usr/bin/
```

**Follow-up:** How does `environment-setup-*` work? (Sets `CC`, `CXX`, `CFLAGS`, `LDFLAGS`, `PKG_CONFIG_PATH`, sysroot  makes native cmake/meson work as cross-compilers)

---

## 🔬 Q6: Yocto for Production  Release Engineering

**Question:** How do you ensure reproducible builds for a product shipping to customers?

**Expected answers:**

1. **Pin SRCREV** for all recipes (no `AUTOREV` in production)
2. **Lock layer revisions**  use a repo manifest (Google repo) or `kas` configuration
3. **Shared sstate cache** on CI server (speeds builds, ensures binary equivalence)
4. **DL_DIR mirror**  local download mirror prevents upstream disappearance
5. **BUILDHISTORY**  track package sizes, installed files, dependencies across builds
6. **License compliance**  `LICENSE_CREATE_PACKAGE = "1"`, generate SPDX/SBOM
7. **Reproducible builds**  `INHERIT += "reproducible_build"` (bit-for-bit identical outputs)
8. **Signed images**  integrate secure boot signing into `do_deploy`

```bitbake
# kas project configuration (kas.yml)
header:
  version: 14
machine: imx8mm-lpddr4-evk
distro: company-distro

repos:
  poky:
    url: https://git.yoctoproject.org/poky
    refspec: kirkstone-4.0.13
  meta-freescale:
    url: https://github.com/Freescale/meta-freescale
    refspec: kirkstone
  meta-company:
    url: git@git.company.com:yocto/meta-company.git
    refspec: release/2.5.0
```

---

## ⚡ Additional Quick Checks

| Question | Key Answer |
|----------|-----------|
| What is `PREFERRED_VERSION`? | Force a specific version of a recipe when multiple exist |
| What is `COMPATIBLE_MACHINE`? | Restricts which MACHINEs a recipe can be built for |
| What is `IMAGE_POSTPROCESS_COMMAND`? | Run shell commands after rootfs is assembled (e.g., remove temp files) |
| What is `do_deploy`? | Task that copies final artifacts (kernel, DTB, bootloader) to `deploy/` directory |
| What is the difference between `-native` and `-cross` recipes? | Native: runs on build host. Cross: runs on host, produces target-arch output. |
| What is `multiconfig`? | Build multiple MACHINE/DISTRO combos in single `bitbake` invocation |
