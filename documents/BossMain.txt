using System.Collections;
using System.Collections.Generic;
using UnityEngine;

enum Stage{ 
    Dashing,
    Slamming,
    BulletHell
}
public class BossMain : MonoBehaviour
{
    [SerializeField] GameObject spiderBody;
    [SerializeField] GameObject orcBody;
    Rigidbody rb;
    EnemyAI enemyScript;

    int maxHP = 100;
    public bool start = true;
    bool faceLeft;
    [SerializeField] Stage stage;
    Stage prevStage;
    bool paused;
    bool isTransition;
    Vector3 pausedVel;
    ParticleSystem particles;

    [SerializeField] SphereCollider activeCollision;
    [SerializeField] GameObject sphereVisual;
    [SerializeField] GameObject key;
    float actColRadius;

    [SerializeField] Transform leftBound;
    [SerializeField] Transform rightBound;
    [SerializeField] Transform bulletHellPos;

    //animations
    [SerializeField] GameObject spider;
    Animator spiderAnim;
    [SerializeField] GameObject orc;
    Animator orcAnim;
    [SerializeField] GameObject bat;
    Animator batAnim;

    //Dashing
    public bool dazed;
    [SerializeField] float dashSpeed;
    [SerializeField] float dazedTimer;
    float runningDazedTimer;
    [SerializeField] int contactDamage;
    bool canDash = true;

    //Slamming
    bool canJump = true;
    bool inAir;
    bool canHurtPlayer = true;
    [SerializeField] float slamRadius;
    [SerializeField] float slamTime;
    float runningSlamTime;
    [SerializeField] Vector3 pivotAdjust;
    [SerializeField] float maxJumpDistance;
    [SerializeField] float flightTime;
    [SerializeField] int slamDamage;
    bool dashWhileSlam;
    bool isSlam;
    bool transitionedToSlam;

    //BulletHell
    public bool finalStage;
    [SerializeField] int bulletHellHealthIncrease;
    float runningHealthIncrease;
    [SerializeField] float wheelTime;
    bool wheelPattern = true;
    [SerializeField] float jiggleTime;
    bool jigglePattern;
    [SerializeField] float seekingTime;
    bool heatSeeking;
    bool isInPattern;
    float runningPatternTime;
    [SerializeField] Transform gunsPivot;
    [SerializeField] float bulletSpeed;
    [SerializeField] float bulletLifeTime;
    [SerializeField] int bulletDamage;
    [SerializeField] List<ShootScript> guns;
    [SerializeField] List<GameObject> boxes;

    [SerializeField] AudioClip screamNoise;
    bool hasDeathScreamed;
    [SerializeField] AudioClip crashSound;
    [SerializeField] AudioClip poof;
    [SerializeField] AudioClip walking;
    bool hasPoofed;
    [SerializeField] AudioClip explode;
    [SerializeField] AudioClip fireworks;

    [SerializeField] GameObject transitionEffect;
    [SerializeField] GameObject dyingEffect;
    bool dieEffect;
    [SerializeField] GameObject finalExplosion;

    [SerializeField] GameObject platform1;
    [SerializeField] GameObject platform2;
    [SerializeField] GameObject sculpture;
    

    // Start is called before the first frame update
    [System.Obsolete]
    void Start()
    {
        rb = GetComponent<Rigidbody>();
        enemyScript = GetComponent<EnemyAI>();
        maxHP = enemyScript.HP;
        actColRadius = activeCollision.radius;
        spiderAnim = spider.GetComponent<Animator>();
        orcAnim = orc.GetComponent<Animator>();
        batAnim = bat.GetComponent<Animator>();
        for (int i = 0; i < guns.Count; i++) {
            guns[i].gameObject.transform.RotateAround(gunsPivot.position, Vector3.right, 60*(i+1));

        }
        particles = GetComponent<ParticleSystem>();
        particles.startLifetime = slamTime;
        particles.startSpeed = slamRadius / slamTime * 1.1f / 2;
        key.SetActive(false);
    }

    // Update is called once per frame
    void Update()
    {
        if (!paused)
        {
            StageCheck();
            if (dazed)
            {
                if (runningDazedTimer <= 0)
                {
                    dazed = false;
                }
                else
                {
                    runningDazedTimer -= Time.deltaTime;
                }
            }
            if (!finalStage)
            {
                if (rb.velocity != Vector3.zero)
                {
                    if (!enemyScript.sfxAudioSource.isPlaying)
                    {
                        enemyScript.sfxAudioSource.PlayOneShot(walking,2f);
                    }
                    if (spiderAnim.enabled)
                    {
                        spiderAnim.SetBool("Moving", true);
                    }
                }
                else
                {
                    if (!dazed)
                    {
                        if (spiderAnim.enabled)
                        {
                            spiderAnim.SetBool("Moving", false);
                        }
                    }
                }
            }
            if (prevStage == stage && !isTransition)
            {
                if (stage == Stage.Dashing)
                {

                    if (!dazed && canDash)
                    {
                        rb.velocity = new Vector3(dashSpeed * (faceLeft ? -1 : 1), rb.velocity.y, 0);
                    }
                }
                else if (stage == Stage.Slamming)
                {
                    if (!inAir && !canJump)
                    {
                        if (runningSlamTime <= 0)
                        {
                            canJump = true;
                        }
                        else
                        {
                            runningSlamTime -= Time.deltaTime;
                        }
                    }
                    if (canJump)
                    {
                        if (Vector3.Distance(transform.position, Gamemanager.Instance.player.transform.position + pivotAdjust) > 10)
                        {
                            Launch();
                        }
                        else {
                            Launch(55,flightTime,true);
                        }
                    }
                }
                else
                {
                    if (runningHealthIncrease > 0) {
                        runningHealthIncrease -= Time.deltaTime;
                    }
                    else
                    {
                        if (enemyScript.HP < maxHP*.3f)
                        {
                            runningHealthIncrease = 1;
                            enemyScript.HP += bulletHellHealthIncrease;
                        }
                    }
                    if (!isInPattern) {
                        isInPattern = true;
                        if (wheelPattern)
                        {
                            StartCoroutine(WheelPattern());
                        }
                        else if (jigglePattern)
                        {
                            StartCoroutine(JigglePattern());
                        }
                        else if (heatSeeking) {
                            StartCoroutine(HeatSeeking());
                        }
                    }
                }
            }
            else if(prevStage != stage)
            {
                //Debug.Break();
                Transition();
                
            }
            prevStage = stage;
            pausedVel = rb.velocity;
        }
    }

    public void HitWall(bool continueDash = false) {
        isTransition = false;
        ////debug.log("Wall hit");
        if (stage == Stage.Dashing) {
            faceLeft = !faceLeft;
            dazed = true;
            runningDazedTimer = dazedTimer;
            rb.velocity = Vector3.zero;
            enemyScript.sfxAudioSource.PlayOneShot(crashSound);
            Gamemanager.Instance.playerScript.sceneCamera.Shake(1000, 2);
            StartCoroutine(DazedRotate(continueDash));
        }
    }
    IEnumerator DazedRotate(bool continueDash = false) {
        int spinDirection = NegativeOrPositive();
        while (runningDazedTimer > 0)
        {
            if (!paused)
            {
                transform.rotation = Quaternion.Euler(0, transform.rotation.eulerAngles.y + (spinDirection * 180 / dazedTimer * Time.deltaTime), 0);
            }
            yield return null;
        }
        //yield return new WaitForSeconds(dazedTimer);
        transform.rotation = Quaternion.Euler(0, 90 * (faceLeft ? -1 : 1), 0);
        if (dashWhileSlam && !continueDash) {
            stage = Stage.Slamming;
            prevStage = stage;
            dashWhileSlam = false;
        }
        activeCollision.gameObject.GetComponent<ActiveCollision>().canHurt = true;
    }

    void StageCheck() {
        if (enemyScript.HP > maxHP * .6f && !finalStage)
        {
            stage = Stage.Dashing;
        }
        else if (enemyScript.HP > maxHP * .2f && !finalStage)
        {
            if (!dashWhileSlam)
            {
                stage = Stage.Slamming;
            }
            else
            {
                stage = Stage.Dashing;
                prevStage = stage;
            }
        }
        else
        {
            stage = Stage.BulletHell;
            finalStage = true;
        }
        
    }
    void Transition()
    {
        if (stage == Stage.BulletHell)
        {
            enemyScript.sfxAudioSource.PlayOneShot(screamNoise);
            StartCoroutine(TransitionToBat());
        }
        else {
            if (stage == Stage.Slamming && !isSlam)
            {
                enemyScript.sfxAudioSource.PlayOneShot(screamNoise);
                StartCoroutine(ScreamToSlam());
            }
        }
    }

    IEnumerator ScreamToSlam() {
        isTransition = true;
        rb.velocity = Vector3.zero;
        float timer = .5f;
        while (timer > 0) {
            if (!paused) {
                timer -= Time.deltaTime;
            }
            yield return null;
        }
        if (orcAnim.enabled)
        {
            orcAnim.SetTrigger("Scream");
        }
        bool temp = false;
        if (dazed) {
            temp = true;
        }
        //Wait for seconds dependent on scream length
        timer = 2;
        while (timer > 0)
        {
            if (!paused)
            {
                timer -= Time.deltaTime;
            }
            yield return null;
        }
        isTransition = false;
        if (!temp) {
            StartCoroutine(Slamming());
        }
    }

    IEnumerator ToBulletHell() {
        yield return new WaitForSeconds(1.5f);
        while (Vector3.Distance(transform.position, bulletHellPos.position) > .2f) {
            if (!paused)
            {
                Vector3 dir = bulletHellPos.position - transform.position;
                rb.velocity = dir.normalized * 15;
            }
            yield return null;
        }
        rb.velocity = Vector3.zero;
        for (int i = 0; i < 15; i++)
        {
            if (!paused)
            {
                transform.Rotate(0, 6 * (faceLeft ? -1 : 1), 0);
                if (transform.rotation.eulerAngles.y > 180 || transform.rotation.eulerAngles.y < -180)
                {
                    i = 16;
                }
            }
            else {
                i--;
            }
            yield return new WaitForSeconds(.05f);
        }
        transform.rotation = Quaternion.Euler(0, 180, 0);
        gunsPivot.rotation = Quaternion.Euler(0, 180, 0);
        canDash = false;
        canJump = false;
        dashWhileSlam = false;
        gunsPivot.gameObject.SetActive(true);
        platform1.SetActive(true);
        platform2.SetActive(true);
        for (int i = 0; i < guns.Count; i++)
        {
            guns[i].bulletDamage = bulletDamage;
            guns[i].bulletSpeed = bulletSpeed;
            guns[i].projectileLifeTime = bulletLifeTime;
        }
        for (int i = 0; i < boxes.Count; i++) {
            boxes[i].SetActive(true);
        }
        isTransition = false;
        rb.useGravity = false;
    }

    IEnumerator TransitionToBat() {
        isTransition = true;
        rb.velocity = Vector3.zero;
        rb.useGravity = false;
        if (orcAnim.enabled)
        {
            orcAnim.SetTrigger("LastHit");
        }
        if (spiderAnim.enabled)
        {
            spiderAnim.SetTrigger("LastHit");
        }
        if (orcAnim.enabled)
        {
            AnimatorStateInfo animInfo = orcAnim.GetCurrentAnimatorStateInfo(0);
            while (!animInfo.IsName("Destroy"))
            {
                animInfo = orcAnim.GetCurrentAnimatorStateInfo(0);
                yield return null;
            }
        }
        Instantiate(transitionEffect, transform.position,transform.rotation);
        Gamemanager.Instance.playerScript.sceneCamera.Shake(1000, 1.5f);
        enemyScript.sfxAudioSource.PlayOneShot(explode);
        spider.SetActive(false);
        bat.SetActive(true);
        if (bulletHellPos.position.x < transform.position.x)
        {
            faceLeft = true;
        }
        else {
            faceLeft = false;
        }
        transform.rotation = Quaternion.Euler(0, 90 * (faceLeft ? -1 : 1), 0);
        GetComponent<SphereCollider>().radius = 2;
        GetComponent<SphereCollider>().center = new Vector3(0, 2f, .08f);
        activeCollision.enabled = false;
        sphereVisual.SetActive(false);
        StartCoroutine(ToBulletHell());
    }
    void Launch(float jumpDistance = 0, float time = 0, bool overrideDist = false)
    {
        ////debug.log("Launch");
        canHurtPlayer = true;
        if (jumpDistance == 0)
        {
            jumpDistance = maxJumpDistance;
        }
        float vec3Dist = Vector3.Distance(transform.position, Gamemanager.Instance.player.transform.position + pivotAdjust);

        if (time == 0)
        {
            time = flightTime;
        }

        float scaling = vec3Dist / maxJumpDistance;
        if (vec3Dist < maxJumpDistance && !overrideDist)
        {
            time *= scaling;
            jumpDistance = vec3Dist;

        }
        float initialXVel = jumpDistance / time;
        float initialYVel;
        //if (Mathf.Abs(transform.position.y - Gamemanager.Instance.player.transform.position.y + pivotAdjust.y) >= .9f)
        //{
        //    initialYVel = ((-.5f * Physics.gravity.y * time * time) + (Gamemanager.Instance.player.transform.position.y + pivotAdjust.y - transform.position.y)) / time;
        //}
        //else
        //{
            initialYVel = Physics.gravity.y * time / -2;
        //}
        rb.velocity = new Vector3(faceLeft ? -initialXVel : initialXVel, initialYVel, 0);
        canJump = false;
        inAir = true;
    }

    IEnumerator Slamming() {
        enemyScript.sfxAudioSource.PlayOneShot(crashSound);
        Gamemanager.Instance.playerScript.sceneCamera.Shake(1000, 2);
        while (runningSlamTime > 0) {
            //Debug.DrawRay(transform.position, (Gamemanager.Instance.player.transform.position + pivotAdjust - transform.position).normalized * slamRadius * (1 - runningSlamTime / slamTime));

            if (!paused && !enemyScript.frozen)
            {
                if (canHurtPlayer)
                {
                    activeCollision.radius = slamRadius * (1 - runningSlamTime / slamTime) / 2;
                    sphereVisual.transform.localPosition = activeCollision.center;
                    sphereVisual.transform.localScale = Vector3.one * activeCollision.radius * 2;
                    sphereVisual.transform.localRotation = Quaternion.Euler(sphereVisual.transform.localRotation.eulerAngles.x, sphereVisual.transform.localRotation.eulerAngles.y + 5, sphereVisual.transform.localRotation.eulerAngles.z);
                }
                else {
                    sphereVisual.transform.localScale = Vector3.zero;
                    sphereVisual.transform.localRotation = Quaternion.Euler(0, 0, 0);
                }
            }
            yield return null;
        }
        dashWhileSlam = true;
        activeCollision.radius = actColRadius;
        sphereVisual.transform.localScale = Vector3.zero;
        sphereVisual.transform.localRotation = Quaternion.Euler(0, 0, 0);
        stage = Stage.Dashing;
        prevStage = stage;

        if (transitionedToSlam)
        {
            float leftDist = Vector3.Distance(transform.position, leftBound.position);
            float rightDist = Vector3.Distance(transform.position, rightBound.position);
            if (faceLeft)
            {
                if (rightDist < leftDist)
                {
                    HitWall(true);
                }
            }
            else
            {
                if (leftDist < rightDist)
                {
                    HitWall(true);
                }
            }
        }
        else {
            transitionedToSlam = true;
        }
    }

    IEnumerator WheelPattern() {
        runningPatternTime = wheelTime;

        while (runningPatternTime > 0) {
            if (!paused)
            {
                runningPatternTime -= Time.deltaTime;
                gunsPivot.rotation = Quaternion.Euler(0, 0, gunsPivot.rotation.eulerAngles.z + (360 / wheelTime * Time.deltaTime));
            }
            yield return null;
        }
        isInPattern = false;
        jigglePattern = true;
        wheelPattern = false;
    }

    IEnumerator JigglePattern() {
        runningPatternTime = jiggleTime;
        int spinDirection = 1;
        while (runningPatternTime > 0) {
            if (!paused)
            {
                for (int i = 0; i < 15; i++)
                {
                    if (!paused)
                    {
                        runningPatternTime -= Time.deltaTime;
                        gunsPivot.Rotate(0, 0, spinDirection * 4);
                    }
                    else {
                        i--;
                    }
                    yield return new WaitForSeconds(.02f);
                }
                spinDirection = -spinDirection;
            }
            yield return null;
        }
        isInPattern = false;
        heatSeeking = true;
        jigglePattern = false;
    }

    IEnumerator HeatSeeking() {
        for (int i = 0; i < guns.Count; i++) {
            guns[i].trackingBullets = true; 
            guns[i].bulletSpeed = bulletSpeed;
            guns[i].projectileLifeTime = bulletLifeTime/5;
        }
        runningPatternTime = seekingTime;
        while (runningPatternTime > 0) {
            if (!paused)
            {
                runningPatternTime -= Time.deltaTime;
            }
            yield return null;
        }
        for (int i = 0; i < guns.Count; i++)
        {
            guns[i].trackingBullets = false;
            guns[i].bulletDamage = bulletDamage;
            guns[i].bulletSpeed = bulletSpeed;
            guns[i].projectileLifeTime = bulletLifeTime;
        }
        isInPattern = false;
        wheelPattern = true;
        heatSeeking = false;
    }

    public void Contact(Collider other) {
        if (rb.velocity != Vector3.zero || activeCollision.radius > actColRadius)
        {
            if (stage == Stage.Dashing && !dazed)
            {
                Gamemanager.Instance.playerScript.Hurt(contactDamage, faceLeft ? -1 : 1);
            }
            else if (stage == Stage.Slamming && canHurtPlayer)
            {
                canHurtPlayer = false;
                if (runningSlamTime > 0)
                {
                    Gamemanager.Instance.playerScript.Hurt(slamDamage, Gamemanager.Instance.player.transform.position.x < transform.position.x ? -1 : 1);
                    activeCollision.radius = actColRadius;
                }
                else
                {

                    Gamemanager.Instance.playerScript.Hurt(contactDamage, faceLeft ? -1 : 1);
                }
            }
        }
    }
    public void Pause(int pauseOverride = 2) {
        if (pauseOverride == 2)
        {
            if (!enemyScript.frozen)
            {
                paused = !paused;
            }
        }
        else if (pauseOverride == 1)
        {
            paused = true;
        }
        else if (pauseOverride == 0) {
            paused = false;
        }

        if (!paused)
        {
            rb.WakeUp();
        }
        else
        {
            rb.Sleep();
        }

        if (paused)
        {
            orcAnim.enabled = false;
            spiderAnim.enabled = false;
            batAnim.enabled = false;
            rb.velocity = Vector3.zero;
            rb.useGravity = false;
            if (particles.isPlaying && !enemyScript.frozen)
            {
                particles.Pause();
            }
        }
        else
        {
            orcAnim.enabled = true;
            spiderAnim.enabled = true;
            batAnim.enabled = true;
            if (pauseOverride == 2)
            {
                rb.velocity = pausedVel;
            }
            if (!finalStage)
            {
                rb.useGravity = true;
            }
            if (particles.isPaused)
            {
                particles.Play();
            }
        }
    }

    int NegativeOrPositive() {
        int value = Random.Range(1, 3);
        if (value == 2)
        {
            value = -1;
        }
        return value;
    }

    public void takedamage() {
        if (!finalStage)
        {
            if (orcAnim.enabled)
            {
                orcAnim.SetTrigger("Hit");
            }
            if (spiderAnim.enabled)
            {
                spiderAnim.SetTrigger("Hit");
            }
        }
        else {
            if (batAnim.enabled && bat.activeSelf)
            {
                batAnim.SetTrigger("Hit");
            }
        }
    }
    public void Death() {
        if (batAnim.enabled)
        {
            batAnim.SetTrigger("LastHit");
        }
        gunsPivot.gameObject.SetActive(false);
        for (int i = 0; i < boxes.Count; i++)
        {
            boxes[i].SetActive(false);
        }
        if (!hasDeathScreamed)
        {
            enemyScript.sfxAudioSource.PlayOneShot(screamNoise);
            hasDeathScreamed = true;
        }
        StartCoroutine(DeathParticles());
    }
    IEnumerator DeathParticles() {
        float timer = 3;
        if (!dieEffect)
        {
            Instantiate(dyingEffect, transform.position + Vector3.up, transform.rotation);
            enemyScript.sfxAudioSource.PlayOneShot(fireworks);
            dieEffect = true;
        }
        if (sculpture)
        {
            StartCoroutine(sculpture.GetComponent<SwitchEvent>().TriggeredEvent());
        }
        while (timer > 0) {
            if (!paused) {
                timer -= Time.deltaTime;
            }
            yield return null;
        }
        if (!hasPoofed)
        {
            //enemyScript.sfxAudioSource.PlayOneShot(poof);
            Instantiate(finalExplosion, transform.position, transform.rotation);
            Gamemanager.Instance.playerScript.sceneCamera.Shake(1000, 1.5f);
            enemyScript.sfxAudioSource.PlayOneShot(explode);
            bat.SetActive(false);
            hasPoofed = true;
        }
        while (enemyScript.sfxAudioSource.isPlaying) {
            yield return null;
        }
        Destroy(gameObject);
    }
    private void OnCollisionEnter(Collision collision)
    {
        if (stage == Stage.Slamming && inAir)
        {
            particles.Play();
            rb.velocity = Vector3.zero; 
            inAir = false;
            runningSlamTime = slamTime;
            StartCoroutine(Slamming());
        }
    }
    private void OnCollisionStay(Collision collision)
    {
        if (stage == Stage.Slamming && inAir && rb.velocity == Vector3.zero)
        {
            particles.Play();
            inAir = false;
            runningSlamTime = slamTime;
            StartCoroutine(Slamming());
        }
    }
    private void OnDestroy()
    {
        if (key)
        {
            key.SetActive(true);
            key.GetComponent<Rigidbody>().useGravity = true;
        }
    }
}
