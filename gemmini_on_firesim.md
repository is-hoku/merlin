# Gemmini on FireSim Workflow
1. `python3 tools/setup.py submodules --submodules-profile core --submodule-sync`  
2. For host compilation, `uv run tools/merlin.py build --profile gemmini --config release`  
3. Remove `torch.operator` and `torch.ao.quantization` from input MLIR

```bash
build/host-merlin-release/install/bin/iree-opt braggnn-gemmini.mlir --iree-plugin=gemmini --torch-match-quantized-custom-ops --torch-fuse-quantized-ops -o braggnn-gemmini-opt.mlir
```

4. Compile targetting for Gemmini

```bash
uv run tools/merlin.py compile braggnn-gemmini-opt.mlir --target gemmini_braggnn --quantized
```

5. Show op counts

```bash
build/host-merlin-release/install/bin/iree-opt braggnn-gemmini-opt.mlir --print-op-stats
```

4. Compile IREE tools for execution on RISC-V
Refer [Merlin docs](https://ucb-bar.github.io/merlin/different_build_types/#cross-compilations-risc-v)

```bash
export WORKSPACE_DIR=${PWD}

# --- SELECT SOURCE ---
# Option A: Forked IREE
export IREE_SRC=${WORKSPACE_DIR}/third_party/iree_bar
# Option B: Standard IREE (Plugin)
# export IREE_SRC=${WORKSPACE_DIR}/third_party/iree
# ---------------------

# Host Paths
export BUILD_HOST_DIR=${WORKSPACE_DIR}/build/host-merlin-release
export INSTALL_HOST_DIR=${BUILD_HOST_DIR}/install

# RISC-V Paths
export RISCV_TOOLCHAIN_ROOT=${WORKSPACE_DIR}/../riscv
export BUILD_RISCV_DIR=${WORKSPACE_DIR}/build-riscv
```

Run `riscv_bootstrap.sh` through `cd ${IREE_SRC} && ./build_tools/riscv/riscv_bootstrap.sh`.  

```bash
unset CFLAGS CXXFLAGS

cmake \
  -G Ninja \
  -B "${BUILD_RISCV_DIR}" \
  -S "${IREE_SRC}" \
  -DCMAKE_TOOLCHAIN_FILE="${IREE_SRC}/build_tools/cmake/riscv.toolchain.cmake" \
  -DIREE_HOST_BIN_DIR="${INSTALL_HOST_DIR}/bin" \
  -DRISCV_CPU=linux-riscv_64 \
  -DIREE_BUILD_COMPILER=OFF \
  -DRISCV_TOOLCHAIN_ROOT="${RISCV_TOOLCHAIN_ROOT}/toolchain/clang/linux/RISCV" \
  -DCMAKE_BUILD_TYPE=Release \
  -DIREE_ENABLE_RUNTIME_TRACING=OFF \
  -DIREE_ENABLE_CPUINFO=OFF \
  -DIREE_HAL_DRIVER_DEFAULTS=OFF \
  -DIREE_HAL_DRIVER_LOCAL_SYNC=ON \
  -DIREE_HAL_DRIVER_LOCAL_TASK=ON \
  -DIREE_BUILD_TESTS=OFF \
  -DIREE_BUILD_SAMPLES=OFF \
  -DCMAKE_BUILD_WITH_INSTALL_RPATH=ON

cmake --build "${BUILD_RISCV_DIR}"
```

4. Create workloads by FireMarshal
Copy IREE tools to FireSim:
```bash
cd build-riscv/tools
cp iree-benchmark-module ~/chipyard/software/firemarshal/br-merlin/overlay/root
```

Create FireMarshal workload like this:
```br-merlin.json
{
    "name" : "br-merlin",
    "base" : "br-base.json",
    "rootfs-size" : "8GB",
    "overlay" : "overlay",
    "command" : "/root/iree-benchmark-module --module=/root/braggnn-gemmini.vmfb --function=main --input=1x1x11x11xf32=@/root/input.npy --benchmark_time_unit=us --benchmark_min_warmup_time=1 --benchmark_repetitions=10 --benchmark_report_aggregates_only=false"
}
```
