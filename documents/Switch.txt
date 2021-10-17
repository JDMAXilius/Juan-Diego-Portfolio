using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class Switch : MonoBehaviour
{
    public bool flipped = false;
    [HideInInspector] public bool prevFlip = false;
    [HideInInspector] public bool flipWasNotEqual;
    public bool canFlip = true;
    public bool triggerOnContact = false;
    Animator anim;
    bool playerContact;
    AudioSource leverSound;
    public GameObject particuleExplode;

    bool paused;
    // Start is called before the first frame update
    void Start()
    {
        anim = GetComponent<Animator>();
        leverSound = GetComponent<AudioSource>();
        HUDScript.pauseGame += Pause;
    }

    // Update is called once per frame
    void Update()
    {
        if (!paused)
        {
            if (playerContact)
            {
                if (canFlip)
                {
                    if (!triggerOnContact)
                    {
                        if (Input.GetKeyDown(KeyCode.E) || (Input.GetButtonDown("Fire1")))
                        {
                            flipped = !flipped;
                            leverSound.Play();
                            GameObject newparticule = (GameObject)Instantiate(particuleExplode, transform.position + new Vector3(Random.Range(-2, 2), Random.Range(0, 3), -10), Quaternion.identity);
    
                            Destroy(newparticule, 1);
                        }
                    }
                }
            }
            if (prevFlip != flipped)
            {
                flipWasNotEqual = true;
                if (!triggerOnContact)
                {
                    if (flipped)
                    {
                        anim.SetTrigger("FlipOn");
                    }
                    else
                    {
                        anim.SetTrigger("FlipOff");
                    }
                }
                canFlip = false;
            }
            else
            {
                flipWasNotEqual = false;
            }
            prevFlip = flipped;
        }
    }
    private void OnTriggerEnter(Collider other)
    {
        if (other.gameObject == Gamemanager.Instance.player)
        {
            playerContact = true;
            if (triggerOnContact) {
                flipped = !flipped;
            }
        }
    }
    private void OnTriggerStay(Collider other)
    {
        if (other.gameObject == Gamemanager.Instance.player) {
            playerContact = true;
        }
    }
    private void OnTriggerExit(Collider other)
    {
        if (other.gameObject == Gamemanager.Instance.player)
        {
            playerContact = false;
        }
    }
    private void OnDestroy()
    {
        HUDScript.pauseGame -= Pause;
    }
    void Pause() {
        paused = !paused;
        if (!triggerOnContact)
        {
            if (paused)
            {
                anim.enabled = false;
            }
            else
            {
                anim.enabled = true;
            }
        }
    }
}
