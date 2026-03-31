# Swapping NPC Schedules at Runtime

This guide covers how to change an NPC's schedule dynamically — for example, switching from a "waiting" schedule to a full daily routine after a quest completes.

## Table of Contents

1. [The Problem](#the-problem)
2. [Key Constraint: Pre-Allocation](#key-constraint-pre-allocation)
3. [The Pattern](#the-pattern)
4. [Complete Example](#complete-example)
5. [Persisting Across Saves](#persisting-across-saves)
6. [API Reference](#api-reference)

## The Problem

A common scenario: your NPC should wait in one spot (e.g. sitting in a restaurant) until a quest completes, then switch to a normal daily routine. You might think you need to disable the schedule entirely and control the NPC manually — but there's a better way.

## Key Constraint: Pre-Allocation

The scheduling system requires all action components to be **pre-allocated during `ConfigurePrefab`**. The system reuses inactive `NPCAction` component instances at runtime — it does **not** create new ones. This is because FishNet (the networking layer) relies on stable component indices.

This means you must declare every action type your NPC will ever need upfront, even if some actions are only used later.

## The Pattern

### 1. Pre-allocate all actions for every schedule phase

In `ConfigurePrefab`, include actions for **both** the waiting phase and the post-quest phase. The system will only activate the ones you assign at runtime.

```csharp
protected override void ConfigurePrefab(NPCPrefabBuilder builder)
{
    builder.WithIdentity("quest_npc", "QuestNPC", "")
        .WithSpawnPosition(tacoTicklersPos)
        .EnsureCustomer()
        .WithSchedule(plan => {
            // Phase 1: Waiting — sit for 24 hours
            plan.SitAtSeatSet("Fast Food Booth", startTime: 0,
                durationMinutes: 1440, warpIfSkipped: true);

            // Phase 2: Normal routine (pre-allocate even though not used yet)
            plan.WalkTo(homePos, 800);
            plan.StayInBuilding(Building.Get<Buildings.NorthApartments>(), 900,
                durationMinutes: 480);
            plan.WalkTo(hangoutPos, 1700);
            plan.SitAtSeatSet("Outdoor Bench", 1730, durationMinutes: 120);
        });
}
```

### 2. Start with the waiting schedule

In `OnCreated`, use `Schedule.ApplyActions()` to activate only the waiting-phase actions:

```csharp
protected override void OnCreated()
{
    base.OnCreated();
    Appearance.Build();
    Region = Region.Downtown;

    if (!_questCompleted)
        ApplyWaitingSchedule();
    else
        ApplyNormalSchedule();

    Schedule.Enable();
}

private void ApplyWaitingSchedule()
{
    Schedule.ApplyActions(new IScheduleActionSpec[]
    {
        new SitSpec
        {
            SeatSetName = "Fast Food Booth",
            StartTime = 0,
            DurationMinutes = 1440,
            WarpIfSkipped = true,
            Name = "WaitForPlayer"
        }
    });
}
```

### 3. Swap to the real schedule on quest completion

```csharp
public void OnQuestCompleted()
{
    _questCompleted = true;

    Schedule.ApplyActions(new IScheduleActionSpec[]
    {
        new WalkToSpec { Destination = homePos, StartTime = 800, Name = "GoToWork" },
        new StayInBuildingSpec { BuildingName = "North apartments", StartTime = 900, DurationMinutes = 480, Name = "WorkShift" },
        new WalkToSpec { Destination = hangoutPos, StartTime = 1700, Name = "GoToHangout" },
        new SitSpec { SeatSetName = "Outdoor Bench", StartTime = 1730, DurationMinutes = 120, Name = "EveningRelax" }
    });

    RequestGameSave();
}
```

`ApplyActions` handles everything in one call: clears existing actions, applies the new specs, rebuilds the action list, and enforces the correct state for the current game time.

## Complete Example

```csharp
using S1API.Entities;
using S1API.Entities.Schedule;
using S1API.Entities.Schedule.ActionSpecs;
using S1API.Map;
using S1API.Map.Buildings;
using S1API.Saveables;
using UnityEngine;

public sealed class WaitingNPC : NPC
{
    public override bool IsPhysical => true;

    [SaveableField("QuestDone")]
    private bool _questCompleted = false;

    private static readonly Vector3 TacoTicklersPos = new(-28f, 1f, 62f);
    private static readonly Vector3 HomePos = new(-53f, 1f, 67f);
    private static readonly Vector3 HangoutPos = new(-40f, 1f, 50f);

    protected override void ConfigurePrefab(NPCPrefabBuilder builder)
    {
        builder.WithIdentity("waiting_npc", "WaitingNPC", "")
            .WithAppearanceDefaults(av =>
            {
                av.Gender = 0f;
                av.Height = 0.5f;
            })
            .WithSpawnPosition(TacoTicklersPos)
            .EnsureCustomer()
            .WithSchedule(plan =>
            {
                // Pre-allocate ALL action types needed across both phases
                // Phase 1: Waiting
                plan.SitAtSeatSet("Fast Food Booth", 0,
                    durationMinutes: 1440, warpIfSkipped: true);

                // Phase 2: Daily routine
                plan.WalkTo(HomePos, 800);
                plan.StayInBuilding(Building.Get<NorthApartments>(), 900,
                    durationMinutes: 480);
                plan.WalkTo(HangoutPos, 1700);
                plan.SitAtSeatSet("Outdoor Bench", 1730, durationMinutes: 120);
            });
    }

    protected override void OnCreated()
    {
        base.OnCreated();
        Appearance.Build();
        Region = Region.Downtown;

        if (_questCompleted)
            ApplyNormalSchedule();
        else
            ApplyWaitingSchedule();

        Schedule.Enable();
    }

    private void ApplyWaitingSchedule()
    {
        Schedule.ApplyActions(new IScheduleActionSpec[]
        {
            new SitSpec
            {
                SeatSetName = "Fast Food Booth",
                StartTime = 0,
                DurationMinutes = 1440,
                WarpIfSkipped = true,
                Name = "WaitForPlayer"
            }
        });
    }

    private void ApplyNormalSchedule()
    {
        Schedule.ApplyActions(new IScheduleActionSpec[]
        {
            new WalkToSpec { Destination = HomePos, StartTime = 800 },
            new StayInBuildingSpec
            {
                BuildingName = "North apartments",
                StartTime = 900,
                DurationMinutes = 480
            },
            new WalkToSpec { Destination = HangoutPos, StartTime = 1700 },
            new SitSpec
            {
                SeatSetName = "Outdoor Bench",
                StartTime = 1730,
                DurationMinutes = 120
            }
        });
    }

    public void OnQuestCompleted()
    {
        _questCompleted = true;
        ApplyNormalSchedule();
        RequestGameSave();
    }
}
```

## Persisting Across Saves

Use `[SaveableField]` to track which schedule phase the NPC is in. In `OnCreated`, check the saved state and apply the correct schedule:

```csharp
[SaveableField("Phase")]
private int _schedulePhase = 0;

protected override void OnCreated()
{
    base.OnCreated();

    switch (_schedulePhase)
    {
        case 0: ApplyWaitingSchedule(); break;
        case 1: ApplyNormalSchedule(); break;
        case 2: ApplyLateGameSchedule(); break;
    }

    Schedule.Enable();
}
```

This ensures the NPC resumes the correct schedule after saving and loading.

## API Reference

### Schedule.ApplyActions

```csharp
public void ApplyActions(IEnumerable<IScheduleActionSpec> specs)
```

Replaces all current schedule actions with the provided specs. This is the primary method for changing an NPC's schedule at runtime. It performs the following steps internally:

1. Clears all existing actions (disables them to preserve network indices)
2. Applies each spec to activate the matching pre-allocated action components
3. Rebuilds and sorts the action list by start time
4. Enforces the correct state for the current game time

### Other Schedule Methods

| Method | Description |
|--------|-------------|
| `Schedule.ClearActions(includeSignals, includeEvents)` | Removes all active actions. Disables components rather than destroying them to preserve network indices. |
| `Schedule.Enable()` / `Schedule.Disable()` | Enable or disable the entire schedule system. |
| `Schedule.EnforceState()` | Forces the schedule manager to evaluate and activate the correct action for the current time. |
| `Schedule.GetActiveActionName()` | Returns the name of the currently running action. |
| `Schedule.GetActionNames()` | Returns a read-only list of all configured action names. |

### Available Action Specs

| Spec | Description |
|------|-------------|
| `SitSpec` | Sit at an `AvatarSeatSet` for a duration |
| `WalkToSpec` | Walk to a world position |
| `StayInBuildingSpec` | Enter and remain in a building |
| `LocationDialogueSpec` | Move to a location and enable dialogue |
| `UseVendingMachineSpec` | Use a vending machine |
| `UseATMSpec` | Use an ATM |
| `DriveToCarParkSpec` | Drive a vehicle to a parking lot |

### Important Rules

1. **Pre-allocate all action types in `ConfigurePrefab`** — the runtime system reuses existing components, it cannot create new ones.
2. **Use `[SaveableField]` to persist the current phase** so the correct schedule is restored on load.
3. **`ApplyActions` handles the full rebuild cycle** — you do not need to call `ClearActions` or `EnforceState` separately.

## See Also

- [Scheduling System](scheduling-system.md) — Full schedule configuration reference
- [Seating Registry](seating-registry.md) — Discovering and using world seats
- [Save System](save-system.md) — Persisting NPC state with `[SaveableField]`
- [Quests System](quests-system.md) — Triggering schedule changes from quest events
