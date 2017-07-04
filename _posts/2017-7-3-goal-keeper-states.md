---
layout: post
title:  "Goal Keeper States"
date:   2017-7-3 00:00:00 +0200
tags: ['c/c++', 'Unreal', 'UE4']
author: "Louis Hofer"
---

With field players running around, passing, and scoring it was becoming more apparent just how static the goal keepers were behaving. (They didn't even have separation turned on)
I've created a goal keeper state machine that the goalies use in their update to determine how to move and act.

Goal keepers currently have four (4) states along with their global state:
TendGoal, ReturnHome, PutBallBackInPlay, and InterceptBall

Similar to the field player states, I'll describe in detail the feature state of the machine: TendGoal

```c
TendGoal* TendGoal::Instance()
{
	static TendGoal instance;

	return &instance;
}

void TendGoal::Enter(AGoalKeeper& keeper)
{
	keeper.GetSteering()->InterposeOn();
	keeper.SetRearInterposeTargetPosition();
}

void TendGoal::Execute(AGoalKeeper& keeper)
{
	keeper.SetRearInterposeTargetPosition();

	if (keeper.BallWithinKeeperRange())
	{
		keeper.GetTeam()->GetBall()->Trap(&keeper);
		keeper.GetTeam()->SetControllingPlayer(&keeper);
		keeper.GetTeam()->GetField()->SetGoalKeeperHasBall(true);
		keeper.GetStateMachine().ChangeState(*PutBallBackInPlay::Instance());

		return;
	}

	if (keeper.BallWithinRangeForIntercept() &&
		!keeper.GetTeam()->InControl())
	{
		keeper.GetStateMachine().ChangeState(*InterceptBall::Instance());
	}

	if (keeper.TooFarFromGoalMouth() &&
		keeper.GetTeam()->InControl())
	{
		keeper.GetStateMachine().ChangeState(*ReturnHome::Instance());

		return;
	}
}

void TendGoal::Exit(AGoalKeeper& keeper)
{
	keeper.GetSteering()->InterposeOff();
}
```

The first thing to notice here is the Enter of the state:

```c
void TendGoal::Enter(AGoalKeeper& keeper)
{
	keeper.GetSteering()->InterposeOn();
	keeper.SetRearInterposeTargetPosition();
}
```

The keeper begins by turning its interpose steering behavior on, a steering behavior we have yet to make use of in the field players.
The plan is for the goal keeper to interpose its position between the back of the goal and the moving ball to provide the best position for blocking a shot.

```c
void TendGoal::Execute(AGoalKeeper& keeper)
{
	keeper.SetRearInterposeTargetPosition();

	if (keeper.BallWithinKeeperRange())
	{
		keeper.GetTeam()->GetBall()->Trap(&keeper);
		keeper.GetTeam()->SetControllingPlayer(&keeper);
		keeper.GetTeam()->GetField()->SetGoalKeeperHasBall(true);
		keeper.GetStateMachine().ChangeState(*PutBallBackInPlay::Instance());

		return;
	}

	if (keeper.BallWithinRangeForIntercept() &&
		!keeper.GetTeam()->InControl())
	{
		keeper.GetStateMachine().ChangeState(*InterceptBall::Instance());
	}

	if (keeper.TooFarFromGoalMouth() &&
		keeper.GetTeam()->InControl())
	{
		keeper.GetStateMachine().ChangeState(*ReturnHome::Instance());

		return;
	}
}
```

The Execute holds the meat of the state, where every frame we update the target position within the goal to interpose with.

```c
void AGoalKeeper::SetRearInterposeTargetPosition()
{
	FVector position = GetTeam()->GetGoal()->GetLocation();

	float goalWidth = GetTeam()->GetGoal()->ScoringArea->GetScaledBoxExtent().X;

	position.Y = GetTeam()->GetField()->GetLocation().Y;
	position.Y -= goalWidth * 0.5f;
	position.Y += GetTeam()->GetBall()->GetLocation().Y * goalWidth / GetTeam()->GetField()->PlayAreaScaledBounds().Y * 2.0f;

	InterposeTarget->SetWorldLocation(position);

	GetSteering()->SetOtherTargetObject(InterposeTarget);
}
```

There is a bit of geometry associated with this method, but the general gist of it is that each frame we project the ball across the goal line to find where it would end up.
This position is then used by the goal keeper's interpose behavior to determine the future midpoint between it and the ball.
As a note, the ball has been set as the goal keeper's primary target at the start of the game as it does not need to change (the only other state that uses it, InterceptBall, wants the ball as the primary as well).

```c
if (keeper.BallWithinKeeperRange())
{
	keeper.GetTeam()->GetBall()->Trap(&keeper);
	keeper.GetTeam()->SetControllingPlayer(&keeper);
	keeper.GetTeam()->GetField()->SetGoalKeeperHasBall(true);
	keeper.GetStateMachine().ChangeState(*PutBallBackInPlay::Instance());

	return;
}
```

This next bit is used to determine whether or not the goal keeper has blocked a shot. By this I mean that the ball is close enough to the goal keeper to be considered blocked.
The trap method of the ball just stops it in place.
Once the ball is trapped, the goal keeper moves on to put the ball back in play.

```c
if (keeper.BallWithinRangeForIntercept() &&
	!keeper.GetTeam()->InControl())
{
	keeper.GetStateMachine().ChangeState(*InterceptBall::Instance());
}
```

If the goal keeper isn't in range to block the ball, it will try to intercept it.
This is done by checking to see if the keeper is within a configurable interception range of the ball.
Additionally, the keeper should not intercept the ball if their team already controls the ball.

```c
if (keeper.TooFarFromGoalMouth() &&
	keeper.GetTeam()->InControl())
{
	keeper.GetStateMachine().ChangeState(*ReturnHome::Instance());

	return;
}
```

Finally, if the keeper is too far from the goal during its interposing (aka the ball is far enough away), the keeper should return home.
Additionally, the keeper shouldn't return home if their team is not in control.

Here is a gif of the goal keeper moving and reacting to the ball and players on its team.
Notice the green dots above the goal, those represent the projected interpose target described above.

![GoalKeeperBehavior](http://louishofer.com/gifs/GoalKeeperBehavior.gif "GoalKeeperBehavior")

The teams aren't very aggressive yet so very few goals are scored.
They often don't have anyone ahead of them to pass to once they regain control of the ball.
This can be improved by creating a state or player type that exists closer to the enemy goal.