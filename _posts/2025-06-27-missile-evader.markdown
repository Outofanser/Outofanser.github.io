---
layout: post
title:  "Missile Evader Blog Post #4"
date:   2025-06-27
categories: Missile Evader update
---

Now includes Fun!
------------

Welcome back! Today is a great update for Missile Evader because today we're adding fun!

Well, today we're addinng flight controls to the player so they too can fly around through the air instead of floating through it
at a constant speed...

Last update, I reformatted the missile to have an aerodynamics model, and made it abstract so I could apply much of the same
code to the player aero model. The player aero model uses a "plane" aero model which has extra terms to better describe
the different aerodynamic properties of a plane. Turns out planes and missiles fly very differently!

I also changed the input system for the player to the new(ish?) input system from Unity. This allows for binding actions
from multiple types of controllers (such as mouse and keyboard and a gamepad).

The player camera also got some tweaks! Instead of just following the player at the same exact, sometimes dizzying angle, we can instead
follow the player's *momentum*. We couldn't do that before because it had no such thing, but now it has aerodynamics the angle of the plane
and the direction of motion are not 100% tied together. This let's the player *feel* the motion better. I also added look actions on the gamepad
so you can look over your shoulder! Not implemented for mouse and keyboard yet, I haven't figured that one out.

The player aerodynamics still needs some tweaking and a few more features, and the code could use some refactoring again now that I have a better grasp of flight dynamics. Even so, the game is now what I would call playable. It doesn't have any gameplay mechanics but it is fun to fly around and get blown out of the sky! So before I get back to boring stuff, it's time to review what I've done so far.

Code
====

Most of the time there's just walls of code to review. This time I hope there's not as much. Mostly just one big thing (plane aero) and
a few small changes within the player controller. Let's start with the new Aero model.

#### PlaneAeroModel

{% highlight C# %}
public class PlaneAeroModel : AerodynamicsModel
{
...
    override public Vector3 CalculateLocalAttitudeMoment(Vector3 attitudeControl)
    {

        float pitchControlCoefficient = 2f * Mathf.PI * m_tailMaxDeflect_deg * Mathf.Deg2Rad;
        float yawControlCoefficient = 2f * Mathf.PI * m_finMaxDeflect_deg * Mathf.Deg2Rad;
        float rollControlCoefficient = 2f * Mathf.PI * m_aerolonMaxDeflect_deg * Mathf.Deg2Rad;

        float maxPitchTorque = 1 * DynPressure * m_tailArea * m_tailDistance;
        float maxYawTorque = 1 * DynPressure * m_finArea * m_tailDistance;
        float maxRollTorque = 2 * DynPressure * m_aerolonArea * m_aerolonDistance;

        float maxControlPitchTorque = maxPitchTorque * pitchControlCoefficient;
        float maxControlYawTorque = maxYawTorque * yawControlCoefficient;
        float maxControlRollTorque = maxRollTorque * rollControlCoefficient;

        Vector3 attitudeMoment_local = Vector3.zero;
        attitudeMoment_local.x = attitudeControl.x * maxControlPitchTorque;
        attitudeMoment_local.y = attitudeControl.y * maxControlYawTorque;
        attitudeMoment_local.z = attitudeControl.z * maxControlRollTorque;

        Vector3 attackAngleAxis = -Vector3.Cross(m_velocity.normalized, transform.forward);
        float attackAngle = Mathf.Asin(attackAngleAxis.magnitude);
        if (attackAngleAxis.magnitude < 0.0001) { attackAngle = 0; }


        Quaternion worldToLocalTransform = Quaternion.Inverse(transform.rotation);
        Vector3 aeroMoment_local = Vector3.zero;
        aeroMoment_local.x = -2f * Mathf.PI * Mathf.Cos(attackAngle) * m_tailDistance * (worldToLocalTransform * m_body.angularVelocity).x / m_velocity.magnitude * maxPitchTorque;
        aeroMoment_local.y = -2f * Mathf.PI * Mathf.Cos(attackAngle) * m_tailDistance * (worldToLocalTransform * m_body.angularVelocity).y / m_velocity.magnitude * DynPressure * m_bodyLongitudinalCrossSection * m_tailDistance * 0.75f;
        aeroMoment_local.z = -2f * Mathf.PI * Mathf.Cos(attackAngle) * 2f * DynPressure * m_wingArea * m_aerolonDistance * m_aerolonDistance * (worldToLocalTransform * m_body.angularVelocity).z / m_velocity.magnitude;

        return attitudeMoment_local + aeroMoment_local; // in local frame

    }
}
{% endhighlight %}

This thing is still pretty big so let's break it down. This extends the AerodynamicsModel from before, which is the abstract base class that defines the lift-draft forces of flight
and calls on the concrete subclasses to define the wing area and the control-attitude moments. As we shall see, I should have probably broken the definitions down into more chunks
as the CalculateLocalAttitudeMoment method is doing a lot.

First, note that the m_ variables are all serialized fields which I can edit in the unity editor, but belong outside this function. We calculate some
coefficients based on the max deflection angle of each control surface, and from that calculate the maximum torque of each surface based
on the dynamic pressure, control surface area, and control surface distance from the center of mass, multiplied by the number of surfaces (wing = 2).

{% highlight C# %}
    float pitchControlCoefficient = 2f * Mathf.PI * m_tailMaxDeflect_deg * Mathf.Deg2Rad;
    float yawControlCoefficient = 2f * Mathf.PI * m_finMaxDeflect_deg * Mathf.Deg2Rad;
    float rollControlCoefficient = 2f * Mathf.PI * m_aerolonMaxDeflect_deg * Mathf.Deg2Rad;

    float maxPitchTorque = 1 * DynPressure * m_tailArea * m_tailDistance;
    float maxYawTorque = 1 * DynPressure * m_finArea * m_tailDistance;
    float maxRollTorque = 2 * DynPressure * m_aerolonArea * m_aerolonDistance;

    float maxControlPitchTorque = maxPitchTorque * pitchControlCoefficient;
    float maxControlYawTorque = maxYawTorque * yawControlCoefficient;
    float maxControlRollTorque = maxRollTorque * rollControlCoefficient;

{% endhighlight %}

The max torque is multiplied by the coefficient to get the max controlled torque. We get the attitude moment
from this torque multiplied by the control signals in the pitch, yaw, and roll dimensions, each of which range from -1 to 1.
These moments are due to the air indicient on the deflected control surface creating a force. I assume aa lift force
that applies a torque, but I do not include drag. 

I also assumed that the air is always traveling stright on toward the control surface. But if the aircraft is rotating
or climbing or turning or whatever, there exists already an angle of attack that can add constructively or destructively
with the angle of incidence of the control surface. I'll try to add that next go. Below are moments due to
the control surface deflections.

{% highlight C# %}
    attitudeMoment_local.x = attitudeControl.x * maxControlPitchTorque;
    attitudeMoment_local.y = attitudeControl.y * maxControlYawTorque;
    attitudeMoment_local.z = attitudeControl.z * maxControlRollTorque;
{% endhighlight %}

Next I calculate the damping moments. When I was writing this code, I stopoped there and reviewed how things were flying.
I was able to have momentum and turning but it didn't feel right. I would yaw, but then I would just keep accelerating my turn.
Then when I let go, I would just keep going in circles. That's not right! And that's because we should have damping.

As the plane body rotates, the control surfaces brush against the air pushing as it rotates. This effectively
adds to the air speed acting on the rotating surfaces, which means our dynamic pressure is changing, which
means an additional force is acting on the surface, which means there's a torque or moment acting on the plane
due to this rotation.

If the force is acting on a surface is acting at a distance `b` (on average) with rotation `w`, then the change in velocity
acting on the surface is `bw` acting perpendicular to the velocity. The angle of the new velocity is `sin a = bw / V` where
`V` is the air speed. For small angles, this means the effective attack angle of the rotating surface is `a = bw / V`.

On the left wing, if we roll left, this force is acting upward to counteract the downward motion of the wing. The right wing, however,
is moving in the opposite direction and the force act upwards. Both of these wings then apply a moment that is in the opposite
direction of the rotation. This acts to slow down the rate of rotation until the plane stops rotating.

When applying a rotation such as a roll, the plane will accelerate its rotation until the torque applied by the control
is equal to the net torque from the damping forces, and the plane will begin to rotate at a constant speed while the control is applied.
Then, it will stop rotating after control is returned to zero. Let's see how this is crudely applied:

{% highlight C# %}
    Quaternion worldToLocalTransform = Quaternion.Inverse(transform.rotation);
    Vector3 aeroMoment_local = Vector3.zero;
    aeroMoment_local.x = -2f * Mathf.PI * Mathf.Cos(attackAngle) * m_tailDistance * (worldToLocalTransform * m_body.angularVelocity).x / m_velocity.magnitude * maxPitchTorque;
    aeroMoment_local.y = -2f * Mathf.PI * Mathf.Cos(attackAngle) * m_tailDistance * (worldToLocalTransform * m_body.angularVelocity).y / m_velocity.magnitude * DynPressure * m_bodyLongitudinalCrossSection * m_tailDistance * 0.75f;
    aeroMoment_local.z = -2f * Mathf.PI * Mathf.Cos(attackAngle) * 2f * DynPressure * m_wingArea * m_aerolonDistance * m_aerolonDistance * (worldToLocalTransform * m_body.angularVelocity).z / m_velocity.magnitude;
{% endhighlight %}

First we define a rotation to go from the world frame to the local frame. We need to do this to make our calculation in the local frame.
Each aero moment is a damping moment, and thus the negative sign on each term. The `2f * Mathf.PI` term is the familiar coefficient I've been using from the flat-sheet aerodynamics assumption
that I've been abusing. the `a = bw/V` term is e.g. ` m_tailDistance * (worldToLocalTransform * m_body.angularVelocity).x / m_velocity.magnitude` where I had to convert the angular velocity to local coordinates.
Finally we multply this by the corresponding torque acting at that attack angle e.g. `maxPitchTorque`. In some instances I need to calculate a different torque than what I used for the control surfaces, because
for example the whole plane body acts to resist the yaw motion and the wings resist the roll.

The `Mathf.Cos(attackAngle)` term is a correction on the assumption that the angle of attack was perpendicular to the rotation, which is not necessarily the case. However,
it shouldn't be he same angle for each axis moment, but it's fine for now. Just another thing to add to the pile of fixes and changes.

Finally we return the total moment `return attitudeMoment_local + aeroMoment_local;`

With that we have plane aerodynamics! Sort of! Still some missing pieces. One very large piece missing from the picture, and one that every flight enthusiast has been yelling at their screen reading this is, the center of lift.
The assumption made in the Aero model is that the force of lift and drag act directly on the center of mass. But that's not true generally for most commercial planes. Instead,
the center of lift (the point on the plane on which the force of lift acts on average) is a bit behind the center of mass. This is by design.

If the center of lift is behind the center of mass (closer to the tail), then when the plane pitches upward, the force of lift applies a torque of its own at the center of lift (or center of pressure) 
which, if it's behind the center of mass, would add a torque that pitches the plane back downwards. This gives the plane stability, and makes it much easier to fly the plane straight.
Furthermore, the tail can be adjusted to "trim" the plane so that the neutral angle of attack is a little positive, so enough lift is generated to keep the plane from falling due to gravity.
Engineering is cool!

Well, I'm not that cool yet. The center of pressure is for next time. And this calls for another refactoring of the code. First, I think that all the forces and torques should be handled by the base class,
with the concrete classes (plane and missile aero) should just be data and knobs that set the relevant lift, drag, and moment coefficients and their respective surfaces and lever arm constants.
Then, we can add center of lift.

We could go even further and make an aero model for each module of the plane. Wing, tail, body... then have the aero model apply all their forces and moments together while keeping a reference to their distances
from the center of mass. But that's hard, and more fidelity than I need for this silly game.

All we have to do now is attach the plane aero model to the player game object, add on a rigidbody, then we're good to go! Oh, and one more thing, the player controls!

#### PlayerController

We've looked at this code before, but we have to make changes to get it to work with our new aerodynamic body.

Let's open up the PlayerMovement method which has been renamed to ApplyPlayerInput

{% highlight C# %}
    void PlayerMovement()
    {
        // get inputs to attitude controls
        Vector3 playerAttitudeInput = new Vector3(-Input.GetAxis("Pitch"), -Input.GetAxis("Yaw"), -Input.GetAxis("Roll"));

        m_AeroModel.SetAttitudeControl(playerAttitudeInput);
    }
{% endhighlight %}

And that's it. That's really it. We just take the input and set it into the aero model. So simple! Well almost, we still need to kick
the plane moving forward, and we'll get to that shortly. However, I wanted to also change the input system
to Unity's new version which has a better time with mapping multiple input types (keyboard vs gamepad, etc).

So I will define some actions using the Unity interface and stick them into the player controller. First,
let's look at how we initialize the controller.

{% highlight C# %}
    void Start()
    {
        m_body.centerOfMass = m_planeCOM.transform.localPosition;
        m_body.AddForce(80f * transform.forward, ForceMode.VelocityChange);

        m_cameraOffset = m_playerCam.transform.localPosition;
        m_cameraLookAngle = m_playerCam.transform.localRotation;

        lookAction = InputSystem.actions.FindAction("FlightControl/Look");
        attitudeAction = InputSystem.actions.FindAction("FlightControl/Attitude");
        yawAction = InputSystem.actions.FindAction("FlightControl/Yaw");
    }
{% endhighlight %}

We see here that first, we give the plane a kick forward to get some velocity. I haven't added any thrust mechanics so that's all you're getting! Don't waste your speed!
We also save some info about the camera position which I'll get to later. But at the bottom we define the actions. Those actions have buttons mapped
to them for the keyboard and gamepad. The lookAction isn't working for keyboard and I haven't cracked that nut, but it works fine on gamepad. 

The attitudeAction controls yaw and pitch in x and y respectively, and the yawAction does just that. Now let's update the ApplyPlayerInput:

{% highlight C# %}
    void ApplyPlayerInput()
    {
        Vector2 attitudeInput = attitudeAction.ReadValue<Vector2>();
        Vector3 playerAttitudeInput = Vector3.zero;

        playerAttitudeInput.x = attitudeInput.y;
        playerAttitudeInput.y = yawAction.ReadValue<float>();
        playerAttitudeInput.z = -attitudeInput.x;

        // clamp the player pitch, we will add control to remove the clamp later
        playerAttitudeInput.x = MinMax(-0.3f, 0.3f, playerAttitudeInput.x);

        m_AeroModel.SetAttitudeControl(playerAttitudeInput);
    }
{% endhighlight %}

This is similar to what I had before, but I have to jumble the action inputs around to get the right values for the playerAttitudeInput.
THe attideAction is a Vector2 for a left thumbstick or wasd control, where the yawAction is a float which ranges from -1 to 1 controlled by the
left and right triggers or q e.

Also, I clamed the x component, i.e. pitch. This is for two reasons. First, it's a bandaid for poor tuning as this limits the amount of pitch you can give to the plane
and the plane still pitches *very* fast, so fast you can easily go into a stall with a simple hard flick of the stick. However, I do want the max pitch to be fast so I can
add a button later that will allow the player to unlock the clamp and effectively *drift* the plane by pitching hard. Cool! Or it would be if it was done :)

Finally I added some camera rotations and this comes in two types: camera rotations due to player lookAction inputs (right thumbstick) and camera rotations
so the camera follows the player *momentum*.

{% highlight C# %}
    void ApplyCameraRotation()
    {
        m_playerCam.transform.localRotation = m_cameraLookAngle;
        m_playerCam.transform.localPosition = m_cameraOffset;

        Vector2 lookInput = lookAction.ReadValue<Vector2>();

        m_playerCam.transform.RotateAround(transform.position, transform.right, maxVertLookAngle_deg * lookInput.y);
        m_playerCam.transform.RotateAround(transform.position, transform.up, maxHorzLookAngle_deg * lookInput.x);

        Vector3 attackAngleAxis = Vector3.Cross(transform.forward, m_body.linearVelocity.normalized);
        float attackAngle = Mathf.Asin(attackAngleAxis.magnitude);
        m_playerCam.transform.RotateAround(transform.position, attackAngleAxis, attackAngle*Mathf.Rad2Deg);

    }
{% endhighlight %}

First we set the local rotation and position of the camera back to its original spot. This is so we don't add rotations on top of rotations during each update.
Then we apply the lookInput (also a Vector2) to rotate the camera around the pitch and yaw directions, so we can look above/below the plane and from the sides.
The angles defined are the max angles we can rotate the camera, which for the vertical is 90 degrees and for the horizontal it is 170 degrees, so pretty
far over the shoulder but not directly behind. Cool!

But this next part is double-cool. I do my familiar trick to get the attack angle of the plane (the angle between the plane's attitude and actual velocity)
as well as the axis of rotation. Then I apply the rotation to the player camera. These rotations must be about the plane's transform.position so
that we keep the camera at the same distance looking at the plane.

What the attack angle rotation does is it places the camera in line with our velocity, so we we pitch up or yaw we can see the plane tilt a little as we turn.
It gievs the player much better feedback about how hard they're turning and where they're moving. And it looks cool! Also, if you do lose control
and stall out, the camera will show you tumbling down like a rock :)

The camera rotation and player inputs are applied each update, and I won't show the code here, just trust. The player input does not need to be on the FixedUpdate
because we are merely setting the AeroModel's controls on each update, then the aero model will effect those controls on the FixedUpdate.

That about does it for this update! There's missiles flying and planes gliding now. There are no points, collectables, or UI yet but the "game" is still pretty fun to mess with.
The missiles are dodgeable but it comes at a cost of speed and it's a limited resource.

You can try the game (or whatever the latest version of it is) [here]. Have fun! Just know that almost every aspect of it it still up to change.

Until next time,


-Outofanser


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
