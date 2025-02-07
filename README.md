# chroot-runner-action

Run tests natively and build images directly from GitHub Actions using a
chroot-based virtualized environment.

With this action, you can:

- run tests in an environment closer to a real embedded system
userland Linux emulation;
- build artifacts in such environment and upload them;
- prepare images that are ready to run on Raspberry Pi and other ARM embedded
devices.

## Usage

Minimal usage is as follows:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: pguyot/arm-runner-action@v2
      with:
        commands: |
            commands to run tests
```

Typical usage to upload an image as an artifact:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: pguyot/arm-runner-action@v2
      id: build_image
      with:
        base_image: raspios_lite:2022-04-04
        commands: |
            commands to build image
    - name: Compress the release image
      if: github.ref == 'refs/heads/releng' || startsWith(github.ref, 'refs/tags/')
      run: |
        mv ${{ steps.build_image.outputs.image }} my-release-image.img
        xz -0 -T 0 -v my-release-image.img
    - name: Upload release image
      uses: actions/upload-artifact@v2
      if: github.ref == 'refs/heads/releng' || startsWith(github.ref, 'refs/tags/')
      with:
        name: Release image
        path: my-release-image.img.xz
```

Several scenarios are actually implemented as [tests](/.github/workflows).

### Host and guest OS

The action has been tested with `ubuntu-24.04` and `ubuntu-24.04-arm`.

### Commands

The repository is copied to the image before the commands script is executed
in the chroot environment. The commands script is copied to /tmp/ and is
deleted on cleanup.

### Inputs

#### `commands`

Commands to execute. Written to a script within the image. Required.

#### `base_image`

Base image to use. By default, uses latest `raspios_lite` image. Please note
that this is not necessarily well suited for continuous integration as
the latest image can change with new releases.

The following values are allowed:

- `raspbian_lite:2020-02-13`
- `raspbian_lite:latest`
- `raspios_lite:2021-03-04`
- `raspios_lite:2021-05-07`
- `raspios_lite:2021-10-30`
- `raspios_lite:2022-01-28`
- `raspios_lite:2022-04-04`
- `raspios_lite:latest` (armhf build, *default*)
- `raspios_lite_arm64:2022-01-28` (arm64)
- `raspios_lite_arm64:2022-04-04` (arm64)
- `raspios_lite_arm64:latest` (arm64)
- `dietpi:rpi_armv6_bullseye`
- `dietpi:rpi_armv7_bullseye`
- `dietpi:rpi_armv8_bullseye` (arm64)
- `raspi_1_bullseye:20220121` (armel)
- `raspi_2_bullseye:20220121` (armhf)
- `raspi_3_bullseye:20220121` (arm64)
- `raspi_4_bullseye:20220121` (arm64)

The input parameter also accepts any custom URL beginning in http(s)://...

More images will be added, eventually. Feel free to submit PRs.

#### `image_additional_mb`

Enlarge the image by this number of MB. Default is to not enlarge the image.

#### `copy_artifact_path`

Source paths(s) inside the image to copy outside after the commands have
executed. Relative to the `/<repository_name>` directory or the directory
defined with `copy_repository_path`. Globs are allowed. To copy multiple paths,
provide a list of paths, separated by semicolons. Default is not to copy.

#### `copy_artifact_dest`

Destination path to copy outside the image after the commands have executed.
Relative to the working directory (outside the image). Defaults to `.`

#### `copy_repository_path`

Absolute path, inside the image, where the repository is copied or mounted.
Defaults to `/<repository_name>`. It is also the working directory where
commands are executed.

The repository is copied unless `bind_mount_repository` is set to true.

#### `bind_mount_repository`

Bind mount the repository within the image instead of copying it. Default is
to copy files.

If mounted, any modification of files within the repository by the target
emulated system will persist after execution. It does not accelerate execution
significantly but can simplify the logic by avoiding the copy artifact step
from the target system.

#### `cpu_info`

Path to a fake cpu_info file to be used instead of `/proc/cpuinfo`. Default is
to not fake the CPU (/proc/cpuinfo will report CPU of the GitHub runner).

Some software checks for features using `/proc/cpuinfo` and this option can be
used to trick them. The path is relative to the action (to use pre-defined
settings) or to the local repository.

Bundled with the action are the following files:
- `cpuinfo/raspberrypi_4b`
- `cpuinfo/raspberrypi_3b` (with a 32 bits system)
- `cpuinfo/raspberrypi_zero_w`
- `cpuinfo/raspberrypi_zero2_w` (with a 32 bits system)
- `cpuinfo/raspberrypi_zero2_w_arm64` (with a 64 bits system)

On real hardware, the `/proc/cpuinfo` file content depends on the CPU being
used in 32 bits or 64 bits mode, which in turn depends on the base image.
Consequently, you may want to use `cpuinfo/raspberrypi_zero2_w_arm64` for
64 bits builds and `cpuinfo/raspberrypi_zero2_w` for 32 bits builds.

#### `optimize_image`

Zero-fill unused filesystem blocks and shrink root filesystem during final clean-up, to make any later
image compression more efficient. Default is to optimize image.

#### `use_systemd_nspawn`

Use `systemd-nspawn` instead of chroot to run commands. Default is to use
chroot.

#### `systemd_nspawn_options`

Additional options passed to `systemd-nspawn`. For example, `-E CI=${CI}` to pass
CI environment variable. See [systemd-nspawn(1)](https://manpages.ubuntu.com/manpages/focal/man1/systemd-nspawn.1.html).

#### `rootpartition`

Index (starting with 1) of the root partition. Default is 2, which is suitable
for Raspberry Pi. NVIDIA Jetson images require 1. This is the partition that is
resized with `image_additional_mb` option.

#### `bootpartition`

Index (starting with 1) of the boot partition which gets mounted at /boot.
Default is 1, which is suitable for Raspberry Pi. If the value is empty,
the partition is not mounted.

#### `shell`

Path to shell or shell name to run the commands in. Defaults to /bin/sh.
If missing, it will be installed. See `shell_package`.
If defined as basename filename, it will be used as long as the shell binary
exists under PATH after the package is installed.

Parameters can be passed to the shell, e.g.:
```yaml
shell: /bin/bash -eo pipefail
```

#### `shell_package`

The shell package to install, if different from shell. It may be handy
with some shells that come packaged under a different package name.

For example, to use `ksh93` as shell, set `shell` to `ksh93` and
`shell_package` to `ksh`.

#### `exit_on_fail`

Exit immediately if a command exits with a non-zero status. Default is to exit.
Set to `no` or `false` to disable exiting on command failure. This only works
with `sh`, `bash` and `ksh` shells.

#### `debug`

Display executed commands as they are executed. Enabled by default.

#### `import_github_env`

Imports variables written so far to `$GITHUB_ENV` to the image. Default is not
to import any environment. This may be useful for sharing external variables with
the virtual environment. Set to `yes` or `true` to enable.

Practically, this setting allows constructs like `${VARIABLE_NAME}` instead of
`${{ env.VARIABLE_NAME }}` within the command set.

#### `export_github_env`

Enables `$GITHUB_ENV` for commands in the image and exports its contents on
completion to subsequent tasks. This option is an alternative to using a
file-based artifact for passing the results of commands outside the image
environment.

Note this parameter does not enable importing any contents written to
`$GITHUB_ENV` ahead of running the commands. For that, use `import_github_env`.

### Outputs

#### `image`

Path to the image, useful after the step to upload the image as an artifact.

