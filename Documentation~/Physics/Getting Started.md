# Getting Started with Latios Physics

This is the second preview version I am releasing out to the public. It
currently only supports a small number of use cases. The number of supported use
cases will grow with each release.

## Authoring

Currently Latios Physics uses the classical Physx components for authoring.
Attach a Sphere Collider or Capsule Collider to the GameObject you wish to
convert.

There’s one exception. I added support for Compound Colliders and this uses the
new Latios Collider Authoring component.

## Colliders in code

```csharp
//How to create collider

//Step 1: Create a sphere collider
var sphere = new SphereCollider(float3.zero, 1f);

//Step 2: Assign it to a collider
Collider collider = sphere;

//Step 3: Attach the collider to an entity
EntityManager.AddComponentData(sceneGlobalEntity, collider);

//How to extract sphere collider

//Step 1: Check type
if (collider.type == ColliderType.Sphere)
{
    //Step 2: Assign to a variable of the specialized type
    SphereCollider sphere2 = collider;

    //Note: With safety checks enabled, you will get an exception if you cast to the wrong type.
}

//EZ PZ, right?
```

## Simple Queries

-   Current

    -   Physics.CalculateAabb

    -   Physics.Raycast

    -   Physics.DistanceBetween (Collider vs Collider only)

-   Future

    -   Physics.DistanceBetween (Point vs Collider)

    -   Physics.AreIntersecting

    -   Physics.ColliderCast

    -   Physics.ComputeContacts

    -   Physics.QuadraticCast

    -   Physics.QuadraticColliderCast

## Simple Modifiers

-   Current

    -   Physics.ScaleCollider

## Scheduling Jobs

Some Physics operations allow for scheduling jobs (as well as executing directly
in a job). These operations expose themselves through a Fluent API with a
specific layout:

```csharp
Physics.SomePhysicsOperation(requiredParams).One(or).MoreExtra(Settings).Scheduler();
```

## Building Collision Layers

You start the Fluent chain by calling `Physics.BuildCollisionLayer`. There are
a couple of variants based on whether you want to build from an `EntityQuery`
or `NativeArray`s.

If you build from an `EntityQuery`, the `EntityQuery` must have the
`Collider` component. If you use Fluent, you can use the extension method
`PatchQueryForBuildingCollisionLayer` when building your `EntityQuery`.

Next in the fluent chain, you have the ability to apply custom settings and
options. You can customize the `CollisionLayerSettings` for better
performance. The settings are given as follows:

-   worldAABB – The AABB from which to construct the multibox

-   worldSubdivisionsPerAxis – How many “buckets” AKA cells to divide the world
    into along each axis

*Important: `worldAABB` and `worldSubdivisionsPerAxis` MUST match when
performing queries across multiple `CollisionLayer`s. I will probably add a
safety check for this in a future release.*

You can also create or pass in a `remapSrcArray` which will map indices in
`FindPairsResult` to the source indices of either your `EntityQuery` or
`NativeArray`s.

Lastly, after customizing things the way you want, you can call one of the many
schedulers:

-   .RunImmediate – Run without scheduling a job. You can call this from inside
    a job. This does not work using an `EntityQuery` as the source.

-   .Run – Run on the main thread with Burst.

-   .ScheduleSingle – Run on a worker thread with Burst.

-   .ScheduleParallel – Run on multiple worker threads with Burst.

*Feedback Request: Anyone have a better name for
PatchQueryForBuildingCollisionLayer?*

## Using FindPairs

`FindPairs` is a broadphase algorithm that lets you immediately process pairs
with any narrow phase or other logic you desire. `FindPairs` has unique
threading properties that ensures the two Entities in the pair can be modified
with `ComponentDataFromEntity`.

`FindPairs` uses a fluent syntax. Currently, there are two steps required.

-   Step 1: Call `Physics.FindPairs`

    -   To find pairs within a layer, only pass in a single `CollisionLayer`.

    -   To find pairs between two layers (Bipartite), pass in both layers. No
        pairs are generated within each individual layer. It is up to you to
        ensure no entity exists simultaneously in both layers or else scheduling
        a parallel job will not be safe.

    -   The final argument in `FindPairs` is an `IFindPairsProcessor`.
        Implement this interface as a struct containing any native containers
        your algorithm relies upon. When done correctly, your struct should
        resemble a non-lambda job struct.

-   Step 2: Call a scheduling method

    -   .RunImmediate – Run without scheduling a job. You can call this from
        inside a job.

    -   .Run – Run on the main thread with Burst.

    -   .ScheduleSingle – Run on a worker thread with Burst.

    -   .ScheduleParallel – Run on multiple worker threads with Burst.

## Why is FindPairs so slow in a build?

Great question! Latios Physics internally schedules multiple generic jobs for
FindPairs dynamically. The Burst Compiler currently does not support generic
jobs in builds without explicit declarations of their concrete types.

Latios.Core introduced a new BurstPatcher in 0.2.0 which should resolve this
problem, but I have not been able to test for all the different platforms which
means it is likely broken on some of them. Check your build for a
LatiosBurstPatched.dll. You can also check for lib_burst_generated.txt and see
if the FindPairs jobs are being listed.

If everything works correctly, FindPairs should be working much faster! In Mono
builds, I typically see over a 100x improvement. And on IL2CPP I see a 50x
improvement.
