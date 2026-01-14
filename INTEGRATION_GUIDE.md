# Integration Guide - Weak Hardware Booster

**Target Engines**: Unity, Unreal Engine, Cocos Creator  
**Language**: Rust with C FFI (Foreign Function Interface)  
**Platforms**: Android (ARM64), iOS (ARM64)

---

## Overview

Weak Hardware Booster integrates with game engines through **native plugins** using FFI. The Rust runtime is compiled into a shared library (`.so` for Android, `.dylib` for iOS) that your game code calls via C-compatible functions.

---

## 1. Unity Integration (C#)

### 1.1 Build Native Plugin

```bash
# For Android ARM64
cargo build --release --target aarch64-linux-android

# For iOS ARM64
cargo build --release --target aarch64-apple-ios

# Output: libweak_hardware_booster.so (Android) or libweak_hardware_booster.dylib (iOS)
```

### 1.2 Unity C# Wrapper

Create `Assets/Scripts/WeakHardwareBooster.cs`:

```csharp
using System;
using System.Runtime.InteropServices;
using UnityEngine;

namespace NexusStudio
{
    [StructLayout(LayoutKind.Sequential)]
    public struct Particle
    {
        [MarshalAs(UnmanagedType.ByValArray, SizeConst = 3)]
        public float[] pos;
        
        [MarshalAs(UnmanagedType.ByValArray, SizeConst = 3)]
        public float[] velocity;
    }

    public static class WeakHardwareBooster
    {
        #if UNITY_ANDROID
        private const string DLL_NAME = "weak_hardware_booster";
        #elif UNITY_IOS
        private const string DLL_NAME = "__Internal";
        #endif

        [DllImport(DLL_NAME)]
        private static extern float accd_process_particles_ffi(
            IntPtr particles, 
            int count, 
            int clusterSize
        );

        /// <summary>
        /// Processes particles using ACCD-OS convergence algorithm.
        /// </summary>
        /// <param name="particles">Array of particles to process</param>
        /// <param name="clusterSize">Convergence cluster size (default: 250)</param>
        /// <returns>Processing latency in milliseconds</returns>
        public static float ProcessParticles(Particle[] particles, int clusterSize = 250)
        {
            if (particles == null || particles.Length == 0)
            {
                Debug.LogWarning("WeakHardwareBooster: Empty particle array");
                return 0f;
            }

            int size = Marshal.SizeOf(typeof(Particle)) * particles.Length;
            IntPtr ptr = Marshal.AllocHGlobal(size);

            try
            {
                // Copy managed array to unmanaged memory
                IntPtr current = ptr;
                foreach (var particle in particles)
                {
                    Marshal.StructureToPtr(particle, current, false);
                    current = IntPtr.Add(current, Marshal.SizeOf(typeof(Particle)));
                }

                // Call native Rust function
                float latency = accd_process_particles_ffi(ptr, particles.Length, clusterSize);
                return latency;
            }
            finally
            {
                Marshal.FreeHGlobal(ptr);
            }
        }
    }
}
```

### 1.3 Unity Usage Example

```csharp
using UnityEngine;
using NexusStudio;

public class ParticleSystemOptimizer : MonoBehaviour
{
    private Particle[] particles;

    void Start()
    {
        // Initialize 10,000 particles
        particles = new Particle[10000];
        for (int i = 0; i < particles.Length; i++)
        {
            particles[i] = new Particle
            {
                pos = new float[] { i * 0.1f, 0f, 0f },
                velocity = new float[] { 1f, 0f, 0f }
            };
        }
    }

    void Update()
    {
        // Process particles every frame
        float latency = WeakHardwareBooster.ProcessParticles(particles, 250);
        
        if (latency > 1f)
        {
            Debug.LogWarning($"High latency detected: {latency}ms");
        }
    }
}
```

### 1.4 Plugin Placement

```
Assets/
  Plugins/
    Android/
      arm64-v8a/
        libweak_hardware_booster.so
    iOS/
      libweak_hardware_booster.dylib
```

---

## 2. Unreal Engine Integration (C++)

### 2.1 Build Native Plugin

Same as Unity (section 1.1).

### 2.2 Unreal C++ Wrapper

Create `Source/YourProject/WeakHardwareBooster.h`:

```cpp
#pragma once

#include "CoreMinimal.h"

// C-compatible structure
struct FParticle
{
    float Pos[3];
    float Velocity[3];
};

// FFI import
extern "C"
{
    __declspec(dllimport) float accd_process_particles_ffi(
        const FParticle* particles,
        size_t count,
        size_t clusterSize
    );
}

// Unreal-friendly wrapper
class YOURPROJECT_API FWeakHardwareBooster
{
public:
    static float ProcessParticles(TArray<FParticle>& Particles, int32 ClusterSize = 250)
    {
        if (Particles.Num() == 0) return 0.0f;
        
        return accd_process_particles_ffi(
            Particles.GetData(), 
            Particles.Num(), 
            ClusterSize
        );
    }
};
```

### 2.3 Unreal Usage Example

```cpp
#include "WeakHardwareBooster.h"

void AMyGameMode::BeginPlay()
{
    Super::BeginPlay();

    // Initialize particles
    TArray<FParticle> Particles;
    Particles.Reserve(10000);
    
    for (int32 i = 0; i < 10000; i++)
    {
        FParticle P;
        P.Pos[0] = i * 0.1f;
        P.Pos[1] = 0.0f;
        P.Pos[2] = 0.0f;
        P.Velocity[0] = 1.0f;
        P.Velocity[1] = 0.0f;
        P.Velocity[2] = 0.0f;
        Particles.Add(P);
    }

    // Process particles
    float Latency = FWeakHardwareBooster::ProcessParticles(Particles);
    UE_LOG(LogTemp, Warning, TEXT("ACCD-OS Latency: %.4f ms"), Latency);
}
```

### 2.4 Library Linking

In `YourProject.Build.cs`:

```csharp
public class YourProject : ModuleRules
{
    public YourProject(ReadOnlyTargetRules Target) : base(Target)
    {
        // ... existing code ...

        if (Target.Platform == UnrealTargetPlatform.Android)
        {
            PublicAdditionalLibraries.Add(
                Path.Combine(ModuleDirectory, "ThirdParty", "Android", "libweak_hardware_booster.so")
            );
        }
        else if (Target.Platform == UnrealTargetPlatform.IOS)
        {
            PublicAdditionalLibraries.Add(
                Path.Combine(ModuleDirectory, "ThirdParty", "iOS", "libweak_hardware_booster.dylib")
            );
        }
    }
}
```

---

## 3. Cocos Creator Integration (JavaScript/TypeScript)

### 3.1 Build Native Plugin

Same as Unity (section 1.1).

### 3.2 JSB Binding

Create `native/weak_hardware_booster_jsb.cpp`:

```cpp
#include "cocos/bindings/jswrapper/SeApi.h"
#include "cocos/bindings/manual/jsb_conversions.h"

extern "C" {
    float accd_process_particles_ffi(const void* particles, size_t count, size_t clusterSize);
}

static bool js_accd_process_particles(se::State& s)
{
    const auto& args = s.args();
    if (args.size() != 3) return false;

    // Extract arguments
    se::Object* particlesObj = args[0].toObject();
    int count = args[1].toInt32();
    int clusterSize = args[2].toInt32();

    // Call native function
    float latency = accd_process_particles_ffi(
        particlesObj->getPrivateData(), 
        count, 
        clusterSize
    );

    s.rval().setFloat(latency);
    return true;
}

SE_BIND_FUNC(js_accd_process_particles);
```

### 3.3 TypeScript Usage

```typescript
// @ts-ignore
import { accd_process_particles } from 'native';

interface Particle {
    pos: [number, number, number];
    velocity: [number, number, number];
}

class ParticleSystemOptimizer {
    private particles: Particle[] = [];

    constructor() {
        // Initialize 10,000 particles
        for (let i = 0; i < 10000; i++) {
            this.particles.push({
                pos: [i * 0.1, 0, 0],
                velocity: [1, 0, 0]
            });
        }
    }

    update() {
        const latency = accd_process_particles(this.particles, this.particles.length, 250);
        if (latency > 1.0) {
            console.warn(`High latency: ${latency}ms`);
        }
    }
}
```

---

## 4. Performance Tuning

### 4.1 Cluster Size Selection

| Particle Count | Recommended Cluster Size | Rationale |
|----------------|--------------------------|-----------|
| <10,000 | 500 | Sparse scenes, minimize overhead |
| 10,000-100,000 | 250 | Balanced (default) |
| 100,000-500,000 | 150 | Dense scenes, finer granularity |
| >500,000 | 100 | Very dense, maximum precision |

**Rule of Thumb**: `cluster_size ≈ sqrt(particle_count)`

### 4.2 Frame Budget Optimization

```csharp
// Unity example: adaptive cluster sizing
public class AdaptiveOptimizer : MonoBehaviour
{
    private const float TARGET_LATENCY_MS = 0.5f; // 0.5ms target
    private int currentClusterSize = 250;

    void Update()
    {
        float latency = WeakHardwareBooster.ProcessParticles(particles, currentClusterSize);

        // Adjust cluster size dynamically
        if (latency > TARGET_LATENCY_MS)
        {
            currentClusterSize += 50; // Increase cluster size (less precision, faster)
        }
        else if (latency < TARGET_LATENCY_MS * 0.5f)
        {
            currentClusterSize = Mathf.Max(50, currentClusterSize - 50); // Decrease (more precision)
        }
    }
}
```

---

## 5. Troubleshooting

### 5.1 Android Build Errors

**Problem**: `UnsatisfiedLinkError: couldn't find libweak_hardware_booster.so`

**Solution**:
```bash
# Verify library is built for correct architecture
adb shell getprop ro.product.cpu.abi
# Should output: arm64-v8a

# Rebuild for ARM64
cargo build --release --target aarch64-linux-android
```

### 5.2 iOS Build Errors

**Problem**: `dyld: Library not loaded`

**Solution**: Ensure library is embedded in Xcode project:
1. Xcode → Build Phases → Link Binary With Libraries
2. Add `libweak_hardware_booster.dylib`
3. Set "Embed" to "Embed & Sign"

### 5.3 Performance Not Meeting Benchmarks

**Checklist**:
- [ ] Compiled with `--release` (not debug)
- [ ] `opt-level=3` in Cargo.toml
- [ ] Running on ARM64 (not x86 emulator)
- [ ] Cluster size appropriate for particle count
- [ ] No unnecessary heap allocations in hot loop

---

## 6. Advanced Integration

### 6.1 GPU Offload (Future)

**Vulkan Compute Shader** (planned Q2 2026):
```glsl
#version 450

layout(local_size_x = 256) in;

layout(binding = 0) buffer ParticleBuffer {
    vec4 positions[];
    vec4 velocities[];
};

void main() {
    uint id = gl_GlobalInvocationID.x;
    // ACCD-OS clustering on GPU
    uint cluster_id = compute_cluster(positions[id].xyz);
    // ...
}
```

### 6.2 Multi-Threading

**Rust Rayon** (parallel processing):
```rust
use rayon::prelude::*;

particles.par_chunks(cluster_size)
    .for_each(|cluster| {
        process_cluster(cluster);
    });
```

---

## 7. Support & Contact

**Technical Issues**:
- Email: nexusstudio100@gmail.com
- GitHub Issues: https://github.com/Tryboy869/weak-hardware-booster/issues

**Commercial Licensing**:
- Email: nexusstudio100@gmail.com
- NDA Available: See [NDA_TEMPLATE.md](../NDA_TEMPLATE.md)

**Documentation**:
- [Technical Whitepaper](WHITEPAPER.md)
- [Benchmark Details](BENCHMARKS.md)
- [Market Analysis](MARKET_ANALYSIS.md)

---

**Copyright © 2026 Nexus Studio. All rights reserved.**  
**Author**: Daouda Abdoul Anzize