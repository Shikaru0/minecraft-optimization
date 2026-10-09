# minecraft-optimization

## JVM Flags

JVM launch flags for low-latency garbage collection and stable memory behavior.

| Flag | Purpose | Supported on |
| --- | --- | --- |
| `-Xms{memory}M` | Sets the minimum and the initial size (in bytes) of the heap. | HotSpot JVM |
| `-Xmx{memory}M` | Specifies the maximum size (in bytes) of the heap. For server deployments, `-Xms` and `-Xmx` are often set to the same value. | HotSpot JVM |
| `-XX:+UseZGC` | Enables the use of the Z garbage collector (ZGC). This is a low latency garbage collector, providing max pause times of a few milliseconds, at some throughput cost. Pause times are independent of what heap size is used. Supports heap sizes from 8MB to 16TB. | Production since JDK 15 (experimental since JDK 11) |
| `-XX:+UseStringDeduplication` | Enables string deduplication: identical `String` objects point to and share the same character array, reducing the memory footprint of `String` objects on the Java heap. | G1 (Oracle's man page requires it); ZGC since JDK 18 |
| `-XX:+UseCompactObjectHeaders` | Enables compact object headers, reducing the size of every object header in the Java heap from 96 or 128 bits to 64 bits and the heap footprint by 4 bytes per object (on average); often improves performance. Limited to 4 million loaded classes. | JDK 24 experimental (needs `-XX:+UnlockExperimentalVMOptions`); product since JDK 25 |
| `-XX:+AlwaysPreTouch` | Requests the VM to touch every page on the Java heap after requesting it from the operating system and before handing memory out to the application. | HotSpot JVM |
| `-XX:TrimNativeHeapInterval=5000` | Interval, in ms, at which the JVM will trim the native heap. Lower values will reclaim memory more eagerly at the cost of higher overhead. A value of 0 (default) disables native heap trimming. | Linux with glibc only |
| `-XX:+UseTransparentHugePages` | Linux only: enables the use of large pages that can dynamically grow or shrink. With ZGC this is the alternative to explicit large pages and must be combined with `-XX:+UseLargePages`. | Linux only; with ZGC needs kernel 4.7+ |

### Requirements

- **JDK 25+ recommended.** Only `-XX:+UseCompactObjectHeaders` needs it: experimental in JDK 24 (add `-XX:+UnlockExperimentalVMOptions`), product since JDK 25. Everything else here runs on JDK 21.
- **`-XX:+ZGenerational`.** Needed on JDK 21 to 23 to opt into generational ZGC. ZGC is generational by default since JDK 23 and the non-generational mode was removed in JDK 24, so drop the flag on JDK 24+ (JEP 439/474/490).
- **`-XX:-ZUncommit`.** Disables ZGC returning unused memory to the OS. This guide runs `-Xms` equal to `-Xmx`, so ZGC never shrinks below `-Xms` and uncommit cannot happen: the flag is unnecessary. Add it only if you set `-Xms` below `-Xmx` and want the memory kept resident.
- **Transparent huge pages.** The JVM also needs `-XX:+UseLargePages`. Check whether they are already enabled:

  ```bash
  cat /sys/kernel/mm/transparent_hugepage/enabled
  cat /sys/kernel/mm/transparent_hugepage/shmem_enabled
  ```

  ZGC needs `[madvise]` in the first and `[advise]` in the second. If they are not selected, enable them (they do not persist across reboots):

  ```bash
  echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
  echo advise  | sudo tee /sys/kernel/mm/transparent_hugepage/shmem_enabled
  ```

  With the common `always` + `shmem_enabled=never` default, every GC except ZGC still gets huge pages, but ZGC does not.

### Usage

Add the flags to the launch command:

```bash
java -Xms4096M -Xmx4096M \
  -XX:+UseZGC \
  -XX:+UseStringDeduplication \
  -XX:+UseCompactObjectHeaders \
  -XX:+AlwaysPreTouch \
  -XX:TrimNativeHeapInterval=5000 \
  -XX:+UseLargePages \
  -XX:+UseTransparentHugePages \
  -jar server.jar nogui
```

On JDK 24 add `-XX:+UnlockExperimentalVMOptions` if you keep `-XX:+UseCompactObjectHeaders`; from JDK 25 on it is a product flag.

### Memory trade-off

Setting `-Xms` equal to `-Xmx` disables ZGC uncommit (it never releases memory below `-Xms`), and with `AlwaysPreTouch` the whole heap is committed at startup: consistent latency for fixed RAM.

Raising `{memory}` above the default is recommended, and a larger heap has few downsides for ZGC beyond the extra resident RAM and a slightly slower first startup.

Oracle's tuning strategy: "Set the maximum heap size to a value that's close to the total physical memory on the platform, but doesn't cause swapping of the application."

Because the pre-touched heap plus the JVM's native overhead (roughly 1 GiB, outside `{memory}`) is resident from the first second, "close" still means leaving enough for the OS and that overhead. Tune by watching the used heap (`jcmd <pid> GC.heap_info`, or the GC logs).

## Mods

Ordered by side. **Both** means it works single-sided on either server or client side; **Server** and **Client** are single-side.

| Mod | Purpose | Side |
| --- | --- | --- |
| [C2ME](https://modrinth.com/mod/c2me-fabric) | A Fabric mod designed to improve the chunk performance of Minecraft. | Both |
| [FerriteCore](https://modrinth.com/mod/ferrite-core) | Memory usage optimizations | Both |
| [Krypton](https://modrinth.com/mod/krypton) | A mod to optimize the Minecraft networking stack | Both |
| [Lithium](https://modrinth.com/mod/lithium) | No-compromises game logic optimization mod, useful for both single-player games and multi-player servers. | Both |
| [ModernFix](https://modrinth.com/mod/modernfix) | All-in-one mod that improves performance, reduces memory usage, and fixes many bugs. Compatible with all your favorite performance mods! | Both |
| [ScalableLux](https://modrinth.com/mod/scalablelux) | A Fabric mod based on Starlight that improves the performance of light updates in Minecraft. | Both |
| [Alternate Current](https://modrinth.com/mod/alternate-current) | An efficient and non-locational redstone dust implementation | Server |
| [C2ME OpenCL Acceleration Module](https://modrinth.com/mod/c2me-ocl) | C2ME addon that provides hardware accelerated world generation through OpenCL | Server |
| [Clumps](https://modrinth.com/mod/clumps) | Clumps XP orbs together to reduce lag | Server |
| [Ksyxis](https://modrinth.com/mod/ksyxis) | Speed up your world loading by removing unneeded chunks. | Server |
| [ServerCore](https://modrinth.com/mod/servercore) | A mod that aims to optimize the minecraft server. | Server |
| [ImmediatelyFast](https://modrinth.com/mod/immediatelyfast) | Speed up immediate mode rendering in Minecraft | Client |
| [Sodium](https://modrinth.com/mod/sodium) | A high-performance rendering engine replacement for Minecraft, which greatly improves frame rates and reduces micro-stutter. | Client |

## Sources

- https://docs.oracle.com/en/java/javase/26/gctuning/other-considerations.html
- https://docs.oracle.com/en/java/javase/26/gctuning/ergonomics.html
- https://docs.oracle.com/en/java/javase/26/gctuning/z-garbage-collector1.html
- https://docs.oracle.com/en/java/javase/26/docs/specs/man/java.html
- https://wiki.openjdk.org/spaces/zgc/pages/34668579/Main
- https://openjdk.org/jeps/450
- https://openjdk.org/jeps/519
- https://openjdk.org/jeps/439
- https://openjdk.org/jeps/474
- https://openjdk.org/jeps/490
- https://modrinth.com
