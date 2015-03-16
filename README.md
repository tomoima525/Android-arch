
今自分の考えてる事の整理用 + チームへの共有用です。

なにか質問、問い合わせなどは[@kgmyshin](https://twitter.com/kgmyshin)まで。

## Androidの設計

### 全体像

![overview](https://raw.githubusercontent.com/kgmyshin/Android-arch/master/art/overview.png)

### パッケージ構成

```
com.kgmyshin.project
├─ HogeApplication.java
├─ ApplicationModule.java
├─ presentation
|  ├─ 考え中
|  └─ PresentationModule.java
├─ domain
|  ├─ entity
|  ├─ value
|  ├─ repository
|  ├─ usecase
|  ├─ sharedusecase
|  └─ DomainModule.java
└─ infra
   ├─ memory
   ├─ prefs
   ├─ provider
   ├─ api
   ├─ repository
   └─ InfraModule.java
```

### 必須使用ライブラリ

- [Dagger](http://square.github.io/dagger/)
- [EventBus](https://github.com/greenrobot/EventBus)

## ドメイン層

### ドメイン層の役割

1. Entity
業務データを保持する為のクラス

2. Value Object
業務データを表すが、同一性は必要ないもの。

2. Repository
業務データを操作する為のメソッドを実装し、UseCase(Service)クラスに提供する
Entityオブジェクトに対するCRUD操作。findAll, find, create, updateなどのメソッド。
ドメイン層ではinterfaceのみ用意する。

3. UseCase(Service)
業務ロジックを実行する為のメソッドを実装し、アプリケーション層に提供する。
業務ロジック内で必要となる業務データは、Repositoryを介して、Entitiyオブジェクトとして取得する

4. SharedUseCase(SharedService)
各UseCaseにおいて共通化できるロジックを各UseCaseにたいして提供する

### Entityの作成

Entityはデータベースのテーブルと1:1となることが多い為、テーブルに従って作成する。

```
public class Hoge {
  public String name;
  public int id;
}
```

### Value Objectの作成

enumなど、永続化する必要のないものなどをここに定義する

### Repositoryの作成

ドメインではinterfaceのみ作成する。(継承クラスはinfra層で実装)。
EntityオブジェクトへのCRUD操作 + αのメソッドをinterfaceに定義する。

```
public interface HogeRepository {
  public ArrayList<Hoge> findAll();
  public ArrayList<Hoge> find(String id);
  public boolean create(Hoge hoge);
  public boolean update(Hoge hoge);
  public boolean delete(Hoge hoge);
}
```

### UseCaseの作成

1ユースケースで1つ作成する。Controllerから呼ばれ、ViewやControllerに結果を返却する。

Daggerでユースケースの取り替えができるようにinterfaceとクラスのセットで定義する。

UIスレッドから呼ばれるときは非同期で行う。(AndroidのServiceから呼ばれる場合は同期でもよい)

具体的には下記のベースクラスを継承して、callメソッドを実装する。結果の返却には[EventBus](https://github.com/greenrobot/EventBus)を用いる。

```
abstract public class AsyncUseCase<T> {

    private ExecutorService mExecutorService = Executors.newSingleThreadExecutor();

    public void start(final T params) {
        mExecutorService.submit(new Runnable() {
            @Override
            public void run() {
                call(params);
            }
        });
    }

    abstract protected void call(T params);
}
```

下記が例。

interfaceを定義

```
public interface CreateHogeUseCase {
    public Hoge hoge;
    public class OnCreated {} // 完了イベント
}
```

parameterクラスを定義。

```
public class CreateHogeUseCaseParams {
    public Hoge hoge;
}
```
ユースケースクラスを定義。

```
public class CreateHogeUseCaseImpl extends AsyncUseCase<CreateHogeUseCaseParams> implements CreateHogeUseCase {

    private HogeRepository hogeRepository;

    public CreateUsageAsyncUseCase(HogeRepository hogeRepository) {
        this.hogeRepository = hogeRepository;
    }

    @Override
    public void call(CreateHogeUseCaseParams params) {
        Hoge hoge = params.hoge;
        hogeRepository.create(usage);
        EventBus.getDefault.post(new OnCreated());
    }
}
```

**(Serviceレイヤの作成はEntity毎につくるケースもあるが、今のところ経験上ユースケース毎の方がすっきりしやすい)** 

### DomainModule.java

ユースケース毎にprovideメソッドを定義。複数インスタンス必要のないものは@Singletonをつける

```
@Module(
        complete = false, library = true)
public class DomainModule {

    @Provides
    @Singleton
    CreateHogeUseCase provideCreateHogeUseCase(HogeRepository hogeRepository) {
        return new CreateHogeUseCaseImpl(hogeRepository);
    }

    :
}
```

## インフラ層

### インフラ層の役割

Repositoryをもとに、memory, ContentProvider(SQLite), api, SharedPreferencesのどれかから値をドメイン層に返却する。1 Entity(Table) 1 Repositoryとなることが多い。

![overview](https://raw.githubusercontent.com/kgmyshin/Android-arch/master/art/infra.png)

データの永続化先がSharedPreferencesではない場合、かならずmemoryとセットで使う。

たとえばContentProviderの場合、findAllなどした場合は、かならずmemoryに保存する。

### memory

メモリキャッシュとして使う。

実装例

```
HogeCache {
  ArrayList<Hoge> hoge;
}
```

### ContentProvider

ContentProvider、SQLIteOpenHelperを通常通り作成する。

テーブル毎にColumnsクラス、DatabseAccessObjectクラスを作成する。

これに関しては、table名、column名を引数としてコマンドラインに打ち込むだけで一式を作られるジェネレータを作成(中)なので、そちらを使っていただけたら。

[ContentProviderGenerator](https://github.com/kgmyshin/ContentProviderGeneretor)

### API

すべて同期で行いRepositoryに返す。

```
public class HogeApi {

    public ArrayList<Hoge> fetchAll() {
        return hogeList;
    }

    public boolean create(Hoge hoge) {
        ・・通信・・
        return result;
    }

    public boolean update(Hoge hoge) {
        ・・通信・・
        return result;
    }
}
```

### Repository


メモリキャッシュ、プロバイダ、APIなどからデータ操作を実装する。

下記が例。


```
public class HogeDataRepository  implements HogeRepository {

    @Inject Application app;

    private HogeCache hogeCache;
    private HogeDao hogeDao;

    public HogeDataRepository() {
        hogeCache = new HogeCache();
        hogeDao = new HogeDao();
    }

    @Override
    public ArrayList<Hoge> findAll(boolean useCache) {
        if (hogeCache.caache == null || !useCache) {
            hogeCache.caache = hogeDao.findAll(app);
        }
        return hogeCache.caache;
    }
    :
}
```

特にProvider系の話だが、` @Inject Application app`としてApplication(Context)をメンバとして持っておく事で、プレゼンテーション層からドメイン層のメソッドを呼ぶ際にContextを渡す必要がなくなる。

### InfraModule.java

各Repositoryクラスをprovideする。すべて@Singletonにする。

```
@Module(
        injects = {
                HogeDataRepository.class,
                :
        },
        complete = false, library = true
)
public class InfrastructureModule {

    @Provides
    @Singleton
    HogeRepository provideHogeRepository() {
        return new HogeDataRepository();
    }
    :

}

```

## アプリケーションクラスおよびApplicationModule.java

ApplicationModuleは下記。Applicationをメンバに持つ。

```
@Module(
        includes = {
                DataSourceModule.class,
                DomainModule.class,
                PresentationModule.class
        }
)
public class ApplicationModule {

    private Application app;

    public ApplicationModule(Application app) {
        this.app = app;
    }

    @Singleton
    @Provides
    Application provideApplication() {
        return app;
    }
}
```

アプリケーションクラスではObjectGraphをメンバに持つようにする。

プレゼンテーション層用にinjectメソッドも用意。

```
public class HogeApplication extends Application {

    private ObjectGraph objectGraph;

    @Override
    public void onCreate() {
        super.onCreate();
        objectGraph = ObjectGraph.create(new LogHackerModule(this));
    }

    public <T> T inject(T instance) {
        objectGraph.inject(instance);
        return instance;
    }

}
```

## TODO

- ContentProviderGeneratorの完成
- Presentation層について
- テストについて
- ドメイン層よりしたのCodeGeneratorをつくる

:

## 考えてる事

- RxAndroidは？

チャットなどイベントが頻繁に飛んでくるようなアプリなら取り入れていいかも。
また同様に、頻繁にイベントが来る画面ではスポットでも可。
それ以外だとイベントをストリームとするメリットあまりないかなという印象。
ドメイン層から導入するとがっつり設計が変わるので、まだリスクが大きいかも。

- Code Generatorについて

Entityの内容と永続化先がどこか決まれば、あとは(ユースケースなどを除いて)ほとんど毎回同じコードです。
なので、entityのクラス名、entityのメンバ、永続化先(providerかapiかprefsか)を指定すればできるのではないかと。
ということで作ってみます。

:
