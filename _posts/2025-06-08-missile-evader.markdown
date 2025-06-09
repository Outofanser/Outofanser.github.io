---
layout: post
title:  "Missile Evader Blog Post #1"
date:   2025-06-08 18:30:00 -0500
categories: Missile Evader update
---

Starting Off
------------

Hello there!

This is the kickoff post for the Missile Evader (working title) game!

The main page will be updated at [Missile Evader](/missile-evader/)
so this blog is meant to be a log of the changes and progress I make.
However, I already started on this project, so there's lots to catch up on and
most of it will be a repeat of what is on the Missile Evader page (as of today).

First, a quick summary of the goals for this game and the current status

-------------

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

Status
======

1. Player controls are very basic and not physics based. Just there for testing with a constant speed and turn rates.

2. Missiles are much more along, but needs code refactoring and balance adjustments. The missiles include aerodynamics,
navigation (acceleration commands) and a PID controller to execute. But there's little to control how well it can track its target,
and right now they are very hit-or-miss and not in a fun way. What it needs is a delay between command and actuation, i.e. an auto-pilot time delay.

3. There are no powerups or pickups or a real sky or a real ground. Just placeholder assets, though I'm proud of my prototype missile.

4. There is a little missile camera that follows a single missile out in flight, mostly for debugging, but it's actually really cool to watch.
I think I will keep this as some sort of feature.

<img src="/Missile_Evader/images/gameView.png" alt="Gameplay" title="Gameplay so far"/>

Code
====

Now we get into the fun stuff. What terrible code have I written so far? Well it's pretty bad right now as it's mostly a product of iterating
on some very technical math and guidance design (that was quite difficult to get working) so let's dive in.

First, right now the player code is pretty simple. Set invisible boundaries and destroy the plane if it takes too much damage.


#### PlayerController
{% highlight C# %}
public class PlayerController : MonoBehaviour
{
...
    void Start()
    {
        planeAudio = GetComponent<AudioSource>();
        initPosition = transform.position;
        initRotation = transform.rotation;
    }

    // Update is called once per frame
    void Update()
    {

        if (health < 0 && !destroyed)
        {
            destroyed = true;
            Explode();
        }

        if (!destroyed)
        {
            PlayerMovement();
            ApplyBoundary();
            propeller.transform.Rotate(new Vector3(0, 0, 1000 * Time.deltaTime));
        }

    }

{% endhighlight %}

The movement could not be simpler. There's no physics yet! So just translate and rotate

{% highlight C# %}

    void PlayerMovement()
    {
        // Move player forward in forward direction (not realistic physics)
        transform.Translate(Vector3.forward * airSpeed * Time.deltaTime);

        // get inputs to attitude controls
        Vector3 playerAttitudeInput = new Vector3(-Input.GetAxis("Pitch"), -Input.GetAxis("Yaw"), -Input.GetAxis("Roll"));
        // scale inputs to body moments (body rates)
        Vector3 rotationRate = Vector3.Scale(playerAttitudeInput, new Vector3(pitchRate, yawRate, rollRate));

        acceleration = Vector3.Cross(Vector3.forward * airSpeed, rotationRate); // may be useful information to use truth data with missile system

        // Rotate player via inputs
        transform.Rotate(rotationRate * Time.deltaTime);
    }
{% endhighlight %}

The boundary code is also very simple, with hardcoded values. I should add them as serialized fields...

{% highlight C# %}
    void ApplyBoundary()
    {
        transform.position = new Vector3(MinMax(-500f, 500f, transform.position.x), MinMax(0f, 200f, transform.position.y), MinMax(-500f, 500f, transform.position.z));
    }
{% endhighlight %}

We destroy the plane if it takes too much damage (or hits the ground). The health is a public field but should be a proper Property or managed indirectly, but w/e.
We explode by playing sound and hiding the plane, then set a timer using the IEnumerator trick to return the player back to its respawn point.

{% highlight C# %}

    private void OnCollisionEnter(Collision collision)
    {
        if (collision.gameObject.CompareTag("Ground"))
        {
            health -= 500;
            Debug.Log("Hit the ground!");
        }
    }

    private void Explode()
    {
        explosionParticle.Play();
        planeAudio.Stop();
        planeAudio.PlayOneShot(explosionSound, 0.2f);
        gameObject.transform.GetChild(0).gameObject.SetActive(false);
        StartCoroutine(Respawn());
    }

    private IEnumerator Respawn()
    {
        yield return new WaitForSeconds(3f);
        gameObject.transform.GetChild(0).gameObject.SetActive(true);
        gameObject.transform.position = initPosition;
        gameObject.transform.rotation = initRotation;
        destroyed = false;
        health = 100;
    }

{% endhighlight %}

#### Missile Controller

This is where the real code is, and it's a real mess. We start with some setup

{% highlight C# %}
public class MissileController : MonoBehaviour
{
...
    void Start()
    {
        body = GetComponent<Rigidbody>();
        body.centerOfMass = missileCOM.transform.localPosition;
        body.AddForce(Vector3.up * airSpeed, ForceMode.VelocityChange);
        missileAudio = GetComponent<AudioSource>();
        thrustLooper = LoopAudio(1f);
        missileAudio.clip = thrustSound;


        //missileAudio.volume = 0.1f;
        //missileAudio.loop = true;

        StartCoroutine(thrustLooper);


        explosionParticle.Stop();
        //StartCoroutine(thrustLooper);
    }

    // Update is called once per frame
    void Update()
    {
        //MissileMovement()
        targetPos = target.GetComponent<PlayerController>().centerOfMass.position;
    }

    void FixedUpdate()
    {
        MissileMovement();
    }
{% endhighlight %}

MissileMovement is simply overloaded with stuff and needs refactoring, but it's born from a process of trial and error to get things working.

{% highlight C# %}
    void MissileMovement()
    {
        // get target position and velocity in inertial world frame
        Vector3 targetPosition = targetPos;
        Vector3 targetVelocity = target.transform.forward * target.GetComponent<PlayerController>().airSpeed;
        // get missile position and velocity in intertial world frame
        Vector3 missilePosition = transform.position;
        ///Vector3 missileVelocity = transform.forward * airSpeed;
        Vector3 missileVelocity = body.velocity;
        // get instantaneous relative position and velocity in world frame (note, relative rotation of local frame and world frame means the frame is relevant!)
        relativePosition = targetPosition - missilePosition;
        Vector3 relativeVelocity = targetVelocity - missileVelocity;
        distance = relativePosition.magnitude;

        tgo = -relativePosition.magnitude * relativePosition.magnitude / Vector3.Dot(relativePosition, relativeVelocity);

        if (tgo < 10 && !gravity_turn)
        {
            armed = true;
        }
        if (tgo < 0 && armed)
        {
            Explode();
        }

        // get acceleration command and body rotation rate (in world frame)
        Vector3 accelerationCmd = PureProNav(missileVelocity, relativePosition, relativeVelocity, gain);// + Vector3.up * 9.8f * gain/2;
        accelerationCmd += -Physics.gravity;
        if (gravity_turn)
        {
            Vector3 rotationDir = Vector3.Cross(missileVelocity, relativePosition).normalized;
            accelerationCmd = Vector3.Cross(rotationDir, missileVelocity).normalized * 9.8f;
            if (accelerationCmd.normalized.y > 0 || accelerationCmd.normalized.y < -0.99f)
            {
                gravity_turn = false;
            }
        }

        Vector3 forward = body.transform.forward;

        vel = missileVelocity.magnitude;
        float dynamicPressure = 0.5f * rho * vel * vel;

        float dAoA = accelerationCmd.magnitude * body.mass / area / dynamicPressure / (2 * Mathf.PI);
        attack_limited = Mathf.Min(dAoA * Mathf.Rad2Deg, aoa_limit_deg);
        Vector3 turnDir = Vector3.Cross(missileVelocity, accelerationCmd).normalized;


        Vector3 desiredForward = Quaternion.AngleAxis(attack_limited, turnDir) * missileVelocity.normalized;
        Vector3 angle = -Vector3.Cross(desiredForward, forward);
        float phi = Mathf.Asin(angle.magnitude) / Time.fixedDeltaTime;
        ///Vector3 control = 1/100f*(phi * angle.normalized - body.angularVelocity)/Time.fixedDeltaTime*body.mass;

        Vector3 control = PIDController(Vector3.zero, phi * angle.normalized);

        Vector3 torque = GenerateControlTorque(control, missileVelocity, 0.1f);


        //body.AddRelativeForce(local_lift,ForceMode.Acceleration);// + drag_force);
        Vector3 aeroForces = GenerateAeroForces(forward, missileVelocity, area);
        body.AddForce(aeroForces);

        //body.AddForce(lift_force, ForceMode.Acceleration);

        if (torque.magnitude > 0.001)
        {
            body.AddTorque(torque);
        }
    }
{% endhighlight %}

This thing does a lot. It calculates a tgo, or time to go, before the missile reaches it's closest point of approach (assuming zero acceleration).
This calculation is based on the relative position and velocity, which will be used extensively.
If the tgo goes to < 0, it's time to detonate!

It calculates an acceleration command based on PureProNav (where the acceleration is from aero-forces only, no thrust).
But when initialized, it will do a gravity turn instead where it flies upward and arcs gradually over before homing on the target.
This prevents the missile from losing speed trying to point to the target before it has steady flight.

The desired angle of attack is then calculated based on the acceleration command, which informs the angle that we want the missile
to be at which will then generate the proper lift. I did attempt to force a limited angle of attack so that the missile 
is a little bit limited on how much it can maneuver, but normally this would be set based on the aerodynamic properties so you don't stall.

It then goes to the PID controller to generate a vector control which ranges from 0 to 1 in magnitude. This then 
tells the GenerateControlTorque how much torque to apply. 

We apply the aerodynamic forces (lift and drag), and then the torque commanded. Then we're done!

Ok let's look at those functions, they are a bit sketch:

{% highlight C# %}

    Vector3 PureProNav(Vector3 Vm, Vector3 relPos, Vector3 relVel, float gain)
    {
        Vector3 LoSRate = Vector3.Cross(relPos, relVel) / (relPos.magnitude * relPos.magnitude);

        float speedRatio = relVel.magnitude / Vm.magnitude;

        Vector3 accelerationCmd = -gain * speedRatio * Vector3.Cross(Vm, LoSRate);

        return accelerationCmd;
    }
{% endhighlight %}

PureProNav is fairly simple to write, harder to understand perhaps. The LOS rate is the line of sight rate. For a successful intercept,
we want the line of site rate of change to be zero, because if it is, we are on a collision course! It looks similar to the inverse of tgo,
but with a cross product. The cross product makes a vector perpendicular to the plane of movement we desire, so we can rotate around it.
But instead we make the acceleration command by applying a gain and taking the cross product of that with the actual missile velocity Vm.
This is so we get an acceleration command which is perpendicular to the missile velocity so that we can use aerodynamic forces
which push the body along its sides, while conserving energy (no thrust, minimal drag).

{% highlight C# %}

    Vector3 PIDController(Vector3 startOrientation, Vector3 endOrientation)
    {
        Vector3 error = (endOrientation - startOrientation) / 2f / Mathf.PI;
        Vector3 dError = (error - errorLast) / Time.fixedDeltaTime;
        Vector3 iError = errorAccumulator + error * Time.fixedDeltaTime;

        if (errorLast == Vector3.zero)
        {
            dError = Vector3.zero;
        }

        Vector3 PID = pGain * error + dGain * dError + iGain * iError;

        if (PID.magnitude > 1)
        {
            PID = PID.normalized;
        }

        errorLast = error;
        errorAccumulator += error;


        return PID;
    }
{% endhighlight %}

The PID controller is a regular PID controller with some normalization. PID stands for Proportional, Integral, Derivative and it's 
a standard controller for closed loop systems where we give feedback based on the difference between the desired and actual output.
In this case, the PID controller is set to be a maximum value of one, similar to a joystick but in 3 dimensions.

{% highlight C# %}

    Vector3 GenerateControlTorque(Vector3 control, Vector3 missileVelocity, float tailArea)
    {
        float max_deflect = 10f;
        float tail2CMDistance = 0.5f;

        float vel = missileVelocity.magnitude;
        float dynamicPressure = 0.5f * rho * vel * vel;
        float lift_coefficient = 2f * Mathf.PI * max_deflect;

        float max_torque = dynamicPressure * lift_coefficient * tailArea * tail2CMDistance;
        Vector3 desired_torque = control * max_torque;

        return desired_torque;

    }
{% endhighlight %}

GenerateControlTorque is not the greatest... or realistic? Maybe it is? I would like to try something where the center of lift is a quantity
and we simply apply our aeroforces on that directly. But this gets the job done. It calculates the max torque that can
be applied based on "tailArea" which is not a true area, but the effective cross-sectional area for generating a rotating moment.
We modulate the "tailArea" using the control to reduce the torque when we don't need to go all out to achieve our heading.

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

Not the best physics here either, and I see that wingArea is not used because I grabbed "area" the object constructor.
I need to retest how the area impacts our ability to guide, 
but this also needs to be moved out into its own component with aero properties and aero force methods.
Its calculation for lift and drag are based on some flat-sheet aerodynamic approximations, where we have a stall-out
case with high angle of attack with the more aggressive drag and diminished lift.

All these methods could be moved into different components. It's even possible to make an ECS system out of this if we end up with
a heck of a lot of missiles. These calculations are probably not cheap? I should trace it...

I'm not done with code yet, we still gotta explode! It's similar to exploding the player, so I'll skip on details.

{% highlight C# %}
    private void Explode()
    {
        if (!exploded)
        {
            Damage();
            exploded = true;
            StopCoroutine(thrustLooper);
            explosionParticle.Play();
            missileAudio.Stop();
            missileAudio.PlayOneShot(explosionSound, 0.2f);
            Destroy(gameObject.transform.GetChild(0).gameObject);
            StartCoroutine(Explosion());
        }
    }

    private void OnCollisionEnter(Collision collision)
    {
        if (collision.gameObject.CompareTag("Player"))
        {
            Explode();
        }
        else if (collision.gameObject.CompareTag("Ground"))
        {
            Explode();
        }
    }

    IEnumerator Explosion()
    {
        yield return new WaitForSeconds(2f);
        Destroy(gameObject);
    }

    private void Damage()
    {
        float distance = (targetPos - transform.position).magnitude;

        float damage = 300f / distance;

        if (damage < 10f)
        {
            return;
        }

        target.GetComponent<PlayerController>().health -= Mathf.Min(damage, 80f);
        Debug.Log("Player hit! Health is now " + target.GetComponent<PlayerController>().health);
    }
{% endhighlight %}

The damage part is more insteresting. We hold a reference to our target to damage its health.
The calculation for the damage is based on the distance we are from the target. If the energy of the blast
is perfectly spherical, it would be 1/r^2, but if we assume some directionality to the blast, it could be 1/r
or even ln(r) dropoff. This is really more about "feels" though and I will change it however. 
Besides, energy != damage and I can do whatever I want! Not everything is physics!

One last function: LoopAudio. This guy is meant to play audio with Unity's audio player which has 3D spatial falloff and Doppler.
The sound is just a bleep, pulled from some "Create With Code" assets, but the code plays the sound at a faster and faster rate
as the missile gets closer. Watch out!

{% highlight C# %}

    IEnumerator LoopAudio(float waitTime)
    {
        while (true)
        {
            float mywait = Mathf.Max(0.1f, Mathf.Min(1f, distance / 1000f)) * waitTime;
            missileAudio.time = 0f;
            missileAudio.Play();
            yield return new WaitForSeconds(mywait);
        }

    }
{% endhighlight %}

<img src="/Missile_Evader/images/EditorView1.png" alt="Dodge" title="Dodging the missile" width="250"/>
<img src="/Missile_Evader/images/EditorView2.png" alt="Explode" title="Missile explodes" width="450" />

That just about does it for code. Lots of terrible code in a bloated controller, but also some cool code and cooler physics.
But the biggest problem is that the missile is not balanced: It will always kill the player. Go ahead and try it, you will die.
It needs a delay added between ask for a commanded acceleration and the actuation of it. This way, the player can maneuver
at the last second and dodge! 

Last bit for this project, as of now, is the spawn manager.

#### SpawnManager

This code is some quickie code to spawn multiple missiles to fly at the player, with scaling "difficulty".
It will find the number of missiles currently alive and spawn a new wave if all have perished.

{% highlight C# %}
    void Start()
    {
        player = GameObject.FindGameObjectWithTag("Player");
        missilePrefab.GetComponent<MissileController>().target = player;
    }

    // Update is called once per frame
    void Update()
    {
        int numMissiles = GameObject.FindGameObjectsWithTag("Missile").Length;

        if (numMissiles <= 0)
        {
            for (int i = 0; i < waveNumber; i++)
            {
                Instantiate(missilePrefab, GenerateSpawnPosition(), Quaternion.Euler(new Vector3(-90, 0, 0)));
            }

            waveNumber++;

        }
    }

    Vector3 GenerateSpawnPosition ()
    {
        float xPos = 0;
        float zPos = 0;
        while (Mathf.Abs(xPos) < minSpawnRadius)
        {
            xPos = Random.Range(-spawnRadius, spawnRadius);
        }

        while (Mathf.Abs(zPos) < minSpawnRadius)
        {
            zPos = Random.Range(-spawnRadius, spawnRadius);
        }
        return new Vector3(xPos, 4f, zPos) + player.transform.position;
    }
{% endhighlight %}

This should be handled by a game manager using an object pool more than likely, but it's a start.

That about wraps it up! This is the state of the game before the blog and next up I will refactor some code
and apply more OOP principles, as well as clean up the physics and add an auto-pilot delay so we can actually dodge the damn thing.

Peace out!

-Outofanser


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
