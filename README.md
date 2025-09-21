# gtolibdump
模组限制加载方式


```/com/gtolib/MixinConfigPlugin.java
@Override
public void onLoad(String mixinPackage) {
    // 1. 注册 Mixin 取消器：阻止黑名单中的 Mixin 加载
    MixinCancellerRegistrar.register((list, name) -> s.contains(name));

    IMixinTransformer transformer;
    Object extensions;

    // 仅在客户端执行以下完整性检测逻辑
    if (FMLEnvironment.dist.isClient() &&
        (transformer = MixinEnvironment.getDefaultEnvironment().getActiveTransformer()) instanceof IMixinTransformer &&
        (extensions = transformer.getExtensions()) instanceof Extensions) {

        Extensions ext = (Extensions) extensions;

        // 创建自定义扩展：用于 preApply 时进行篡改检测
        IExtension customExtension = new IExtension() {
            @Override
            public boolean checkActive(MixinEnvironment environment) {
                return true;
            }

            @Override
            public void preApply(ITargetClassContext context) {
                try {
                    ClassInfo classInfo = context.getClassInfo();

                    // 接口不检查
                    if (classInfo.isInterface()) return;

                    String className = classInfo.getClassName();

                    // 获取当前类被应用的所有 Mixin（通过反射调用 Embeddium 的私有方法）
                    if (m == null) {
                        // 查找并缓存 MethodHandle
                        Field field = MixinTaintDetector.class.getDeclaredField("GET_MIXINS_ON_CLASS_INFO");
                        field.setAccessible(true);
                        m = (MethodHandle) field.get(null);
                    }

                    // 情况 1: GregTech 类（com.gregtechceu）
                    if (className.startsWith("com.gregtechceu")) {
                        @SuppressWarnings("unchecked")
                        List<IMixinInfo> mixins = (List<IMixinInfo>) m.invokeExact(classInfo);

                        for (IMixinInfo mixin : mixins) {
                            String mixinClassName = mixin.getConfig().getName();

                            // 允许来自 GTOLib / GTOCore / AllTheLeaks 的 Mixin
                            if (mixinClassName.startsWith("com.gtolib.mixin.") ||
                                mixinClassName.startsWith("com.gtocore.mixin.") ||
                                mixinClassName.startsWith("dev.uncandango.alltheleaks.mixin.core.main.")) {
                                continue;
                            }

                            // 否则视为“污染”，抛出异常
                            throw new RuntimeException("Detected taint in " + className + "!");
                        }
                    }
                    // 情况 2: 禁止任何对 GTOCore 或 GTOLib 自身的 Mixin
                    else if (className.startsWith("com.gtocore") || className.startsWith("com.gtolib")) {
                        throw new RuntimeException("Detected taint in " + className + "!");
                    }
                    // 情况 3: 对 AE2 类限制最大 Mixin 数量
                    else if (className.startsWith("appeng")) {
                        int mixinCount = m.invokeExact(classInfo).size();
                        int maxAllowed = a.getOrDefault(className, 1); // 默认最多 1 个

                        if (mixinCount > maxAllowed) {
                            throw new RuntimeException("Detected taint in " + className + "!");
                        }
                    }
                } catch (Throwable t) {
                    throw new RuntimeException(t);
                }
            }

            @Override public void postApply(ITargetClassContext context) { }
            @Override public void export(MixinEnvironment env, String name, boolean forceExport, ClassNode node) { }
        };

        // 使用反射将自定义 Extension 添加到 Mixin 扩展链中
        try {
            Field extensionsField = ext.getClass().getDeclaredField("extensions");
            extensionsField.setAccessible(true);
            e = (List<IExtension>) extensionsField.get(ext);

            // 移除 Embeddium 自带的 Taint Detector（避免重复报错）
            e.removeIf(iExt -> iExt instanceof MixinTaintDetector);

            // 添加我们自己的检测器
            e.add(customExtension);

            // 更新 activeExtensions 列表（同样需要移除原 detector）
            Field activeField = ext.getClass().getDeclaredField("activeExtensions");
            activeField.setAccessible(true);

            ObjectArrayList newList = new ObjectArrayList(e);
            activeField.set(ext, Collections.unmodifiableList(newList));
        } catch (ReflectiveOperationException | RuntimeException ex) {
            throw new RuntimeException(ex);
        }
    }

    // 最后：关闭所有非错误级日志（减少启动噪音）
    Configurator.setRootLevel(Level.ERROR);
}
```
