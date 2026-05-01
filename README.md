# 🎮 Kitchen - симулятор готовки

[![Unity Version](https://img.shields.io/badge/Unity-2022.3+-blue.svg)](https://unity.com/)


## 🎥 Видео геймплея

| Геймплей 1 | Геймплей 2 |
|------------|------------|
| <video src="https://github.com/user-attachments/assets/94ec70d7-78db-47f3-b38b-8c4f77eaf73b" controls width="400"></video> | <video src="https://github.com/user-attachments/assets/673a9110-a17d-49b0-a732-bfdcc4875c4b" controls width="400"></video> |

## 📝 Описание

**Первая моя игра на Unity. Суть - готовка блюд из продуктов с помощью нарезки, плиты, кастрюль и сковородок. 
Приготовленное видео оценивается в соответствии с выбранным рецептом**  

**Ключевые особенности:**
- 🕹️ Настройки управления (назначения клавиш), яркости, аудио, разрешения;
  и модификатора времени игровых событий;
- 🎨 Уникальный визуальный стиль;
- 🎵 Фоновая музыка и звуки;
- 🧩 Уровни представлены в виде записей в БД SQLite и JSON с продуктами рецепта и их обработкой;
  
## 🔧 Технические особенности
- Графика реализована с помощью URP, текстуры статичных объектов запечены с освещением;
- Для управления используется New Input System, реализована переназнача клавиш;
- UI реализован с помощью MVC;
- Так же в архитектуре используются паттерны Singletone, Strategy, Prototype и другие;
- Все настройки и информация об уровнях сохраняются в БД SQLite (требования дипломной, иначе использовался бы только JSON);
- У игры 2 сцены: главное меню и геймлей, при выборе уровня загружается сцена геймплей, во время загрузки сцена берёт
из БД информацию о выбранном рецепте. Сцены переключаются ассинхронно, с анимированной заставкой;
- Рецепт представляет собой сериализованный в JSON файл класс со списком продуктов, он хранится в папке Resources;
- Все продукты представлены классом с информацией о продукте, информация о продуктах хранится в ScriptableObject в папке Resources;
  
```csharp
[CreateAssetMenu(fileName = "FoodInfo", menuName = "Scriptable Objects/FoodInfo")]
[System.Serializable]
public class FoodInfo : ScriptableObject
{
    public string FoodId;
    public string FoodName;
    public float GramsWeight;
    public bool IsPour;
    public CutType[] AllCutTypes;
    [HideInInspector]
    public List<FoodParametr> Params = new List<FoodParametr>();
    [HideInInspector]
    public CutType CurrentCutType;
    public FoodInfo Clone()
    {
        FoodInfo clone = this.MemberwiseClone() as FoodInfo;
        List<FoodParametr> cloneParams = new List<FoodParametr>();
        foreach (FoodParametr param in Params)
        {
            cloneParams.Add(param.Clone());
        }
        clone.Params = cloneParams;
        return clone;
    }
    public SerializableFoodInfo GetSerializableFoodInfo()
    {
        SerializableFoodInfo sfoodInfo = new SerializableFoodInfo();
        sfoodInfo.FoodId = FoodId;
        sfoodInfo.FoodName = FoodName;
        sfoodInfo.GramsWeight= GramsWeight;
        sfoodInfo.IsPour = IsPour;
        sfoodInfo.AllCutTypes = AllCutTypes;
        List<SerializableFoodParameter> sparams = new List<SerializableFoodParameter>(); 
        foreach(var p in Params)
        {
            sparams.Add(p.GetSerializbleFoodParameter());
        }
        sfoodInfo.Params = sparams;
        sfoodInfo.CurrentCutType = CurrentCutType;
        return sfoodInfo;
    }
}
```

- Продукты так же хранят список своих параметров

```csharp
[CreateAssetMenu(fileName = "FoodParametr", menuName = "Scriptable Objects/FoodParametr")]
[System.Serializable]
public class FoodParametr : ScriptableObject
{
    public string ParamName;
    [HideInInspector]
    public float ParamValue;
    public string Desc = "";
    public string ValueChar = "";
    public string TooMuchError = "";
    public string TooLittleError = "";
    public string NoThisError = "";
    public string NoNeedThisError = "";

    public bool IsSpice = false;
    public FoodParametr Clone()
    {
        return this.MemberwiseClone() as FoodParametr;
    }
    public SerializableFoodParameter GetSerializbleFoodParameter()
    {
        SerializableFoodParameter sfoodParam = new SerializableFoodParameter();
        sfoodParam.ParamName = ParamName;
        sfoodParam.ParamValue = ParamValue;
        sfoodParam.IsSpice = IsSpice;
        return sfoodParam;
    }
}
```

- Параметры хранятся в пуле параметров и загружаются из ScriptableObject при необходимости добавления (т.к. они могут применяться к любому продукту),
пример параметров - солёность продукта, жаренность, варёность. В полях так же представлены строки для оценки на случай
если продукту не хватает параметров или их значений (например недоварен) или наоборот их больше чем должно быть.
- Продукты можно нарезать, из-за чего спавнятся новые продукты, старый продукт удаляется, передавая информацию о себе новым
(но к примеру вес картошки передаётся её разрезанным частям разделенным на количество частей

```csharp
public void Cut()
{
    if (nextCuttedPart==null)
    {
        return;
    }
    Plate plateForParts = plate;
    if(plate!=null)plate.RemoveFood(this);
    for (int i = 0; i < CutCount; i++)
    {
        var part = Instantiate(nextCuttedPart, transform.position+new Vector3(UnityEngine.Random.Range(-0.10f, 0.10f),0, UnityEngine.Random.Range(-0.10f, 0.10f)), Quaternion.identity,  transform.parent);
        var partFoodComponent = part.GetComponent<FoodComponent>();
        partFoodComponent.FoodInfo = FoodInfo.Clone();
        partFoodComponent.FoodInfo.GramsWeight = FoodInfo.GramsWeight/(float)CutCount;
        foreach(var param in partFoodComponent.FoodInfo.Params)
        {
            if (param.IsSpice)
            {
                param.ParamValue /= CutCount;
            }
        }
        if(plateForParts != null)
        {
            partFoodComponent.OnPull = new UnityEvent<FoodComponent>();
            if (plateForParts.TryAddFood(partFoodComponent))
            {
                var dgo = part.GetComponent<DraggableObject>();
                if(dgo!=null)
                {
                    dgo.OffRigidbody();
                    dgo.CanDrag = false;
                }
            }
        }
        else
        {
            part.gameObject.transform.parent = Parents.GetInstance().FoodParent.transform;
        }
    }
    Destroy(gameObject);
    
}
```

- Продукты могут храниться в посуде, перемещаясь в неё визуально (тарелку) или скрываясь (кастрюля);
  
```csharp
public class Plate : MonoBehaviour, IListable, IFinish
{

    public bool CanPull { get; private set; } = true;
    [HideInInspector]
    public UnityEvent<string> OnUpdateInfo;
    [SerializeField]
    float maxWeight = 500f;
    [SerializeField]
    bool canFinish = true;
    [SerializeField]
    Vector3 offset = Vector3.zero;
    [SerializeField]
    Vector3 randomRange = Vector3.zero;
    [SerializeField]
    GameObject content;
    bool isShowedContent = false;
    ShowObjectInfo info;
    MeshRenderer contentMaterial;
    Texture2D contentEtalonTexture;

    [HideInInspector]
    public float MaxWeight { get => maxWeight; }
    public ObservableCollection<FoodComponent> Foods { get; set; }
    public bool HasWater { get; set; }
    public bool CanFinish { get=> canFinish; }

    private void Awake()
    {
        OnUpdateInfo = new UnityEvent<string>();
        HasWater = false;
    }
    private void Start()
    {
        Foods = new ObservableCollection<FoodComponent>();
        Foods.CollectionChanged += UpdateInfo;
        if(content!= null)
        {
            contentMaterial = content.GetComponent<MeshRenderer>();
            if (contentMaterial != null)
            {
                contentEtalonTexture = Resources.Load<Texture2D>("Levels/" + SettingsInit.CurrentLevelName + "/" + BellFinish.Level.RecipeFoodPictureName);
                contentMaterial.material.mainTexture = contentEtalonTexture;
            }
            content.SetActive(false);
        }
        info = GetComponent<ShowObjectInfo>();
        info.ObjectData = $"Макс вес - {maxWeight} г";
    }
    public bool TryAddFood(FoodComponent food)
    {
        if (food == null) return false;
        if (this.Foods.Sum(f => f.FoodInfo.GramsWeight) + food.FoodInfo.GramsWeight > maxWeight)
        {
            UIElements.ShowToast($"{food.FoodInfo.FoodName}: не добавлено. Максимальная масса {maxWeight}г превышена!");
            return false;
        }
        else
        {
            if (food.plate != null)
            {
                var plate2 = food.plate;
                plate2.RemoveFood(food);
            }

            this.Foods.Add(food);
            Vector3 randomOffset = Vector3.zero;
            randomOffset.x = UnityEngine.Random.Range(-randomRange.x, randomRange.x);
            randomOffset.y = UnityEngine.Random.Range(-randomRange.y, randomRange.y);
            randomOffset.z = UnityEngine.Random.Range(-randomRange.z, randomRange.z);

            food.gameObject.transform.parent = transform;
            food.gameObject.transform.localPosition = Vector3.zero + offset + randomOffset;
            food.gameObject.transform.localRotation = Quaternion.identity;
            if(isShowedContent && content!=null)
            {
                food.gameObject.transform.localScale = Vector3.zero;
            }
            food.plate = this;
            food.OnPull.RemoveAllListeners();
            food.OnPull.AddListener(PutFood);
            return true;
        }
    }
    public void RemoveFood(FoodComponent food)
    {
        if (food == null) return;
        this.Foods.Remove(food);
        food.plate = null;

        if(food.gameObject.layer != LayerMask.NameToLayer("DraggableObject"))
        {

            food.gameObject.layer = LayerMask.NameToLayer("DraggableObject");
            foreach (Transform child in food.transform)
            {
                child.gameObject.layer = LayerMask.NameToLayer("DraggableObject");
            }
        }
    }
    public void SpiceFood(SpiceComponent spice)
    {
        foreach(FoodComponent food in Foods)
        {
            spice.AddSpiceTo(food);
        }
    }
```
- Сама посуда (кастрюля и сковородка) может располагаться на плите (или в духовке) и нагреваться, воздействуя на параметры находящейся в ней еды;
- Еду можно перемещать из одной посуды в другую всю или по отдельности;

БД располагается в специальной папке StreamingAssets в формате .bytes
Структура БД:

<img width="490" height="710" alt="image" src="https://github.com/user-attachments/assets/dd1b1fcf-0cfe-446e-9072-e4184d188a46" />

- Levels - информация об уровнях для UI (название, описание, фото...), так же хранится рекорд игрока по времени и оценке (звёздам);
- CurrentLevel - одна запись об ID текущего загруженном уровня, он подхватывается во время загрузки сцены геймлпея и
загружается соответствующий рецепт;
- LockedLevels - информация о неоткрытых уровнях, чтобы те нельзя было выбрать из списка. 
Поле NeedForUnlockId - Id уровня, который нужно пройти для открытия этого. При открытии уровень удаляется из этой таблицы;
- Recipes - информация о рецептах, хранится название JSON файла, в котором хранится сериализованный рецепт;
- SettingsControls - информация для привязки назначенных клавиш;
- SettingsValues - настройки хранимые одной переменной, например уровень яркости или сенсы мыши.

### В итоге 
Cтруктура уровня выглядит примерно так: информация об выбранном уровне подтягивается из БД,
оттуда берётся имя JSON файла с сериализованным рецептом (списком продуктов с их параметрами), после манипуляций с продуктами
(нарезкой, варкой, жаркой, приправой специями) эти продукты, находящиеся в одной тарелке, сравниваются с этим итоговым эталонным
списком продуктов из JSON и на основе этого выдается финишная информация: все ли продукты есть в блюде, правильно ли они обработаны
и за какое время рецепт приготовлен. Если рецепт правильный - проверяется есть ли в БД записи с закрытыми уровнями, которым требуется 
пройденный, если есть - такие записи удаляются, разблокируя уровни.


## 🎯 Геймплей

### В главном меню кнопка настроек (скриншот 1):
- вкладка игра (настройки чувствительности мыши, скорости течения времени - как быстро внутриигровое время течет относительно
реального, от этого зависит напрмиер как быстро закипит и выкипит кастрюля с водой) (скриншот 2)
- графика (разрешение, яркость, полноэкранный режим) (скриншот 3)
- аудио (общая громкость, громкость звуков, музыки) (скриншот 4)
- назначение клавиш (скриншот 5)
- обучение - набор картинок с обучением основным механикам игры (скриншот 6)
- список уровней (скриншот 7)

### Передвижение
Возможность ходить, прыгать, приседать

### Взаимодействие
- возможность открывать дверцы шкафчиков и холодильников наведя на них и нажав клавишу; (скриншот 8)
- можно взять продукт или посуду в руки и перенести, наведя и нажав клавишу, после отпустить и бросить; (скриншот 9)
- возможность положить продукт в посуду, взяв его в руки и наведясь + нажав клавишу на посуду; (скриншот 10)
- так же можно положить продукты из одной посуды в другую подобным образом; (скриншот 11)
- из посуды продукт так же можно достать взяв в руки тарелку, наведя на посуду и открыв список её продуктов. 
Можно выбрать нужный, нажать UI кнопку и выбрать количество, чтобы переложить его в тарелку; (скриншот 12)
- возможность достать нож и нарезать продукты, если они находятся в посуде - все нарезанные продукты автоматически в неё переместятся; (скриншот 13)
- возможность расположить кастрюлю или сковородку на плите (или в духовке), взяв посуду в руки и нажав на место на плите; (скриншот 14)
- возможность регулировать огонь на этих конфорках; (скриншот 15)
- возможность налить воды в кастрюлю, взяв её в руки и нажав на кран; (скриншот 16)
- возможность добавить специи в продукт, взяв их в руки и нажав на продукт (либо на посуду чтобы например посолить всё блюдо); (скриншот 17)
- возможность нажать на колокол, чтобы оценить блюдо (перед этим тарелку нужно поместить на столе с колоколом); (скриншот 18)

### Другие механики
- текущий рецепт уровня отображается на доске, его можно закрепить на экране нажав кнопку; (скриншот 19)
- при наведении на объекты они подсвечиваются, показывая визуальные подсказки и информацию; (скриншот 20)
- у посуды эта информация - список её продуктов, в случае с кастрюлей и сковородкой - температура и время на плите; (скриншот 21)
- на плите посуда постепенно нагревается, при снятии - остывает; 
- вода в кастрюле выкипает на огне через некоторое время; (при достижении температуры кипения),
чтобы продолжить варку нужно долить воды, иначе продукты внутри будут жариться; (скриншот 22)
- при неправильном действии высвечиваются уведомления; (например если мы пытаемся вынуть из посуды продукт, держа в руках заполненную тарелку);
(скриншот 23)
- при нажатии на escape высвечивается меню (настройки, начать заново, подсказки, выход в меню); (скриншот 24)
- из кастрюли суп можно набрать в суповую тарелку. Если в супе правильные ингридиенты - у него появляется соответствующая текстура; (скриншот 25)
- окно финиша (при нажатии на колокол) оценивает блюдо; (скриншот 26, 27)
- в игре идёт таймер, он используется для оценки приготовления блюда;

## 🖼️ Скриншоты

### Главное меню

1. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/2af76415-879b-4b61-833b-f809b2c58f54" />

2. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/ac39f788-4dbc-4106-994d-43010180a2d2" />

3. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/861d0f15-f111-47c9-9487-47da976fe68c" />

4. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/fab2d48a-8caa-46a5-91f3-4bc81155af75" />

5. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/96b97af6-d6bd-408f-b3bb-e1d1fc5a1eda" />

6. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/468cb54f-3456-40fc-bb43-c85c5ce11e44" />

7. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/43c184ca-898d-48b9-9153-25072c919f26" />

### Геймплей

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/c7e9a2e4-d59f-4645-bc2b-76d17f68e3fa" />

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/e3833562-0f75-4a92-bb8d-2e23cefb5554" />

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/71dd670a-cc64-4e95-a0cb-ac6d6d034d1e" />

8. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/5a307de6-ede6-4922-9a7c-d55382ccf6ca" />

9. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/5d5f413b-3054-4f16-99df-57623c1ec4c3" />

10. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/75b1c10e-b53a-4b92-ba48-b59af563a813" />

11. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/4244f655-1cd4-4e67-86ff-d70e28861603" />

12. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/82f93084-4992-46ec-9b5f-fbf0e56e1e36" />

13. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/17137bc6-ed00-4726-bb76-29683a398959" />

14. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0468dd9f-8724-4e59-aefc-0ae8fee38bce" />

15. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/acf13b16-5548-47d0-9bde-b2d55fa85eb0" />

16. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/667ffab7-0037-4531-921d-55e7a398fdc2" />

17. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/9ef23151-8fa1-480f-9982-cd9c132957ae" />

18. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/24adb989-c4e8-40ca-b503-96f952f6a3fd" />

19. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/32167207-7057-408d-a23f-9ce74fdb4ed0" />

20. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/09b5fd08-9124-4f06-a235-48b0a1cfed90" />

21. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0f921f69-ee9f-4e64-8c28-0e707ce50a3e" />

22. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/8917f515-c19e-4db9-80b6-6646ad431d63" />

23. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/2d079e67-1f5d-4659-ba3d-69eb11501c41" />

24. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/2787667f-3d71-4fe0-b7b3-57d23c28fd5d" />

25. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b5e7c27e-a8c9-4476-8cd5-97d130585b02" />

26. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/83784a60-7092-4bac-b9cc-8e1314b86181" />

27. <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/4b0de09a-0f36-4772-be2a-b77d275cec8e" />
