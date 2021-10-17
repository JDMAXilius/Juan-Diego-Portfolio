using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class TurretEnemy : MonoBehaviour
{


    [SerializeField] bool isDynamic;
    [SerializeField] Transform playerPos;
    [SerializeField] GameObject model;
    [SerializeField] ShootScript shootScript;
    [SerializeField] float extraDetection;
    [HideInInspector] public Animator modelAnim;
    [SerializeField] Vector3 pivotAdjust;
    Vector3 prevDir;
    Rigidbody rb;
    public bool paused;
    public bool shootScriptOnStart;

    private void Awake()
    {
        shootScript.enabled = false;
    }
    // Start is called before the first frame update
    void Start()
    {
        playerPos = Gamemanager.Instance.player.transform;
        modelAnim = model.GetComponent<Animator>();
        rb = GetComponent<Rigidbody>();
        shootScript = GetComponent<ShootScript>();
        shootScriptOnStart = shootScript.enabled;
    }

    // Update is called once per frame
    void Update()
    {
        if (!paused)
        {
            //**Note**
            //Many of the transform.up operations require the *-1 for the sprite. Depending on the asset used, change/remove this value so the sprite
            //points correctly and the projectile moves in the right direction

            rb.velocity = Vector3.zero;
            //Change rotation if spider is a dynamic spider
            if (isDynamic)
            {
                //Debug.DrawRay(transform.position, -1 * transform.up.normalized * ((shootScript.bulletSpeed * shootScript.projectileLifeTime) + extraDetection), Color.green);
                if (Physics.Raycast(transform.position, (playerPos.position + pivotAdjust - transform.position), out RaycastHit hit, (shootScript.bulletSpeed * shootScript.projectileLifeTime) + extraDetection))
                {
                    if (hit.collider.CompareTag("Player"))
                    {
                        shootScript.enabled = true;

                        prevDir = transform.up;
                        transform.up = new Vector3(transform.position.x - playerPos.position.x, transform.position.y - playerPos.position.y) - (Vector3)pivotAdjust;

                        if (prevDir != transform.up)
                        {
                            ////debug.log("Turn true");
                            modelAnim.SetTrigger("Turn");
                        }
                        else
                        {
                            modelAnim.SetTrigger("Idle");
                        }
                    }
                    else
                    {
                        shootScript.enabled = false;
                    }
                }
                else
                {
                    shootScript.enabled = false;
                }
            }
        }
    }

    public void Pause(int pauseOverride = 2)
    {
        if (pauseOverride == 2)
        {
            paused = !paused;
        }
        else if (pauseOverride == 1)
        {
            paused = true;
        }
        else if (pauseOverride == 0)
        {
            paused = false;
        }

        if (paused)
        {
            modelAnim.enabled = false;
        }
        else
        {
            modelAnim.enabled = true;
        }
    }
}

