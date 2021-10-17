using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class SlamEnemy : MonoBehaviour
{
    [Header("Jumping")]
    [SerializeField] float maxJumpDistance;
    [SerializeField] float flightTime;
    [SerializeField] float jumpDelay;
    float runningDelay;
    bool canJump = true;
    
    [Space]
    [Header("Basic Properties")]
    [SerializeField] bool faceLeft;
    [SerializeField] float triggerRange;
    public Vector3 pivotAdjust;
    public bool paused;
    Vector3 pausedVel;

    [Space]
    [Header("Slamming")]
    [SerializeField] float slamTriggerSize;
    [SerializeField] float slammingTime;
    [SerializeField] int slamDamage;
    float slamTimer;
    bool slamming;
    bool canSlam;
    bool inAir;
    [HideInInspector] public bool canHurtPlayer = true;

    Rigidbody rb;
    ParticleSystem particles;
    [SerializeField] GameObject model;
    [HideInInspector] public Animator modelAnim;
    public SphereCollider slamTrigger;
    float slamTriggerRadius;
    float lookAroundTimer;
    
    [System.Obsolete]
    // Start is called before the first frame update
    void Start()
    {
        rb = GetComponent<Rigidbody>();
        particles = GetComponent<ParticleSystem>();
        modelAnim = model.GetComponent<Animator>();
        particles.startLifetime = slammingTime;
        particles.startSpeed = slamTriggerSize / slammingTime * 1.2f;
        slamTriggerRadius = slamTrigger.radius;
        lookAroundTimer = Random.Range(5f, 10f);
    }

    // Update is called once per frame
    void Update()
    {
        if (!paused)
        {
            if (lookAroundTimer > 0)                         //
            {                                                //
                lookAroundTimer -= Time.deltaTime;           //
            }                                                //
            else                                             //Timer to player idle animation
            {                                                //
                if (!slamming && canJump)                    //
                {                                            //
                    modelAnim.SetTrigger("Look");            //
                    lookAroundTimer = Random.Range(5f, 10f); //
                }                                            //
            }                                                //

            if (!IsStuck())       //If the enemy isnt stuck,
            {                     //
                if (isInRange())  //and the player is in range of the enemy,
                {                 //
                    if (canJump)  //and the enemy can jump,
                    {             //
                        Launch(); //then jump to the player
                    }             //
                }                 //
            }                     //

            if (runningDelay > 0)               //
            {                                   //
                runningDelay -= Time.deltaTime; //
            }                                   //
            else                                //
            {                                   //Timer between jumps,
                if (!slamming && !inAir)        //detects if enemy is still in the sequence
                {                               //
                    canJump = true;             //
                }                               //
            }                                   //

            if (transform.position.x < Gamemanager.Instance.player.transform.position.x) //
            {                                                                            //
                faceLeft = false;                                                        //
                model.transform.localRotation = Quaternion.Euler(Vector3.up * 90);       //
            }                                                                            //Direction check
            else                                                                         //
            {                                                                            //
                faceLeft = true;                                                         //
                model.transform.localRotation = Quaternion.Euler(Vector3.down * 90);     //
            }                                                                            //

            EnoughSpeed();
            Slamming();
            pausedVel = rb.velocity;
        }
    }
    void EnoughSpeed() { //If the enemy has enough speed to create the slam field
        if (rb.velocity.y < -3)
        {
            canSlam = true;
        }
        else {
            canSlam = false;
        }
    }
    bool isInRange()//Checks if player is within range and the enemy can see the player
    {
        if (Vector3.Distance(transform.position, Gamemanager.Instance.player.transform.position + pivotAdjust) <= triggerRange) {
            ////debug.log("In Distance");
            //Debug.DrawRay(transform.position, (Vector2)Gamemanager.Instance.player.transform.position - (Vector2)transform.position, Color.red);
            if (Physics.Raycast(transform.position, (Vector2)Gamemanager.Instance.player.transform.position + (Vector2)pivotAdjust - (Vector2)transform.position, out RaycastHit hit, triggerRange)) {
                ////debug.log("Raycast shot");
                Debug.DrawRay(transform.position, (Vector2)Gamemanager.Instance.player.transform.position + (Vector2)pivotAdjust - (Vector2)transform.position, Color.red);
                if (hit.collider.gameObject.CompareTag("Player")) {
                    ////debug.log("In Range");
                    return true;
                }
                
            }
        }
        ////debug.log("Not In Range");
        return false;
    }
    void Launch(float jumpDistance = 0, float time = 0, bool overrideDist = false)//Setting velocities for the enemy
    {
        canHurtPlayer = true;
        ////debug.log("Launch" + jumpDistance.ToString() + time.ToString());
        if (jumpDistance == 0)                                                                                               //
        {                                                                                                                    //
            jumpDistance = maxJumpDistance;                                                                                  //Setting the maximum distance
        }                                                                                                                    //
        float vec3Dist = Vector3.Distance(transform.position, Gamemanager.Instance.player.transform.position + pivotAdjust); //Getting distance between player end enemy
        if (time == 0)                                                                                                       //
        {                                                                                                                    //
            time = flightTime;                                                                                               //Setting the max flight time
        }                                                                                                                    //
        float scaling = vec3Dist / maxJumpDistance;                                                                          //
        if (vec3Dist < maxJumpDistance && !overrideDist) {                                                                   //
            time *= scaling;                                                                                                 //Scaling the arc based on how close the player is
            jumpDistance = vec3Dist;                                                                                         //

        }                                                       

        float initialXVel = jumpDistance / time;                                                                                                                         //X velocity
        float initialYVel;                                                                                                                                               //
        if (Mathf.Abs(transform.position.y - Gamemanager.Instance.player.transform.position.y + pivotAdjust.y) >= .9f)                                                   //
        {                                                                                                                                                                //
            initialYVel = ((-.5f * Physics.gravity.y * time * time) + (Gamemanager.Instance.player.transform.position.y + pivotAdjust.y - transform.position.y)) / time; //
        }                                                                                                                                                                //Set y velocity based on vertical difference
        else                                                                                                                                                             //
        {                                                                                                                                                                //
            initialYVel = Physics.gravity.y * time / -2;                                                                                                                 //
        }                                                                                                                                                                //

        rb.velocity = new Vector3(faceLeft ? -initialXVel : initialXVel, initialYVel, 0); //Set velocity
        ////debug.log(rb.velocity);
        canJump = false;
        inAir = true;
        runningDelay = flightTime + jumpDelay;
    }

    void Slamming() {
        if (inAir) {
            if (Mathf.Sqrt(Mathf.Pow(rb.velocity.x,2) + Mathf.Pow(rb.velocity.y,2)) > 5)
            {
                if (Physics.Raycast(transform.position, Gamemanager.Instance.player.transform.position + pivotAdjust - transform.position, out RaycastHit hit, slamTrigger.radius))
                {
                    if (hit.collider.CompareTag("Player") && canHurtPlayer)
                    {
                        Contact(hit.collider);
                    }
                }
            }
        }
        if (slamming) {
            float scale = Time.deltaTime / slammingTime;
            slamTimer -= Time.deltaTime;
            slamTrigger.radius += scale * slamTriggerSize;
            if (Physics.Raycast(transform.position, (Vector2)Gamemanager.Instance.player.transform.position + (Vector2)pivotAdjust - (Vector2)transform.position, out RaycastHit hit, slamTrigger.radius))
            {
                if (hit.collider.gameObject.CompareTag("Player") && canHurtPlayer)
                {
                    Contact(hit.collider);
                }
            }
            if (slamTimer <= 0) {
                slamming = false;
                slamTrigger.radius = slamTriggerRadius;
                canHurtPlayer = true;
                //particles.Clear();
            }
        }
    }
    public void Contact(Collider other) {
        if (other.gameObject.CompareTag("Player") && canHurtPlayer) {
            slamTimer = 0;
            canHurtPlayer = false;
            if (!faceLeft)
            {
                Gamemanager.Instance.playerScript.Hurt(slamDamage, 1);
            }
            else
            {
                Gamemanager.Instance.playerScript.Hurt(slamDamage, -1);
            }
            
        }
    }
    bool IsStuck() {//Check if their is a wall between enemy and player if enemy can see the player. If stuck, jumps back a little bit.
        if (canJump)
        {
            ////debug.log("Is Stuck");
            if (!faceLeft)
            {
                if (Physics.Raycast(transform.position, Vector3.right, out RaycastHit hit,1))
                {
                    ////debug.log("Raycast right");
                    if (!hit.collider.gameObject.CompareTag("Player") && !hit.collider.gameObject.CompareTag("Enemy"))
                    {
                        faceLeft = true;
                        model.transform.localRotation = Quaternion.Euler(Vector3.down * 90);
                        Launch(2, 1, true);
                        return true;
                    }
                }
            }
            else
            {
                if (Physics.Raycast(transform.position,Vector3.left, out RaycastHit hit, 1))
                {
                    ////debug.log("Raycast left");
                    if (!hit.collider.gameObject.CompareTag("Player") && !hit.collider.gameObject.CompareTag("Enemy"))
                    {
                        faceLeft = false;
                        model.transform.localRotation = Quaternion.Euler(Vector3.up * 90);
                        Launch(2, 1, true);
                        return true;
                    }
                }
            }
        }
        return false;
    }
    private void OnCollisionEnter(Collision collision)
    {
        if (transform.position.y - collision.gameObject.transform.position.y >= .9f && canSlam) {
            slamming = true;
            slamTimer = slammingTime;
            runningDelay += slamTimer;
            particles.Play();
        }
        inAir = false;
        rb.velocity = Vector3.zero;
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
            rb.velocity = Vector3.zero;
            rb.useGravity = false;
            particles.Pause();
        }
        else
        {
            modelAnim.enabled = true;
            if (pauseOverride == 2)
            {
                rb.velocity = pausedVel;
            }
            rb.useGravity = true;
            if (particles.isPaused)
            {
                particles.Play();
            }
        }
    }
}
