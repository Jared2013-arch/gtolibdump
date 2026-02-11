# gtolibdump
模组限制加载方式


```package com.gtolib;

import com.google.common.collect.ImmutableList;
import com.gtolib.api.annotation.DataGeneratorScanned;
import com.gtolib.api.misc.AbstractMixinConfigPlugin;
import com.gtolib.utils.reflect.FieldReference;
import com.gtolib.utils.reflect.MethodReference;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.Map;
import net.minecraftforge.fml.loading.FMLLoader;
import org.apache.logging.log4j.Level;
import org.apache.logging.log4j.core.config.Configurator;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.spongepowered.asm.mixin.MixinEnvironment;
import org.spongepowered.asm.mixin.transformer.IMixinTransformer;
import org.spongepowered.asm.mixin.transformer.ext.Extensions;
import org.spongepowered.asm.mixin.transformer.ext.IExtension;
import org.spongepowered.asm.mixin.transformer.ext.IExtensionRegistry;

@DataGeneratorScanned
public final class MixinConfigPlugin extends AbstractMixinConfigPlugin {

   private static final Logger LOGGER = LoggerFactory.getLogger("GTO Core");
   public static String hash = null;

   @Override
   public void onLoad(String mixinPackage) {
      try {
         ClassLoader classLoader = this.getClass().getClassLoader();
         
         MethodReference.fromClass(classLoader.loadClass("gto.native0.plugins.GTOProvider"),
                         "init_gtolib", ClassLoader.class)
                 .invoke(classLoader);

        
         if (FMLLoader.isProduction()) {
            Configurator.setRootLevel(Level.ERROR);
         }

         
         IMixinTransformer transformer = (IMixinTransformer) MixinEnvironment.getDefaultEnvironment()
                 .getActiveTransformer();
         Extensions extensions = (Extensions) transformer.getExtensions();
         Object CustomExtensionProvider;
         extensions.add(CustomExtensionProvider.CUSTOM_EXTENSION);
      } catch (Throwable t) {
         throw new RuntimeException("Failed to load MixinConfigPlugin", t);
      }
   }

   @Override
   public List<String> getMixins() {
      try {
         removeCustomExtensions();
         return null; 
      } catch (Throwable t) {
         throw new RuntimeException("Failed to get mixins", t);
      }
   }

   @Override
   public boolean shouldApplyMixin(String targetClassName, String mixinClassName) {
      return true;
   }
   
   private static void removeCustomExtensions() {
      IMixinTransformer transformer = (IMixinTransformer) MixinEnvironment.getDefaultEnvironment()
              .getActiveTransformer();
      IExtensionRegistry registry = transformer.getExtensions();

      
      FieldReference extensionMapField = FieldReference.fromInstance(registry, "extensionMap");
      Map<?, ?> extensionMap = (Map<?, ?>) extensionMapField.get();
      extensionMap.entrySet().removeIf(entry ->
              CustomExtensionProvider.IS_CUSTOM.test((IExtension) entry.getValue())
      );

     
      FieldReference extensionsField = FieldReference.fromInstance(registry, "extensions");
      List<IExtension> extensions = (List<IExtension>) extensionsField.get();
      extensions.removeIf(CustomExtensionProvider.IS_CUSTOM);

      
      FieldReference activeExtensionsField = FieldReference.fromInstance(registry, "activeExtensions");
      Collection<IExtension> activeExtensions = new ArrayList<>((Collection<IExtension>) activeExtensionsField.get());
      activeExtensions.removeIf(CustomExtensionProvider.IS_CUSTOM);
      activeExtensionsField.set(ImmutableList.copyOf(activeExtensions));
   }
}

```
