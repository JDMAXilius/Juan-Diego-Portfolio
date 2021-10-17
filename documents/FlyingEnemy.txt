using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public enum FlyingStage
{
    RandomTarget,
    PlayerTarget,
    VerticalPatrol
}

public class FlyingEnemy : MonoBehaviour
{
    [HideInInspector]
    public Rigidbody rb;
    [SerializeField] GameObject model;
    [SerializeField] GameObject modelBody;
    ShootScript shootScript;
    //float rotateBody = -50;
    [HideInInspector] public Animator modelAnim;

    [Header("Basic properties")]
    public float speed = 5;
    public int contactDamage = 10;
    public bool shootTowardsPlayer;
    public bool paused;

    //Space for birds to freely fly in
    BoxCollider flySpace;

    //
    //Variables used throughout the program
    //
    Vector3 target;
    Vector3 otherSide;
    //Distance from target to bird that the bird considers "Complete"
    float threshold = .25f;

    float distanceToTarget;

    //Distance from center of flyspace to corner of flyspace    
    //float outOfBoundsDistance;

    //Vector3 prevPos;

    //If the bird gets stuck
    //float stalledXFrameCount = 0;
    //float stalledYFrameCount = 0;

    [Space]
    public FlyingStage pathing;
    [Space]
    [Header("Player chasing")]
    [SerializeField] bool canChasePlayer;
    [SerializeField] float playerFollowingTime;
    float playerRunningTimer;
    [Range(0, 100)]
    [SerializeField] int percentChanceToChasePlayer = 30;
    [SerializeField] Vector3 pivotAdjust;
    [Space]
    [Header("Vertical Patrol")]
    [SerializeField] float verticalPatrolDistance;
    public bool shootScriptOnStart;

    [Space]
    [Header("Attacking")]
    bool attacking;
    [SerializeField] float attackDistance;
    [SerializeField] float attackDelay;
    float attackTimer;
    [SerializeField] float extraDetection;
    // Start is called before the first frame update
    void Start()
    {
        flySpace = GetComponentInParent<BoxCollider>();
        rb = GetComponent<Rigidbody>();
        modelAnim = model.GetComponent<Animator>();

        //if (flySpace)
        //{
            //outOfBoundsDistance = Mathf.Sqrt(((flySpace.size.x / 2) * (flySpace.size.x / 2)) + ((flySpace.size.y / 2) * (flySpace.size.y / 2)));
        //}

        if (pathing != FlyingStage.VerticalPatrol)
        {
            NewTarget();
            BoundsCheck();
        }
        else
        {
            target = new Vector2(transform.position.x, transform.position.y + verticalPatrolDistance / 2);
            otherSide = target;
            otherSide.Set(otherSide.x, otherSide.y - verticalPatrolDistance, otherSide.z);
            transform.up = Vector3.up;
            rb.velocity = transform.up * speed;
        }
        shootScript = GetComponent<ShootScript>();
        shootScriptOnStart = shootScript.enabled;
    }

    // Update is called once per frame
    void Update()
    {
        if (!paused)
        {
            Debug.DrawRay(transform.position, (Gamemanager.Instance.player.transform.position + pivotAdjust - transform.position).normalized * (shootScript.projectileLifeTime*shootScript.bulletSpeed + extraDetection), Color.green);
            if (Vector3.Distance(transform.position, Gamemanager.Instance.player.transform.position + pivotAdjust) < (shootScript.projectileLifeTime * shootScript.bulletSpeed + extraDetection))
            {
                shootScript.canShoot = true;
                if (shootScript.runningTimer < 1)
                {
                    modelAnim.SetTrigger("Attack02");
                    rb.velocity = Vector3.zero;
                    attacking = true;
                    if (shootTowardsPlayer)
                    {
                        transform.up = Gamemanager.Instance.player.transform.position + pivotAdjust - transform.position;
                        transform.Rotate((transform.up.x < 0 ? Vector3.up : Vector3.down) * 90);
                    }
                }
                else
                {
                    attacking = false;
                    transform.up = target - transform.position;
                    transform.Rotate((transform.up.x < 0 ? Vector3.up : Vector3.down) * 90);
                }
            }
            else
            {
                shootScript.canShoot = false;
                //Check if player in range of melee attack
                if (pathing == FlyingStage.PlayerTarget && !attacking && attackTimer <= 0)
                {
                    if (Physics.Raycast(transform.position, Gamemanager.Instance.player.transform.position + pivotAdjust - transform.position, out RaycastHit hit, attackDistance))
                    {
                        if (hit.collider.CompareTag("Player"))
                        {
                            modelAnim.SetTrigger("Attack01");
                            StartCoroutine(Attack(hit.collider));
                            attacking = true;
                            attackTimer = attackDelay;
                        }
                    }
                }
            }

            if (pathing == FlyingStage.VerticalPatrol)
            {
                if (!attacking)
                {
                    modelAnim.SetBool("Forward", false);
                }
                Debug.DrawLine(target, transform.position, Color.red);
                Debug.DrawLine(otherSide, transform.position, Color.red);
                if (transform.position.x < Gamemanager.Instance.player.transform.position.x)
                {
                    transform.rotation = Quaternion.Euler(transform.rotation.x, -90, transform.rotation.z);
                }
                else
                {
                    transform.rotation = Quaternion.Euler(transform.rotation.x, 90, transform.rotation.z);
                }
                distanceToTarget = Vector3.Distance(target, transform.position);
                if (distanceToTarget <= threshold)
                {
                    verticalPatrolDistance *= -1;
                    NewTarget();
                    transform.up = Vector3.up;
                }

            }
            else
            {
                distanceToTarget = Vector3.Distance(target, transform.position);//Update distance to target
                if (!attacking)
                {
                    modelAnim.SetBool("Forward", true);
                }
                if (distanceToTarget <= threshold)//Target reached
                {
                    if (playerRunningTimer <= 0 && canChasePlayer)// If can chase player
                    {
                        if (Random.Range(0, 101) < percentChanceToChasePlayer)//Random chance to chase player
                        {
                            pathing = FlyingStage.PlayerTarget;
                            playerRunningTimer = playerFollowingTime;
                        }
                        else
                        {
                            pathing = FlyingStage.RandomTarget;
                        }
                    }
                    if (!attacking)
                    {
                        NewTarget();//Set new target now that original has been reached
                    }
                    distanceToTarget = Vector3.Distance(target, transform.position);//Update distance
                }

                if (playerRunningTimer > 0)                  //
                {                                            //If still chasing the player,
                    playerRunningTimer -= Time.deltaTime;    //lower timer,
                    NewTarget();                             //set target to updated player location
                }                                            //
                else                                         //
                {                                            //
                    if (pathing == FlyingStage.PlayerTarget) //
                    {                                        //
                        pathing = FlyingStage.RandomTarget;  //
                        NewTarget();                         //if the timer ran out, set a random target
                    }                                        //
                }                                            //

                if (attackTimer > 0) {
                    attackTimer -= Time.deltaTime;
                }

                BoundsCheck();
                if (attacking)                              //
                {                                           //
                    rb.velocity = Vector3.zero;             //
                }                                           //
                else {                                      //
                    rb.velocity = transform.up * speed;     //
                }                                           //
            }
        }
    }
   
    IEnumerator Attack(Collider collide) {//Delayed attack to pair with animation

        float timer = 1;
        //modelAnim.SetTrigger("Attack01");
        while (timer > .5f) {//change the right side of this to pair with animation
            if (!paused)
            {
                timer -= Time.deltaTime;
            }
            yield return null;
        }
        Contact(collide);
        while (timer > 0) {
            if (!paused)
            {
                timer -= Time.deltaTime;
            }
            yield return null;
        }
        attacking = false;
    }
    private void OnDrawGizmos()
    {
        if (pathing == FlyingStage.VerticalPatrol)
        {
            Debug.DrawLine(new Vector2(transform.position.x, transform.position.y + verticalPatrolDistance / 2), transform.position, Color.red);
            Debug.DrawLine(new Vector2(transform.position.x, transform.position.y - verticalPatrolDistance / 2), transform.position, Color.red);
        }
    }
    void NewTarget()
    {
        if (pathing == FlyingStage.RandomTarget && playerRunningTimer <= 0) //New random target    //////////// this line creates the erros////////////////////////////////////////////////////////////////////
        {
            target.Set(Random.Range(flySpace.transform.position.x - flySpace.size.x / 2, flySpace.transform.position.x + flySpace.size.x / 2),
                                 Random.Range(flySpace.transform.position.y - flySpace.size.y / 2, flySpace.transform.position.y + flySpace.size.y / 2),0);
            int limit = 100;
            while (Physics.Raycast(transform.position, target - transform.position, Vector3.Distance(transform.position, target)) && limit > 0) {
                limit--;
                target.Set(Random.Range(flySpace.transform.position.x - flySpace.size.x / 2, flySpace.transform.position.x + flySpace.size.x / 2),
                                 Random.Range(flySpace.transform.position.y - flySpace.size.y / 2, flySpace.transform.position.y + flySpace.size.y / 2),0);
            }
            transform.up = target - transform.position;
            transform.Rotate((transform.up.x < 0 ? Vector3.up : Vector3.down) * 90);
            rb.velocity = transform.up * speed;
        }
        else if (pathing == FlyingStage.PlayerTarget)//Update based on the player position
        {
            target = Gamemanager.Instance.player.transform.position;
            transform.up = target - transform.position;
            transform.Rotate((transform.up.x < 0 ? Vector3.up : Vector3.down) * 90);
            if (!attacking)
            {
                rb.velocity = transform.up * speed;
            }
        }
        else//Set target to other side of vertical patrol
        {
            target = new Vector2(target.x, target.y + verticalPatrolDistance);
            otherSide = target;
            otherSide.Set(otherSide.x, otherSide.y - verticalPatrolDistance, otherSide.z);
            speed = -speed;
            rb.velocity = transform.up * speed;
        }
    }
    void NewTarget(Vector2 targ)//Set target to a specific point
    {
        target = targ;
        transform.up = target - transform.position;
        transform.Rotate((transform.up.x < 0 ? Vector3.up : Vector3.down) * 90);
        rb.velocity = transform.up * speed;
    }
    void BoundsCheck()//Check if bat is outside 
    {
        if (pathing == FlyingStage.RandomTarget)
        {
            if ((transform.position.x < (flySpace.transform.position.x - flySpace.size.x / 2) || transform.position.x > (flySpace.transform.position.x + flySpace.size.x / 2)) || (transform.position.y < (flySpace.transform.position.y - flySpace.size.y / 2) || transform.position.y > (flySpace.transform.position.y + flySpace.size.y / 2)))
            {
                NewTarget(flySpace.transform.position);
            }
        }
    }
    private void OnCollisionEnter(Collision collision)//If collision, set a new target and stop following the player
    {
        if (pathing != FlyingStage.VerticalPatrol)
        {
            pathing = FlyingStage.RandomTarget;
            playerRunningTimer = 0;
        }
        NewTarget();
    }
    private void OnCollisionStay(Collision collision)
    {
        if (pathing != FlyingStage.VerticalPatrol)
        {
            pathing = FlyingStage.RandomTarget;
            playerRunningTimer = 0;
        }
        NewTarget();
    }
    public void Contact(Collider other)
    {
        if (other.gameObject.tag == "Player")
        {
            if (transform.position.x < other.gameObject.transform.position.x)
            {
                Gamemanager.Instance.playerScript.Hurt(contactDamage, 1);
            }
            else
            {
                Gamemanager.Instance.playerScript.Hurt(contactDamage, -1);
            }
            if (pathing != FlyingStage.VerticalPatrol)
            {
                ////debug.log("Random Flying Stage");
                playerRunningTimer = 0;
                pathing = FlyingStage.RandomTarget;
                NewTarget();
                BoundsCheck();
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
            rb.velocity = Vector3.zero;
        }
        else
        {
            modelAnim.enabled = true;
            rb.velocity = transform.up * speed;
        }
    }
}

