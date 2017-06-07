---
layout: post
title:  "Writing Steering Behaviors"
date:   2017-5-30 00:00:00 +0200
tags: ['c/c++', 'Unreal', 'UE4']
author: "Louis Hofer"
---

With a world (or soccer field) built in a thoughful way, I'm now able to begin implementing behaviors.
Before I can get to the interesting things like support spot calculation, I have to be able to move players.
But I want to move players in a generic, yet thoughtful way.
I decided to implement some known steering behaviors that will be useful for playing soccer.
Below is the list along with descriptions, visualizations, and the code I used to implement them.

1 Seek
The seek steering behavior provides a point for the player to move towards at its maximum possible speed:

![seek](http://louishofer.com/gifs/Seek2.gif "Seek")

```c
FVector SteeringBehavior::Seek(FVector Target)
{
	FVector desiredVelocity = Target - PlayerBase->GetActorLocation();
	desiredVelocity.Normalize();
	desiredVelocity *= PlayerBase->MaximumSpeed;

	FVector projectedPosition = PlayerBase->GetActorLocation() + (desiredVelocity * Delta);

	if (FVector::Dist(projectedPosition, Target) > FVector::Dist(PlayerBase->GetActorLocation(), Target))
	{
		return FVector::ZeroVector;
	}

	return desiredVelocity;
}
```

2 Arrive
As you could see above, the seek behavior can be jittery when the player actually reaches their goal, as they are constantly moving at their maximum speed.
The arrive steering behavior makes the player move to a target position with deceleration to ease into the desired spot.
In the following visualization, the red players are using a normal deceleration while two blue players are fast and two are slow:

![arrive](http://louishofer.com/gifs/Arrive2.gif "Arrive")

```c
const float SteeringBehavior::DecelerationTweaker = 0.3f;

UENUM(BlueprintType)
enum class EDecelerationType : uint8
{
	fast = 1,
	normal = 2,
	slow = 3,
};

FVector SteeringBehavior::Arrive(FVector Target, EDecelerationType Deceleration)
{
	FVector toTarget = Target - PlayerBase->GetActorLocation();
	float distance = toTarget.Size();

	FVector desiredVelocity = FVector::ZeroVector;

	if (distance > 0.0f)
	{
		float speed = distance / ((uint8)Deceleration * DecelerationTweaker);
		speed = FMath::Min(speed, PlayerBase->MaximumSpeed);

		desiredVelocity = (toTarget * speed / distance);
	}

	return desiredVelocity;
}
```

3 Pursuit
Now that I can have a player move quickly or gracefully to a point, its time to move it towards a moving object.
The pursuit steering behavior moves a player towards the position of a given object.
This is done more thoughtfully, however, by predicting where the object will be at a certain look-ahead time.
Some people like to seek to their predicted location, but for this visualization I arrive there.
Each player follows a different player on its team, while the lead players follow the ball which I kick around with debug keys.

![pursuit](http://louishofer.com/gifs/Pursuit2.gif "Pursuit")

```c
FVector SteeringBehavior::Pursue(UPrimitiveComponent* Object)
{
	FVector toTarget = Object->GetComponentLocation() - PlayerBase->GetActorLocation();

	FVector heading = PlayerBase->GetVelocity();
	heading.Normalize();

	FVector objectHeading = Object->GetPhysicsLinearVelocity();
	objectHeading.Normalize();

	float relativeHeading = FVector::DotProduct(heading, objectHeading);

	if ((FVector::DotProduct(toTarget, heading) > 0.0f) &&
		(relativeHeading < -0.95)) // acos(0.95) = 18 degrees
	{
		//return Seek(Object->GetComponentLocation());

		return Arrive(Object->GetComponentLocation(), EDecelerationType::fast);
	}

	float lookAheadTime = toTarget.Size() / (PlayerBase->MaximumSpeed + Object->GetPhysicsLinearVelocity().Size());

	//return Seek(Object->GetComponentLocation() + Object->GetPhysicsLinearVelocity() * lookAheadTime);

	return Arrive(Object->GetComponentLocation() + Object->GetPhysicsLinearVelocity() * lookAheadTime, EDecelerationType::fast);
}
```

4 Interpose
A steering behavior quite relevant to playing soccer is the interpose behavior.
This involves moving towards the expected midpoint of two objects.
This can be used to attempt intercepting the ball from the receiving player by getting between them.
In this visualization each player interposes themself between their teammate and the ball:

![interpose](http://louishofer.com/gifs/Interpose2.gif "Interpose")

```c
FVector SteeringBehavior::Interpose(UPrimitiveComponent* ObjectA, UPrimitiveComponent* ObjectB)
{
	FVector midpoint = (ObjectA->GetComponentLocation() + ObjectB->GetComponentLocation()) / 2.0f;

	float timeToReachMidpoint = FVector::Dist(PlayerBase->GetActorLocation(), midpoint) / PlayerBase->MaximumSpeed;

	FVector futureA = ObjectA->GetComponentLocation() + (ObjectA->GetPhysicsLinearVelocity() * timeToReachMidpoint);
	FVector futureB = ObjectB->GetComponentLocation() + (ObjectB->GetPhysicsLinearVelocity() * timeToReachMidpoint);

	midpoint = (futureA + futureB) / 2.0f;

	return Arrive(midpoint, EDecelerationType::fast);
}
```

5 Separation
The final steering behavior is a group behavior that takes into account surrounding objects.
The separation behavior attempts to move the player away from a list of neighbors that are tagged based on their proximity.
If they are close enough, the player will factor them into the best direction to move to avoid objects.
The player will move at its maximum speed towards the best direction:

![separate](http://louishofer.com/gifs/Separate2.gif "Separate")

```c
FVector SteeringBehavior::Separation(TArray<UPrimitiveComponent*> Neighbors)
{
	FVector result = FVector::ZeroVector;

	for (UPrimitiveComponent* neighbor : Neighbors)
	{
		FVector fromNeighbor = PlayerBase->GetActorLocation() - neighbor->GetComponentLocation();
		float distance = fromNeighbor.Size();

		fromNeighbor.Normalize();

		result += fromNeighbor / distance;
	}

	if (result == FVector::ZeroVector)
	{
		return result;
	}

	result.Normalize();

	return result * PlayerBase->MaximumSpeed;
}
```