---
layout: page
title: Missile Evader
permalink: /missile-evader/
---

Overview
--------
Missile Evader (title not final) is a personal Unity project to make a small game about flying a plane and dodging missile.
This project was started as part of the self-study portion of the Unity Learn with Code series.
It's mostly a sandbox for me to apply my physics background while learning flight dynamics and Unity physics, as well as other basic Unity gameplay features.

<img src="/Missile_Evader/images/gameView.png" alt="Gameplay" title="Gameplay so far"/> 
--------------

Goals
=====
The goals for this project are to have the following features:

1. Player controlled plane with yaw-pitch-roll controls and quasi-realistic flight dynamics.

2. A missile with quasi-realistic flight dynamics and a navigation system that can track the player.
This navigation system should be tunable to provide a scalable level of difficulty for the player to dodge.
The missile will also detonate at the point of closest approach to its target.

3. The player consumes fuel as they fly and must acquire fuel to stay in flight. The player can use afterburners
which consumes fuel to improve maneuverabiltiy.

4. The player may acquire other power ups besides fuel, including powerups to repair damage, or give the player consumable
flairs to help them dissuade incoming missiles.

5. The game manager (or similar entity) will spawn multiple missiles and missile may have varying maneuverability to ramp up the difficulty.

There are many other things that could be added. The missile spawners could be enemies and the player could be armed with guns/missiles, 
or there could be other collectables for the player to use to optimize their score. But for now, the main focus is on getting
missile and player flight to feel good and for dodging missiles to be fun. I am not there yet -.-

<img src="/Missile_Evader/images/EditorView1.png" alt="Dodge" title="Dodging the missile" width="250"/> 
<img src="/Missile_Evader/images/EditorView2.png" alt="Explode" title="Missile explodes" width="450" />

---------------

Status
======

1. Player controls are very basic and not physics based. Just there for testing with a constant speed and turn rates.

2. Missiles are much more along, but needs code refactoring and balance adjustments. The missiles include aerodynamics, 
navigation (acceleration commands) and a PID controller to execute. But there's little to control how well it can track its target,
and right now they are very hit-or-miss and not in a fun way. What it needs is a delay between command and actuation, i.e. an auto-pilot time delay.

3. There are no powerups or pickups or a real sky or a real ground. Just placeholder assets, though I'm proud of my prototype missile.

4. There is a little missile camera that follows a single missile out in flight, mostly for debugging, but it's actually really cool to watch. 
I think I will keep this as some sort of feature.

#### Aero Forces Code Snippet
<figure tile="AeroForces Code snippet">
{% highlight C# %}
    Vector3 GenerateAeroForces(Vector3 forward, Vector3 missileVelocity, float wingArea)
    {

        Vector3 temp1 = Vector3.Cross(forward, missileVelocity);
        Vector3 temp2 = Vector3.Cross(missileVelocity, temp1);
        Vector3 lift_direction = temp2.normalized;

        float vel = missileVelocity.magnitude;
        float dynamicPressure = 0.5f * rho * vel * vel;
        attackAngle = Vector3.Angle(forward, missileVelocity);

        float lift_coefficient = 2f * Mathf.PI * attackAngle * Mathf.Deg2Rad;
        float drag_coefficient = 1f * Mathf.Pow(attackAngle * Mathf.Deg2Rad, 2f);
        //float moment_coefficient = 0.01f * attackAngle * Mathf.Deg2Rad;
        if (attackAngle > 12)
        {
            lift_coefficient = Mathf.Sin(2f * attackAngle * Mathf.Deg2Rad);
            drag_coefficient = 1f - Mathf.Cos(2f * attackAngle * Mathf.Deg2Rad);
        }


        Vector3 lift_force = lift_direction * lift_coefficient * dynamicPressure * area;
        Vector3 drag_force = -missileVelocity.normalized * drag_coefficient * dynamicPressure * area;

        return lift_force + drag_force;
    }
{% endhighlight %}
<figcaption>Code to Generate the aerodynamic forces -- needs refactoring and some minor changes</figcaption>
</figure>

Todo
====
1. Convert to Unity 6

2. Apply auto-pilot delay to missile controller

3. Refactor missile controller and abstract out aerodynamics module

4. Apply aerodynamics and physics based controls to player (easier said than done).

[jekyll-organization]: https://github.com/jekyll
