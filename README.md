
using System.Collections.Generic;
using UnityEngine;

public class KizilTutulma2D : MonoBehaviour
{
    // Bu ustteki public Sprite alanlari Unity Inspector'da kutu olarak gorunur.
    // Sen gorselleri bu kutulara surukleyince oyun renkli kare yerine o gorseli kullanir.
    // Bir kutu bos kalirsa sorun olmaz; kod yine renkli kare cizer.
    [Header("Harita Gorselleri")]
    public Sprite wallSprite;
    public Sprite floorSprite;
    public Sprite floorSprite2;
    public Sprite floorSprite3;
    public Sprite lavaSprite;
    public Sprite magmaSprite;

    [Header("Karakter ve Kapi")]
    public Sprite playerSprite;
    public Sprite doorSprite;

    [Header("Dusman Gorselleri")]
    public Sprite enemySprite;
    public Sprite enemyFastSprite;
    public Sprite enemyHeavySprite;
    public Sprite enemyShieldSprite;
    public Sprite enemyArcherSprite;
    public Sprite bossSprite;

    [Header("Bonus Gorselleri")]
    public Sprite heartSprite;
    public Sprite swordSprite;
    public Sprite maxHeartSprite;

    // Oyunun su an hangi ekranda oldugunu tutar.
    // Menu = ana menu, Game = oyun, Skills = yetenek/yukseltme ekrani.
    // Dead = olum ekrani, Victory = bolum bitirme ekrani, Powerups = kalici guclendirme ekrani.
    enum Mode { Menu, Game, Skills, Dead, Victory, Powerups }

    // Yerden alinabilecek bonus turleri.
    // Heal can verir, Attack saldiri verir, MaxHealth maksimum cani artirir.
    enum PickupType { Heal, Attack, MaxHealth }

    // Dusmanin ustunde sakladigimiz bilgiler.
    // Unity'de her dusman bir GameObject, bu class da onun can/saldiri/hangi karede oldugu bilgisini tutar.
    class EnemyData : MonoBehaviour
    {
        public int health;
        public int attack;
        public int bonus;
        public Vector2Int cell;
        public bool isArcher;   // Okcu dusman mi? Oyleyse belirli araliklarla oyuncuya ok atar.
        public float shotTimer; // Bir sonraki oka kadar kalan sure.
    }

    // Bonus objesinin ustunde sakladigimiz bilgiler.
    // Bonus hangi tur ve haritada hangi karede duruyor, burada tutulur.
    class PickupData : MonoBehaviour
    {
        public PickupType type;
        public Vector2Int cell;
    }

    // Okcunun attigi okun ustunde sakladigimiz bilgiler.
    // Ok, kucuk yesil bir kare olarak okcudan oyuncuya dogru ucar.
    class ArrowData
    {
        public GameObject obj;
        public Vector3 startWorld;
        public Vector3 direction;      // Okun ucus yonu (normalize).
        public float speed;            // Saniyede kat ettigi dunya mesafesi.
        public float traveledDistance; // Su ana kadar kat ettigi mesafe.
        public float hitCheckDistance; // Oyuncunun atis anindaki karesine olan mesafe; burada hitbox kontrolu yapilir.
        public float maxDistance;      // Isabet olmazsa okun devam edip yok olacagi mesafe (harita kenarina kadar).
        public Vector2Int targetCell;  // Ok atildiginda oyuncunun bulundugu kare.
        public bool hitChecked;        // Hitbox kontrolu bir kere yapilsin diye.
        public int damage;
    }

    // Ekranda ucusan hasar yazilarinin bilgisi.
    // Nereden basladi, nereye kadar cikacak, kac saniye kaldi ve kritik mi burada tutulur.
    class DamageText
    {
        public string text;
        public Vector3 startWorld;
        public Vector3 endWorld;
        public float age;
        public bool critical;
    }

    // Harita 25 kare genislik, 13 kare yukseklik.
    // Eski Python kodundaki ekran_g = 25, ekran_y = 13 mantigi.
    const int GridWidth = 24;
    const int GridHeight = 13;
    const float DamageTextDuration = 0.8f; // Hasar yazisinin ekranda kalma suresi. Bunu 0.6f/1.0f gibi degistirebilirsin.
    const float EnemyCounterDelay = 0.5f; // Oyuncu vurduktan kac saniye sonra dusman cevap vurusu yapacak.
    const float ArcherShotInterval = 2.2f; // Okcu kac saniyede bir ok atar.
    const int ArcherRange = 6; // Okcu oyuncuyu bu kare mesafesine kadar gorup ok atabilir.
    const float ArcherArrowDuration = 0.45f; // Okun okcudan oyuncuya ulasma suresi.
    const float ReplayRewardMultiplier = 0.5f; // Daha once bitirilmis bir bolum tekrar oynanirsa odul carpani.
    const int BossFloorNumber = 7; // Boss'un cikacagi kat (SpawnFloorEnemies'teki "floor > 6" ile ayni mantik). Kat gostergesinde "su an / bu deger" seklinde kullanilir.

    // Oyunda sonradan olusan dusmanlar, bonuslar ve harita kareleri bu listelerde tutulur.
    // Boylece gerektiginde hepsini silebilir, gizleyebilir veya kontrol edebiliriz.
    readonly List<EnemyData> enemies = new List<EnemyData>();
    readonly List<PickupData> pickups = new List<PickupData>();
    readonly List<DamageText> damageTexts = new List<DamageText>();
    readonly List<ArrowData> arrows = new List<ArrowData>();
    readonly Dictionary<Vector2Int, GameObject> tiles = new Dictionary<Vector2Int, GameObject>();

    // Oyunun baslangic durumu.
    // Baslangicta menudeyiz, oyuncu sol ust tarafa yakin baslar, kapi sag alttadir.
    Mode mode = Mode.Menu;
    Vector2Int playerCell = new Vector2Int(1, 1);
    Vector2Int doorCell = new Vector2Int(22, 11);

    // Sahnedeki ana objeler.
    // player = oyuncu objesi, door = kapi objesi, mainCamera = kameramiz.
    GameObject player;
    GameObject door;
    Camera mainCamera;

    // Oyunun sayisal degerleri.
    // Bunlar Python kodundaki karakter.health, attack, para, mana, exp gibi degiskenlerin Unity hali.
    int[,] map;
    int currentChapter = 1;
    int completedChapters = 0;
    int playerLevel = 1;
    int expToNextLevel = 100;
    int health = 100;
    int maxHealth = 100;
    int attack = 5;
    int money = 10000;
    int mana = 250;
    int exp = 0;
    int expExtra = 0;
    int floor = 0;
    int manaUpgrade = 0;
    int critChancePercent = 0;
    int healthUpgradeLevel = 0;
    int attackUpgradeLevel = 0;
    int critUpgradeLevel = 0;
    int healthUpgradeCost = 5;
    int attackUpgradeCost = 5;
    int manaUpgradeCost = 5;
    int critUpgradeCost = 5;
    int enemiesKilled = 0;
    int powerupTokens = 0;
    bool chapter1Completed = false;
    bool lastVictoryGavePowerup = false;
    bool moneyBoostUnlocked = false;
    bool maxHealthBoostUnlocked = false;
    bool attackBoostUnlocked = false;

    // Ana menude hangi bolumun secili oldugunu ve bolum secim menusunun acik olup olmadigini tutar.
    int selectedChapter = 1;
    bool chapterMenuOpen = false;
    Vector2 chapterMenuScroll = Vector2.zero;

    // Su anki oynanis daha once bitirilmis bir bolumun tekrari mi? Oyleyse odul azalir.
    bool isReplayRun = false;

    // Envanter panelinin acik olup olmadigini tutar (mana, exp, kat, kritik oran gibi detaylar burada).
    bool inventoryOpen = false;

    int enemy1Min = 5;
    int enemy1Max = 10;
    int enemy2Min = 40;
    int enemy2Max = 60;
    int enemy3Min = 30;
    int enemy3Max = 70;
    int enemy4Min = 20;
    int enemy4Max = 30;
    int enemy5Min = 80;
    int enemy5Max = 100;
    int enemy6Min = 10;
    int enemy6Max = 18;

    // Mana zamanla dolsun diye timer kullaniyoruz.
    // GUIStyle ve Texture2D alanlari da ekrandaki yazilar/barlar icin.
    float manaTimer;
    GUIStyle titleStyle;
    GUIStyle textStyle;
    GUIStyle buttonStyle;
    GUIStyle deathTitleStyle;
    GUIStyle damageTextStyle;
    GUIStyle criticalDamageTextStyle;
    Texture2D manaBarTexture;
    Texture2D manaBarBackTexture;
    Texture2D healthBarTexture;
    Texture2D healthBarBackTexture;
    Texture2D expBarTexture;
    Texture2D expBarBackTexture;
    Texture2D bloodTexture;
    Texture2D deathBackTexture;
    Texture2D victoryBackTexture;
    Texture2D panelBackTexture;
    float enemyCounterTimer = 0f;
    int pendingEnemyDamage = 0;

    void Start()
    {
        // Start, oyun baslayinca bir kere calisir.
        // Burada kamera hazirlaniyor, harita olusturuluyor, oyuncu ve kapi sahneye konuyor.
        mainCamera = Camera.main;
        if (mainCamera == null)
        {
            var cameraObject = new GameObject("Main Camera");
            mainCamera = cameraObject.AddComponent<Camera>();
            cameraObject.tag = "MainCamera";
        }

        mainCamera.orthographic = true;
        mainCamera.orthographicSize = 7.1f;
        mainCamera.transform.position = new Vector3(12, 6, -10);
        mainCamera.backgroundColor = new Color(0.10f, 0.12f, 0.16f);

        map = CreateMap();
        DrawMap();
        CreatePlayer();
        CreateDoor();
        SetGameObjectsVisible(false);
    }

    void Update()
    {
        // Update her karede calisir.
        // Ama sadece oyun ekranindaysak hareket, mana, bonus toplama gibi seyleri kontrol ediyoruz.
        if (mode != Mode.Game)
            return;

        if (pendingEnemyDamage > 0)
        {
            // Oyuncu dusmana vurduktan sonra dusman hemen degil, kisa gecikmeyle cevap verir.
            // Bu sure boyunca oyuncu hareket edemez; cevap vurusu bitince tekrar hareket serbest olur.
            enemyCounterTimer -= Time.deltaTime;
            if (enemyCounterTimer <= 0f)
            {
                health -= pendingEnemyDamage;
                SpawnDamageText(playerCell, pendingEnemyDamage, false);
                pendingEnemyDamage = 0;
            }
        }

        manaTimer += Time.deltaTime;
        if (manaTimer >= 1f)
        {
            // Her 1 saniyede mana biraz dolar. Mana 250'yi gecemez.
            mana = Mathf.Min(250, mana + 1 + manaUpgrade);
            manaTimer = 0f;
        }

        UpdateDamageTexts();
        UpdateArchers();
        UpdateArrows();

        // Klavye hareketleri.
        // W yukari, S asagi olacak sekilde ayarlandi.
        if (pendingEnemyDamage <= 0)
        {
            if (Input.GetKeyDown(KeyCode.D)) TryMove(Vector2Int.right);
            if (Input.GetKeyDown(KeyCode.A)) TryMove(Vector2Int.left);
            if (Input.GetKeyDown(KeyCode.W)) TryMove(Vector2Int.down);
            if (Input.GetKeyDown(KeyCode.S)) TryMove(Vector2Int.up);
        }

        if (Input.GetKeyDown(KeyCode.I))
            inventoryOpen = !inventoryOpen;

        CollectPickups();

        if (playerCell == doorCell)
            mana = 250;

        if (enemies.Count == 0 && pickups.Count == 0 && playerCell == doorCell)
            NextFloor();

        if (health <= 0)
            DieAndReturnToMenu();

        HandleLevelUp();
    }

    void HandleLevelUp()
    {
        // Exp bari dolunca oyuncu level atlar.
        // Level atlayinca can/mana dolar ve bir sonraki level icin gereken exp biraz artar.
        while (exp >= expToNextLevel)
        {
            exp -= expToNextLevel;
            playerLevel++;
            expToNextLevel += 25;
            health = maxHealth;
            mana = 250;
        }
    }

    void OnGUI()
    {
        // OnGUI ekrandaki menu, buton, yazilar ve barlari cizer.
        // Yani oyun mantigi degil, ekranda gorunen arayuz burada.
        BuildGuiStyles();

        if (mode == Mode.Menu)
            DrawMenu();
        else if (mode == Mode.Skills)
            DrawSkills();
        else if (mode == Mode.Dead)
            DrawDeathScreen();
        else if (mode == Mode.Victory)
            DrawVictoryScreen();
        else if (mode == Mode.Powerups)
            DrawPowerups();
        else
        {
            DrawGameHud();
            DrawDamageTexts();
        }
    }

    int[,] CreateMap()
    {
        // Harita sayilardan olusur.
        // 0 = duvar, 21 = lav, 22 = magma, diger sayilar = farkli zemin tipleri.
        // Dis cerceve hep duvar, ic taraf rastgele zemin.
        var newMap = new int[GridWidth, GridHeight];
        for (int y = 0; y < GridHeight; y++)
        {
            for (int x = 0; x < GridWidth; x++)
            {
                bool border = x == 0 || y == 0 || x == GridWidth - 1 || y == GridHeight - 1;
                newMap[x, y] = border ? 0 : Random.Range(1, 21);
            }
        }

        AddHazardTiles(newMap);
        return newMap;
    }

    void AddHazardTiles(int[,] targetMap)
    {
        // Lav ve magma bloklarini haritaya ekler.
        // Her lavin yanina en az 1 magma koymaya calisiriz; magma lavin yari hasarini verir.
        int lavaCount = currentChapter == 1 ? 3 : 5;
        for (int i = 0; i < lavaCount; i++)
        {
            Vector2Int lavaCell = RandomSafeMapCell(targetMap);
            targetMap[lavaCell.x, lavaCell.y] = 21;
            AddMagmaNearLava(targetMap, lavaCell);
        }
    }

    void AddMagmaNearLava(int[,] targetMap, Vector2Int lavaCell)
    {
        // Lavin etrafindaki rastgele uygun komsu karelere magma koyar.
        // En az 1 magma deniyoruz, bazen 2-3 tane de olabilir.
        int magmaCount = Random.Range(1, 4);
        Vector2Int[] directions = { Vector2Int.up, Vector2Int.down, Vector2Int.left, Vector2Int.right };

        for (int i = 0; i < magmaCount; i++)
        {
            Vector2Int magmaCell = lavaCell + directions[Random.Range(0, directions.Length)];
            if (IsInsideWalkableArea(magmaCell) && targetMap[magmaCell.x, magmaCell.y] != 21)
                targetMap[magmaCell.x, magmaCell.y] = 22;
        }
    }

    Vector2Int RandomSafeMapCell(int[,] targetMap)
    {
        // Oyuncunun baslangici, kapi ve duvarlar haric rastgele bir kare secer.
        // Lav/magma gibi ozel bloklar da ust uste binmesin diye kontrol edilir.
        Vector2Int cell;
        do
        {
            cell = new Vector2Int(Random.Range(2, GridWidth - 2), Random.Range(2, GridHeight - 2));
        }
        while (cell == playerCell || cell == doorCell || targetMap[cell.x, cell.y] == 21 || targetMap[cell.x, cell.y] == 22);

        return cell;
    }

    void DrawMap()
    {
        // Haritadaki sayilara bakip sahneye gercek GameObject kareleri koyar.
        // Sprite verildiyse gorsel kullanir, verilmediyse renkli kare kullanir.
        foreach (var tile in tiles.Values)
            Destroy(tile);
        tiles.Clear();

        for (int y = 0; y < GridHeight; y++)
        {
            for (int x = 0; x < GridWidth; x++)
            {
                Color color;
                Sprite sprite;
                int value = map[x, y];
                if (value == 0)
                {
                    color = currentChapter == 1 ? new Color(0.20f, 0.21f, 0.25f) : new Color(0.16f, 0.18f, 0.23f);
                    sprite = wallSprite;
                }
                else if (value == 21)
                {
                    color = new Color(0.95f, 0.05f, 0.02f);
                    sprite = lavaSprite;
                }
                else if (value == 22)
                {
                    color = new Color(1.00f, 0.42f, 0.05f);
                    sprite = magmaSprite;
                }
                else if (value <= 15)
                {
                    color = currentChapter == 1 ? new Color(0.34f, 0.31f, 0.27f) : new Color(0.28f, 0.32f, 0.36f);
                    sprite = floorSprite;
                }
                else if (value == 16 || value == 17 || value == 19)
                {
                    color = currentChapter == 1 ? new Color(0.30f, 0.35f, 0.39f) : new Color(0.23f, 0.28f, 0.34f);
                    sprite = floorSprite2 != null ? floorSprite2 : floorSprite;
                }
                else
                {
                    color = currentChapter == 1 ? new Color(0.40f, 0.25f, 0.24f) : new Color(0.34f, 0.24f, 0.36f);
                    sprite = floorSprite3 != null ? floorSprite3 : floorSprite;
                }

                var tile = CreateSquare("Tile", new Vector2Int(x, y), color, 0.96f, sprite);
                tile.transform.SetParent(transform);
                tiles[new Vector2Int(x, y)] = tile;
            }
        }
    }

    void CreatePlayer()
    {
        player = CreateSquare("Oyuncu", playerCell, new Color(0.86f, 0.18f, 0.14f), 0.72f, playerSprite);
    }

    void CreateDoor()
    {
        door = CreateSquare("Kapi", doorCell, new Color(0.95f, 0.72f, 0.22f), 0.76f, doorSprite);
    }

    GameObject CreateSquare(string objectName, Vector2Int cell, Color color, float scale, Sprite sprite = null)
    {
        // Bu fonksiyon oyundaki her gorunen seyi olusturur:
        // zemin, duvar, oyuncu, kapi, dusman, bonus.
        // Eger sprite varsa onu kullanir, yoksa tek renkli kare yapar.
        var obj = new GameObject(objectName);
        obj.transform.position = CellToWorld(cell);
        obj.transform.localScale = Vector3.one * scale;

        var renderer = obj.AddComponent<SpriteRenderer>();
        renderer.sprite = sprite != null ? sprite : MakeSprite(color);
        renderer.sortingOrder = objectName == "Tile" ? 0 : 5;

        var collider = obj.AddComponent<BoxCollider2D>();
        collider.isTrigger = true;

        return obj;
    }

    Sprite MakeSprite(Color color)
    {
        var texture = new Texture2D(1, 1);
        texture.SetPixel(0, 0, color);
        texture.Apply();
        texture.filterMode = FilterMode.Point;
        return Sprite.Create(texture, new Rect(0, 0, 1, 1), new Vector2(0.5f, 0.5f), 1f);
    }

    Vector3 CellToWorld(Vector2Int cell)
    {
        // Harita kare koordinatini Unity sahne pozisyonuna cevirir.
        // Grid sisteminde "1 kare saga git" gibi dusunuyoruz.
        return new Vector3(cell.x, GridHeight - 1 - cell.y, 0);
    }

    void TryMove(Vector2Int direction)
    {
        // Oyuncu hareket etmeye calisinca burasi calisir.
        // Once mana yetiyor mu, hedef kare haritanin icinde mi diye bakar.
        int moveCost = 4 + floor;
        if (mana < moveCost)
            return;

        var target = playerCell + direction;
        if (target.x <= 0 || target.y <= 0 || target.x >= GridWidth - 1 || target.y >= GridHeight - 1)
            return;

        mana -= moveCost;

        var enemy = EnemyAt(target);
        if (enemy != null)
        {
            // Hedef karede dusman varsa oyuncu oraya yurumez.
            // Bunun yerine dusmana vurur, dusman da oyuncuya vurur.
            int playerDamage = RollPlayerDamage(out bool criticalHit);
            enemy.health -= playerDamage;
            SpawnDamageText(enemy.cell, playerDamage, criticalHit);

            if (enemy.health <= 0)
                KillEnemy(enemy);
            else
            {
                // Okcu dibimize girince normal dusman gibi davranir, ama menzil silahi
                // olmasindan dolayi yakinda orijinal hasarinin %80'i kadar vurur.
                int counterDamage = enemy.isArcher
                    ? Mathf.Max(1, Mathf.RoundToInt(enemy.attack * 0.8f))
                    : enemy.attack;
                StartEnemyCounter(counterDamage);
            }

            return;
        }

        playerCell = target;
        player.transform.position = CellToWorld(playerCell);
        ApplyHazardDamage(playerCell);
    }

    void ApplyHazardDamage(Vector2Int cell)
    {
        // Oyuncu lav veya magma uzerine basarsa hasar alir.
        // Lav maksimum canin %10'u, magma maksimum canin %5'i kadar vurur.
        int tileValue = map[cell.x, cell.y];
        int damage = 0;

        if (tileValue == 21)
            damage = Mathf.Max(1, Mathf.RoundToInt(maxHealth * 0.10f));
        else if (tileValue == 22)
            damage = Mathf.Max(1, Mathf.RoundToInt(maxHealth * 0.05f));

        if (damage <= 0)
            return;

        health -= damage;
        SpawnDamageText(playerCell, damage, false);
    }

    EnemyData EnemyAt(Vector2Int cell)
    {
        for (int i = 0; i < enemies.Count; i++)
        {
            if (enemies[i].cell == cell)
                return enemies[i];
        }

        return null;
    }

    int RollPlayerDamage(out bool criticalHit)
    {
        // Oyuncu saldirirken kritik sansi burada hesaplanir.
        // Kritik olursa hasar her zaman 2x olur.
        criticalHit = Random.Range(0, 100) < critChancePercent;
        return criticalHit ? attack * 2 : attack;
    }

    void SpawnDamageText(Vector2Int cell, int damage, bool critical)
    {
        // Hasar yiyen kisinin/canavarin ustunde beyaz hasar yazisi olusturur.
        // Yazinin baslangici biraz rastgele kayar, boylece her vurus ayni noktada cikmaz.
        Vector3 baseWorld = CellToWorld(cell);
        Vector3 randomOffset = new Vector3(Random.Range(-0.28f, 0.28f), Random.Range(0.10f, 0.36f), 0f);
        Vector3 start = baseWorld + randomOffset;

        damageTexts.Add(new DamageText
        {
            text = damage.ToString(),
            startWorld = start,
            endWorld = start + Vector3.up * 0.5f,
            age = 0f,
            critical = critical
        });
    }

    void UpdateDamageTexts()
    {
        // Hasar yazilarinin suresini ilerletir.
        // Sure dolunca yaziyi listeden sileriz, yoksa gereksiz yere birikir.
        for (int i = damageTexts.Count - 1; i >= 0; i--)
        {
            damageTexts[i].age += Time.deltaTime;
            if (damageTexts[i].age >= DamageTextDuration)
                damageTexts.RemoveAt(i);
        }
    }

    void StartEnemyCounter(int enemyDamage)
    {
        // Oyuncu dusmana vurdu ama dusman olmediyse cevap vurusu hazirlanir.
        // Cevap vurusu 0.5 saniye sonra gelir; oyuncu bu arada hareket edemez.
        pendingEnemyDamage = enemyDamage;
        enemyCounterTimer = EnemyCounterDelay;
    }

    void UpdateArchers()
    {
        // Okcular oyuncuyla ayni satir/sutunda ve menzil icindeyse belirli araliklarla ok atar.
        // Bu, oyuncunun hareketinden bagimsiz calisan, zamana dayali bir saldiridir.
        foreach (var enemy in enemies)
        {
            if (!enemy.isArcher)
                continue;

            enemy.shotTimer -= Time.deltaTime;
            if (enemy.shotTimer > 0f)
                continue;

            bool sameRowOrColumn = enemy.cell.x == playerCell.x || enemy.cell.y == playerCell.y;
            int distance = Mathf.Abs(enemy.cell.x - playerCell.x) + Mathf.Abs(enemy.cell.y - playerCell.y);

            if (sameRowOrColumn && distance > 1 && distance <= ArcherRange)
            {
                FireArrow(enemy);
                enemy.shotTimer = ArcherShotInterval;
            }
            else
            {
                // Menzil disindaysa kisa kisa tekrar dener, oyuncu araya girince hemen ok atsin.
                enemy.shotTimer = 0.2f;
            }
        }
    }

    void FireArrow(EnemyData archer)
    {
        // Okcudan cikan, oyuncuya dogru ucan kucuk yesil kareyi olusturur.
        // Ana dusman karesinden daha kucuk ve daha acik yesil oluyor ki "ok" gibi ayirt edilsin.
        // Ok, atildigi andaki oyuncu karesine (targetCell) dogru gider ve hitbox kontrolu orada yapilir.
        // Oyuncu kacip ok ISKALARSA, ok yok olmaz; ayni yonde harita kenarina (duvara) kadar ucmaya devam eder.
        var arrowColor = new Color(0.55f, 0.95f, 0.45f);
        var obj = CreateSquare("Ok", archer.cell, arrowColor, 0.30f);

        Vector2Int farCell = archer.cell;
        if (archer.cell.y == playerCell.y)
        {
            // Ayni satirda: x ekseninde ilerleyip haritanin sol ya da sag duvarina kadar gider.
            int direction = playerCell.x >= archer.cell.x ? 1 : -1;
            farCell = new Vector2Int(direction > 0 ? GridWidth - 1 : 0, archer.cell.y);
        }
        else
        {
            // Ayni sutunda: y ekseninde ilerleyip haritanin ust ya da alt duvarina kadar gider.
            int direction = playerCell.y >= archer.cell.y ? 1 : -1;
            farCell = new Vector2Int(archer.cell.x, direction > 0 ? GridHeight - 1 : 0);
        }

        Vector3 startWorld = CellToWorld(archer.cell);
        Vector3 targetWorld = CellToWorld(playerCell);
        Vector3 farWorld = CellToWorld(farCell);

        float hitCheckDistance = Vector3.Distance(startWorld, targetWorld);
        float maxDistance = Mathf.Max(hitCheckDistance, Vector3.Distance(startWorld, farWorld));
        float speed = hitCheckDistance > 0.01f ? hitCheckDistance / ArcherArrowDuration : maxDistance / ArcherArrowDuration;

        arrows.Add(new ArrowData
        {
            obj = obj,
            startWorld = startWorld,
            direction = (farWorld - startWorld).normalized,
            speed = speed,
            traveledDistance = 0f,
            hitCheckDistance = hitCheckDistance,
            maxDistance = maxDistance,
            targetCell = playerCell,
            hitChecked = false,
            damage = archer.attack
        });
    }

    void UpdateArrows()
    {
        // Havadaki oklari yonlerinde ilerletir.
        // Ok, oyuncunun atis anindaki karesine (hitCheckDistance) ulastiginda BIR KERE hitbox kontrolu yapar:
        // oyuncu hala o karedeyse hasar alir ve ok yok olur; degilse (iskalarsa) ok durmaz,
        // ayni yonde harita kenarina (maxDistance) kadar ucmaya devam eder, sonra yok olur.
        for (int i = arrows.Count - 1; i >= 0; i--)
        {
            var arrow = arrows[i];
            arrow.traveledDistance += arrow.speed * Time.deltaTime;

            if (arrow.obj != null)
                arrow.obj.transform.position = arrow.startWorld + arrow.direction * arrow.traveledDistance;

            if (!arrow.hitChecked && arrow.traveledDistance >= arrow.hitCheckDistance)
            {
                arrow.hitChecked = true;

                if (playerCell == arrow.targetCell)
                {
                    health -= arrow.damage;
                    SpawnDamageText(playerCell, arrow.damage, false);

                    if (arrow.obj != null)
                        Destroy(arrow.obj);

                    arrows.RemoveAt(i);
                    continue;
                }
            }

            if (arrow.traveledDistance >= arrow.maxDistance)
            {
                if (arrow.obj != null)
                    Destroy(arrow.obj);

                arrows.RemoveAt(i);
            }
        }
    }

    void KillEnemy(EnemyData enemy)
    {
        // Dusman olunce para ve exp verir.
        // Daha once bitirilmis bir bolum tekrar oynaniyorsa (isReplayRun) odul azalir.
        // Sansina gore oldugu yere kalp/kilic/maksimum can bonusu birakir.
        int moneyGain = ApplyMoneyBoost(Random.Range(8, 17));
        int expGain = Random.Range(10, 21);

        if (isReplayRun)
        {
            moneyGain = Mathf.Max(1, Mathf.RoundToInt(moneyGain * ReplayRewardMultiplier));
            expGain = Mathf.Max(1, Mathf.RoundToInt(expGain * ReplayRewardMultiplier));
        }

        money += moneyGain;
        exp += expGain;
        enemiesKilled++;

        if (enemy.bonus > 0)
            SpawnPickup(enemy.cell, (PickupType)(enemy.bonus - 1));

        enemies.Remove(enemy);
        Destroy(enemy.gameObject);
    }

    void SpawnPickup(Vector2Int cell, PickupType type)
    {
        // Bonus olusturur.
        // Heal = kalp, Attack = kilic, MaxHealth = pembe kalp gibi dusun.
        Color color = type == PickupType.Heal
            ? new Color(0.95f, 0.18f, 0.35f)
            : type == PickupType.Attack
                ? new Color(0.75f, 0.82f, 0.94f)
                : new Color(1.00f, 0.45f, 0.72f);

        Sprite sprite = type == PickupType.Heal
            ? heartSprite
            : type == PickupType.Attack
                ? swordSprite
                : maxHeartSprite;

        var obj = CreateSquare(type.ToString(), cell, color, 0.46f, sprite);
        var pickup = obj.AddComponent<PickupData>();
        pickup.type = type;
        pickup.cell = cell;
        pickups.Add(pickup);
    }

    void CollectPickups()
    {
        // Oyuncu bonusun ustune gelirse bonusu alir.
        // Alinan bonus sahneden silinir.
        for (int i = pickups.Count - 1; i >= 0; i--)
        {
            if (pickups[i].cell != playerCell)
                continue;

            if (pickups[i].type == PickupType.Heal)
                health = Mathf.Min(maxHealth, health + GetHeartHealAmount());
            else if (pickups[i].type == PickupType.Attack)
                attack += GetAttackPickupAmount();
            else
                maxHealth += GetMaxHealthPickupAmount();

            Destroy(pickups[i].gameObject);
            pickups.RemoveAt(i);
        }
    }

    int GetHeartHealAmount()
    {
        // Kalp artik devasa can basmiyor.
        // Kat arttikca biraz daha iyi olur ama kontrolsuz buyumez.
        return 12 + floor * 3;
    }

    int GetMaxHealthPickupAmount()
    {
        // Pembe kalp maksimum can verir.
        // Eski 4^floor formulu cok hizli patladigi icin daha dengeli yaptik.
        int amount = 6 + floor * 2;
        return maxHealthBoostUnlocked ? Mathf.RoundToInt(amount * 1.5f) : amount;
    }

    int GetAttackPickupAmount()
    {
        // Kilic bonusu saldiriyi artirir.
        // Artik 2^floor degil; katla beraber yavas buyur.
        int amount = 1 + Mathf.CeilToInt(floor / 2f);
        return attackBoostUnlocked ? amount * 2 : amount;
    }

    int GetHealthUpgradeAmount()
    {
        // Yetenekten gelen can artisi.
        // Cok hizli buyumesin diye sabit agirlikli tuttuk.
        int amount = 8 + Mathf.CeilToInt(healthUpgradeLevel / 3f);
        return maxHealthBoostUnlocked ? Mathf.RoundToInt(amount * 1.5f) : amount;
    }

    int GetAttackUpgradeAmount()
    {
        // Yetenekten gelen saldiri artisi.
        // Her seferinde level kadar artmak yerine daha yavas buyur.
        int amount = 2 + Mathf.FloorToInt(attackUpgradeLevel / 4f);
        return attackBoostUnlocked ? amount * 2 : amount;
    }

    int ApplyMoneyBoost(int baseMoney)
    {
        // Kalici para guclendirmesi acildiysa dusmandan gelen parayi 1.2x yapar.
        // RoundToInt kullaniyoruz ki para yine tam sayi kalsin.
        return moneyBoostUnlocked ? Mathf.RoundToInt(baseMoney * 1.2f) : baseMoney;
    }

    void NextFloor()
    {
        // Kapiya gidip tum dusman/bonuslar bittiyse yeni kata gecilir.
        // Oyuncu basa alinir, floor artar, yeni dusmanlar dogar.
        playerCell = new Vector2Int(1, 1);
        player.transform.position = CellToWorld(playerCell);
        floor++;

        if (floor == 8)
        {
            CompleteChapter();
            return;
        }

        SpawnFloorEnemies();
    }

    void CompleteChapter()
    {
        // Bolum bitince burasi calisir.
        // Oyuncuya tebrik ekrani gosterir, 1 guclendirme hakki verir ve oyun alanini gizler.
        lastVictoryGavePowerup = currentChapter > completedChapters;
        completedChapters = Mathf.Max(completedChapters, currentChapter);
        chapter1Completed = completedChapters >= 1;
        if (lastVictoryGavePowerup)
            powerupTokens++;
        floor = 0;
        mana = 250;
        ResetEnemyPower();
        ClearEnemiesAndPickups();
        SetGameObjectsVisible(false);
        mode = Mode.Victory;
    }

    void ReturnToMenuAfterVictory()
    {
        // Tebrik ekranindaki butona basilinca ana menuye doner.
        // Oyuncu isterse artik Guclendirmeler ekranina girip odulunu harcayabilir.
        health = maxHealth;
        playerCell = new Vector2Int(1, 1);
        player.transform.position = CellToWorld(playerCell);
        mode = Mode.Menu;
        SetGameObjectsVisible(false);
    }

    void SpawnFloorEnemies()
    {
        // Kat numarasina gore dusmanlari olusturur.
        // Kat arttikca dusman sayisi ve gucu artar.
        if (floor >= 1 && floor <= 6)
        {
            enemy1Min = 4 + floor * 3;
            enemy1Max = 8 + floor * 6;
            for (int i = 0; i < 5 + floor; i++)
                SpawnEnemy("Dusman", RandomCell(), Random.Range(enemy1Min, enemy1Max + 1), Random.Range(enemy1Min, enemy1Max + 1), new Color(0.55f, 0.08f, 0.08f), enemySprite);
        }

        if (floor >= 2 && floor <= 6)
        {
            enemy2Min = 12 + floor * 5;
            enemy2Max = 20 + floor * 8;
            for (int i = 0; i < 2; i++)
                SpawnEnemy("Hizli Dusman", RandomCell(), Random.Range(10 + floor * 3, 18 + floor * 5), Random.Range(enemy2Min, enemy2Max + 1), new Color(0.75f, 0.20f, 0.09f), enemyFastSprite);
        }

        if (floor >= 2 && floor <= 6)
        {
            // Okcu: yesil kare govdeli, uzaktan (ok atarak) saldiran dusman.
            enemy6Min = 10 + floor * 4;
            enemy6Max = 18 + floor * 6;
            for (int i = 0; i < 2; i++)
                SpawnEnemy("Okcu", RandomCell(), Random.Range(enemy6Min, enemy6Max + 1), Random.Range(6 + floor * 2, 12 + floor * 3), new Color(0.16f, 0.62f, 0.22f), enemyArcherSprite, -1, true);
        }

        if (floor >= 3 && floor <= 6)
        {
            enemy3Min = 18 + floor * 7;
            enemy3Max = 35 + floor * 10;
            SpawnEnemy("Olucu", RandomCell(), Random.Range(45 + floor * 15, 80 + floor * 25), Random.Range(enemy3Min, enemy3Max + 1), new Color(0.38f, 0.16f, 0.45f), enemyHeavySprite);
        }

        if (floor >= 5 && floor <= 6)
        {
            enemy4Min = 25 + floor * 6;
            enemy4Max = 45 + floor * 8;
            SpawnEnemy("Kalkanli", RandomCell(), Random.Range(110 + floor * 25, 170 + floor * 35), Random.Range(enemy4Min, enemy4Max + 1), new Color(0.13f, 0.36f, 0.50f), enemyShieldSprite);
        }

        if (floor > 6)
        {
            enemy5Min = 55;
            enemy5Max = 85;
            SpawnEnemy("Ana Boss", RandomCell(), Random.Range(260, 361), Random.Range(enemy5Min, enemy5Max + 1), new Color(0.08f, 0.02f, 0.02f), bossSprite, 2);
        }
    }

    void SpawnEnemy(string enemyName, Vector2Int cell, int enemyHealth, int enemyAttack, Color color, Sprite sprite = null, int forcedBonus = -1, bool isArcher = false)
    {
        // Tek bir dusman olusturur.
        // Dusman oyuncunun veya kapinin ustune denk gelirse baska kare secer.
        while (cell == playerCell || cell == doorCell || EnemyAt(cell) != null || IsHazardCell(cell))
            cell = RandomCell();

        var obj = CreateSquare(enemyName, cell, color, 0.68f, sprite != null ? sprite : enemySprite);
        var enemy = obj.AddComponent<EnemyData>();
        enemy.health = enemyHealth;
        enemy.attack = enemyAttack;
        enemy.cell = cell;
        enemy.bonus = forcedBonus >= 0 ? forcedBonus : RollBonus();
        enemy.isArcher = isArcher;
        enemy.shotTimer = isArcher ? Random.Range(0.4f, ArcherShotInterval) : 0f;
        enemies.Add(enemy);
    }

    Vector2Int RandomCell()
    {
        Vector2Int cell;
        do
        {
            cell = new Vector2Int(Random.Range(1, GridWidth - 1), Random.Range(1, GridHeight - 1));
        }
        while (IsHazardCell(cell));

        return cell;
    }

    bool IsHazardCell(Vector2Int cell)
    {
        // Lav ve magma tehlikeli bloktur.
        // Dusmanlari bu bloklara koymuyoruz.
        return map[cell.x, cell.y] == 21 || map[cell.x, cell.y] == 22;
    }

    bool IsInsideWalkableArea(Vector2Int cell)
    {
        // Haritanin dis duvarlari haric iceride kalan alan mi diye bakar.
        return cell.x > 0 && cell.y > 0 && cell.x < GridWidth - 1 && cell.y < GridHeight - 1;
    }

    int RollBonus()
    {
        // Dusman olunce bonus dusup dusmeyecegini belirler.
        // 1 = can, 2 = saldiri, 3 = maksimum can, 0 = bonus yok.
        int chance = Random.Range(0, 51);
        if (chance > 0 && chance <= 15) return 1;
        if (chance > 15 && chance <= 25) return 2;
        if (chance > 25 && chance <= 40) return 3;
        return 0;
    }

    void StartGame()
    {
        StartChapter(1);
    }

    void StartChapter(int chapter)
    {
        // Menudeki "BASLA" butonuna basilinca ya da bolum menusunden bir bolum secilince burasi calisir.
        // Secilen bolume gore harita yenilenir ve oyun baslatilir.
        // Secilen bolum daha once bitirilmisse (tekrar oynaniyorsa) odul dusuk verilir.
        currentChapter = chapter;
        isReplayRun = chapter <= completedChapters;
        map = CreateMap();
        DrawMap();
        ResetEnemyPower();
        ClearEnemiesAndPickups();
        floor = 0;
        mode = Mode.Game;
        playerCell = new Vector2Int(1, 1);
        player.transform.position = CellToWorld(playerCell);
        mana = 250;
        inventoryOpen = false;
        SetGameObjectsVisible(true);
        NextFloor();
    }

    void DieAndReturnToMenu()
    {
        // Oyuncunun cani 0 olursa burasi calisir.
        // Artik hemen menuye atmiyoruz; once ozel olum ekranini gosteriyoruz.
        mode = Mode.Dead;
        SetGameObjectsVisible(false);
    }

    void RestartAfterDeath()
    {
        // Olum ekranindaki Tekrar Basla butonuna basilinca burasi calisir.
        // Direkt oyuna sokmuyoruz; ana menuye donduruyoruz.
        // Boylece oyuncu kazandigi parayla once yetenek yukseltmesi yapabilir.
        health = maxHealth;
        mana = 250;
        exp = 0;
        playerLevel = 1;
        expToNextLevel = 100;
        floor = 0;
        ResetEnemyPower();
        ClearEnemiesAndPickups();
        playerCell = new Vector2Int(1, 1);
        player.transform.position = CellToWorld(playerCell);
        mode = Mode.Menu;
        SetGameObjectsVisible(false);
    }

    void ResetEnemyPower()
    {
        // Floor sifirlaninca dusman gucleri de sifirlanmali.
        // Yoksa oyuncu olse bile enemy1Min/enemy1Max gibi degerler artmis halde kalir
        // ve yeni oyunda daha ilk katta bile eski guclu dusmanlar gelir.
        enemy1Min = 5;
        enemy1Max = 10;
        enemy2Min = 40;
        enemy2Max = 60;
        enemy3Min = 30;
        enemy3Max = 70;
        enemy4Min = 20;
        enemy4Max = 30;
        enemy5Min = 80;
        enemy5Max = 100;
        enemy6Min = 10;
        enemy6Max = 18;
    }

    void ClearEnemiesAndPickups()
    {
        // Dusman, bonus, ok ve ekrandaki hasar yazilarini temizler.
        // Yeni oyun/olum sonrasi eski efektler ekranda kalmasin diye hepsini sifirliyoruz.
        foreach (var enemy in enemies)
            Destroy(enemy.gameObject);
        enemies.Clear();

        foreach (var pickup in pickups)
            Destroy(pickup.gameObject);
        pickups.Clear();

        foreach (var arrow in arrows)
            if (arrow.obj != null) Destroy(arrow.obj);
        arrows.Clear();

        damageTexts.Clear();
        pendingEnemyDamage = 0;
        enemyCounterTimer = 0f;
    }

    void SetGameObjectsVisible(bool visible)
    {
        foreach (var tile in tiles.Values)
            tile.SetActive(visible);

        if (player != null) player.SetActive(visible);
        if (door != null) door.SetActive(visible);

        foreach (var enemy in enemies)
            enemy.gameObject.SetActive(visible);

        foreach (var pickup in pickups)
            pickup.gameObject.SetActive(visible);

        foreach (var arrow in arrows)
            if (arrow.obj != null) arrow.obj.SetActive(visible);
    }

    void DrawMenu()
    {
        // Ana menuyu cizer.
        // Artik sabit "Bolum 1 / Bolum 2" butonlari yok; tek bir BASLA butonu var.
        // Hangi bolumun oynanacagi sol ustteki "Bolumler" menusunden secilir. Boylece
        // ileride bolum sayisi 50'ye ciksa bile menu tasmaz, kaydirilabilir liste kullanilir.
        GUI.Label(new Rect(0, 70, Screen.width, 70), "KIZIL TUTULMA", titleStyle);

        if (GUI.Button(new Rect(24, 24, 150, 46), "Bolumler", buttonStyle))
            chapterMenuOpen = !chapterMenuOpen;

        GUI.Label(new Rect(Screen.width - 260, 20, 240, 36), "Tamamlanan Bolum: " + completedChapters, textStyle);

        if (chapterMenuOpen)
        {
            DrawChapterMenu();
        }
        else
        {
            GUI.Label(new Rect(0, 150, Screen.width, 40), "Secili Bolum: " + selectedChapter, textStyle);

            if (GUI.Button(CenterRect(260, 76, 30), "BASLA", buttonStyle))
                StartChapter(selectedChapter);

            if (GUI.Button(CenterRect(240, 70, 130), "Yetenekler", buttonStyle))
                mode = Mode.Skills;

            if (chapter1Completed && GUI.Button(CenterRect(260, 70, 220), "Guclendirmeler", buttonStyle))
                mode = Mode.Powerups;
        }

        GUI.Label(new Rect(0, Screen.height - 105, Screen.width, 56), "Para: " + money, textStyle);
    }

    void DrawChapterMenu()
    {
        // Su ana kadar acilmis olan bolum sayisi kadar buton listeler (tamamlanan + oynanabilecek 1 yeni bolum).
        // Liste sabit buyuklukte bir kaydirma alani icinde ciziliyor, boylece bolum sayisi artsa da tasma olmaz.
        int maxUnlocked = completedChapters + 1;

        GUI.Label(new Rect(0, 150, Screen.width, 40), "BOLUM SEC", textStyle);

        float panelWidth = 320f;
        float panelHeight = 300f;
        float startX = Screen.width / 2f - panelWidth / 2f;
        float startY = 210f;
        float buttonHeight = 54f;
        float spacing = 62f;

        Rect scrollRect = new Rect(startX, startY, panelWidth, panelHeight);
        Rect viewRect = new Rect(0, 0, panelWidth - 24, Mathf.Max(panelHeight, maxUnlocked * spacing));

        chapterMenuScroll = GUI.BeginScrollView(scrollRect, chapterMenuScroll, viewRect);
        for (int chapter = 1; chapter <= maxUnlocked; chapter++)
        {
            Rect buttonRect = new Rect(0, (chapter - 1) * spacing, viewRect.width, buttonHeight);
            string label = "Bolum " + chapter + (chapter <= completedChapters ? "  (Tekrar Oyna)" : "");
            if (GUI.Button(buttonRect, label, buttonStyle))
            {
                selectedChapter = chapter;
                chapterMenuOpen = false;
            }
        }
        GUI.EndScrollView();

        if (GUI.Button(new Rect(Screen.width / 2f - 90, startY + panelHeight + 20, 180, 50), "Kapat", buttonStyle))
            chapterMenuOpen = false;
    }

    void DrawSkills()
    {
        // Yetenek/yukseltme ekranini cizer.
        GUI.Label(new Rect(0, 45, Screen.width, 60), "YETENEKLER", titleStyle);

        if (GUI.Button(new Rect(24, 24, 130, 46), "Geri", buttonStyle))
            mode = Mode.Menu;

        DrawUpgradeButton(0, "Can", maxHealth.ToString(), healthUpgradeCost, BuyHealth);
        DrawUpgradeButton(1, "Saldiri", attack.ToString(), attackUpgradeCost, BuyAttack);
        DrawUpgradeButton(2, "Mana Yenilenmesi", manaUpgrade.ToString(), manaUpgradeCost, BuyMana);
        DrawUpgradeButton(3, "Kritik Oran", "%" + critChancePercent, critUpgradeCost, BuyCrit);

        GUI.Label(new Rect(0, Screen.height - 105, Screen.width, 56), "Para: " + money, textStyle);
    }

    void DrawUpgradeButton(int row, string label, string value, int cost, System.Action buyAction)
    {
        // Yetenek satirlarini biraz sik yerlestiriyoruz.
        // Kritik orani eklenince 4 satir oldu, bu yuzden para yazisina binmesin.
        float y = 130 + row * 82;
        GUI.Label(new Rect(Screen.width / 2f - 315, y + 6, 210, 60), label + ":", textStyle);
        GUI.Label(new Rect(Screen.width / 2f - 120, y + 6, 90, 60), value, textStyle);

        if (GUI.Button(new Rect(Screen.width / 2f + 30, y, 210, 58), "Al - " + cost, buttonStyle))
            buyAction();
    }

    void DrawGameHud()
    {
        // Oyun oynanirken ekrandaki bilgileri cizer.
        // Can/Saldiri alt solda sabit. Mana ve Exp barlari eski (orijinal) yerlerinde.
        // Kat bilgisi artik WASD yazisinin yaninda, boss katina gore "su an / max" seklinde.
        // Para sag ustte. Kritik Oran ise Envanter panelinde kaliyor.
        GUI.Label(new Rect(16, 10, 300, 46), "Hareket: W A S D", textStyle);
        GUI.Label(new Rect(300, 10, 200, 46), "Kat: " + floor + " / " + BossFloorNumber, textStyle);

        GUI.Label(new Rect(Screen.width - 220, 10, 200, 40), "Para: " + money, textStyle);

        float hudY = Screen.height - 64;
        GUI.Label(new Rect(16, hudY - 4, 390, 32), "Can: " + health + " / " + maxHealth + "   Saldiri: " + attack, textStyle);
        DrawBar(new Rect(70, hudY + 31, 180, 18), health, maxHealth, healthBarTexture, healthBarBackTexture);

        GUI.Label(new Rect(Screen.width / 2f - 82, hudY - 4, 130, 32), "Exp: " + exp, textStyle);
        DrawExpBar(new Rect(Screen.width / 2f - 70, hudY + 31, 180, 18));

        Rect manaBar = new Rect(Screen.width / 2f + 145, hudY + 17, 120, 18);
        DrawBar(manaBar, mana, 250, manaBarTexture, manaBarBackTexture);
        GUI.Label(new Rect(manaBar.x + manaBar.width + 10, hudY, 120, 52), "Mana: " + mana, textStyle);

        if (GUI.Button(new Rect(Screen.width - 170, hudY, 150, 46), "Envanter (I)", buttonStyle))
            inventoryOpen = !inventoryOpen;

        if (inventoryOpen)
            DrawInventoryPanel();
    }

    void DrawExpBar(Rect rect)
    {
        // Exp bari bir sonraki levele ne kadar yaklastigini gosterir.
        // Bar rengi sari, icindeki level yazisi koyu renk; boylece birbirinden ayrilir.
        DrawBar(rect, exp, expToNextLevel, expBarTexture, expBarBackTexture);

        Color oldColor = textStyle.normal.textColor;
        textStyle.normal.textColor = new Color(0.08f, 0.06f, 0.02f);
        GUI.Label(rect, "Lvl: " + playerLevel, textStyle);
        textStyle.normal.textColor = oldColor;
    }

    void DrawInventoryPanel()
    {
        // Mana ve Exp eski yerlerine (HUD) geri donduruldugu icin envanterde sadece Kritik Oran kaldi.
        float panelWidth = 220f;
        float panelHeight = 84f;
        float x = Screen.width - panelWidth - 16;
        float y = Screen.height - 64 - panelHeight - 16;

        GUI.DrawTexture(new Rect(x, y, panelWidth, panelHeight), panelBackTexture);
        GUI.Label(new Rect(x, y + 6, panelWidth, 30), "ENVANTER", textStyle);
        GUI.Label(new Rect(x + 10, y + 40, panelWidth - 20, 36), "Kritik Oran: %" + critChancePercent, textStyle);
    }

    void DrawDamageTexts()
    {
        // Hasar yazilarini ekrana cizer.
        // Yasina gore yukari gider ve yavasca kaybolur.
        for (int i = 0; i < damageTexts.Count; i++)
        {
            DamageText damageText = damageTexts[i];
            float progress = Mathf.Clamp01(damageText.age / DamageTextDuration);
            Vector3 worldPosition = Vector3.Lerp(damageText.startWorld, damageText.endWorld, progress);
            Vector3 screenPosition = mainCamera.WorldToScreenPoint(worldPosition);

            GUIStyle style = damageText.critical ? criticalDamageTextStyle : damageTextStyle;
            Color oldColor = style.normal.textColor;
            style.normal.textColor = new Color(oldColor.r, oldColor.g, oldColor.b, 1f - progress);

            float width = damageText.critical ? 120f : 90f;
            float height = damageText.critical ? 48f : 36f;
            Rect rect = new Rect(screenPosition.x - width / 2f, Screen.height - screenPosition.y - height / 2f, width, height);
            GUI.Label(rect, damageText.text, style);

            style.normal.textColor = oldColor;
        }
    }

    void DrawDeathScreen()
    {
        // Olum ekranini cizer.
        // Koyu zemin, kirmizi yazi ve yazidan asagi akan boya/kan damlalari var.
        GUI.DrawTexture(new Rect(0, 0, Screen.width, Screen.height), deathBackTexture);

        Rect titleRect = new Rect(0, Screen.height / 2f - 145, Screen.width, 90);
        GUI.Label(titleRect, "OLDUN", deathTitleStyle);
        DrawBloodDrips(titleRect);

        GUI.Label(new Rect(0, Screen.height / 2f - 45, Screen.width, 42), "Karanlik seni yakaladi.", textStyle);

        if (GUI.Button(CenterRect(260, 72, 80), "Ana Menuye Don", buttonStyle))
            RestartAfterDeath();
    }

    void DrawVictoryScreen()
    {
        // Bolum bitince gelen tebrik ekrani.
        // Olum ekranina benzer sekilde tek ekranda mesaj verir, sonra ana menuye yollar.
        GUI.DrawTexture(new Rect(0, 0, Screen.width, Screen.height), victoryBackTexture);
        GUI.Label(new Rect(0, Screen.height / 2f - 150, Screen.width, 80), "TEBRIKLER", titleStyle);
        string rewardText = lastVictoryGavePowerup ? "1 guclendirme hakki kazandin." : "Bu bolumun odulunu zaten aldin.";
        GUI.Label(new Rect(0, Screen.height / 2f - 55, Screen.width, 50), "Bolum " + currentChapter + " tamamlandi. " + rewardText, textStyle);

        if (GUI.Button(CenterRect(300, 72, 80), "Ana Menuye Don", buttonStyle))
            ReturnToMenuAfterVictory();
    }

    void DrawPowerups()
    {
        // Kalici guclendirmeler ekrani.
        // Bolum bitirdikce token kazanirsin, burada o tokenlari kalici bonuslara harcarsin.
        GUI.Label(new Rect(0, 45, Screen.width, 60), "GUCLENDIRMELER", titleStyle);

        if (GUI.Button(new Rect(24, 24, 130, 46), "Geri", buttonStyle))
            mode = Mode.Menu;

        GUI.Label(new Rect(0, 105, Screen.width, 40), "Hak: " + powerupTokens, textStyle);
        DrawPowerupButton(0, "Para Kazanci", "1.2x", moneyBoostUnlocked, BuyMoneyBoost);
        DrawPowerupButton(1, "Maksimum Can", "1.5x", maxHealthBoostUnlocked, BuyMaxHealthBoost);
        DrawPowerupButton(2, "Saldiri", "2x", attackBoostUnlocked, BuyAttackBoost);
    }

    void DrawPowerupButton(int row, string label, string value, bool unlocked, System.Action buyAction)
    {
        // Tek bir kalici guclendirme satirini cizer.
        // Acildiysa "Acik" yazar; acilmadiysa 1 hak karsiliginda alinabilir.
        float y = 170 + row * 88;
        GUI.Label(new Rect(Screen.width / 2f - 310, y + 8, 220, 58), label + ": " + value, textStyle);
        string buttonText = unlocked ? "Acik" : "1 Hak Kullan";

        GUI.enabled = !unlocked && powerupTokens > 0;
        if (GUI.Button(new Rect(Screen.width / 2f + 30, y, 230, 58), buttonText, buttonStyle))
            buyAction();
        GUI.enabled = true;
    }

    void DrawBloodDrips(Rect titleRect)
    {
        // Bu damlalar gercek fizik degil; sadece goruntu efekti.
        // Farkli uzunluklarda kirmizi ince dikdortgenler cizerek akan boya hissi veriyoruz.
        float centerX = Screen.width / 2f;
        float dripTop = titleRect.y + 70;
        float[] xOffsets = { -145f, -92f, -36f, 18f, 72f, 132f };
        float[] heights = { 46f, 78f, 36f, 62f, 92f, 50f };
        float[] widths = { 10f, 14f, 8f, 12f, 16f, 9f };

        for (int i = 0; i < xOffsets.Length; i++)
        {
            Rect drip = new Rect(centerX + xOffsets[i], dripTop, widths[i], heights[i]);
            GUI.DrawTexture(drip, bloodTexture);
            GUI.DrawTexture(new Rect(drip.x - 4, drip.y + drip.height - 3, drip.width + 8, drip.width + 8), bloodTexture);
        }
    }

    void DrawBar(Rect rect, int value, int maxValue, Texture2D fillTexture, Texture2D backTexture)
    {
        // Can ve mana barini cizen ortak fonksiyon.
        // Once arka plan cizer, sonra icini orana gore doldurur.
        float fill = maxValue <= 0 ? 0f : Mathf.Clamp01((float)value / maxValue);
        GUI.DrawTexture(rect, backTexture);
        GUI.DrawTexture(new Rect(rect.x + 2, rect.y + 2, (rect.width - 4) * fill, rect.height - 4), fillTexture);
    }

    void BuyHealth()
    {
        // Yetenek ekraninda can satin alma.
        if (money < healthUpgradeCost) return;

        healthUpgradeLevel++;
        money -= healthUpgradeCost;
        healthUpgradeCost = Mathf.RoundToInt(healthUpgradeCost * 1.25f);
        int gain = GetHealthUpgradeAmount();
        health += gain;
        maxHealth += gain;
    }

    void BuyAttack()
    {
        // Yetenek ekraninda saldiri satin alma.
        if (money < attackUpgradeCost) return;

        attackUpgradeLevel++;
        money -= attackUpgradeCost;
        attackUpgradeCost = Mathf.RoundToInt(attackUpgradeCost * 1.25f);
        attack += GetAttackUpgradeAmount();
    }

    void BuyMana()
    {
        // Yetenek ekraninda mana yenilenmesi satin alma.
        if (money < manaUpgradeCost) return;

        manaUpgrade++;
        money -= manaUpgradeCost;
        manaUpgradeCost = Mathf.RoundToInt(manaUpgradeCost * 1.25f);
    }

    void BuyCrit()
    {
        // Yetenek ekraninda kritik oran satin alma.
        // Her satin alma +5% kritik sansi verir, 100% ustune cikmasin diye sinirliyoruz.
        if (money < critUpgradeCost || critChancePercent >= 100) return;

        critUpgradeLevel++;
        money -= critUpgradeCost;
        critChancePercent = Mathf.Min(100, critChancePercent + 5);
        critUpgradeCost = Mathf.RoundToInt(critUpgradeCost * 1.2f);
    }

    void BuyMoneyBoost()
    {
        // Kalici para guclendirmesi.
        // Bundan sonra dusmanlardan gelen para 1.2x olur.
        if (powerupTokens <= 0 || moneyBoostUnlocked) return;

        powerupTokens--;
        moneyBoostUnlocked = true;
    }

    void BuyMaxHealthBoost()
    {
        // Kalici maksimum can guclendirmesi.
        // Mevcut maksimum cani hemen 1.5x yapar, sonraki max can artislarini da guclendirir.
        if (powerupTokens <= 0 || maxHealthBoostUnlocked) return;

        powerupTokens--;
        maxHealthBoostUnlocked = true;
        maxHealth = Mathf.RoundToInt(maxHealth * 1.5f);
        health = maxHealth;
    }

    void BuyAttackBoost()
    {
        // Kalici saldiri guclendirmesi.
        // Mevcut saldiriyi hemen 2x yapar, sonraki saldiri artislarini da guclendirir.
        if (powerupTokens <= 0 || attackBoostUnlocked) return;

        powerupTokens--;
        attackBoostUnlocked = true;
        attack *= 2;
    }

    Rect CenterRect(float width, float height, float yOffset)
    {
        return new Rect(Screen.width / 2f - width / 2f, 190 + yOffset, width, height);
    }

    void BuildGuiStyles()
    {
        // Yazilarin, butonlarin ve bar renklerinin ayarlari.
        // Bunu her karede bastan yapmamak icin bir kez olusturup sonra kullaniyoruz.
        if (titleStyle != null)
            return;

        titleStyle = new GUIStyle(GUI.skin.label)
        {
            alignment = TextAnchor.MiddleCenter,
            fontSize = 48,
            fontStyle = FontStyle.Bold,
            normal = { textColor = new Color(0.95f, 0.18f, 0.12f) }
        };

        textStyle = new GUIStyle(GUI.skin.label)
        {
            alignment = TextAnchor.MiddleCenter,
            fontSize = 21,
            fontStyle = FontStyle.Bold,
            clipping = TextClipping.Overflow,
            normal = { textColor = Color.white }
        };

        buttonStyle = new GUIStyle(GUI.skin.button)
        {
            fontSize = 22,
            fontStyle = FontStyle.Bold,
            clipping = TextClipping.Overflow
        };

        deathTitleStyle = new GUIStyle(GUI.skin.label)
        {
            alignment = TextAnchor.MiddleCenter,
            fontSize = 72,
            fontStyle = FontStyle.Bold,
            clipping = TextClipping.Overflow,
            normal = { textColor = new Color(0.86f, 0.02f, 0.02f) }
        };

        damageTextStyle = new GUIStyle(GUI.skin.label)
        {
            alignment = TextAnchor.MiddleCenter,
            fontSize = 22,
            fontStyle = FontStyle.Bold,
            clipping = TextClipping.Overflow,
            normal = { textColor = Color.white }
        };

        criticalDamageTextStyle = new GUIStyle(GUI.skin.label)
        {
            alignment = TextAnchor.MiddleCenter,
            fontSize = 29,
            fontStyle = FontStyle.Bold,
            clipping = TextClipping.Overflow,
            normal = { textColor = Color.white }
        };

        manaBarTexture = MakeGuiTexture(new Color(0.16f, 0.48f, 1.00f));
        manaBarBackTexture = MakeGuiTexture(new Color(0.03f, 0.08f, 0.16f));
        healthBarTexture = MakeGuiTexture(new Color(0.20f, 0.82f, 0.28f));
        healthBarBackTexture = MakeGuiTexture(new Color(0.07f, 0.14f, 0.08f));
        expBarTexture = MakeGuiTexture(new Color(0.95f, 0.72f, 0.18f));
        expBarBackTexture = MakeGuiTexture(new Color(0.18f, 0.13f, 0.04f));
        bloodTexture = MakeGuiTexture(new Color(0.65f, 0.00f, 0.00f));
        deathBackTexture = MakeGuiTexture(new Color(0.06f, 0.00f, 0.00f));
        victoryBackTexture = MakeGuiTexture(new Color(0.03f, 0.07f, 0.05f));
        panelBackTexture = MakeGuiTexture(new Color(0.05f, 0.05f, 0.08f, 0.92f));
    }

    Texture2D MakeGuiTexture(Color color)
    {
        var texture = new Texture2D(1, 1);
        texture.SetPixel(0, 0, color);
        texture.Apply();
        return texture;
    }
}
