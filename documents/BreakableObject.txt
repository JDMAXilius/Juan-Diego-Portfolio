using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class BreakableObject : MonoBehaviour
{
    EnemyAI enemyScript;
    [SerializeField] GameObject broken;
    [SerializeField] MeshRenderer mesh;
    BoxCollider box;
    Rigidbody rb;
    //int maxHP;
    bool paused;
    // Start is called before the first frame update
    void Start()
    {
        enemyScript = GetComponent<EnemyAI>();
        //maxHP = enemyScript.HP;
        box = GetComponent<BoxCollider>();
        rb = GetComponent<Rigidbody>();
    }

    // Update is called once per frame
    //void Update()
    //{
    //    if (enemyScript.HP == maxHP)
    //    {
    //        //debug.log("Stage 0");
    //    }
    //    else if (enemyScript.HP > maxHP * 2 / 3)
    //    {
    //        //debug.log("Stage 1");
    //    }
    //    else if (enemyScript.HP > maxHP / 3)
    //    {
    //        //debug.log("Stage 2");
    //    }
    //    else {
    //        //debug.log("Stage 3");
    //    }
    //}
    private void Update()
    {
        paused = enemyScript.paused;
        if (paused) {
            if (!Gamemanager.Instance.playerScript.paused) {
                enemyScript.Pause();
                //Debug.Break();
            }
        }
    }
    private void OnCollisionEnter(Collision collision)
    {
        if (collision.gameObject.CompareTag("Enemy") && gameObject.layer == 0) {
            enemyScript.takeDamage(100);
        }
    }
    public void ChangeShape() {
        mesh.enabled = false;
        box.size = new Vector3(box.size.x, .1f, box.size.z);
        broken.SetActive(true);
        gameObject.layer = 5;
        Destroy(gameObject,2);
    }
}
