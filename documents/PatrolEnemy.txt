using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PatrolEnemy : MonoBehaviour
{
    EnemyAI enemyScript;
    Rigidbody rb;
    [Header("Object reference for prefab")]
    [SerializeField] GameObject face;
    [SerializeField] GameObject model;
    [SerializeField] GameObject parent;
    [SerializeField] GameObject chaseDetection;
    [HideInInspector] public Animator modelAnim;
    float lookAroundTimer;//Timer to determine if enemy should play idle animation
    bool isWaiting;//Waiting for idle anim to end;

    [Space]
    [Header("Basic properties")]
    public float basicSpeed;//Walking speed
    float speed;//Speed the enemy is moving at
    public bool faceLeft = true;//Direction currently facing
    bool prevFacing;//Facing direction from previous fram
    private int targetSide;//Target side for hurting player
    public bool paused;//Is paused?

    #region Attack Variables
    [Space]
    [Header("Attack properties")]
    public float attackDelay = 3;//Delay between each attack
    float attackTimer = 0;//Timer to count the attack delay
    public int damage = 25;
    public float attackDistance = 2;
    public float attackAngle = 90;//Angle of attack
    [SerializeField] Vector3 pivotAdjust;//Adjust where the enemy is looking, relative to the player's position
    [HideInInspector] public bool canAttack = true;
    
    [Space]
    public float afterAttackDelay;//Delay where the enemy cannot do anything after attacking
    float afterAttackTimer = 0;//Timer to count after attack delay
    float attackingDelay = .15f;//Time to sync with animation
    float runningattackingTimer;//Timer to count for attacking delay
    bool hasAttacked = true;
    
    [Space]
    [Tooltip("Amount of time to react to changing direction")]
    public float chaseDelay;//Delay between changing directions to respond to the player
    [SerializeField] float chasingSpeed;//Running speed
    [HideInInspector] public float chaseTimer;//Timer to count the chase delay
    [HideInInspector] public bool chasePlayer = false;//Is chasing the palyer?
    #endregion

    bool playAnimation;

    // Start is called before the first frame update
    void Start()
    {
        rb = GetComponent<Rigidbody>();
        enemyScript = GetComponent<EnemyAI>();
        enemyScript.parent = parent;
        modelAnim = model.GetComponent<Animator>();

        if (faceLeft)                              //
        {                                          //
            rb.velocity = Vector2.left * speed;    //
            targetSide = -1;                       //
        }                                          //Set the inital direction of movement
        else                                       //
        {                                          //
            rb.velocity = Vector2.right * speed;   //
            targetSide = 1;                        //
        }                                          //

        lookAroundTimer = Random.Range(10f, 20f);//Set idle delay
    }

    // Update is called once per frame
    void Update()
    {
        if (!paused) {

            if (chasePlayer && !isWaiting)
            {
                if (prevFacing != Gamemanager.Instance.player.transform.position.x < transform.position.x) //If the player is currently behind the enemy
                {                                                                                          //
                    ////debug.log("faceLeft != prevFacing");                                                 //
                    chaseTimer = chaseDelay;                                                               //
                }                                                                                          //
                else
                {
                    if (chaseTimer <= 0)                                                                                    //
                    {                                                                                                       //
                        faceLeft = Gamemanager.Instance.player.transform.position.x < transform.position.x ? true : false;  //Set the new facing direction 
                    }                                                                                                       //
                    chaseTimer -= Time.deltaTime;                                                                           //
                }
                speed = chasingSpeed;//Set to running speed
            }
            else
            {
                speed = basicSpeed;//Walking speed
            }
            if (enemyScript.afterDamageTimer <= 0 && afterAttackTimer <= 0 && !isWaiting) //
            {                                                                             //Set velocity based on direction
                rb.velocity = (faceLeft ? Vector3.left : Vector3.right) * speed;          //
            }                                                                             //

            if (lookAroundTimer <= 0)                                                            //Play idle animation
            {                                                                                    //
                if (!chasePlayer && enemyScript.afterDamageTimer <= 0 && afterAttackTimer <= 0)  //If not in the middle of fighting the player
                {                                                                                //
                    rb.velocity = Vector3.zero;                                                  //Stop enemy
                    int trigger = Random.Range(1, 3);                                            //Random choice to scream or look around
                    if (trigger == 1)                                                            //
                    {                                                                            //
                        modelAnim.SetTrigger("Look");                                            //
                    }                                                                            //
                    else                                                                         //
                    {                                                                            //
                        modelAnim.SetTrigger("Scream");                                          //
                    }                                                                            //
                    lookAroundTimer = Random.Range(10f, 20f);                                    //Set the timer
                }                                                                                //
            }                                                                                    //
            else                                                                                 //
            {                                                                                    //
                lookAroundTimer -= Time.deltaTime;                                               //Decrement the timer if > 0
            }                                                                                    //

            if (faceLeft)                                                             //Set the models and triggers
            {                                                                         //depending on facing direction
                face.transform.localPosition = (Vector3.left + Vector3.up) * .5f;     //
                model.transform.localRotation = Quaternion.Euler(Vector3.down * 90);  //  
                model.transform.localPosition = Vector3.right * .04f + Vector3.down;  //
                chaseDetection.transform.localPosition = Vector3.left * 1.25f;        //
            }                                                                         //
            else                                                                      //
            {                                                                         //
                face.transform.localPosition = (Vector3.right + Vector3.up) * .5f;    //
                model.transform.localRotation = Quaternion.Euler(Vector3.up * 90);    //
                model.transform.localPosition = Vector3.left * .04f + Vector3.down;   //
                chaseDetection.transform.localPosition = Vector3.right * 1.25f;       //
            }                                                                         //
            Attack();
            AfterAttack();
            ChooseAnimation();
            prevFacing = Gamemanager.Instance.player.transform.position.x < transform.position.x;
            //if (Input.GetButtonDown("Fire1"))
            //{
            //   modelAnim.SetTrigger("Dizzy");
            //}
        }
    }

    public void SwapDirection() //Changing directions
    {
        rb.velocity = new Vector2(rb.velocity.x * -1, rb.velocity.y);
        faceLeft = !faceLeft;
        face.transform.localPosition = new Vector3(face.transform.localPosition.x * -1, face.transform.localPosition.y, face.transform.localPosition.z);
        targetSide *= -1;
    }

    #region Attack
    void Attack()
    {
        //Draw attack angle lines
        float y = Mathf.Tan(Mathf.Deg2Rad * (attackAngle/2));
        Debug.DrawLine(transform.position, ((faceLeft ? Vector3.left : Vector3.right) + (Vector3.up * y)).normalized * attackDistance + transform.position, Color.red);
        Debug.DrawLine(transform.position, ((faceLeft ? Vector3.left : Vector3.right) + (Vector3.down * y)).normalized * attackDistance + transform.position, Color.red);
        Debug.DrawRay(transform.position, (faceLeft ? Vector3.left : Vector3.right) * attackDistance, Color.red);

        if (Vector3.Distance(transform.position, Gamemanager.Instance.player.transform.position + pivotAdjust) <= attackDistance && !isWaiting) //Check if in range and not in idle anim
        {
            
            //modelAnim.SetTrigger("Scream");
            //if (modelAnim.GetCurrentAnimatorStateInfo(0).IsName("Scream"))
            //{
            //    rb.velocity = Vector3.zero;
            //}

            Vector3 targetDir = Gamemanager.Instance.player.transform.position + pivotAdjust - transform.position; //direction towards player

            float angle = Vector3.Angle(targetDir, (faceLeft ? Vector3.left : Vector3.right) * attackDistance); //Angle of the direction
            Debug.DrawRay(transform.position, targetDir,Color.red);
            if (Physics.Raycast(transform.position, targetDir, out RaycastHit r, attackDistance)) //Check if the player is in range
            {                                                                                     //of the enemy's attack distance
                if (angle <= attackAngle && r.collider.gameObject.CompareTag("Player"))           //
                {                                                                                 //
                    rb.velocity = Vector3.zero;                                                   //Stop the enemy
                    if (canAttack && attackTimer <= 0)                                            //
                    {                                                                             //
                        chasePlayer = true;                                                       //Set the enemy to attack and change its
                        modelAnim.SetTrigger("Attack");                                           //state to chase the player
                        Attacking();                                                              //
                        attackTimer = attackDelay;                                                //
                    }                                                                             //
                }                                                                                 //
            }                                                                                     //
        }                                                                                         //

        if (attackTimer > 0) //Attack cooldown timer
        {
            attackTimer -= Time.deltaTime;
        }
    }
    void Attacking()//Set variables to indicate in the middle of attacking
    {
        canAttack = false;
        rb.velocity = Vector3.zero;
        afterAttackTimer = afterAttackDelay;
        runningattackingTimer = attackingDelay;
        hasAttacked = false;
        //Gamemanager.Instance.playerScript.Hurt(damParam, targetSide);//this the fuction for the player get damage
    }
    public void HitPlayer() { //Rycast to make sure player is still in range, hit if is
        Vector3 targetDir = Gamemanager.Instance.player.transform.position + pivotAdjust - transform.position;

        float angle = Vector3.Angle(targetDir, (faceLeft ? Vector3.left : Vector3.right) * attackDistance);
        if (Physics.Raycast(transform.position, targetDir, out RaycastHit r, attackDistance))
        {
            if (angle <= attackAngle && r.collider.gameObject.CompareTag("Player"))
            {
                Gamemanager.Instance.playerScript.Hurt(damage, targetSide);//this the fuction for the player get damage
            }
        }
    }
    void AfterAttack()
    {
        if (runningattackingTimer > 0)               //Timer to delay hit until animation catches up
        {                                            //
            runningattackingTimer -= Time.deltaTime; //
        }                                            //
        else {                                       //
            if (!hasAttacked)                        //
            {                                        //
                HitPlayer();                         //Actual attacking function
                hasAttacked = true;                  //
            }                                        //
        }                                            //

        if (afterAttackTimer > 0)                //
            afterAttackTimer -= Time.deltaTime;  //
        else                                     //Delay to reset ability to attack
        {                                        //
            canAttack = true;                    //
        }                                        //
    }
    void ChooseAnimation() {
        if (!modelAnim.GetCurrentAnimatorStateInfo(0).IsName("Look") && !modelAnim.GetCurrentAnimatorStateInfo(0).IsName("Scream")) //
        {                                                                                                                           //
            isWaiting = false;                                                                                                      //
        }                                                                                                                           //If in the middle of idle anim,
        else {                                                                                                                      //waiting == true,
            isWaiting = true;                                                                                                       //otherwise false
        }                                                                                                                           //

        float absVel = Mathf.Abs(rb.velocity.x); //
        if (absVel != 0) {                       //
            if (absVel == basicSpeed)            //
            {                                    //
                modelAnim.SetTrigger("Walk");    //
            }                                    //Set animation based on speed
            else {                               //
                modelAnim.SetTrigger("Run");     //
            }                                    //
        }                                        //
        else {                                   //
            if (!isWaiting)                      //
            {                                    //
                modelAnim.SetTrigger("Idle");    //
            }                                    //
        }                                        //
    }
    #endregion
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
        else if (pauseOverride == 0) {
            paused = false;
        }
        if (paused)
        {
            modelAnim.enabled = false;
        }
        else {
            modelAnim.enabled = true;
        }
    }

    public void ResetAnim()
    {
        playAnimation = false;
    }
}
