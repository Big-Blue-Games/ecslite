# LeoEcsLite - Lightweight C# Entity Component System framework
Performance, zero or minimal allocations, minimizing memory usage, no dependencies on any game engine are the main goals of this framework.

> **IMPORTANT!** Don't forget to use `DEBUG` development builds and `RELEASE` release builds: all internal checks/exceptions will only work in `DEBUG` versions and have been removed to improve performance in `RELEASE `-versions.

> **IMPORTANT!** The LeoEcsLite framework is **not thread-safe** and never will be! If you need multithreading, you must implement it yourself and integrate synchronization in the form of an ecs system.

# Content
* [Social Resources](#Social Resources)
* [Installation](#Installation)
* [as a unity module](#as a unity module)
* [As source](# As-source)
* [Basic types](#Basic-types)
* [Entity](#Entity)
* [Component](#Component)
* [System](#System)
* [Data Sharing](#Data Sharing)
* [Special types](#Special-types)
* [EcsPool](#EcsPool)
* [EcsFilter](#EcsFilter)
* [EcsWorld](#EcsWorld)
* [EcsSystems](#EcsSystems)
* [Integration with engines](#Integration-with-engines)
* [Unity](#Unity)
* [Custom-engine](#Custom-engine)
* [Articles](#Articles)
* [Projects using LeoECS Lite](#Projects using LeoECS-Lite)
* [With sources](#With sources)
* [Extensions](#Extensions)
* [License](#License)
* [FAQ](#FAQ)

# Social resources
[![discord](https://img.shields.io/discord/404358247621853185.svg?label=enter%20to%20discord%20server&style=for-the-badge&logo=discord)](https://discord.gg/ 5GZVde6)

# Installation

## As a unity module
Installation as a unity module via a git link in the PackageManager or direct editing of `Packages/manifest.json` is supported:
```
"com.leopotam.ecslite": "https://github.com/Leopotam/ecslite.git",
```
By default, the latest release version is used. If you need a version "in development" with the latest changes, you should switch to the `develop` branch:
```
"com.leopotam.ecslite": "https://github.com/Leopotam/ecslite.git#develop",
```

## As source
The code can also be cloned or obtained as an archive from the releases page.

# Basic types

## Entity
By itself, it does not mean anything and does not exist, it is only a container for components. Implemented as `int`:
```c#
// Create a new entity in the world.
int entity = _world.NewEntity();

// Any entity can be deleted, first all components will be automatically deleted and only then the entity will be considered destroyed.
world.DelEntity(entity);
```

> **IMPORTANT!** Entities cannot exist without components and will be automatically destroyed when the last component on them is removed.

## Component
It is a container for user data and should not contain logic (minimal helpers are allowed, but not pieces of the main logic):
```c#
struct Component1 {
public int Id;
public stringName;
}
```
Components can be added, requested or removed via [component pools](#ecspool).

## System
It is a container for the main logic for processing filtered entities. Exists as a custom class that implements at least one of the `IEcsInitSystem`, `IEcsDestroySystem`, `IEcsRunSystem` (and other supported) interfaces:
```c#
class UserSystem : IEcsPreInitSystem, IEcsInitSystem, IEcsRunSystem, IEcsDestroySystem, IEcsPostDestroySystem {
public void PreInit(IEcsSystems systems) {
// Will be called once when IEcsSystems.Init() runs and before IEcsInitSystem.Init() runs.
}
    
public void Init(IEcsSystems systems) {
// Will be called once when IEcsSystems.Init() runs and after IEcsPreInitSystem.PreInit() runs.
}
    
public void Run(IEcsSystems systems) {
// Will be called once per IEcsSystems.Run() run.
}

public void Destroy(IEcsSystems systems) {
// Will be called once when IEcsSystems.Destroy() runs and before IEcsPostDestroySystem.PostDestroy() runs.
}
    
public void PostDestroy(IEcsSystems systems) {
// Will be called once when IEcsSystems.Destroy() runs and after IEcsDestroySystem.Destroy() runs.
}
}
```

# Sharing data
An instance of any custom type (class) can be simultaneously connected to all systems:
```c#
class SharedData {
public string PrefabsPath;
}
...
SharedData sharedData = new SharedData { PrefabsPath = "Items/{0}" };
IEcsSystems systems = new EcsSystems(world, sharedData);
systems
.Add(new TestSystem1())
.init();
...
class TestSystem1 : IEcsInitSystem {
public void Init(IEcsSystems systems) {
SharedData shared = systems.GetShared<SharedData>();
string prefabPath = string.Format(shared.PrefabsPath, 123);
// prefabPath = "Items/123" at this point.
}
}
```

# Special types

## EcsPool
It is a container for components, provides an api for adding / requesting / removing components on an entity:
```c#
int entity = world.NewEntity();
EcsPool<Component1> pool = world.GetPool<Component1>();

// Add() adds a bean to an entity. If the component already exists, an exception will be thrown in the DEBUG version.
ref Component1 c1 = ref pool.Add(entity);

// Has() checks for the presence of a bean on an entity.
bool c1Exists = pool.Has(entity);

// Get() returns the component that exists on the entity. If the component does not exist, an exception will be thrown in the DEBUG version.
ref Component1 c1 = ref pool.Get(entity);

// Del() removes the bean from the entity. If there was no component, no errors will be generated. If it was the last component, the entity will be deleted automatically.
pool.del(entity);
```

> **IMPORTANT!** Once removed, the component will be pooled for later reuse. All component fields will be reset to default values automatically.

##EcsFilter
It is a container for storing filtered entities by the presence or absence of certain components:
```c#
class WeaponSystem : IEcsInitSystem, IEcsRunSystem {
public void Init(IEcsSystems systems) {
// Get the default world instance.
EcsWorld world = systems.GetWorld();
        
// Create a new entity.
int entity = world.NewEntity();
        
// And add the "Weapon" component to it.
varweapons = world.GetPool<Weapon>();
weapons.Add(entity);
}

public void Run(IEcsSystems systems) {
EcsWorld world = systems.GetWorld();
// We want to get all entities with a "Weapon" component and without a "Health" component.
// The filter can be built dynamically every time, or it can be cached somewhere.
var filter = world.Filter<Weapon> ().Exc<Health> ().End ();
        
// The filter stores only entities, the data itself is in the "Weapon" component pool.
// The pool can also be cached somewhere.
varweapons = world.GetPool<Weapon>();
        
foreach (int entity in filter) {
ref Weapon weapon = ref weapons.Get(entity);
weapon.Ammo = System.Math.Max(0, weapon.Ammo - 1);
}
}
}
```

Additional requirements for filtered entities can be added via the `Inc<>()` / `Exc<>()` methods.

> **IMPORTANT!** Filters support any number of component requirements, but the same component cannot be in the "include" and "exclude" lists.

## EcsWorld
It is a container for all entities, pools and filters components, the data of each instance is unique and isolated from other worlds.

> **IMPORTANT!** You must call `EcsWorld.Destroy()` on the world instance when it is no longer needed.

## EcsSystems
It is a container for systems that will handle the `EcsWorld` world instance:
```c#
class Startup : MonoBehavior {
EcsWorld _world;
IEcsSystems_systems;

void Start() {
// Create environment, connect systems.
_world = new EcsWorld();
_systems = new EcsSystems(_world);
_systems
.Add(new WeaponSystem())
.init();
}
    
void Update() {
// Execute all connected systems.
_systems?.run();
}

void OnDestroy() {
// Destroy connected systems.
if (_systems != null) {
_systems.Destroy();
_systems = null;
}
// Clear the environment.
if (_world != null) {
_world.Destroy();
_world = null;
}
}
}
```

> **IMPORTANT!** You must call `IEcsSystems.Destroy()` on a system group instance if it is no longer needed.

# Integration with engines

## Unity
> Tested on Unity 2020.3 (does not depend on it) and contains asmdef descriptions for compiling as separate assemblies and reducing the recompilation time of the main project.

[Unity editor integration](https://github.com/Leopotam/ecslite-unityeditor) contains code templates and also provides world state monitoring.

## Custom engine
> C# 7.3 or higher is required to use the framework.

Each part of the example below must be correctly integrated into the correct place where the code is executed by the engine:
```c#
using Leopotam.EcsLite;

class EcsStartup {
EcsWorld _world;
IEcsSystems_systems;

// Initialize the environment.
void Init() {
_world = new EcsWorld();
_systems = new EcsSystems(_world);
_systems
// Additional instances of worlds
// must be registered here.
// .AddWorld(customWorldInstance, "events")
            
// Systems with core logic should
// be registered here.
// .Add(new TestSystem1())
// .Add(new TestSystem2())
            
.init();
}

// Method must be called from
// the main update-cycle of the engine.
void UpdateLoop() {
_systems?.run();
}

// Clearing the environment.
void Destroy() {
if (_systems != null) {
_systems.Destroy();
_systems = null;
}
if (_world != null) {
_world.destroy();
_world = null;
}
}
}
```

# Articles

* ["Creating a dungeon crawler with LeoECS Lite. Part 1"](https://habr.com/en/post/661085/)
[![](https://habrastorage.org/r/w1560/getpro/habr/upload_files/372/b1c/ad3/372b1cad308788dac56f8db1ea16b9c9.png)](https://habr.com/ru/post/661085/)
* ["Creating a dungeon crawler with LeoECS Lite. Part 2"](https://habr.com/en/post/673926/)
[![](https://habrastorage.org/r/w1560/getpro/habr/upload_files/63f/3ef/c47/63f3efc473664fdaaf1a249f258e2486.png)](https://habr.com/ru/post/673926/)
* ["All you need to know about ECS"](https://habr.com/en/post/665276/)
[![](https://habrastorage.org/r/w1560/getpro/habr/upload_files/3fd/5bc/544/3fd5bc5442b03a20d52a8003576056d4.png)](https://habr.com/ru/post/665276/)

# Projects using LeoECS Lite
## With sources
* ["3D Platformer"](https://github.com/supremestranger/3D-Platformer-Lite)

[![](https://camo.githubusercontent.com/dcd2f525130d73f4688c1f1cfb12f6e37d166dae23a1c6fac70e5b7873c3ab21/68747470733a2f2f692e6962622e636f2f686d374c726d342f506c6174666f726d65722e706e67)](https://github.com/supremestranger/3D-Platformer-Lite)


* ["SharpPhysics2D"](https://github.com/7Bpencil/sharpPhysics)

[![](https://github.com/7Bpencil/sharpPhysics/raw/master/pictures/preview.png)](https://github.com/7Bpencil/sharpPhysics)


* ["YourVostok"](https://github.com/7Bpencil/YourVostok)

[![](https://github.com/7Bpencil/YourVostok/raw/master/Previews/preview.gif)](https://github.com/7Bpencil/YourVostok)


# Extensions
* [Dependency Injection](https://github.com/Leopotam/ecslite-di)
* [Extended Systems](https://github.com/Leopotam/ecslite-extendedsystems)
* [Multithreading Support](https://github.com/Leopotam/ecslite-threads)
* [Unity editor integration](https://github.com/Leopotam/ecslite-unityeditor)
* [Unity uGui Support](https://github.com/Leopotam/ecslite-unity-ugui)
* [UniLeo - Unity scene data converter](https://github.com/voody2506/UniLeo-Lite)
* [Unity Physx events support](https://github.com/supremestranger/leoecs-lite-physics)
* [Multiple Shared injection](https://github.com/GoodCatGames/ecslite-multiple-shared)
* [EasyEvents](https://github.com/7Bpencil/ecslite-easyevents)
* [Entity command buffer](https://github.com/JimboA/EcsLiteEntityCommandBuffer)

# License
The framework is released under two licenses, [details here](./LICENSE.md).

In cases of licensing under the terms of MIT-Red, you should not rely on
personal advice or any guarantees.

# FAQ

### How is it different from the old version of LeoECS?

I prefer to call them `lite` (ecs-lite) and `classic` (leoecs). The main differences between `light` are as follows:
* The code base of the framework has decreased by 2 times, it has become easier to maintain and expand.
* Absence of any static data in the kernel.
* Lack of component caches in filters, this reduces memory consumption and increases the speed of shifting entities through filters.
* Quick access to any component on any entity (not just filtered and through the filter cache).
* There is no limit on the number of requirements / restrictions on the components for filters.
* The overall linear performance is close to `classic`, but access to components, shifting entities through filters has become disproportionately faster.
* Focus on the use of multi-worlds - several instances of worlds simultaneously with the division of data on them to optimize memory consumption.
* Lack of reflection in the kernel, it is possible to use aggressive cutting of unused code by the compiler (code stripping, dead code elimination).
* Sharing common data between systems happens without reflection (if it is allowed, it is recommended to use the `ecslite-di` extension from the list of extensions).
* The implementation of entities has returned to the usual `int` type, this has reduced memory consumption. If entities need to be stored somewhere, they still need to be packaged in a special structure.
* Small core, all additional functionality is implemented through the connection of optional extensions.
* All new functionality will be released only to the `light` version, `classic` has been switched to support mode for bug fixes.

### I want to call one system in `MonoBehaviour.Update()` and another system in `MonoBehaviour.FixedUpdate()`. How can i do this?

To separate systems based on different methods from `MonoBehaviour`, you need to create a separate `IEcsSystems` group for each method:
```c#
IEcsSystems_update;
IEcsSystems_fixedUpdate;

void Start() {
EcsWorld world = new EcsWorld();
_update = new EcsSystems(world);
_update
.Add(new UpdateSystem())
.init();
_fixedUpdate = new EcsSystems(world);
_fixedUpdate
.Add(new FixedUpdateSystem())
.init();
}

void Update() {
_update?.run();
}

void FixedUpdate() {
_fixedUpdate?.Run();
}
```

### I'm not happy with the default values for component fields. How can I set it up?

Components support custom value setting through the implementation of the `IEcsAutoReset<>` interface:
```c#
struct MyComponent : IEcsAutoReset<MyComponent> {
public int Id;
public object LinkToAnotherComponent;

public void AutoReset(ref MyComponent c) {
c.Id = 2;
c.LinkToAnotherComponent = null;
}
}
```
This method will automatically be called for all new components, as well as for all newly removed ones, before they are placed in the pool.
> **IMPORTANT!** If `IEcsAutoReset` is used, all additional clearing/checking of component fields is disabled, which can lead to memory leaks. The responsibility lies with the user!

### I want to store a reference to an entity in a component. How can i do this?

To save a reference to an entity, it must be packed into one of the special containers (`EcsPackedEntity` or `EcsPackedEntityWithWorld`):
```c#
EcsWorld world = new EcsWorld();
int entity = world.NewEntity();
EcsPackedEntity packed = world.PackEntity(entity);
EcsPackedEntityWithWorld packedWithWorld = world.PackEntityWithWorld(entity);
...
// At the time of unpacking, we check if this entity is alive or not.
if (packed. Unpack (world, out int unpacked)) {
// "unpacked" is a valid entity and we can use it.
}

// At the time of unpacking, we check if this entity is alive or not.
if (packedWithWorld.Unpack (out EcsWorld unpackedWorld, out int unpackedWithWorld)) {
// "unpackedWithWorld" is a valid entity and we can use it.
}
```

### I want to add reactivity and handle world change events myself. How can I do it?

> **IMPORTANT!** This is not recommended due to performance degradation.

To activate this functionality, add `LEOECSLITE_WORLD_EVENTS` to the list of compiler directives, and then add an event listener:

```c#
class TestWorldEventListener : IEcsWorldEventListener {
public void OnEntityCreated(int entity) {
// The entity has been created - the method will be called when world.NewEntity() is called.
}

public void OnEntityChanged(int entity) {
// The entity has been changed - the method will be called at the moment pool.Add() / pool.Del() is called.
}

public void OnEntityDestroyed(int entity) {
// Entity is destroyed - the method will be called when world.DelEntity() is called or when the last component is removed.
}

public void OnFilterCreated(EcsFilter filter) {
// The filter has been created - the method will be called when world.Filter().End() is called if the filter did not exist before.
}

public void OnWorldResized(int newSize) {
// The world has changed sizes - the method will be called in case of changing the sizes of the caches under the entity at the moment of calling world.NewEntity().
}

public void OnWorldDestroyed(EcsWorld world) {
// The world is destroyed - the method will be called when world.Destroy() is called.
}
}
...
varworld = new EcsWorld();
var listener = new TestWorldEventListener();
world.AddEventListener(listener);
```

### I want to add reactivity and handle filter change events. How can i do this?

> **IMPORTANT!** This is not recommended due to performance degradation.

To activate this functionality, add `LEOECSLITE_FILTER_EVENTS` to the list of compiler directives, and then add an event listener:

```c#
class TestFilterEventListener : IEcsFilterEventListener {
public void OnEntityAdded(int entity) {
// The entity is added to the filter.
}

public void OnEntityRemoved(int entity) {
// Entity removed from filter.
}
}
...
varworld = new EcsWorld();
var filter = world.Filter<C1>().End();
var listener = new TestFilterEventListener();
filter.AddEventListener(listener);
```

