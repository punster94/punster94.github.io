---
layout: post
title:  "Team and Field Player State Machines"
date:   2017-6-18 00:00:00 +0200
tags: ['c/c++', 'Unreal', 'UE4']
author: "Louis Hofer"
---

I've created basic state machines before that make use of enumerations and switch statements due to their simplicity and the fact that they probably won't need to be expanded upon later.
In the case of this soccer simulation, I may want to be updating the number of states as well as how to transition between them several times during the simulation's development.
Due to this it is important to more closely follow the state pattern by defining classes for each state and generically handle the transitions between.

I accomplished this with a templated class StateMachine:

```c
template <class T>
class StateMachine
{

public:

	StateMachine(const T& owner);

	virtual ~StateMachine();

	void SetCurrentState(State<T>& s);

	void SetPreviousState(State<T>& s);

	void SetGlobalState(State<T>* s); // can be nullptr

	void Update() const;

	void ChangeState(State<T>& newState);

	void RevertToPreviousState();

	bool IsInState(const State<T>& s) const;

	State<T>& GetCurrentState() const;

	State<T>& GetPreviousState() const;

	State<T>& GetGlobalState() const;

private:

	T& Owner;

	State<T>* CurrentState;

	State<T>* PreviousState;

	State<T>* GlobalState;
};

#include "StateMachine.inl"
```

In this class, the set global state will execute every update, followed by the current state.
The majority of the state magic happens in the ChangeState method.
This method maintains the previous state pointer, exits the current state, and enters the new state.
It is up to the State instance to determine what exiting and entering should do, as well as what happens on execution during update.

The State class itself is a very simple abstract class with an Enter, Execute, and Exit method:

```c
template <class T>
class State
{

public:

	virtual ~State() {};

	virtual void Enter(T&) = 0;

	virtual void Execute(T&) = 0;

	virtual void Exit(T&) = 0;
};
```

With the framework for a state pattern set up, specific states can now be implemented.
I started with team states, which I can show all of as there are only three.
Each state acts as a singleton to allow other states to obtain a reference to switch without the need to construct.
Additionally, the states themselves do not have "state" (data members) so the same static instance can be used by several objects at once:

```c
Attacking* Attacking::Instance()
{
	static Attacking instance;

	return &instance;
}

void Attacking::Enter(ASoccerTeam& team)
{
	team.SetAllPlayersToAttackMode();

	team.UpdateTargetsOfWaitingPlayers();

	GEngine->AddOnScreenDebugMessage(-1, 10.0f, FColor::Green, TEXT("Entering Attack State"));
}

void Attacking::Execute(ASoccerTeam& team)
{
	if (!team.InControl())
	{
		team.GetStateMachine().ChangeState(*Defending::Instance());
	}
	else
	{
		//team.DetermineSupportSpot();
	}
}

void Attacking::Exit(ASoccerTeam& team)
{
	team.SetSupportingPlayer(nullptr);
}

Attacking::Attacking()
{
}

Defending* Defending::Instance()
{
	static Defending instance;

	return &instance;
}

void Defending::Enter(ASoccerTeam& team)
{
	team.SetAllPlayersToDefenseMode();

	team.UpdateTargetsOfWaitingPlayers();

	GEngine->AddOnScreenDebugMessage(-1, 10.0f, FColor::Yellow, TEXT("Entering Defense State"));
}

void Defending::Execute(ASoccerTeam& team)
{
	if (team.InControl())
	{
		team.GetStateMachine().ChangeState(*Attacking::Instance());
	}
}

void Defending::Exit(ASoccerTeam& team)
{
	team.SetSupportingPlayer(nullptr);
}

Defending::Defending()
{
}

PrepareForKickoff* PrepareForKickoff::Instance()
{
	static PrepareForKickoff instance;

	return &instance;
}

void PrepareForKickoff::Enter(ASoccerTeam& team)
{
	team.SetControllingPlayer(nullptr);
	team.SetSupportingPlayer(nullptr);
	team.SetReceiver(nullptr);
	team.SetPlayerClosestToBall(nullptr);

	team.UpdateTargetsOfWaitingPlayers();

	GEngine->AddOnScreenDebugMessage(-1, 10.0f, FColor::Red, TEXT("Entering PrepareForKickoff State"));
}

void PrepareForKickoff::Execute(ASoccerTeam& team)
{
	if (team.AllPlayersAtHome() && team.GetOpponentTeam()->AllPlayersAtHome())
	{
		team.GetStateMachine().ChangeState(*Defending::Instance());
	}
}

void PrepareForKickoff::Exit(ASoccerTeam& team)
{
	team.GetField()->SetGameOn(true);
}

PrepareForKickoff::PrepareForKickoff()
{
}
```

I have the basic states for field players (non-goalies) set up but several functions they perform are spec-ed out as of now.
Here is a demonstration of the players moving based on their team and individual states:

![TeamStateDemo](http://louishofer.com/gifs/TeamStateDemo.gif "TeamStateDemo")

Now comes finalizing the functionality of actions performed in the states of the field players.