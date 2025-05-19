```shell
sudo pacman -S python-pipx

# 使用 Meson 构建系统初始化一个新项目。build-riscv 是构建目录的名称。
meson setup build-riscv --cross-file config/riscv/riscv64_linux_gcc

cd build-riscv
# 使用 Ninja 构建系统进行编译。
ninja


```
