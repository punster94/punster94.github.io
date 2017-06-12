---
layout: post
title:  "Support Spot Calculation"
date:   2017-6-12 00:00:00 +0200
tags: ['c/c++', 'Unreal', 'UE4']
author: "Louis Hofer"
---

Now that players can move around the field space, its important to give them a sense of where to go.
This involves breaking the field down into an area that we can properly iterate over and analyze.
I've tackled this by putting a UBoxComponent in the ASoccerField class used to define the playable area.
Given the x and y (and z) bounds of the area as well as a configurable number of spots along the X and Y axis, spots can be generated like so:

```c
FVector bounds = Field->SupportAreaScaledBounds();

float sliceX = bounds.X * 2.0f / XSpots;
float sliceY = bounds.Y * 2.0f / YSpots;

FVector fieldLocation = Field->GetLocation();

float left = fieldLocation.X - bounds.X + sliceX / 2.0f;
float right = fieldLocation.X + bounds.X - sliceX / 2.0f;
float top = fieldLocation.Y - bounds.Y + sliceY / 2.0f;

for (int x = 0; x < (XSpots / 2); ++x)
{
	for (int y = 0; y < YSpots; ++y)
	{
		if (Team == ETeam::red)
		{
			Spots.push_back(SupportSpot(FVector(left + x * sliceX, top + y * sliceY, 0.0f), 0.0f));
		}
		else
		{
			Spots.push_back(SupportSpot(FVector(right - x * sliceX, top + y * sliceY, 0.0f), 0.0f));
		}
	}
}
```

With the field sliced up into points, its possible to evaluate the area for what spots would be good to send a player to.
But what makes a good support spot?
For this simulation, three general factors will be used to evaluate the strength of a spot:

1 Whether the ball can be passed from the controlling player to the spot free from opponents
1 Whether a shot can be made from the spot without opponents intercepting the shot
1 The distance the spot is from the controlling player (the closer to the configured optimal distance the better)

My implementation looks like this:

```c
void ASoccerTeam::SupportSpotCalculator::DetermineBestSupportingPosition()
{
	BestSupportingSpot = nullptr;

	float bestSpotSoFar = 0.0f;

	for (SupportSpot& spot : Spots)
	{
		spot.Score = 1.0f;

		//Test for pass safety
		if (mTeam->IsPassSafeFromAllOpponents(mTeam->ControllingPlayer->GetLocation(),
			spot.Position, nullptr, 1000.0f))
		{
			spot.Score += mTeam->CanPassStrength;
		}

		FVector shotDirection;

		//Test for whether the support spot 
		if (mTeam->CanShoot(spot.Position, mTeam->MaxShootingStrength, shotDirection))
		{
			spot.Score += mTeam->CanScoreFromPositionStrength;
		}

		//Test for how far the pass would be
		if (mTeam->SupportingPlayer != nullptr)
		{
			float distance = FVector::Distance(mTeam->ControllingPlayer->GetLocation(), spot.Position);
			float distanceFromOptimal = FMath::Abs(mTeam->OptimalDistanceFromControllingPlayer - distance);

			if (distanceFromOptimal < mTeam->OptimalDistanceFromControllingPlayer)
			{
				GEngine->AddOnScreenDebugMessage(-1, 1.0f, FColor::Blue, FString::Printf(TEXT("%f"), spot.Score));
				spot.Score += mTeam->DistanceFromControllingPlayerStrength *
					((mTeam->OptimalDistanceFromControllingPlayer - distanceFromOptimal) /
						mTeam->OptimalDistanceFromControllingPlayer);
			}
		}

		//Set if best
		if (spot.Score > bestSpotSoFar)
		{
			bestSpotSoFar = spot.Score;
			BestSupportingSpot = &spot;
		}


		//Draw each spot based on their score multiplied by a base size
		DebugSpot(spot, 5.0f);
	}
}
```

Here is a visualization of the support spots updating:

![SupportSpotCalculation](http://louishofer.com/gifs/SupportSpotCalculation.gif "SupportSpotCalculation")

Now we can move the currently designated supporting player to the best support spot whenever it is calculated:

![SupportMovingToBestSpot](http://louishofer.com/gifs/SupportMovingToBestSpot.gif "SupportMovingToBestSpot")

The next step in the process is controlling when and who is designated as the controlling and supporting players on a given team.
For this, team and player state machines should be made.