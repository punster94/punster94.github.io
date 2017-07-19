---
layout: post
title:  "Dynamic Team Positions"
date:   2017-7-19 00:00:00 +0200
tags: ['c/c++', 'Unreal', 'UE4']
author: "Louis Hofer"
---

Watching the new teams play against each other, it became clear that they do not have any knowledge of their team's composition.
Each player would just traverse to its hand-picked attack or defense position and wait to be the closest player to the ball.
This caused the players to move forwards and backwards frequently, which looks very unnatural and incorrect.
I've attempted to prevent this behavior by refactoring the team structure to instead make use of relative offsets from the "team position".
The roles that were creted for the players will now be used to determine how far away from the team position they should be.
Additionally, the number of players of that role will be used to determine the specific spot the player should occupy.
Here is a look at what I've come up with.

```c
FVector ASoccerTeam::CalculatePlayerHomeRegion(class AFieldPlayer* Player)
{
	FVector regionPosition = FVector::ZeroVector;

	int playerNumber;
	int numberOfType = NumberOfPlayersOfType(Player->PlayerRole, playerNumber, Player);

	FVector bounds = Field->PlayAreaScaledBounds() * 2.0f;
	FVector fieldLocation = Field->GetLocation();

	float sliceY = bounds.Y / numberOfType;

	regionPosition.Y = fieldLocation.Y - (bounds.Y * 0.5f);
	regionPosition.Y += (sliceY * playerNumber) + (sliceY * 0.5f);

	float sqDistanceFromMiddle = TeamRoleDistances[Player->PlayerRole];
	sqDistanceFromMiddle *= sqDistanceFromMiddle;

	float sqYDistance = Location.Y - regionPosition.Y;
	sqYDistance *= sqYDistance;

	regionPosition.X = sqDistanceFromMiddle - sqYDistance;
	regionPosition.X = FMath::Sqrt(regionPosition.X);

	if (Team == ETeam::blue)
	{
		regionPosition.X = fieldLocation.X - regionPosition.X;
	}
	else
	{
		regionPosition.X = fieldLocation.X + regionPosition.X;
	}

	return regionPosition;
}
```

The team is responsible for determining the position a character should use.
This is determined by first counting the number of players on the team with the given player's role.
Then the field's Y-bounds are sliced evenly for each player in that role and each player is assigned a slice in order of when they are found in the list of players.
For example, if a midfielder requested to find its home region and was found to be the third of four midfielders, they would be assigned the vertical slice with the index of 2.

Next, the pythagorian theorem is used to determine the X value of their position using a configurable map of euclidean distances that each role should be from the team position.
The team position is positioned to be in front of all players, so the X position is modified based on which team the player is on.
On the blue team all player X positions should be to the left of the team and on the red team the opposite is true.

```c
void APlayerBase::SetHomeRegion(int Region)
{
	AFieldPlayer* fieldPlayer = Cast<AFieldPlayer>(this);

	if (fieldPlayer != nullptr)
	{
		HomeRegion = Team->CalculatePlayerHomeRegion(fieldPlayer);
	}
	else
	{
		HomeRegion = Team->GetField()->GetPositionOfRegionID(Region);
	}

	OffsetFromTeam = HomeRegion - Team->GetLocation();
}

void APlayerBase::TargetHome()
{
	Steering->SetTargetLocation(Team->GetLocation() + OffsetFromTeam);
}
```

This is used by players to calculate an offset from their team that they should seek to whenever they are to wait.
Whenever the player's state machine requests that they target home, they will look at the team's location to determine what position to wait at.
The offsets are recalculated any time the ball changes teams rather than every frame to prevent some of the heavier math from occuring frequently.
While the offset should not change throughout the simulation as of now, I plan on adding functionality for compressing the play area.
I'll describe this later.

```c
void ASoccerTeam::UpdateAdvancement(float DeltaTime)
{
	float xOffset = FMath::Abs(Location.X - InitialLocation.X);

	switch (AdvancementState)
	{
	case ETeamAdvancementState::advance:
		xOffset += AdvancementSpeed * DeltaTime;

		break;

	case ETeamAdvancementState::fallback:
		xOffset -= AdvancementSpeed * DeltaTime;

		break;
	}

	xOffset = FMath::Clamp(xOffset, 0.0f, MaximumAdvancementDistance);

	Location.X = InitialLocation.X + xOffset;
}
```

Now that the players wait in positions relative to their team, it is important to move the teams in some way.
For now, this movement is modeled as whether the team is advancing forward or falling backwards.
A configurable advancement speed on the team dictates how far the team can travel in one second in the x direction.
Advancement states are changed in the team state machine.
When the team is in the attack state they advance whereas in defense they fall back.
To prevent the teams from moving through their own goal, or the goal of the other team, the offset from the simulation's initial position is clamped between 0 and a configurable maximum distance.

Here are two gifs of the team forming up at start.
The first has three sweepers, four midfielders, and three forwards:

![ThreeFourThree](http://louishofer.com/gifs/ThreeFourThree.gif "threefourthree")

The second has three sweepers, five midfielders, and two forwards:

![ThreeFiveTwo](http://louishofer.com/gifs/ThreeFiveTwo.gif "threefivetwo")