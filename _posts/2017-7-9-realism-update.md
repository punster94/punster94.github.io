---
layout: post
title:  "Realism Update"
date:   2017-7-9 00:00:00 +0200
tags: ['c/c++', 'Unreal', 'UE4']
author: "Louis Hofer"
---

Rather than making an aggressive state for field players, I thought it more interesting to bring the simulation closer to a real soccer match.
The first thing that you will notice is the size of the teams:

![Large Teams](http://louishofer.com/images/large%20teams.png "Large Teams")

Each team contains 11 players: 10 field players and one goal keeper.
Their placements are akin to the configuration of a real soccer team and as such each player has one of the following roles:

* sweeper
* centerback
* wingback
* midfield
* forward

In addition to creating defined roles on the team for each player, I've added a system that degrades a player's stamina as they play.
Players have separate speeds, stamina costs, stamina regeneration rates, and stamina thresholds that determine what speed to move at.
Each player has different values that I have assigned manually to give a bit of realism to their behaviors.
For example, the players I have assigned as forwards I have also assigned higher movement speeds but higher stamina costs.
In this way, they can move significantly faster than other players, but can only do so when they haven't been moving for too long.

I'll explain how the stamina system works and how it is used by steering behaviors:

```c
void APlayerBase::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);

	UpdateStamina(DeltaTime);
	UpdateSpeed();
	Steer(DeltaTime);
}
```

Let's start with the update method of the player (Tick in Unreal).

Every frame the player updates its current stamina value.
This stamina value is used to update the player's speed accordingly.
Finally, the updated current speed is used by the player's steering controller to determine how quickly to move in a direction.

Now let's get into the specifics of what it means to update the player's stamina and speed.

```c
void APlayerBase::UpdateStamina(float DeltaTime)
{
	float Cost = GetVelocity().Size() * DeltaTime * StaminaCost;
	float Recovery = StaminaRegenerationRate * DeltaTime;

	CurrentStamina = FMath::Clamp(CurrentStamina - Cost + Recovery, 0.0f, MaxStamina);

	if (CurrentStaminaState == EStaminaState::Normal &&
		CurrentStamina <= MinimumSpeedThreshold)
	{
		CurrentStaminaState = EStaminaState::Regenerating;
	}

	if (CurrentStaminaState == EStaminaState::Regenerating &&
		CurrentStamina >= StaminaRegenerationThreshold)
	{
		CurrentStaminaState = EStaminaState::Normal;
	}
}
```

Updating stamina involves calculating the current cost of moving and the amount the player can recover in the current frame.
The player properties I mentioned earlier, stamina cost and stamina regeneration rate, are used here.
Stamina cost can be thought of like the amount of stamina lost per unit of distance traveled (in the case of Unreal, stamina / cm).
Stamina regeneration is the amount of stamina per second regained by the player.

The current stamina of the player is calculated by adding the regeneration to the player's previous stamina amount, then subtracting the cost of their current movement.

After this, the player maintains its current stamina state.
This refers to whether the player has passed its minimum speed threshold and is currently regenerating or if they are fit enough to run like normal.

```c
void APlayerBase::UpdateSpeed()
{
	switch (CurrentStaminaState)
	{
	case EStaminaState::Normal:
		CurrentSpeed = MaximumSpeed;

		if (CurrentStamina <= MaximumSpeedThreshold)
		{
			// this is currently linear, but could be better
			CurrentSpeed *= CurrentStamina / MaxStamina;
			CurrentSpeed = FMath::Max(CurrentSpeed, MinimumSpeed);
		}

		break;

	case EStaminaState::Regenerating:
		CurrentSpeed = MinimumSpeed;

		break;
	}
}
```

Updating the player's speed depends on that stamina state we just updated.
If the player is in the normal state, the player starts with the assumption that they can move at their maximum speed.
The player determines if their stamina is below a threshold and if so, modifies their current speed linearly based on their current stamina.
If the player is regenerating, their current speed is set to their minimum speed until they recover enough stamina to start moving faster again.

Here is a gif of showing the large teams passing, running, and getting tired:

![StaminaSystem](http://louishofer.com/gifs/StaminaSystem.gif "StaminaSystem")
