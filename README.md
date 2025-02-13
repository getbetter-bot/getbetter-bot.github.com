using UnityEngine;
using Photon.Pun;
using UnityEngine.AI;
using UnityEngine.UI;

public class SettingsManager : MonoBehaviour
{
    public Dropdown graphicsDropdown;
    public Dropdown gameplayDropdown;

    void Start()
    {
        graphicsDropdown.onValueChanged.AddListener(SetGraphicsSettings);
        gameplayDropdown.onValueChanged.AddListener(SetGameplaySettings);
    }

    void SetGraphicsSettings(int index)
    {
        switch (index)
        {
            case 0: // Low
                QualitySettings.SetQualityLevel(0);
                break;
            case 1: // Medium
                QualitySettings.SetQualityLevel(2);
                break;
            case 2: // High
                QualitySettings.SetQualityLevel(5);
                break;
        }
    }

    void SetGameplaySettings(int index)
    {
        switch (index)
        {
            case 0: // Casual
                PlayerController.moveSpeed = 4f;
                AIEnemy.detectionRange = 8f;
                break;
            case 1: // Competitive
                PlayerController.moveSpeed = 6f;
                AIEnemy.detectionRange = 12f;
                break;
        }
    }
}

public class PlayerController : MonoBehaviourPunCallbacks
{
    public static float moveSpeed = 5f;
    public float lookSpeed = 2f;
    public Camera playerCamera;
    public GameObject bulletPrefab;
    public Transform firePoint;
    public float bulletSpeed = 20f;
    public int playerLevel = 1;
    public int killstreak = 0;
    public GameObject[] weaponPrefabs;
    public Transform weaponHolder;
    public bool canBuild = true;
    public GameObject buildPrefab;
    public LayerMask buildableLayer;

    private Rigidbody rb;
    private float xRotation = 0f;
    private GameObject currentWeapon;

    void Start()
    {
        rb = GetComponent<Rigidbody>();
        if (!photonView.IsMine)
        {
            playerCamera.enabled = false;
        }
        EquipWeapon(0);
    }

    void Update()
    {
        if (!photonView.IsMine) return;
        
        MovePlayer();
        LookAround();
        
        if (Input.GetMouseButtonDown(0))
        {
            Shoot();
        }

        if (Input.GetKeyDown(KeyCode.B))
        {
            TryBuild();
        }
    }

    void MovePlayer()
    {
        float moveX = Input.GetAxis("Horizontal") * moveSpeed;
        float moveZ = Input.GetAxis("Vertical") * moveSpeed;
        
        Vector3 move = transform.right * moveX + transform.forward * moveZ;
        rb.AddForce(new Vector3(move.x, 0, move.z), ForceMode.VelocityChange);
    }

    void LookAround()
    {
        float mouseX = Input.GetAxis("Mouse X") * lookSpeed;
        float mouseY = Input.GetAxis("Mouse Y") * lookSpeed;
        
        xRotation -= mouseY;
        xRotation = Mathf.Clamp(xRotation, -90f, 90f);
        
        transform.Rotate(Vector3.up * mouseX);
        playerCamera.transform.localRotation = Quaternion.Euler(xRotation, 0f, 0f);
    }

    void Shoot()
    {
        GameObject bullet = PhotonNetwork.Instantiate(bulletPrefab.name, firePoint.position, firePoint.rotation);
        Rigidbody bulletRb = bullet.GetComponent<Rigidbody>();
        bulletRb.velocity = firePoint.forward * bulletSpeed;
    }

    void TryBuild()
    {
        if (canBuild)
        {
            RaycastHit hit;
            if (Physics.Raycast(transform.position, Vector3.down, out hit, 5f, buildableLayer))
            {
                Instantiate(buildPrefab, hit.point, Quaternion.identity);
            }
        }
    }

    void EquipWeapon(int index)
    {
        if (currentWeapon != null)
        {
            Destroy(currentWeapon);
        }
        currentWeapon = Instantiate(weaponPrefabs[index], weaponHolder.position, weaponHolder.rotation, weaponHolder);
    }
}

public class AIEnemy : MonoBehaviour
{
    public enum Difficulty { Easy, Hard }
    public Difficulty aiDifficulty = Difficulty.Easy;
    public static float detectionRange = 10f;
    public float attackRange;
    public float bulletSpeed;
    public float fireRate;
    public GameObject bulletPrefab;
    public Transform firePoint;
    private Transform target;
    private float nextFireTime = 0f;
    private NavMeshAgent agent;

    void Start()
    {
        GameObject[] players = GameObject.FindGameObjectsWithTag("Player");
        target = players.Length > 0 ? players[Random.Range(0, players.Length)].transform : null;
        agent = GetComponent<NavMeshAgent>();
        SetDifficulty();
    }

    void Update()
    {
        if (target == null) return;

        float distance = Vector3.Distance(transform.position, target.position);
        if (distance <= attackRange && Time.time >= nextFireTime)
        {
            Shoot();
            nextFireTime = Time.time + 1f / fireRate;
        }
        else if (distance <= detectionRange)
        {
            agent.SetDestination(target.position);
        }
    }

    void SetDifficulty()
    {
        if (aiDifficulty == Difficulty.Easy)
        {
            attackRange = 7f;
            bulletSpeed = 10f;
            fireRate = 0.5f;
        }
        else if (aiDifficulty == Difficulty.Hard)
        {
            attackRange = 14f;
            bulletSpeed = 18f;
            fireRate = 1.5f;
        }
    }
}

public class LobbyManager : MonoBehaviour
{
    public Button teamRedButton;
    public Button teamBlueButton;
    private string selectedTeam;

    void Start()
    {
        teamRedButton.onClick.AddListener(() => SelectTeam("Red"));
        teamBlueButton.onClick.AddListener(() => SelectTeam("Blue"));
    }

    void SelectTeam(string team)
    {
        selectedTeam = team;
        Debug.Log("Selected Team: " + selectedTeam);
        PhotonNetwork.NickName = team + "_" + PhotonNetwork.NickName;
    }
}
