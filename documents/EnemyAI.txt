using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.AI;

public class EnemyAI : MonoBehaviour
{
    [HideInInspector] public GameObject parent;
    [HideInInspector] public Rigidbody rb;
    //[SerializeField] public MeshRenderer mesh;
    public GameObject FlotingTextPrefab;
    public GameObject DropOrb;
    public GameObject ManaOrb;
    public GameObject particuleExplode;
    public GameObject particuleHit;
    public GameObject particuleAttack;

    [HideInInspector] public PatrolEnemy patrolScript = null;
    [HideInInspector] public TurretEnemy turretScript = null;
    [HideInInspector] public FlyingEnemy flyScript = null;
    [HideInInspector] public SlamEnemy slamScript = null;
    [HideInInspector] public ShootScript shootScript = null;
    [HideInInspector] public BreakableObject breakScript = null;
    [HideInInspector] public BossMain bossScript = null;

    bool deathHasBeenTriggered;
    float destroyTimer;

    public int HP = 100;
    bool isdeath = false;
    public bool paused;

    #region Damaged Variables
    private bool canTakeDamage = true;
    [SerializeField] float afterDamageDelay;
    [HideInInspector] public float afterDamageTimer = 0;
    bool pausedUsedInDeath;
    Animator anim = null;
    public bool frozen;
    [SerializeField] float frozenTimer;
    float runningFrozen;
    [SerializeField] bool useOrbs = true;
    //public Vector3 damageForce;
    #endregion

    public AudioSource sfxAudioSource;
    //public ParticleSystem ParticuleSource;
    //public AnimatorFunctions animatorFunctions;
    //
    public Vector3 moveDirection;
    bool playAnimation;

    #region Unused variables(Has link to other comments of code)
    //private bool isAttacking = false;
    //RaycastHit hit;
    //public NavMeshAgent agent;
    //private bool attackAgin = true;
    //public int attackRange = 3;

    #endregion

    // Start is called before the first frame update
    void Start()
    {
        //Gamemanager.Instance.animatorFunctionsScript.PlayDieEmitParticles();
        //AnimatorFunctions.Instance.PlayDieEmitParticles();
        //mesh = GetComponent<MeshRenderer>();
        rb = GetComponent<Rigidbody>();
        // mesh = GetComponent<MeshRenderer>();
        TryGetComponent<PatrolEnemy>(out patrolScript);
        TryGetComponent<TurretEnemy>(out turretScript);
        TryGetComponent<FlyingEnemy>(out flyScript);
        TryGetComponent<SlamEnemy>(out slamScript);
        TryGetComponent<ShootScript>(out shootScript);
        TryGetComponent<BreakableObject>(out breakScript);
        TryGetComponent<BossMain>(out bossScript);

        HUDScript.pauseGame += Pause;
    }

    // Update is called once per frame
    void Update()
    {
        
       
        if (!paused)
        {
            if (transform.position.z != 0)
            {
                transform.position.Set(transform.position.x, transform.position.y, 0);
            }
            if (runningFrozen > 0)
            {
                runningFrozen -= Time.deltaTime;
            }
            else {
                frozen = false;
                if (patrolScript)
                {
                    patrolScript.Pause(0);
                }
                else if (flyScript)
                {
                    flyScript.Pause(0);
                    if (shootScript && flyScript.shootScriptOnStart)
                    {
                        shootScript.enabled = true;
                    }
                }
                else if (slamScript)
                {
                    slamScript.Pause(0);
                }
                else if (turretScript)
                {
                    turretScript.Pause(0);
                    if (shootScript && turretScript.shootScriptOnStart)
                    {
                        shootScript.enabled = true;
                    }
                }
                else if (bossScript) {
                    bossScript.Pause(0);
                }
            }

            if (frozen) {
                if (patrolScript)
                {
                    //patrolScript.Pause(1);
                    patrolScript.modelAnim.SetTrigger("Dizzy");
                }
                else if (flyScript)
                {
                    flyScript.Pause(1);
                }
                else if (slamScript)
                {
                    slamScript.Pause(1);
                    if (slamScript.GetComponent<ParticleSystem>().isPaused)
                    {
                        slamScript.GetComponent<ParticleSystem>().Play();
                    }
                }
                else if (turretScript)
                {
                    turretScript.Pause(1);
                }
                else if (bossScript) {
                    bossScript.Pause(1);
                }
                if (shootScript) {
                    shootScript.enabled = false;
                }
            }
            //agent.destination = player.transform.position;
            Health();
            AfterDamage();
        }
        //if (Input.GetButtonDown("Fire1"))
        //{
        //    patrolScript.modelAnim.SetTrigger("Dizzy");
        //}
    }

    #region Damage
    public void takeDamage(int damage, bool freezeHit = false)
    {
        ////debug.log("Enemy got hit");
        frozen = freezeHit;
        if (frozen) {
            runningFrozen = frozenTimer;
            //if (patrolScript)
            //{
            //    patrolScript.modelAnim.SetTrigger("Dizzy");
            //}
        }
        if (canTakeDamage)
        {
            canTakeDamage = false;
            if (!bossScript && !flyScript)
            {
                rb.velocity = Vector3.zero;
            }
            //mesh.material.color = Color.red;
            afterDamageTimer = afterDamageDelay;

            if (patrolScript)
            {
                patrolScript.canAttack = false;
                patrolScript.chasePlayer = true;
                patrolScript.chaseTimer = patrolScript.chaseDelay;
            }

            HP -= damage;
            if (HP > 0)
            {
                if (patrolScript)
                {
                    patrolScript.modelAnim.SetTrigger("Hit");
                }
                else if (flyScript)
                {
                    flyScript.modelAnim.SetTrigger("Hit");
                }
                else if (slamScript)
                {
                    slamScript.modelAnim.SetTrigger("Hit");
                }
                else if (turretScript)
                {
                    turretScript.modelAnim.SetTrigger("Hit");
                }
                else if (bossScript) {
                    bossScript.takedamage();
                }
            }
            Gamemanager.Instance.animatorFunctionsScript.PlayOnHitSound();
            GameObject newparticule = (GameObject)Instantiate(particuleHit, transform.position + new Vector3(0, Random.Range(0, 2)), Quaternion.identity);
           
            Destroy(newparticule, 1);
            if (!frozen && !bossScript && !gameObject.name.Contains("DestructableBox"))
            {
                KnockbackHit();
            }

            //mesh.material.color = Color.red;
            StartCoroutine(FreezeEffect(.1f, .3f));
            afterDamageTimer = afterDamageDelay;
            canTakeDamage = false;
            ////debug.log("Enemy got hit");

            if (patrolScript)
            {
                patrolScript.canAttack = false;
                patrolScript.chasePlayer = true;
                patrolScript.chaseTimer = patrolScript.chaseDelay;
            }
            if (FlotingTextPrefab != null && HP > 0 && !gameObject.name.Contains("DestructableBox"))
            {
                ShowFlotingText();
            }
            //rb.AddForce((patrolScript.targetSide * 4,0,0),1);
            //rb.transform.localPosition.y += 4f;
            Health();
           
        }
    }

    public IEnumerator FreezeEffect(float length, float timeScale)
    {
        //mesh.material.color = Color.white;
        //Time.timeScale = timeScale;
        yield return new WaitForSeconds(length);
        //mesh.material.color = Color.red;
        //Time.timeScale = 1;

    }
    public void KnockbackHit()
    {
        if (!paused && !frozen)
        {
            moveDirection = ((Vector2)transform.position - (Vector2)Gamemanager.Instance.player.transform.position).normalized;
            rb.AddForce((Vector2)moveDirection * 200f);
        }
        //rb.(moveDirection * -500f);
    }
    void AfterDamage()
    {
        if (afterDamageTimer > 0)
            afterDamageTimer -= Time.deltaTime;
        else
        {
            //mesh.material.color = Color.red;
            canTakeDamage = true;
        }
    }

    #endregion

    private void Health()
    {
        //If health < 0, destroy me
        if (HP <= 0)
        {
            if (!isdeath)
            {
                if (useOrbs)
                {
                    var go = Instantiate(DropOrb, transform.position + new Vector3(Random.Range(-2, 2), Random.Range(0, 3)), Quaternion.identity);
                    go = Instantiate(ManaOrb, transform.position + new Vector3(Random.Range(-2, 2), Random.Range(0, 3)), Quaternion.identity);
                    int amountToDrop = Random.Range(1, 4);
                    ////debug.log("Dropped Orb amount: " + amountToDrop);
                    for (int i = 0; i < amountToDrop; i++)
                    {
                        int rand = Random.Range(1, 101);
                        ////debug.log("Random Number: " + rand);
                        if (rand > 25)
                        {
                            go = Instantiate(DropOrb, transform.position + new Vector3(Random.Range(0, 2), Random.Range(0, 3)), Quaternion.identity);
                            ////debug.log("Health Orb");
                        }
                        else
                        {
                            go = Instantiate(ManaOrb, transform.position + new Vector3(Random.Range(0, 2), Random.Range(0, 3)), Quaternion.identity);
                            ////debug.log("Mana Orb");
                        }
                    }
                }
                Gamemanager.Instance.playerScript.sceneCamera.Shake(5, 0.2f);
                //Gamemanager.Instance.animatorFunctionsScript.PlayDieEmitParticles();
                Instantiate(particuleExplode, transform.position /*+ new Vector3(0, Random.Range(0,2))*/, Quaternion.identity);
                isdeath = true;
            }
            else
            {
                //animatorFunctions.PlayDieSound();
                //sfxAudioSource.Play(Gamemanager.Instance.animatorFunctionsScript.PlayDieSound());
                Death();
            }
        }
    }
    private void Death()
    {
        if (!bossScript)
        {
            if (breakScript)
            {
                breakScript.ChangeShape();
                return;
            }
            if (patrolScript)
            {
                anim = patrolScript.modelAnim;
                patrolScript.enabled = false;
            }
            else if (turretScript)
            {
                anim = turretScript.modelAnim;
                turretScript.enabled = false;
            }
            else if (flyScript)
            {
                anim = flyScript.modelAnim;
                flyScript.enabled = false;
            }
            else if (slamScript)
            {
                anim = slamScript.modelAnim;
                slamScript.enabled = false;
            }
            anim.enabled = true;
            if (shootScript)
            {
                shootScript.enabled = false;
            }

            if (!deathHasBeenTriggered)
            {
                anim.SetBool("Death", true);
                rb.useGravity = true;
                deathHasBeenTriggered = true;
            }
            else
            {
                AnimatorStateInfo info = anim.GetCurrentAnimatorStateInfo(0);
                if (info.IsName("Destroy"))
                {
                    DestroySelf();
                    ////debug.log("Death");
                }
                //DestroySelf();
            }
        }
        else {
            bossScript.Death();
        }
        ////debug.log("Death");
    }
    void DestroySelf()
    {
        //Destroy(parent);
        //Destroy(gameObject);
        if (patrolScript)
        {
            //Instantiate(particuleExplode, transform.position /*+ new Vector3(0, Random.Range(0,2))*/, Quaternion.identity);
            Destroy(parent);//Do not change this, deleting the parent deletes all of the children
            //Destroy(gameObject);
        }
        else
        {
            //Instantiate(particuleExplode, transform.position /*+ new Vector3(0, Random.Range(0,2))*/, Quaternion.identity);
            Destroy(gameObject);
        }
    }

    void ShowFlotingText()
    {
        var go = Instantiate(FlotingTextPrefab, transform.position, Quaternion.identity, transform);
        go.GetComponent<TextMesh>().text = HP.ToString();
    }

    public void Pause() {
        paused = !paused;
        if (paused)
        {
            rb.Sleep();
        }
        else {
            rb.WakeUp();
        }

        if (patrolScript)
        {
            patrolScript.Pause();
        }
        else if (flyScript)
        {
            flyScript.Pause();
        }
        else if (turretScript)
        {
            turretScript.Pause();
        }
        else if (slamScript)
        {
            slamScript.Pause();
        }
        else if (bossScript) {
            bossScript.Pause();
        }
        if (shootScript)
        {
            shootScript.Pause();
        }

    }
    private void OnDestroy()
    {
        HUDScript.pauseGame -= Pause;
    }
}

