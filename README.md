# gtolibdump
模组限制加载方式


```package com.gtolib;

import com.gtolib.api.annotation.DataGeneratorScanned;
import com.gtolib.api.misc.AbstractMixinConfigPlugin;
import com.gtolib.utils.reflect.MethodReference;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.List;

/**
 * Mixin 配置插件，用于在加载时初始化 GTO 原生组件，
 * 并控制 Mixin 的注入行为。
 */
@DataGeneratorScanned
public final class MixinConfigPlugin extends AbstractMixinConfigPlugin {

    static final Logger LOGGER = LoggerFactory.getLogger("GTO Core");
    public static String hash = null;

    /**
     * 插件加载时执行：加载 GTOProvider 并调用其初始化方法，
     * 然后执行原生初始化逻辑。
     *
     * @param string 配置参数（未使用）
     * @throws RuntimeException 若任何步骤失败则包装抛出
     */
    @Override
    public void onLoad(String string) {
        try {
            ClassLoader classLoader = this.getClass().getClassLoader();
            // 通过反射调用 gto.native0.plugins.GTOProvider#init_gtolib(ClassLoader)
            MethodReference.fromClass(
                    classLoader.loadClass("gto.native0.plugins.GTOProvider"),
                    "init_gtolib",
                    ClassLoader.class
            ).invoke(classLoader);

            // 执行原生辅助类的初始化
            NativeInitializer.initialize();
        } catch (Throwable throwable) {
            throw new RuntimeException(throwable);
        }
    }

    /**
     * 返回需要应用的 Mixin 列表。
     * 当前实现返回 null，但会先调用原生辅助类的准备方法（可能产生副作用）。
     *
     * @return null
     * @throws RuntimeException 若准备过程中发生错误
     */
    @Override
    public List<String> getMixins() {
        try {
            NativeInitializer.prepareMixins();
        } catch (Throwable throwable) {
            throw new RuntimeException(throwable);
        }
        return null;
    }

    /**
     * 决定是否应用指定的 Mixin 到目标类。
     * 当前实现始终允许应用。
     *
     * @param targetClassName  目标类名
     * @param mixinClassName   Mixin 类名
     * @return true
     */
    @Override
    public boolean shouldApplyMixin(String targetClassName, String mixinClassName) {
        return true;
    }
}

```
