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

[Early Demo](https://play.unity.com/en/games/e7c47b99-b777-4a4a-968b-e232121982e1/missile-evader)

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

1. (UPDATE) Player has an aerodynamic model, but it's not tuned and is missing some features. It also features
a dynamic camera that follows player momentum and has a lookInput control.

2. Missiles are much more along, but need balance adjustments. The missiles include aerodynamics, 
navigation (acceleration commands) and a PID controller to execute.

3. There are no powerups or pickups or a real sky or a real ground. Just placeholder assets, though I'm proud of my prototype missile.

4. There is a little missile camera that follows a single missile out in flight, mostly for debugging, but it's actually really cool to watch. 
I think I will keep this as some sort of feature.

#### Aero Forces Code Snippet
<figure tile="AeroForces Code snippet">
{% highlight C# %}
    virtual public Vector3 CalculateAeroForces()
    {
        Vector3 body2velAngle = Vector3.Cross(m_body.transform.forward, m_velocity);
        Vector3 liftDirection = Vector3.Cross(m_velocity, body2velAngle).normalized;

        float attackAngle = Vector3.Angle(m_body.transform.forward, m_velocity);

        float liftCoefficient = 2f * Mathf.PI * attackAngle * Mathf.Deg2Rad;
        float dragCoefficient = 1f * Mathf.Pow(attackAngle * Mathf.Deg2Rad, 2f);

        if (attackAngle > m_stallAoA_deg) // Stalling condition
        {
            liftCoefficient = Mathf.Sin(2f * attackAngle * Mathf.Deg2Rad);
            dragCoefficient = 1f - Mathf.Cos(2f * attackAngle * Mathf.Deg2Rad);
        }


        Vector3 liftForce = liftDirection * liftCoefficient * DynPressure * WingArea;
        Vector3 dragForce = -m_velocity.normalized * dragCoefficient * DynPressure * WingArea;

        return liftForce + dragForce; // in world frame
    }
{% endhighlight %}
<figcaption>Code to Generate the aerodynamic forces -- needs refactoring and some minor changes</figcaption>
</figure>

Todo
====

1. More aerodynamics code refactoring - get the forces and moments organized

2. Add center of lift to aero model.

3. Add thrust and fuel to player

4. Add UI to display speed, altitude (eventually, add artificial horizon)


<img src="/Missile_Evader/images/CameraRotation.png" alt="Inbound" title="Missiles Inbound with camera rotation"/>

[jekyll-organization]: https://github.com/jekyll
