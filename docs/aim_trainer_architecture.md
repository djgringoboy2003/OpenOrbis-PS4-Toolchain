# High-Level Architecture for PS4 Aim Trainer (OpenOrbis SDK)

This document outlines a C++-oriented architecture tailored for a low-latency first-person aim trainer on PlayStation 4 using the OpenOrbis SDK. The focus is on predictable input handling, fixed-step simulation, and efficient rendering integration.

## Input Handling (Low Latency)

### Goals
- Minimize latency between controller events and game state updates.
- Provide stable analog stick behavior via radial deadzones and configurable sensitivity curves.
- Keep processing deterministic by sampling input on a fixed timestep and buffering for rendering.

### Class Structure
- **`InputDevice` (interface):** Abstracts a physical controller; exposes raw axis/buttons state.
- **`DualshockController : public InputDevice`:** Polls DualShock 4 via OpenOrbis APIs, caches timestamped samples.
- **`StickFilter`**: Encapsulates deadzone and sensitivity curve computation.
- **`InputFrame`**: Immutable snapshot of processed inputs for a simulation tick.
- **`InputManager`**: Owns controllers, applies `StickFilter`, and produces `InputFrame` instances for the game loop.

### Key Data Types
```cpp
struct StickState {
    float x; // raw in [-1, 1]
    float y; // raw in [-1, 1]
};

struct StickSettings {
    float radialDeadzone = 0.12f;    // D_R
    float sensitivity = 1.0f;        // S
    float powerCurve = 1.0f;         // alpha; 1.0 means linear
};

struct InputFrame {
    StickState leftFiltered;
    StickState rightFiltered;
    uint64_t   sampleTimeUs; // high-resolution timestamp
};
```

### Radial Deadzone + Sensitivity (Pseudo-Code)
```cpp
StickState StickFilter::process(const StickState& raw, const StickSettings& cfg) const {
    // radial magnitude
    float mag = sqrtf(raw.x * raw.x + raw.y * raw.y);
    if (mag < cfg.radialDeadzone) {
        return {0.0f, 0.0f};
    }

    // re-normalize after removing deadzone
    float scale = (mag - cfg.radialDeadzone) / (1.0f - cfg.radialDeadzone);
    scale = powf(std::clamp(scale, 0.0f, 1.0f), cfg.powerCurve); // non-linear response
    scale *= cfg.sensitivity; // sensitivity multiplier

    float normX = raw.x / mag; // preserve direction
    float normY = raw.y / mag;
    return {normX * scale, normY * scale};
}
```

### Polling/Buffering Strategy
- **Fixed sampling**: Poll the controller once per simulation tick (e.g., 120 Hz) to align with physics.
- **Double-buffered frames**: Maintain `InputFrame current` and `InputFrame previous`; swap after each tick to avoid locking across threads.
- **Threading**: If using a dedicated input thread, timestamp samples and push to a lock-free ring buffer; the game loop pops the most recent sample per tick.
- **Latency checks**: Expose debug counters for input age (`now - sampleTimeUs`) to tune polling frequency and scheduling.

### InputManager Pseudo-Code
```cpp
class InputManager {
public:
    void initialize();
    void update(float fixedDt) {
        controller.poll(rawState); // reads from OpenOrbis input API
        processed.leftFiltered  = stickFilter.process(rawState.left, leftCfg);
        processed.rightFiltered = stickFilter.process(rawState.right, rightCfg);
        processed.sampleTimeUs  = clock.getMicroseconds();
        swapBuffers(processed);
    }

    const InputFrame& getFrame() const { return currentFrame; }

private:
    DualshockController controller;
    StickFilter stickFilter;
    StickSettings leftCfg, rightCfg;
    InputFrame currentFrame, previousFrame, processed;
};
```

## Core Game Loop & Engine Structure

### Principles
- **Fixed timestep simulation** (e.g., 120 FPS) for physics and deterministic gameplay.
- **Decoupled rendering** running as fast as possible, consuming interpolated simulation state to avoid visual jitter.
- **Minimal allocations** in the main loop; pre-allocate entity pools and render resources.

### High-Level Modules
- **`Game` (interface):** Owns systems and orchestrates initialization/shutdown.
- **`GameLoop`**: Manages fixed-step updates and rendering scheduling.
- **`Renderer`**: Records command buffers and submits to a low-level API (Gnm/Gnmx via OpenOrbis abstraction, or Vulkan/OpenGL for portability testing).
- **`PhysicsSystem`**: Fixed-step integrator for projectiles/targets; uses input frames from `InputManager`.
- **`ECS`**: Stores components (Transform, Renderable, TargetBehavior, Weapon, Lifetime); provides system iteration.

### Game Loop Pseudo-Code
```cpp
class GameLoop {
public:
    void run() {
        const float fixedDt = 1.0f / 120.0f;
        double accumulator = 0.0;
        double lastTime = clock.now();

        while (!quitRequested) {
            double now = clock.now();
            double frameTime = now - lastTime;
            lastTime = now;
            accumulator += frameTime;

            // process OS/SDK events here to keep the queue drained
            platformPumpEvents();

            while (accumulator >= fixedDt) {
                input.update(fixedDt);
                physics.integrate(fixedDt, input.getFrame());
                gameplay.update(fixedDt);
                accumulator -= fixedDt;
            }

            float alpha = static_cast<float>(accumulator / fixedDt);
            render(alpha); // interpolate between last two simulation states
        }
    }

    void render(float alpha) {
        renderer.beginFrame();
        renderer.uploadDynamicData(alpha); // interpolate transforms, muzzle flashes, UI
        renderer.recordCommands();
        renderer.submit();
    }
};
```

### Rendering Integration Points
- **Command buffers per frame**: Pre-allocate and recycle; avoid per-frame heap allocations.
- **Double/triple buffering**: Maintain CPU-GPU sync via fences; render thread should not block the simulation thread.
- **Frame graph**: Optional lightweight pass graph to organize G-buffer, lighting, and UI passes.
- **GPU submission**: Renderer owns a `RenderQueue` abstraction to batch draw calls for targets, weapons, and HUD with minimal state changes.

### Physics & Gameplay Modules
- **PhysicsSystem**: Simple Euler/Verlet integrator sufficient for targets/projectiles; deterministic at fixedDt.
- **Hit Detection**: Use bounding spheres/boxes; compute raycasts from camera to target per tick.
- **GameplaySystem**: Spawns targets, handles scoring/combo logic, and updates UI data consumed by the renderer.

### Entity Management Options
- **Component-Based (lightweight)**: Arrays/vectors of components keyed by entity ID for low memory overhead.
- **ECS Pattern (performance-focused)**:
  - **Entity**: 32-bit ID with generation counter.
  - **Component Stores**: Struct-of-arrays for cache-friendly iteration (e.g., `Transforms`, `Renderables`, `Targets`).
  - **Systems**: Functions operating on filtered views (e.g., `UpdateTargets`, `AnimateWeapons`, `RenderSubmission`).
  - **Lifecycle**: Free-list allocator for entity IDs; preallocate pools to avoid runtime malloc.

### Threading Suggestions
- **Job system**: Small task scheduler to parallelize rendering prep (culling, sorting) and gameplay tasks within the frame budget.
- **Affinity**: Pin input/physics to one core for consistent latency; allow renderer/streaming jobs on others.

### Debug/Profiling Hooks
- **Frame markers**: Insert GPU/CPU markers to track latency (input-to-photon).
- **Input graphs**: On-screen display of raw vs filtered stick magnitude for tuning deadzones and curves.
- **Telemetry**: Per-frame logs for accumulator, buffer age, and render queue depth.

## Summary Checklist (Low-Resource Friendly)
- Fixed 120 Hz simulation; render interpolated frames as fast as possible.
- Radial deadzone + power curve filtering on sticks; double-buffered `InputFrame` objects.
- Preallocated ECS pools; minimal per-frame allocations.
- Command buffers + render queue with aggressive state batching.
- Lightweight job system; avoid blocking sync between simulation and rendering.
