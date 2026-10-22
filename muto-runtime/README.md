# Muto Runtime: Dynamic Bytecode Hot-Swapping & Classloader Isolation in Modded JVMs

Modding the JVM has historically been an exercise in patience. Anyone who has developed mods for modern Minecraft knows the routine: you tweak a render coordinate, fix an off-by-one bounding box check, hit compile, and then stare at a console log for two and a half minutes while Gradle re-packages the jar and the JVM spins up assets, sound banks, and world data.

`Muto Runtime` was born out of pure impatience. We set out to answer a simple question: **Why can't we load, unload, and update Fabric mod jars at runtime with zero JVM restarts?**

Standard JVM hot-swapping via Debugger APIs (JDWP) only allows mutating method bodies without changing class schema, adding fields, or touching method signatures. In the Minecraft modding ecosystem, things are ten times worse: mods mutate game bytecode dynamically using **Mixin**, inject code into core engine loops, and hook directly into LWJGL render passes. If you reload a class naively, the game crashes instantly with a `ClassCastException` or blows up Metaspace memory.

Here is the engineering post-mortem of how we built `Muto Runtime`, defeated classloader leakage, and achieved sub-200ms hot-reloads live in game.

## The Architecture of the Mixin Deadlock

To understand why hot-swapping mods is notoriously difficult, you have to look at how SpongePowered's Mixin operates. Mixin transforms vanilla Minecraft classes on the fly during classloading:

```mermaid
graph LR
  A["Mod Jar Bytecode"] --> B["Fabric Loader KnotClassLoader"]
  B --> C["Mixin Transformer Pipeline"]
  C --> D["Mutated Minecraft Runtime Class"]
  D --> E["JVM Metaspace Execution"]
```

When a mod injector applies an `@Inject` or `@Overwrite` to `net.minecraft.client.render.GameRenderer`, the underlying bytecode in memory is permanently altered. If you attempt to reload the mod class:

1. The new class definition is rejected by the bootstrapper because the old class is already pinned in the KnotClassLoader hierarchy.
2. Even if you redefine the class via instrumentation, any static references or event handler delegates stored in vanilla singletons still point to the old class instances.
3. Every attempted reload leaks several megabytes of JNI handles and Metaspace memory, guaranteeing an `OutOfMemoryError: Metaspace` within 10 reloads.

## Ephemeral Child-First Classloader Topology

The core breakthrough in Muto Runtime is **Child-First Ephemeral Classloader Isolation**. Instead of loading the mod directly through the global Fabric `KnotClassLoader`, Muto wraps every hot-swappable module inside an isolated `MutoModuleClassLoader` that intercepts load requests before delegating upwards.

```java
public class MutoModuleClassLoader extends URLClassLoader {
    private final ClassLoader parent;
    private final Set<String> isolatedPackages;

    public MutoModuleClassLoader(URL[] urls, ClassLoader parent, Set<String> isolated) {
        super(urls, null); // orphan parent delegation
        this.parent = parent;
        this.isolatedPackages = isolated;
    }

    @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        // Child-first priority for mod classes
        if (isIsolated(name)) {
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                c = findClass(name);
            }
            if (resolve) resolveClass(c);
            return c;
        }
        // Fallback to parent for engine and engine symbols
        return parent.loadClass(name);
    }
}
```

When Muto detects a file change on disk or receives a hot-reload trigger:

1. **State Snapshot**: Muto notifies the mod via a `@PreReload` hook to serialize its active memory state into an in-memory byte buffer.
2. **Unbinding Event Buses**: All listeners registered by the old classloader in Fabric's event manifolds are unhooked.
3. **Classloader Severing**: The reference to the old `MutoModuleClassLoader` is severed, dropping all associated classes, reflection caches, and static members into the garbage collector.
4. **Instant Spin-up**: A fresh `MutoModuleClassLoader` is spun up, the newly compiled jar is read from memory, and the state snapshot is re-injected via `@PostReload`.

## Benchmark: Development Iteration Velocity

We benchmarked iteration latency across three distinct development scenarios on a standard Fabric 1.21.1 setup:

| Workflow Pipeline | Cold Restart Time | Hot Reload Latency | Metaspace Leaked per Iteration |
| :--- | :--- | :--- | :--- |
| **Traditional Gradle Launch** | 148.2 seconds | N/A (Restart Required) | 0 KB |
| **Standard JDWP HotSwap** | N/A | Fails on Schema/Mixin change | 4.8 MB |
| **Muto Runtime Hot-Swap** | **0.0s (In-Game)** | **184 milliseconds** | **< 12 KB (Zero-Leak)** |

> "The difference between waiting 2 minutes per UI tweak versus seeing your changes update instantaneously in 180 milliseconds turns modding from a chore into flow state."
> — fy2ne, Muto Lead Architect

## Verifying Memory Reclamation

To guarantee that classes actually get collected from Metaspace and do not cause JVM exhaustion over hours of rapid iteration, Muto runs a memory reaper using `PhantomReference` monitors. 

If a reference to an old class survives past 2 minor garbage collection cycles, Muto dumps the root reference path directly to console, alerting developers immediately if a third-party library is holding onto a stale static instance.

The runtime engine is open-source and active on [GitHub: fy2ne/Muto-Runtime](https://github.com/fy2ne/Muto-Runtime).
