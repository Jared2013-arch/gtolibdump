# gtolibdump
模组限制加载方式


```package com.gtolib;

import com.google.common.collect.ImmutableSet;
import org.objectweb.asm.tree.ClassNode;
import org.spongepowered.asm.mixin.extensibility.IMixinConfigPlugin;
import org.spongepowered.asm.mixin.extensibility.IMixinInfo;
import java.lang.invoke.MethodHandle;
import java.util.List;
import java.util.Set;

/**
 * GTOLib Mixin 控制插件
 * 负责在运行时决定哪些字节码修改应当生效。
 */
public final class MixinConfigPlugin implements IMixinConfigPlugin {

    // 伴生对象，存储静态变量
    public static final class Companion {
        public String hash; // 可能用于版本校验或安全检查的哈希值
        private ImmutableSet<String> s; // 内部使用的黑名单或白名单集合
        private MethodHandle m; // 用于高性能反射调用的句柄
        private List<?> e;      // 存储 Mixin 扩展实例
    }

    /**
     * 当 Mixin 配置文件被加载时调用
     * 逻辑：初始化 Native 环境，加载底层 C++ 库
     */
    @Override
    public void onLoad(String mixinPackage) {
        // Native 逻辑可能在这里检查当前游戏版本、模组冲突，
        // 并根据结果初始化内部的静态变量。
    }

    /**
     * 决定一个 Mixin 是否应该应用到目标类上
     * @param targetClassName 目标类名 (如: net.minecraft.world.level.Level)
     * @param mixinClassName  Mixin 类名
     * @return 如果返回 false，该 Mixin 将被跳过
     */
    @Override
    public boolean shouldApplyMixin(String targetClassName, String mixinClassName) {
        // 核心逻辑：
        // 1. 检查 targetClassName 是否在黑名单中
        // 2. 检查前置模组是否存在（如：只有安装了 Botania 才会应用植物魔法相关的 Mixin）
        // 3. 校验 hash 值是否匹配
        return true; 
    }

    /**
     * 在 Mixin 应用前对字节码进行预处理
     */
    @Override
    public void preApply(String targetClassName, ClassNode targetClass, 
                         String mixinClassName, IMixinInfo mixinInfo) {
        // 在这里可以使用 ASM 直接修改 targetClass 的字节码
    }

    /**
     * 在 Mixin 应用后进行后期处理
     */
    @Override
    public void postApply(String targetClassName, ClassNode targetClass, 
                          String mixinClassName, IMixinInfo mixinInfo) {
        // 确认修改是否成功应用
    }

    // --- 以下为标准接口实现，通常返回默认值 ---

    @Override
    public String getRefMapperConfig() { return null; }

    @Override
    public List<String> getMixins() { return null; }

    @Override
    public void acceptTargets(Set<String> myTargets, Set<String> otherTargets) {
        // 接收目标类列表
    }
}
```
