---
layout: post
title:  "Missile Evader Blog Post #3"
date:   2025-06-21
categories: Missile Evader update
---

Even More Refactoring
------------

So last time I mentioned that I am doing the Unity Learn thing and my goal was to apply the OOP principles:

1. Abstraction
2. Encapsulation
3. Inheritance
4. Polymorphism

Last time, we focused on Abstraction and Encapsulation and we did those by refactoring the monolithic Missile Controller
into sensible components with more tightly controlled interfaces between components.

Great!

Now do it again! But the goal of this refactoring is to make the Aerodynamics Model applicable to more than just the missile. We will
do this by applying inheritance and polymorphism. There will be a base Aerodynamics class which defines the basic
operations and properties of the aero model, such as applying forces, attitude controls, and determining the dynamic pressure. Then there will
be a concrete subclass (Missile Aero Model) which determines *how* to calculate the attitude control forces while holding fields which determine
the aero properties of the control surfaces.

Great!

But there's a problem. See, the point of doing this is to eventually add an aero model for the player, which is flying a plane. This means
we will have to have a control axis for yaw, pitch, and roll, and we will want the controls to operate on each of these axes independently.
The current design is not very suitable for this, as we defined a control vector which applied a torque in the world coordinates,
which should just be along the axes for yaw and pitch (treating them the same for the missile) but it was not obvious that it would.
Not only was it hard to reason, it was very inflexible and hard to debug.

Furthermore, the PID controller also operated on this three dimensional vector that just exists in some world space.
It's meant to represent something more restricted, a two dimensional control for yaw/pitch. Even further-furthermore,
there was only one PID controller, which was a component of the missile. How can we adjust the gains for yaw-pitch-roll
independently if we only have one PID controller?

The answer is to apply our controls in the local coordinate frame and use indpenedent PID controllers for each axis.
This will make my life a lot easier once it comes to making the player aero model.

In the meantime...
We have some additional refactoring to do.

Also, while I'm at it, there's some physics and math going on that could really use some explaining!

Code
====

Before we get any inheritance/polymorphism, we got some code to fix. First, let's redefine our PID controllers. Instead
of being a Vector3 whose magnitude ranges from 0 to 1 let it be a float that ranges from -1 to 1:

#### PIDController
{% highlight C# %}
public class PIDController
{
...
    public float PID(float error)
    {
        float dError = (error - m_errorLast) / Time.fixedDeltaTime;
        float iError = m_errorAccumulator + error * Time.fixedDeltaTime;

        if (m_errorLast == 0)
        {
            dError = 0;
        }

        float PID = pGain * error + dGain * dError + iGain * iError;

        if (PID > 1)
        {
            PID = 1;
        }
        else if (PID < -1)
        {
            PID = -1;
        }

        m_errorLast = error;
        m_errorAccumulator += error;

        return PID;
    }

    public void Reset()
    {
        m_errorLast = 0;
        m_errorAccumulator = 0;
    }
{% endhighlight %}

First thing to notice is this is no longer a Monobehavior. We do not want to attach this to the missile as a component.
This is because we need multiple controllers and we would need to bind them anyway, so we will let the Autopilot take care of it.
I also added a Reset function to reset the errorLast and errorAccumulator, though I don't use this function anywhere, it seemed logical to have it.
This is pretty straight forward so we shall move on.

Next let's go to the Auto Pilot, which has a few important changes itself.

#### AutoPilotController
First, it's now responsible for the PID controllers, so we'll initialize them.

{% highlight C# %}
public class AutoPilotController : MonoBehaviour
{
...
    [Header("Vertical Attitude PID")]
    [SerializeField] private PIDController m_vertPIDController;
    [Header("Horizontal Attitude PID")]
    [SerializeField] private PIDController m_horzPIDController;
    [Header("Roll Attitude PID")]
    [SerializeField] private PIDController m_rollPIDController;
...
    void Awake()
    {
        //m_pidController = GetComponent<PIDController>();
        m_vertPIDController = new PIDController();
        m_horzPIDController = new PIDController();
        m_rollPIDController = new PIDController();

        m_AeroModel = GetComponent<AerodynamicsModel>();
        m_body = GetComponent<Rigidbody>();
    }
{% endhighlight %}

We initialize the controllers in Awake now. In order to make them modifiable in the Unity editor, we add the Header and Serialize Field
to their definitions. This gives me the option to change the gains within the AutoPilotController component while using
a descriptive header to organize my gains together. Hurray.

Next, we will edit the GetControl function, which is now called ComputeAutoPilotControl!

{% highlight C# %}
    public Vector3 ComputeAutoPilotControl()
    {
        float area = m_AeroModel.WingArea;
        float dynamicPressure = m_AeroModel.DynPressure;

        // Lift Force ~= 2pi * AoA * dynP * area
        float desiredAngleOfAttack = m_accelerationCmd.magnitude * m_body.mass / area / dynamicPressure / (2 * Mathf.PI);
        float attackLimited = Mathf.Min(desiredAngleOfAttack * Mathf.Rad2Deg, m_angleOfAttackLimit_deg);

        // Compute the desired forward direction to achieve the AoA
        Vector3 turnAxis = Vector3.Cross(m_velocity, m_accelerationCmd).normalized;
        Vector3 desiredForward = Quaternion.AngleAxis(attackLimited, turnAxis) * m_velocity.normalized;

        // Calculate the error angle from this new desiredForward
        Vector3 misalignmentAxis = -Vector3.Cross(desiredForward, m_body.transform.forward);
        float misalignmentAngle = Mathf.Asin(misalignmentAxis.magnitude);

        Quaternion worldToBodyRotation = Quaternion.Inverse(transform.rotation);

        Vector3 alignmentError_local = worldToBodyRotation * misalignmentAxis.normalized * misalignmentAngle;  // z component should be zero

        // get the error rate to calculate the control for this time step
        Vector3 alignmentErrorRate_local = alignmentError_local / m_timeConstant; // Slew control to moderate the error rate
        Vector3 attitudeControl = Vector3.zero;
        attitudeControl.x = m_vertPIDController.PID(alignmentErrorRate_local.x/2f/Mathf.PI);
        attitudeControl.y = m_horzPIDController.PID(alignmentErrorRate_local.y/2f/Mathf.PI);

        // skip doing roll control for now I don't need it -- need a better method
        //Vector3 angularVelocity_local = worldToBodyRotation * m_body.angularVelocity;
        //attitudeControl.z = m_rollPIDController.PID(-angularVelocity_local.z/2f/Mathf.PI);

        // could also add a cascade PID control: sets a desired rate from the error in outer PID and an inner PID controls the error rate

        return attitudeControl;
    }
{% endhighlight %}

This code is actually mostly the same as before, despite appearances. I renamed a lot of stuff. I actually asked chatGPT if the code even made any sense.
Surprisingly, it did understand what I was trying to do. Then it gave me some ways to optimize the code that either wouldn't actually do anything
or was tedious so I moved on. But it did have some naming suggestions! I prefer to only use AI to name variables than to write my code anyhow...

Let's review for real this time what this code *actually* does, there's some cool physics!

First, we get the desired angle of attack. This equation is based on a flat-sheet aerodynamics model where
the lift force is proportional to the angle of attack (measured in radians)
`L = 2π * AoA * dynamicPressure * SurfaceArea`
Solving this equation for AoA gets us the desired angle of attack.
{% highlight C# %}
// Lift Force ~= 2pi * AoA * dynP * area
float desiredAngleOfAttack = m_accelerationCmd.magnitude * m_body.mass / area / dynamicPressure / (2 * Mathf.PI);
float attackLimited = Mathf.Min(desiredAngleOfAttack * Mathf.Rad2Deg, m_angleOfAttackLimit_deg);
{% endhighlight %}

*Buuut* if we are going slow or the target is moving fast, our acceleration command may lead to us making an extreme turn.
This is bad, because the nice, proportional lift force becomes quadratic with the angle of attack which kills our ability to fly.
We go from sailing through the air to falling like a rock. Or at least we should, I suppose I don't have to obey
the laws of physics. To avoid falling out of sky, the best solution is to limit our angle of attack.

Next, we determine the direction to orient the missile. 
The way I do this is by defining the axis of rotation between the missile's velocity and the acceleration command.
We want the velocity to change in this axis, but we have the AoA defined as the orientation angle relative to
the current velocity.
{% highlight C# %}
// Compute the desired forward direction to achieve the AoA
Vector3 turnAxis = Vector3.Cross(m_velocity, m_accelerationCmd).normalized;
Vector3 desiredForward = Quaternion.AngleAxis(attackLimited, turnAxis) * m_velocity.normalized;
{% endhighlight %}
We define the desiredForward as the rotation of the velocity about this axis.
The product of the "Quaternion" and the vector is a rotation. I use quaternions in quotes because Unity hides
the actual implementation of Quaternion rotation to make it easy for the plebs.

I could make an entire blogpost on Quaternions but it would quickly devolve into a rant about geometric algebra
and how the electromagnetic force is actually a Bivector in space-time. For now, just know that Quaternions are
four numbers that encode rotations of 3D vectors, and Unity is lying to you. Always lying...

We have our desired orientation, but we need to know how far off we are from our goal so we can calculate an error singal.
{% highlight C# %}
// Calculate the error angle from this new desiredForward
Vector3 misalignmentAxis = -Vector3.Cross(desiredForward, m_body.transform.forward);
float misalignmentAngle = Mathf.Asin(misalignmentAxis.magnitude);
{% endhighlight %}
The misalignment axis gets the axis of rotation of our missile forward and the desired forward, and the magnitude
of this cross product is `sin(misalignmnetAngle)`. Note how this is different than the turnAxis we defined before,
it's not the angle between our velocity and our desired orientation, but the angle between our physical orientation
and the desired orientation.

If we just started our maneuver, the desiredForward would be the same as the missile velocity doesn't change immediately,
but the misalignment angle would decrease as we approach the appropriate angle of attack.

Ok, next we put things into local coordinates. This is so that we can define our attitude controls relative to the missile body,
which is the coordinate system our fins actually operate in. We do this with more Quaternions!
{% highlight C# %}
Quaternion worldToBodyRotation = Quaternion.Inverse(transform.rotation);

Vector3 alignmentError_local = worldToBodyRotation * misalignmentAxis.normalized * misalignmentAngle;  // z component should be zero
{% endhighlight %}

The `transform.rotation` defines a rotation from world space to the object's local orientation. The quaternion defines a rotation as the inverse of the transform's rotation, so that it will transform a world space vector to local coordinates.
The error signal we get then is the misalignment axis in local coordinates (which should be only in x-y) multiplied by the angle.

Next, we do some normalization as we did before and apply our PID controllers
{% highlight C# %}
// get the error rate to calculate the control for this time step
Vector3 alignmentErrorRate_local = alignmentError_local / m_timeConstant; // Slew control to moderate the error rate
Vector3 attitudeControl = Vector3.zero;
attitudeControl.x = m_vertPIDController.PID(alignmentErrorRate_local.x/2f/Mathf.PI);
attitudeControl.y = m_horzPIDController.PID(alignmentErrorRate_local.y/2f/Mathf.PI);
{% endhighlight %}

The `m_timeConstant` works as a slew rate controller which keeps the missile from correcting too quickly. It's not completely realistic
but it works great for what I want it to do, make the missile lag a little.
We then define a control vector, now in local coordinates, and apply a PID to each component. We also normalize
the error by `2π` to make tuning the PIDs a little easier.

We could also include a z axis for roll, and I did try it but it's buggy and right now it's not necessary.

We then return the attitudeControl to the missile so it can hand it off to the new Aero model.

Speaking of which, it's now finally time to talk about the Aero model. We'll actually
start with the concrete subclass MissileAeroModel.

#### MissileAeroModel

The only responsibility of the Missile Aero Model is to determine the attitude control characteristics.
It defines the familiar CalculateLocalAttitudeMoment, which is a renamed version of GenerateControlTorque.
It still takes a Vector3 control input, but now the control is in local coordinates.

There are also a few fixes and changes to the calculation as we shall see

{% highlight C# %}
public class MissileAeroModel : AerodynamicsModel
{
    [SerializeField]
    private float m_finArea = 0.1f;
    [SerializeField]
    private float m_finMaxDeflect_deg = 40f;
    [SerializeField]
    private float m_tailToCMDistance = 2.5f;
    [SerializeField]
    private float m_missileRadius = 0.3f;
    [SerializeField]
    private float m_wingArea = 1f;
    public override float WingArea { get { return m_wingArea; } }

    override public Vector3 CalculateLocalAttitudeMoment(Vector3 attitudeControl)
    {

        if (attitudeControl.magnitude > 1)
        {
            attitudeControl = attitudeControl.normalized;
        }

        float liftCoefficient = 2f * Mathf.PI * m_finMaxDeflect_deg * Mathf.Deg2Rad; //* Mathf.Deg2Rad;

        float maxTorque = 4 * DynPressure * liftCoefficient * m_finArea * m_tailToCMDistance;
        float maxRollTorque = 4 * DynPressure * liftCoefficient * m_finArea * m_missileRadius;
        Vector3 attitudeMoment_local = Vector3.zero;
        attitudeMoment_local.x = attitudeControl.x * maxTorque;
        attitudeMoment_local.y = attitudeControl.y * maxTorque;
        attitudeMoment_local.z = attitudeControl.z * maxRollTorque;

        return attitudeMoment_local; // in local frame

    }
}
{% endhighlight %}

We see that the fields are all related to the control surfaces of the missile.

We have one method which implements the base class, CalculateLocalAttitudeMoment, which has a few small changes.
It is now responsible for normalizing the control input.
I also fixed the lift coefficient (based on the same flat-sheet lift model) to convert the max find deflection to radians.
This change required some readjustment of some of the other aero properties and the PID gains to get the missile
to behave the way I liked again. Now, I like it even more!

We calculate the maximum possible torque we can apply, which will scale down by our control. We do the same for roll
using a different lever distance, which would reduce the amount of roll torque we get by moving our fins. We don't use
the roll controls though but it's there.

We finally get our attitude Moments but these are in the local coordinate frame. We will see in the base class how this is applied.

#### AerodynamicsModel

The AerodyanicsModel base class will apply our forces to the rigid body as well as accept control inputs. Let's take a look.

{% highlight C# %}
abstract public class AerodynamicsModel : MonoBehaviour
{
...
    protected Vector3 m_control;
    public abstract float WingArea { get; }
    
    void Awake()
    {
        m_body = GetComponent<Rigidbody>();
        m_velocity = Vector3.zero;
    }

    void Update()
    {
        m_velocity = m_body.linearVelocity;
    }
{% endhighlight %}

We start by defining the control vector we will feed to the CalculateLocalAttitudeMoment. The Aero model will
be in charge of setting this vector. It also initializes the model. So far so boring. Next, let's take
a look at a familiar method, which as been renamed to CalculateAeroForces.

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

Note that this method is virtual, so it's overridable, but it defines a default implementation.
That default is the flat-sheet lift model I mentioned earlier. Let's explain this one a little more too.

{% highlight C# %}
Vector3 body2velAngle = Vector3.Cross(m_body.transform.forward, m_velocity);
Vector3 liftDirection = Vector3.Cross(m_velocity, body2velAngle).normalized;
{% endhighlight %}
To get the lift direction, we get the direction perpendicular to our velocity and our forward orientation, 
i.e. rotation axis for the the angle of attack.
The lift direction is then perpendicular to this and the velocity.

We define the angle of attack and calculate the lift and drag coefficients using the flat-sheet model.
{% highlight C# %}
float attackAngle = Vector3.Angle(m_body.transform.forward, m_velocity);

float liftCoefficient = 2f * Mathf.PI * attackAngle * Mathf.Deg2Rad;
float dragCoefficient = 1f * Mathf.Pow(attackAngle * Mathf.Deg2Rad, 2f);
{% endhighlight %}

The drag component is quadratic, according to this simple model. How is this model derived? Well, the study
of aerodynamics is witchcraft and alchemy rolled into a vortex that applies a sheer force... or something like that.
I don't really understand it myself, maybe I'll learn and make a blog post about it. But now we move on.

What if we have a high angle of attack? Don't we stall? Yes we do!
{% highlight C# %}
if (attackAngle > m_stallAoA_deg) // Stalling condition
{
    liftCoefficient = Mathf.Sin(2f * attackAngle * Mathf.Deg2Rad);
    dragCoefficient = 1f - Mathf.Cos(2f * attackAngle * Mathf.Deg2Rad);
}
{% endhighlight %}

Now we have the more "traditional" quadratic terms in our stall condition. This I can explain where it comes from.
If we assume the incoming air to the wing are just balls of momentum, the force impacted would create a force
in the direction of the attack angle with magnitude `sin(AoA)`. This force then decomposes into a lift and drag
component by multiplying by an additional `cos(AoA)` and `sin(AoA)` respectively. Then it's trig to simplify the coefficients.

We will use the coefficients to calculate the lift and drag forces

{% highlight C# %}
Vector3 liftForce = liftDirection * liftCoefficient * DynPressure * WingArea;
Vector3 dragForce = -m_velocity.normalized * dragCoefficient * DynPressure * WingArea;

return liftForce + dragForce; // in world frame
{% endhighlight %}

The lift force is the force unit vector times the coefficient times the pressure times the area. F = P * A.
The drag force is the same but its direction is in the opposite direction of velocity.
We then return these forces. Note they are calculated in the world frame because all our vector calculates
are from vectors in this frame.

Let's look at a couple more methods this class has:

{% highlight C# %}
    abstract public Vector3 CalculateLocalAttitudeMoment(Vector3 attitudeControl);

    virtual public void SetAttitudeControl(Vector3 control)
    {
        m_control = control; // Could add slew rates
    }
{% endhighlight %}

We define the abstract CalculateLocalAttitudeMoment which each subclass must implement. 
We also define a vritual SetAttitudeControl which by default will just act as a basic setter for m_control.
But we could add slew controls or actuators for our fins, etc, and do some cool stuff! But later.

Finally for this class, the Aeromodel will now be in charge of applying the aerodynamic forces.

{% highlight C# %}
    void FixedUpdate()
    {

        Vector3 torque_local = CalculateLocalAttitudeMoment(m_control);
        Vector3 aeroForces = CalculateAeroForces();

        m_body.AddForce(aeroForces);

        if (torque_local.magnitude > 0.001)
        {
            m_body.AddRelativeTorque(torque_local);
        }
    }
{% endhighlight %}

Note the torque is in the local frame, so we use AddRelativeTorque instead.
The m_control must be set by the missile and is held by the Aero model. Think of it like the aero model
is holding a reference to its fin positions and such, but less complicated.
This method is mostly the same as was in Missile, so how does the Missile class change?

#### Missile

Finally, we get to our last bit of change, the missile!
It's very anticlimatic. We remove the FixedUpdate method as we do all that in the Aero model.
Instead, we pass the controls from the AutoPilot to the Aeromodel

{% highlight C# %}
    void Update()
    {
    ...
        Vector3 attitudeControl = m_AutoPilot.ComputeAutoPilotControl();

        m_AeroModel.SetAttitudeControl(attitudeControl);
    }
{% endhighlight %}

That's it! Very simple change there. We don't even have to change how we get the m_AeroModel, we still use
`m_AeroModel = GetComponent<AerodynamicsModel>();` and simply attach the MissileAeroModel component to the missile game object.
It will be found because MissileAeroModel is an AerodynamicsModel. Inheritance! Polymorphism! Are we done here?

I think with this we conclude our most recent changes. We are setting ourselves up
to apply an Aero model to the player, and with that, actual flight physics for the player. Soon, the player will get to have
fun too! What a concept!

But this also concludes the Unity Learn With Code series for me it seems like. I applied the four principles and I've
completed the lessons it had for me. There will be other lessons, but it's also put me on a path of self learning.

That does it for this post. I hope that you learned something from reading this!

Until next time,

-Outofanser


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
