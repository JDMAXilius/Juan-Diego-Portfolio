using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class SpikeController : MonoBehaviour
{
    [SerializeField] float stayHitTimer = 2;
    float runningTimer;
    public int damage = 25;
    public void OnTriggerEnter(Collider col)
    {
        //if (col.gameObject.name == "Player")
        if (col.gameObject == Gamemanager.Instance.player)
        {
            Gamemanager.Instance.playerScript.Hurt(damage,0);
            runningTimer = stayHitTimer;
        }
    }
    public void OnTriggerStay(Collider col)
    {
        //if (col.gameObject.name == "Player")
        if (!Gamemanager.Instance.playerScript.paused)
        {
            if (col.gameObject == Gamemanager.Instance.player)
            {
                runningTimer -= Time.deltaTime;
                if (runningTimer <= 0)
                {
                    runningTimer = stayHitTimer;
                    Gamemanager.Instance.playerScript.Hurt(damage, 0);
                }
            }
        }
    }
    // Start is called before the first frame update
    void Start()
    {
      
    }
    void Hurt()
    {
        Gamemanager.Instance.playerScript.Hurt(damage, 0);
    }
    
}
