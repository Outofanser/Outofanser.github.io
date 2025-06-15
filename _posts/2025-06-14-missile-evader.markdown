---
layout: post
title:  "Missile Evader Blog Post #2"
date:   2025-06-14
categories: Missile Evader update
---

The Great Refactoring
------------

Part of the reason this project was started was the Unity Learn With Code series.
It started as the self-study project. But now I'm on the final lesson, and part of
that lesson is to take a project and apply the four principles of Object Oriented design.

1. Abstraction
2. Encapsulation
3. Inheritance
4. Polymorphism

Today we are focusing on the first two. Part of the refactoring is making chunks of code that make sense
as their own components. The component owners don't need to know the details of its inner workings, just
an interface to get what it needs from the component. This is abstraction.

Abstraction works best if different components can only 'talk' to eachother through purposeful interfaces,
where we hide internal methods and fields while exposing public methods and properties. This makes it easier
to modify what is intended. Protecting the data from unintended uses like this is called encapsulation.
I will also adopt a variable naming convention that emphasises the encapsulation.

Refactoring the missile controller is a great way to practice these two principles! Also, it was
really really *really* necessary. Before this, the missile controller did everything missile related.
It determined the aerodynamics of the missile, the autopilot controls, the arming and exploding,
and all the sound management. 

Well today we're changing the missile controller by splitting up some of its biggest parts, the aerodyanimics
and the autopilot, into separate components. We're also renaming the MissileController into just Missile
because I was getting sick of everything being called "Controller" but that's less important.

Also! I added a small way to make the missile more evadable! Yay! We will see how this is done in the AutoPilot section.

Let's get into some changes.

Code
====

We got a few new components to add to our missile, let's start with the Aerodynamics Model.
First, I want to show an example of how I've learned to do proper variable naming and encapsulation.

#### AerodynamicsModel
{% highlight C# %}
public class AerodynamicsModel : MonoBehaviour
{
    private const float c_rho = 1.293f;
    [SerializeField]
    private float m_wingArea = 1f;
    [SerializeField]
    private float m_tailArea = 0.1f;
    [SerializeField]
    private float m_maxDeflect = 10f;
    [SerializeField]
    private float m_tail2CMDistance = 0.5f;
    private Rigidbody m_body;
    [SerializeField]
    private Vector3 m_velocity;
    public float DynPressure
    {
        get
        {
            return 0.5f * c_rho * Mathf.Pow(m_velocity.magnitude, 2);
        }
    }
    public float WingArea { get { return m_wingArea; } private set { m_wingArea = value; } }
{% endhighlight %}

Here, m_varName is a private field (though C# suggests just \_varName for private fields),
c_varName is a constant, and the PascalCase (e.g. DynPressure) is a public property.
I won't review this for every component, but I want to show how it can make
the code easier to read because you can distinguish fields from local variables and the likes.

With that out of the way, let's review the rest of the code.

{% highlight C# %}
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

This component holds a reference to the rigid body (many of the components do) and updates its velocity, which
as you see above is used to calculate the DynPressure (dynamic pressure).

Next we will see some familiar methods which used to be in MissileController.

{% highlight C# %}
    public Vector3 GenerateAeroForces()
    {
        Vector3 body2velAngle = Vector3.Cross(m_body.transform.forward, m_velocity);
        Vector3  lift_direction= Vector3.Cross(m_velocity, body2velAngle).normalized;

        float attackAngle = Vector3.Angle(m_body.transform.forward, m_velocity);

        float lift_coefficient = 2f * Mathf.PI * attackAngle * Mathf.Deg2Rad;
        float drag_coefficient = 1f * Mathf.Pow(attackAngle * Mathf.Deg2Rad, 2f);

        if (attackAngle > 12) // Stalling condition
        {
            lift_coefficient = Mathf.Sin(2f * attackAngle * Mathf.Deg2Rad);
            drag_coefficient = 1f - Mathf.Cos(2f * attackAngle * Mathf.Deg2Rad);
        }


        Vector3 lift_force = lift_direction * lift_coefficient * DynPressure * m_wingArea;
        Vector3 drag_force = -m_velocity.normalized * drag_coefficient * DynPressure * m_wingArea;

        return lift_force + drag_force;
    }

    public Vector3 GenerateControlTorque(Vector3 control)
    {


        float lift_coefficient = 2f * Mathf.PI * m_maxDeflect;

        float max_torque = DynPressure * lift_coefficient * m_tailArea * m_tail2CMDistance;
        Vector3 desired_torque = control * max_torque;

        return desired_torque;

    }
{% endhighlight %}

Note some of the subtle changes. GenerateAeroForces no longer takes arguments; the values it needs
are a part of the Aero model. Most everything else is the same except some changes in naming.
GenerateControlTorque is also mostly the same but with only one argument, the control.
These changes reflect that the forces and torques are a response to the missile aerodynamic properties
to its velocity.

Next up, AutoPilot!

#### AutoPilotController

The auto pilot is in charge of determining the control used to generate the torque (i.e. operate the fins).
It determines the logic of the missile guidance, both in the "gravity turn" and in pursuit of the target.
It holds a reference to the Aerodynamics model `m_AeroModel` and a PID controller `m_pidController`,
which is also a new component I'll talk about after this.

{% highlight C# %}
public class AutoPilotController : MonoBehaviour
{
...
    void Update()
    {
        Vector3 targetPosition = m_target.GetComponent<PlayerController>().centerOfMass.position;
        Vector3 targetVelocity = m_target.transform.forward * m_target.GetComponent<PlayerController>().airSpeed;
        RelativePosition = targetPosition - transform.position;
        RelativeVelocity = targetVelocity - m_body.linearVelocity;

        m_velocity = m_body.linearVelocity;

        Tgo = -RelativePosition.magnitude * RelativePosition.magnitude / Vector3.Dot(RelativePosition, RelativeVelocity);

        // determine acceleration command Strategy
        if (IsInPursuit)
        {
            m_accelerationCmd = PursuitCommand();
        }
        else
        {
            m_accelerationCmd = GravityTurnCommand();
        }

        // Leave gravity turn when we are stable
        if (!IsInPursuit && (m_accelerationCmd.normalized.y > 0 || m_accelerationCmd.normalized.y < -0.99f))
        {
            IsInPursuit = true;
        }

    }

    Vector3 GravityTurnCommand()
    {

        Vector3 rotationDir = Vector3.Cross(m_velocity, RelativePosition).normalized;
        Vector3 accelerationCmd = Vector3.Cross(rotationDir, m_velocity).normalized * 9.8f;
        return accelerationCmd;

    }

    Vector3 PursuitCommand()
    {
        Vector3 accelerationCmd = PureProNav(RelativePosition, RelativeVelocity, Gain);// + Vector3.up * 9.8f * gain/2;
        accelerationCmd += -Physics.gravity;
        return accelerationCmd;

    }
{% endhighlight %}

In the update, we determine the relative position and velocity of the target and missile. 
We add this to a property that the Missile can use as we will see later. It also
calculates Tgo (time to go) for the Missile as well. After this,
it sets the acceleration command based on two strategies: the GravityTurnCommand and the
PursuitCommand. It also controls when the autopilot transitions from gravity turn to pursuit.

Next we will see how to set the auto pilot control signal.

{% highlight C# %}
    public Vector3 GetControl()
    {

        float area = m_AeroModel.WingArea;
        float dynamicPressure = m_AeroModel.DynPressure;

        // Lift Force ~= 2pi * AoA * dynP * area
        float desiredAngleOfAttack = m_accelerationCmd.magnitude * m_body.mass / area / dynamicPressure / (2 * Mathf.PI);
        float attackLimited = Mathf.Min(desiredAngleOfAttack * Mathf.Rad2Deg, m_angleOfAttackLimit);

        // what even is this bit?
        Vector3 turnDir = Vector3.Cross(m_velocity, m_accelerationCmd).normalized;
        Vector3 desiredForward = Quaternion.AngleAxis(attackLimited, turnDir) * m_velocity.normalized;
        Vector3 angle = -Vector3.Cross(desiredForward, m_body.transform.forward);
        float phi = Mathf.Asin(angle.magnitude);

        // get the error rate to calculate the control for this time step
        Vector3 error = phi * angle.normalized / 2f / Mathf.PI;
        Vector3 errorRate = error / m_timeConstant; // Slew control to moderate the error rate
        Vector3 control = m_pidController.PID(errorRate);

        return control;
    }

    Vector3 PureProNav(Vector3 relPos, Vector3 relVel, float gain)
    {
        Vector3 LoSRate = Vector3.Cross(relPos, relVel) / (relPos.magnitude * relPos.magnitude);

        float speedRatio = relVel.magnitude / m_velocity.magnitude;

        Vector3 accelerationCmd = -gain * speedRatio * Vector3.Cross(m_velocity, LoSRate);

        return accelerationCmd;
    }
{% endhighlight %}

The GetControl method gets the control signal (which is still a Vector3) in order to generate the torque.
This code is similar to what was in MissileController, with a little bit of cleaning up.

One very important detail however to point out is a subtle change to the error calculation.
We now call it error rate where before, we divided by `Time.deltaTime` but now the code is
`errorRate = error / m_timeConstant`. This autopilot time constant is the slew rate, or the rate
at which the actuators/fin controls would execute the needed motion to complete the command.
Essentially, this adds a delay from when the missile is issued a command and when it is completed.

This means that the missile will lag in response to a maneuver, which for us is good! We can use
the time constant as one method of making the missile easier to dodge with good player maneuvering.
Is it sufficient? Probably not, as we will probably want to add a delay in the missile even recognizing the player
has changed direction, but that will be for another day.

Also included is the PureProNav from last time. It's the same as before, and included here. It could potentially
be a module where we could swap out other acceleration command generating functions, but for now it's just here.

Next up, the PID controller.

#### PIDController

So this was also in MissileController, but instead of putting it in the AutoPilot, it becomes its own component. Why?
Becuase A) it has state in the form of the errorLast and errorAccumulator, and B) because we can more neatly organize
the gains when it's in its own component and modify them in the editor. Otherwise, it's exactly the same
as before but we can repeat ourselves a little.

{% highlight C# %}
public class PIDController : MonoBehaviour
{
    private Vector3 m_errorLast = Vector3.zero;
    private Vector3 m_errorAccumulator = Vector3.zero;
    [SerializeField]
    private float pGain = 1f;
    [SerializeField]
    private float iGain = 0.001f;
    [SerializeField]
    private float dGain = 0.2f;

    public Vector3 PID(Vector3 error)
    {
        Vector3 dError = (error - m_errorLast) / Time.fixedDeltaTime;
        Vector3 iError = m_errorAccumulator + error * Time.fixedDeltaTime;

        if (m_errorLast == Vector3.zero)
        {
            dError = Vector3.zero;
        }

        Vector3 PID = pGain * error + dGain * dError + iGain * iError;

        if (PID.magnitude > 1)
        {
            PID = PID.normalized;
        }

        m_errorLast = error;
        m_errorAccumulator += error;


        return PID;
    }
}
{% endhighlight %}

Next up is the old Missile Controller, now called just Missile. It was a monsterous load of code, so what does it look like now?

{% highlight C# %}
public class Missile : MonoBehaviour
{
...
    void Awake()
    {
        m_body = GetComponent<Rigidbody>();
        missileAudio = GetComponent<AudioSource>();
        m_AeroModel = GetComponent<AerodynamicsModel>();
        m_AutoPilot = GetComponent<AutoPilotController>();
    }

    void Start()
    {

        m_body.centerOfMass = m_missileCOM.transform.localPosition;
        m_body.AddForce(Vector3.up * m_launchSpeed, ForceMode.VelocityChange);

        thrustLooper = LoopAudio(1f);
        missileAudio.clip = thrustSound;

        m_AutoPilot.Target = m_target;

        StartCoroutine(thrustLooper);

        explosionParticle.Stop();

    }
{% endhighlight %}

The code starts out with initializing its references to the other components and it assigns a target to the AutoPilot.
So far very similar, but we will see some major cuts coming up.

{% highlight C# %}
    void Update()
    {
        float tgo = m_AutoPilot.Tgo;

        if (tgo < 10 && tgo > 0 && m_AutoPilot.IsInPursuit)
        {
            m_armed = true;
        }
        if (tgo < 0 && m_armed)
        {
            Explode();
        }
    }

    void FixedUpdate()
    {
        Vector3 control = m_AutoPilot.GetControl();

        Vector3 torque = m_AeroModel.GenerateControlTorque(control);

        Vector3 aeroForces = m_AeroModel.GenerateAeroForces();
        m_body.AddForce(aeroForces);

        if (torque.magnitude > 0.001)
        {
            m_body.AddTorque(torque);
        }
    }
{% endhighlight %}

There is no MissileMovement() function, all that functionality has been moved into Update and FixedUpdate and boy it is a lot smaller now.
Update is in charge of arming and exploding the missile, which makes sense as it's not really physics based.
FixedUpdate uses the auto pilot to get the control, which it feeds into the Aero model to get the torque,
then calls the aero model to calculate the forces so it can add them to the missile body. And that's it!
Makes it really clean and simple!

The rest of the code is still mostly the same, with some changes to references. For example,
the Damage method now has a reference to the auto pilot to get the distance.

{% highlight C# %}
    private void Damage()
    {
        float distance = m_AutoPilot.RelativePosition.magnitude;
{% endhighlight %}

Lastly, there's a small change in the SpawnManager class to assign the target to the missile.
Before, it would change the prefab to do the assignment, now it does it on the instance.

{% highlight C# %}
public class SpawnManager : MonoBehaviour
{
...
        GameObject missile = Instantiate(missilePrefab, GenerateSpawnPosition(), Quaternion.Euler(new Vector3(-90, 0, 0)));
        missile.GetComponent<Missile>().Target = player;
{% endhighlight %}

That's all the refactoring for today! There's still more that can be changed and organized
with these scripts, but next I want to work more on their functionality, starting with the Aero model.

But with even the most subtle change we made it a lot easier for the player to dodge the missile! Just a little slew delay,
and that's pretty good! I have more ideas, such as adding a Kalman filter (or something that emulates one) so that
there will be a lag response to player acceleration. But I also want to work on the player flight controls,
as right now it's the most pathetic controller you can write for a flying plane. You can quote me on that.

So until next time, thanks for reading and journeying on with me,

-Outofanser


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
