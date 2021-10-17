using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MovingPlatformScript : MonoBehaviour
{
    public Vector3[] points;
    public int pointNumber = 0;
    private Vector3 currentTarget;

    public float tolerance;
    public float speed;
    public float delayTime;

    private float delayStart;

    public bool automatic;

    bool paused;
    // Start is called before the first frame update
    void Start()
    {
        if (points.Length > 0)
        {
            currentTarget = points[0];
        }
        tolerance = speed/100;
        HUDScript.pauseGame += Pause;
    }

    // Update is called once per frame
    void FixedUpdate()
    {
        if (!paused)
        {
            if (transform.position != currentTarget)
            {
                MovePlatform();
            }
            else
            {
                UpdateTarget();
            }
        }
    }

    private void MovePlatform()
    {
        Vector3 heading = currentTarget - transform.position;
        transform.position += (heading / heading.magnitude) * speed * Time.deltaTime;

        if (heading.magnitude < tolerance)
        {
            transform.position = currentTarget;
            delayStart = Time.time;
        }
    }

    private void UpdateTarget()
    {
        if (automatic)
        {
            if (Time.time - delayStart > delayTime)
            {
                NextPlatform();
            }
        }
    }

    public void NextPlatform()
    {
        pointNumber++;
        if (pointNumber >= points.Length)
        {
            pointNumber = 0;
        }
        currentTarget = points[pointNumber];
    }

    private void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag("Player") || other.name.Contains("Slam Enemy"))
        {
            //Gamemanager.Instance.GetComponent<Animator>().enabled = false;
            other.transform.parent = transform;
        }
    }
    private void OnTriggerExit(Collider other)
    {
        if (other.CompareTag("Player") || other.name.Contains("Slam Enemy"))
        {
            //Gamemanager.Instance.GetComponent<Animator>().enabled = true;
            other.transform.parent = null;
        }
    }
    private void OnDestroy()
    {
        HUDScript.pauseGame -= Pause;
    }
    void Pause() {
        paused = !paused;
    }
}