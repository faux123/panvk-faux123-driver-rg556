# panvk-faux123-driver-rg556

Mesa **PanVK**, the open source Vulkan driver for Arm Mali GPUs, built for
Android and packaged for [AdrenoTools](https://github.com/bylaws/libadrenotools)
style driver loading on the **Anbernic RG556** (Mali-G57). The driver itself
needs no root.

The driver talks to the `mali_kbase` kernel driver that is already on the
device, and the system driver stays in place.

I apply fixes that are not upstream in mesa, and I verify every one of them on
my own hardware before I release it. Each release lists what it changes.

**This is an early, experimental driver. It is not conformant, and it has one
known rendering fault, described under
[What works and what does not](#what-works-and-what-does-not).**

**This is a proof of concept, released as is, with no support.** See
[No support](#no-support).

**Everything here is verified on one device: my Anbernic RG556, Mali-G57 MC4,
running a custom kernel.** I have not run it on an RG556 with the stock kernel,
so I can make no promises there or anywhere else. See
[Other devices](#other-devices-not-tested-no-guarantees).

Grab the latest `.adpkg.zip` from [Releases](../../releases) and import it in an
app whose driver picker accepts a PanVK package on Mali.

---

## Where the driver comes from

Nothing here is a repackaged binary from somewhere else. It is built from source.

| | |
|---|---|
| Source | Mesa 26.3.0-devel, from [`funnymdzz/mesa`](https://github.com/funnymdzz/mesa), a Mesa tree that adds a `mali_kbase` backend |
| Mali-G57 generation support | the patch series from [`Noysz/panvk-g99-jm`](https://github.com/Noysz/panvk-g99-jm) |
| My patches | 29, summarized under [What I added](#what-i-added) |
| Toolchain | Android NDK 27.3, meson cross build, `aarch64`, API level 33 |
| Packaging | `.adpkg.zip`: `libvulkan_panfrost.so` plus `meta.json` |

**Every release names the exact build**, in its release notes and in the
`meta.json` inside the package.

The build container, the patches, the test harnesses and a recreate guide live in
a private companion repository.

### Why the stock kernel driver and not Panfrost

Upstream PanVK expects the open source Panfrost or Panthor kernel drivers. A
retail Android handheld ships Arm's `mali_kbase` kernel driver instead. PanVK on
top of `mali_kbase` is meant to need no kernel change. My own RG556 runs a custom
kernel, though, so whether this package works on the stock RG556 kernel is not
verified.

### What I added

- Every job failed on this device until the job layout matched Arm's own
  `mali_kbase`. The patch series was developed on MediaTek's variant, which
  differs.
- The screen flickered badly on the device's own panel. The driver now presents
  compressed frames, as the stock driver does.
- Speed: the first working build took 1.88 times as long per frame as the stock
  driver. A series of fixes brought that to about 1.06.
- One scene ran at 3.5 frames a second and now runs at about 29.
- Indirect draws drew every model with the same texture.
- Geometry shaders, tessellation, and transform feedback come from the newer
  Noysz patch series since v0.13.0, with three speed fixes for that series that
  I made on my Retroid Pocket 4 Pro build, and the job layout fix redone for it.
- Memory that an app hands the driver, and that a game writes through a cached
  mapping, is written out of the processor's cache before each submit since
  v0.14.0. On my Retroid Pocket 4 Pro the lack of this showed as black dashes in
  a game's textures. This device did not show them; the fix is carried so both
  builds stay the same.
- The Android swapchain, GPU clock, and repeated-submit fixes from my
  [Retroid Pocket 4 Pro build](https://github.com/faux123/panvk-faux123-driver-rp4pro).

---

## How it was tested

Everything below was measured on **one device**: my Anbernic RG556, firmware
V1.16, Android 13, Unisoc T820, Mali-G57 MC4, custom kernel 5.4.276, Arm
`mali_kbase` r40p0. The bootloader is unlocked and the device is rooted. My
measuring tools use root. The driver does not.

The test rig is the Khronos [Vulkan-Samples](https://github.com/KhronosGroup/Vulkan-Samples)
app, rebuilt with AdrenoTools linked in so it loads a chosen driver directly. No
Wine, no DXVK, no Box64, no emulation. One variable: the driver `.so`.

- 29 of 29 offscreen graphics and compute tests pass.
- All 114 samples were launched one at a time: 67 run, 42 refuse to start
  because a feature is missing, 3 give no result, 2 have no assets, and none
  crash. The stock driver runs 58.
- My feature validator passes 14 checks on this driver against 6 on the stock
  driver. That count is from an earlier version of the validator. With the
  current one, v0.13.0 passes 103 items and v0.12.0 passed 89.

The sample launch count and the frame times below were measured on v0.11.0 and
v0.12.0. On v0.13.0 I timed nine scenes against v0.12.0 and they take the same
time within run-to-run spread; the full set was not run again.

Frame times against the stock Mali driver, over the 52 samples both drivers run:
1.06 times as long by geometric mean. Nine samples are faster than stock by more
than 5% and 22 are slower. The worst like-for-like one is `instancing` at 1.17
times. That was measured on the build just before this one, which differs only
in review fixes. This build measured 1.08, with a newer version of the test app
than the stock baseline used.

Those are single sweeps, so a difference of a few percent is within run-to-run
spread. A game running through a translation layer is usually limited by the CPU
rather than the driver, so do not expect these ratios in a game.

## Other devices: not tested, no guarantees

I own one RG556, and it runs a custom kernel. Nothing here has been run on
anything else. That includes an RG556 on the stock kernel, other Unisoc T820
devices, and other Mali-G57 devices.

Concretely, on any other device:

- The driver may not load at all. Each vendor's `mali_kbase` differs. This
  package uses Arm's job layout and fails on MediaTek's `mali_kbase`, so do not
  use it on a MediaTek device.
- Every fix I ship was written for the hardware I have. On a different device it
  may be unnecessary, and it may not be harmless.
- Nothing on this page was measured on that hardware, so none of the numbers
  apply to it.
- I will not look into problems on hardware I do not own.

Use it if you want to, but that is the honest state of it, and you are on your
own with it.

I build one driver per device I own, because that is the only way I can test
it: [Retroid Pocket 4 Pro](https://github.com/faux123/panvk-faux123-driver-rp4pro)
and [GameForce Ace](https://github.com/faux123/panvk-faux123-driver-ace).

## Installing

**Read this first: you need an app that can load it.** Most driver pickers
were written for Adreno and refuse or ignore a Mali package.
[GameNative-Mali](https://github.com/faux123/GameNative-Mali/releases) can import
it from version 1.2.1-mali.11: open **Driver Manager**, import the zip, then
select it for a game under **Graphics**. That import screen is new and has had
little use.

1. Download `panvk_faux123_rg556_<version>.adpkg.zip` from
   [Releases](../../releases). Do not unzip it.
2. Import it in your app's driver manager, the same way you would import a Turnip
   package on an Adreno device.
3. Select it, then restart the container or game.

The driver has to be loaded by an app that opts in. Android does not let you
replace the system Vulkan driver for arbitrary apps without root, so a normal
Play Store game cannot use this.

To go back, select the system driver again. Nothing on the device is replaced.

---

## What works and what does not

**Known fault: indirect draws fail now and then.** In the `multi_draw_indirect`
sample, about half of the launches lose some draws during the first second, and
then it stops. The stock driver does not do this. I have not found the cause.

**Not tested on this build:** any game other than Giana Sisters: Twisted Dreams,
and long sessions.

**Not offered by this driver on the Mali-G57**, so an app that needs them refuses
to start rather than failing later:

Timestamp queries.

Geometry shaders, tessellation shaders, and transform feedback are offered since
v0.13.0. On this device they have been checked with three small tests and one
sample scene, not with a conformance run and not with a game.

This driver does offer features the stock driver lacks, which is why it runs
more of the samples: push descriptors, graphics pipeline library, host image
copy, pipeline binary, and vertex input dynamic state among them.

---

## No support

This is a proof of concept. I built this driver for my own device, and I am
releasing it so the community can see what is possible and build on it.

- There is no support from me. I do not answer bug reports or questions.
- I do not take requests for devices I do not own. If I do not have the device,
  I will not develop for it.
- I do not do remote debugging or beta testing.
- I may update this driver for myself from time to time and post it here. There
  is no schedule and no promise.

If you want to help, support the original developers listed under
[Credits](#credits).

---

## Credits

- **Mesa and the Panfrost team** wrote PanVK. This repository is a build of
  their work with patches on top. All the hard parts are theirs.
- **[funnymdzz](https://github.com/funnymdzz/mesa)** for the `mali_kbase`
  backend this build sits on.
- **[Noysz](https://github.com/Noysz/panvk-g99-jm)** for the patch series that
  brings PanVK up on this GPU generation, developed on a Mali-G57.
- **[bylaws](https://github.com/bylaws/libadrenotools)** for AdrenoTools, which
  is the only reason a custom Vulkan driver can be loaded without root.
- **Khronos** for Vulkan-Samples, which turned out to be a far better driver
  test bench than any game.

## License

PanVK is [MIT licensed](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/docs/license.rst),
and so are the patches in this build. See [LICENSE](LICENSE).

The driver is compiled against Arm's `mali_kbase` interface headers, which are
GPL-2.0 with the Linux syscall note. That note is what allows a program to use
a kernel interface without taking on the kernel's license.

This project is not affiliated with or endorsed by Arm, Unisoc, Anbernic, Google,
the Khronos Group, or the Mesa project.
