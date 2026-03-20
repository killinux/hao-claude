# hao-claude

claude的学习笔记保存到这里

mame的代码需要 https://github.com/killinux/emularity 和 https://github.com/killinux/mame


# 环境使用：

cd emsdk

./emsdk activate 3.1.50

source ./emsdk_env.sh

# 编译的时候使用

## 编译 shadfrce（用 REGENIE=1 应用新的 genie.lua 配置）
emmake make SUBTARGET=shadfrce SOURCES=src/mame/technos/shadfrce.cpp REGENIE=1 -j$(sysctl -n hw.logicalcpu)

## 之后编译其他游戏就不需要 REGENIE=1 了（genie.lua 只改一次）
emmake make SUBTARGET=psikyo SOURCES=src/mame/psikyo/psikyo.cpp -j$(sysctl -n hw.logicalcpu)

