---
layout: post
title:  "Passing Tweaks"
date:   2017-6-25 00:00:00 +0200
tags: ['c/c++', 'Unreal', 'UE4']
author: "Louis Hofer"
---

Similar to the team's state machines, each field player has their own state machine running to determine what particular action they should perform.
These involve knowing when to transition between states and what actions to perform in any given state.

Field players currently have seven (7) states along with their global state:
Wait, ChaseBall, ReturnToHomeRegion, KickBall, Dribble, SupportAttacker, and ReceiveBall

These states have beefy implementations, so I'll only provide one state as an example.
The KickBall state is responsible for taking shots, passing to available players, and dribbling given the correct conditions.
As one of the more complicated but telling states I'll provide my current implementation below:

```c
void KickBall::Enter(AFieldPlayer& player)
{
	player.GetTeam()->SetControllingPlayer(&player);

	if (!player.IsReadyForNextKick())
	{
		player.GetStateMachine().ChangeState(*ChaseBall::Instance());
	}
}

void KickBall::Execute(AFieldPlayer& player)
{
	FVector toBall = player.GetTeam()->GetBall()->GetLocation() - player.GetLocation();
	toBall.Normalize();

	float dot = FVector::DotProduct(player.GetHeading(), toBall);

	if (player.GetTeam()->GetReceiver() != nullptr ||
		player.GetTeam()->GetField()->GoalKeeperHasBall() ||
		dot < 0.0f)
	{
		player.GetStateMachine().ChangeState(*ChaseBall::Instance());

		return;
	}

	FVector ballTarget;

	float power = player.GetTeam()->GetMaxShootingStrength() * dot;

	if (player.GetTeam()->CanShoot(player.GetTeam()->GetBall()->GetLocation(), power, ballTarget) ||
		FMath::FRand() < player.GetTeam()->GetChancePlayerAttemptsPotShot())
	{
		ballTarget = player.AddNoiseToKick(player.GetTeam()->GetBall()->GetLocation(), ballTarget);

		FVector kickDirection = ballTarget - player.GetTeam()->GetBall()->GetLocation();

		GEngine->AddOnScreenDebugMessage(-1, 2.0f, FColor::Green, TEXT("Attempting Shot At Goal"));

		player.GetTeam()->GetBall()->Kick(kickDirection, power);

		player.GetStateMachine().ChangeState(*Wait::Instance());

		player.FindSupport();

		return;
	}

	APlayerBase* receiver = nullptr;

	power = player.GetTeam()->GetMaxPassingStrength() * dot;

	if (player.IsThreatened() &&
		player.GetTeam()->FindPass(player, receiver, ballTarget, power, player.GetTeam()->GetMinPassingDistance()))
	{
		ballTarget = player.AddNoiseToKick(player.GetTeam()->GetBall()->GetLocation(), ballTarget);

		FVector kickDirection = ballTarget - player.GetTeam()->GetBall()->GetLocation();

		GEngine->AddOnScreenDebugMessage(-1, 2.0f, FColor::Green, TEXT("Attempting Pass"));

		player.GetTeam()->GetBall()->Kick(kickDirection, power);

		AFieldPlayer* receivingPlayer = Cast<AFieldPlayer>(receiver);
		
		if (receivingPlayer != nullptr)
		{
			receivingPlayer->MessageReceiveBall(ballTarget);
		}

		player.GetStateMachine().ChangeState(*Wait::Instance());

		player.FindSupport();
	}
	else
	{
		player.FindSupport();

		player.GetStateMachine().ChangeState(*Dribble::Instance());
	}
}
```

As you can see, the KickBall state begins by checking if the player is allowed to kick the ball right now.
This prevents the player from frequently kicking the ball (which could happen several frames in a row if the ball remains close enough).
Once the player knows that they can kick the ball, it sets a timer for itself that prevents the IsReadyForNextKick from returning true while it is still ticking.
Its important to note that the player should only be in the KickBall state for a duration one frame; it is a pass-through state that invokes the Kick method on the ball.

```c
FVector toBall = player.GetTeam()->GetBall()->GetLocation() - player.GetLocation();
toBall.Normalize();

float dot = FVector::DotProduct(player.GetHeading(), toBall);

if (player.GetTeam()->GetReceiver() != nullptr ||
	player.GetTeam()->GetField()->GoalKeeperHasBall() ||
	dot < 0.0f)
{
	player.GetStateMachine().ChangeState(*ChaseBall::Instance());

	return;
}
```

The start of the execute method determines the orientation of the player with respect to the ball and its corresponding dot product.
If the player is not heading in the general direction to the ball, the team already has a designated receiver for a pass, or the goalkeeper has the ball (implemented as return false for now), the player should continue to chase the ball instead of kicking.

```c
FVector ballTarget;

float power = player.GetTeam()->GetMaxShootingStrength() * dot;

if (player.GetTeam()->CanShoot(player.GetTeam()->GetBall()->GetLocation(), power, ballTarget) ||
	FMath::FRand() < player.GetTeam()->GetChancePlayerAttemptsPotShot())
{
	ballTarget = player.AddNoiseToKick(player.GetTeam()->GetBall()->GetLocation(), ballTarget);

	FVector kickDirection = ballTarget - player.GetTeam()->GetBall()->GetLocation();

	GEngine->AddOnScreenDebugMessage(-1, 2.0f, FColor::Green, TEXT("Attempting Shot At Goal"));

	player.GetTeam()->GetBall()->Kick(kickDirection, power);

	player.GetStateMachine().ChangeState(*Wait::Instance());

	player.FindSupport();

	return;
}
```

This next block calculates the power the player should attempt to shoot the ball towards the goal with, given the previous dot product.
If the team deems that the player can make the shot given the ball's location and the power of the kick, the Kick method of the ball will be executed on the returned ball destination.
Additionally, a player has a small random chance of attempting a potshot at the goal to spice things up.

Additionally spicy is the AddNoiseToKick method which randomly adds a bit of angle to the kick by slightly changing the ball's destination based on the team's specified kicking accuracy.
When the player kicks the ball, they stop at their location and wait for the shot to complete.
They also find the best supporting player on their team to ensure that someone will exist at a useful location in the event that their shot misses.

```c
APlayerBase* receiver = nullptr;

power = player.GetTeam()->GetMaxPassingStrength() * dot;

if (player.IsThreatened() &&
	player.GetTeam()->FindPass(player, receiver, ballTarget, power, player.GetTeam()->GetMinPassingDistance()))
{
	ballTarget = player.AddNoiseToKick(player.GetTeam()->GetBall()->GetLocation(), ballTarget);

	FVector kickDirection = ballTarget - player.GetTeam()->GetBall()->GetLocation();

	GEngine->AddOnScreenDebugMessage(-1, 2.0f, FColor::Green, TEXT("Attempting Pass"));

	player.GetTeam()->GetBall()->Kick(kickDirection, power);

	AFieldPlayer* receivingPlayer = Cast<AFieldPlayer>(receiver);
		
	if (receivingPlayer != nullptr)
	{
		receivingPlayer->MessageReceiveBall(ballTarget);
	}

	player.GetStateMachine().ChangeState(*Wait::Instance());

	player.FindSupport();
}
```

This third block deals with the player passing to a teammate (the cool stuff!).
Passes are only invoked if the player is threatened (there is an opponent close enough to them).
The player would otherwise just keep dribbling themself towards the opponent's goal as there is no need for passing.
Additionally, the player needs to determine if there are any acceptable across their team.
The FindPass method takes a APlayerBase*& and sets it if there are any valid and safe passes between the player and its teammates.
This pass is picked based on proximity to the best support spot.

Noise is added to this pass just like a shot at the goal and the receiving player is notified about the incoming pass.

```c
else
{
	player.FindSupport();

	player.GetStateMachine().ChangeState(*Dribble::Instance());
}
```

This last bit being executed implies that the player should not shoot or pass the ball, and should instead keep it in their posession.
The player can dribble the ball, which involves either changing their direction to be more oriented to the opponent's goal or moving the ball forward faster than the player can move with it in their vicinity.

Here is a gif of the two teams scoring and passing along the way (both requiring this KickBall method)

![PassingAndScoring](http://louishofer.com/gifs/PassingAndScoring.gif "PassingAndScoring")

The next step involves getting those static goalies to do something about their opponents shots!