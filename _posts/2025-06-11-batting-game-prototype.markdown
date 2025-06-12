---
layout: post
title:  "Batting Game Prototype Blog Post #1"
date:   2025-06-08 22:00:00 -0500
categories: Batting Game update
---

Starting Off
------------

Hello there!

This is the kickoff post for the Batting Game Prototype!

The main page will be updated at [Batting Game Prototype](/batting-game-prototype/)
so this blog is meant to be a log of the changes and progress I make.
This project is part of the Unity learn "Create with Code" series,
so its main purpose is just to test out and try new concepts.

First, a quick summary of the goals for this game and the current status

-------------

Goals
=====

The goals for this project are to have the following features:

1. Player can swing the bat once per round

2. Player hits the ball to collect gems to earn points

3. More gems spawn for each gem earned in previous rounds

4. UI to display score, rounds left, and a game over screen to restart

5. Persistent high score

6. Powerups that will grow the ball or spawn multiple balls to hit


Status
======

The game meets the requirement for the Unity Learn lesson, but goals 5 and 6 are not implemented.
I may come back to do these things in the future, but it's not a high priority.

Code
====

Most of the code is pretty basic. Let's start with the most complicated bit, the bat controller.
It's job is to let the bat loose when the player hits space. But only once per round!

#### BatController
{% highlight C# %}
public class BatController : MonoBehaviour
{
...
    void Update()
    {
        if (Input.GetButtonDown("Jump") && !hasSwung)
        {
            //Debug.Log("swing");
            isSwingActive = true;
            hasSwung = true;
        }
    }

    // Update is called once per frame
    void FixedUpdate()
    {
        if (isSwingActive)
        {
            batterRb.AddForceAtPosition(new Vector3(swingSpeed, 0, 0), new Vector3(0, 0, 0), ForceMode.Impulse);
            isSwingActive = false;
        }
        if (transform.rotation.eulerAngles.z > 60f)
        {
            //Debug.Log("Stop");
            batterRb.angularVelocity = Vector3.zero;
        }
    }
{% endhighlight %}

We use Update to get the player input so that the input doesn't get "missed" in FixedUpdate.
We check if the player has swung already and stop that shiz if they have.

We also need to set if the swing is currently active so that we only swing the bat once,
which we do in the FixedUpdate. There are other ways of accomplishing this I'm sure, but I stopped
thinking about it and moved on pretty quick once I get something that worked. I was eager to move
to my next lesson :)

We then restrict the bat's movement once it reaches some ending angle (hard coded).

{% highlight C# %}
    public void Reset()
    {
        transform.rotation = Quaternion.Euler(new Vector3(0, -90, 0));
        hasSwung = false;
        isSwingActive = false;
    }
{% endhighlight %}

We then include a method for the Game manager to reset the bat to home position for each round.
Speaking of Game manager, let's tackle that turkey.

#### GameManager

The traditional monolithic "manager" that does perhaps more than it should, the game manager is in charge
of setting the rules for launching the ball at the bat, resetting rounds, and even restarting the game.
It's not a singleton, I'm skeptical of using singletons until I need to but there's only one scene
in this game so far (may want an starting screen scene later on).

{% highlight C# %}
public class GameManager : MonoBehaviour
{
...
    void Start()
    {
        ballSpawnPosition = gameObject.transform.position;
        isFirstRound = true;
        isBallInPlay = false;
        isBallLaunching = false;

    }

    // Update is called once per frame
    void Update()
    {

        if (!isGameStarted)
        {
            StartRound();
            isGameStarted = true;
        }
        scoreText.text = "Score: " + score;
        ballsText.text = "Balls: " + swingsLeft;

        if (swingsLeft < 0)
        {
            return; // End Game
        }

        CheckBallInPlay();

    }
{% endhighlight %}

We initialize by setting flags for whether the ball is in play or mid-launch sequence.
This launch sequence is a gap between rounds where the ball will not spawn for a short delay.

In the update we set the UI text and determine if the game has started and if not, start it.
It lastly checks if the ball is in play, which will modify the above flags depending
if we have a ball in play and start the round.

{% highlight C# %}
    void CheckBallInPlay()
    {
        if (!isBallInPlay && !isBallLaunching)
        {
            StartRound();
            return;
        }

        GameObject[] ballsInPlay = GameObject.FindGameObjectsWithTag("Ball");

        //Debug.Log("ballsInPlay: " + ballsInPlay.Length);

        foreach (GameObject ball in ballsInPlay)
        {
            if (ball.transform.position.y < -10f)
            {
                Destroy(ball);
            }
        }
        if (ballsInPlay.Length == 0 && isBallInPlay)
        {
            isBallInPlay = false;
        }
    }
{% endhighlight %}

We destroy the balls if they fall offscreen. There is normally only one ball, but if I add a multi-ball powerup later
I want to track all the balls I set loose. We also start the round if there are no balls in play and if we're not already
launching a ball due to a new round. THis isn't well organized code, and it should be in a different place,
probably in update with this guy just checking isBallInPlay. What should destroy the ball then? Moving on.

{% highlight C# %}
    void StartRound()
    {
        if (swingsLeft <= 0)
        {
            gameOverScreen.gameObject.SetActive(true);
            return;

        }
        gemsNum = 2 + score / 10;

        if (isFirstRound)
        {
            gemsNum += 1;
            isFirstRound = false;
        }


        for (int i = 0; i < gemsNum; i++)
        {
            GameObject pooledGems = ObjectPooler.SharedInstance.GetPooledObject();
            Debug.Log(pooledGems);
            if (pooledGems != null)
            {
                Debug.Log(i);
                pooledGems.SetActive(true); // activate it
                pooledGems.transform.position = GenerateRandomPosition(); // position it
            }
        }

        playerBat.GetComponent<BatController>().Reset();
        waitToFire = WaitToFire(1f);
        isBallLaunching = true;
        StartCoroutine(waitToFire);

    }
{% endhighlight %}

This guy starts a round. We set the number of gems to be `gemsNum = 2 + score/10`.
The score each gem adds is ten, so this is a lazy way of saying the the gemsNum is
2 plus the number of gems collected thus far (which resets on new game). We also
add one more gem in the first round only just to give the player more chances.

The StartRound method is also in charge of setting up the GameOver screen for some reason.
That really shouldn't be there but hey... I wanted to make sure the game over
waited until the last ball was out of play and this was the lazy way to do it.

This function also spawns the gem using an ObjectPool. The object pool is a singleton
that holds a reference to a prefab (gem) and it initializes an array of them
at Start which are set unactive. We use the object pool to request an unactive one
to set it active and set its position using GenerateRandomPosition.

GenerateRandomPosition is nothing special, just picks a point within a box space.

We also reset the bat controller and call a "WaitToFire" method to start the ball launch.

{% highlight C# %}

    IEnumerator WaitToFire(float time)
    {
        yield return new WaitForSeconds(time);
        isBallInPlay = true;
        isBallLaunching = false;
        LaunchBall();
    }

    void LaunchBall()
    {
        swingsLeft--;
        ballPrefab.GetComponent<BallLaunch>().ballSpeed = ballSpeed;
        Instantiate(ballPrefab, ballSpawnPosition, ballPrefab.transform.rotation);
    }
{% endhighlight %}

We use the Unity's favorite Coroutine trick of defining an IEnumerator
and have it wait for some time before we will launch a ball.
We call Launch ball while setting the proper flags, and we also set
the ball's speed. The balls have a component BallLaunch which simply
adds an impulse force to the left via the ballSpeed property.

Lastly, we have a RestartGame method which is called by a button that appears on the gameover screen.

{% highlight C# %}
    public void RestartGame()
    {
        SceneManager.LoadScene(0);
    }
{% endhighlight %}

That about does it for the game manager, we have one script left, and the most important one!
The GemPickup script, which meets the only requirement of the Unity lesson

#### GemPickup
{% highlight C# %}
public class GemPickup : MonoBehaviour
{
    private GameManager gameManager;
    [SerializeField] public int value = 10;
    // Start is called once before the first execution of Update after the MonoBehaviour is created
    void Start()
    {
        gameManager = GameObject.Find("Game Manager").GetComponent<GameManager>();


    }

    void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag("Ball"))
        {
            gameManager.score += value;
            gameObject.SetActive(false);
        }
    }
}
{% endhighlight %}

That is the whole script! Increase the score (by 10) and remove the gem (since it's an object pool item, we just set it inactive).

So there is some code refactoring that's begging to be done as well as some other feature I could add. I would like to add
score persistence for a high-score UI, which is a topic which was covered in a later Unity lesson. That would be fun!
We do need *something* to explain that getting more gems = spawning more gems, though I think players would *eventually* get that.
Also, I could add a powerup system. It could be good for practicing polymorphism, having a base powerup extended by concrete powerups.

But since this was a quickie project, I don't suspect I will have a ton more updates on it, and particularly none too technical.
But it's a nice little game, and I like the exponential gem growth mechanic! Maybe we can fight on who can get the highest score.

Peace out,

-Outofanser


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
