
## 概述
> MVP模式解决了以往MVC模式中，View与Model耦合度较高的问题，通过在Model和View之间架上一个Presenter桥梁，将主要逻辑操作从View层转移到Presenter层，同时减少view层代码，也可实现presenter中逻辑复用；

#### MVP各指什么，如何工作
- M(Model):数据仓库，给Presenter提供，同时有本地缓存和网络缓存两部分，在数据集合较小时，可以考虑内存缓存；
- v(View):在Android中一般指activity或者fragment，首先会进行布局控件初始化工作，主要是根据用户操作获取意图，将主要需要实现的逻辑传递给Presenter,让Presenter去进行复杂的逻辑处理。view中暴露一个接口用于与presenter进行绑定，进而可以调用presenter进行相关逻辑处理。
- p(presenter):是M层和V层的桥梁，其中主要包含一些逻辑处理。在初始化时持有一个view引用和一个Model引用，当view传递命令过来时，中间通过逻辑处理后，会调用model层的数据操作处理，需要得到Model的响应时，可以将一个callback传递给model,model处理完相关事务后会回调这个callback，进而presenter在它里面能获取到响应信息，进而可以继续一些逻辑操作后，再通过view的引用通知view进行页面响应。

![MVP图示](https://note.youdao.com/yws/public/resource/4fbcae57ed9c225f22300c9968c9ee8b/xmlnote/8A0D7511CA3F4C38A6355A76070ED45C/15289)

#### 通过任务列表管理demo展示具体工作方式

- **BaseView接口**
```
public interface BaseView<T> {
    void setPresenter(T presenter);
}
```
BaseView暴露一个setPresenter()接口，让其可以持有presenter引用；

- **BasePresenter接口**
```
public interface BasePresenter {
    //一般activity调用onStart()方法时调用
    void start();
}
```
此时我们已经拥有了两个公共接口，是所有view和presenter都必须实现的方法，因为所有的view都必须持有一个presenter的引用才可以形成桥梁的一边；

- **Contract用于统一管理一对view和presenter**
```
public interface TasksContract {

    interface View extends BaseView<Presenter> {
        /*显示任务列表*/
        void showTasks(List<Task> tasks);
        ...
    }

    interface Presenter extends BasePresenter {
        /*加载列表的逻辑*/
        void loadTasks(boolean forceUpdate);
        ...
    }
}
```
可以看到上方基本是对于一个列表管理的部分接口，并且将页面展示放在view中，逻辑处理放在presenter中，那么十分明显，接下来应该实现这两个接口，给出具体操作，解耦性强，例如现在想要修改保存成功消息的展示，只需要修改view的实现类，而presenter这边完全不需要动；

- **model的主接口**
```
public interface TasksDataSource {

    interface LoadTasksCallback {

        void onTasksLoaded(List<Task> tasks);

        void onDataNotAvailable();
    }

    interface GetTaskCallback {

        void onTaskLoaded(Task task);

        void onDataNotAvailable();
    }

    void getTasks(@NonNull LoadTasksCallback callback);
    void saveTask(@NonNull Task task);
```
对于所有列表管理model的实现类可能有多个，有可能从网络、本地或内存获取，因此需要对这些类进行接口统一，然后创建一个主体类对这几种方式进行管理，根据目前应用情况选择合适的数据加载方式；

- **远程请求数据model实现**（伪网络请求）
```
public class FakeTasksRemoteDataSource implements TasksDataSource {

    private static FakeTasksRemoteDataSource INSTANCE;

    private static final Map<String, Task> TASKS_SERVICE_DATA = new LinkedHashMap<>();

    // Prevent direct instantiation.
    private FakeTasksRemoteDataSource() {}

    public static FakeTasksRemoteDataSource getInstance() {
        if (INSTANCE == null) {
            INSTANCE = new FakeTasksRemoteDataSource();
        }
        return INSTANCE;
    }

    @Override
    public void getTasks(@NonNull LoadTasksCallback callback) {
        callback.onTasksLoaded(Lists.newArrayList(TASKS_SERVICE_DATA.values()));
    }

    @Override
    public void saveTask(@NonNull Task task) {
        TASKS_SERVICE_DATA.put(task.getId(), task);
    }
    ...
}
```
FakeTasksRemoteDataSource实现了getTasks、saveTask等接口，因此如果调用他们，就会通过网络去进行数据处理；

- **本地数据操作model实现**
```
/**
 * Concrete implementation of a data source as a db.
 */
public class TasksLocalDataSource implements TasksDataSource {

    private static volatile TasksLocalDataSource INSTANCE;

    private TasksDao mTasksDao;

    private AppExecutors mAppExecutors;

    // Prevent direct instantiation.
    private TasksLocalDataSource(@NonNull AppExecutors appExecutors,
            @NonNull TasksDao tasksDao) {
        mAppExecutors = appExecutors;
        mTasksDao = tasksDao;
    }

    public static TasksLocalDataSource getInstance(@NonNull AppExecutors appExecutors,
            @NonNull TasksDao tasksDao) {
        if (INSTANCE == null) {
            synchronized (TasksLocalDataSource.class) {
                if (INSTANCE == null) {
                    INSTANCE = new TasksLocalDataSource(appExecutors, tasksDao);
                }
            }
        }
        return INSTANCE;
    }

    /**
     * Note: {@link LoadTasksCallback#onDataNotAvailable()} is fired if the database doesn't exist
     * or the table is empty.
     */
    @Override
    public void getTasks(@NonNull final LoadTasksCallback callback) {
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                //此过程在子线程执行
                //从数据库中获取数据
                final List<Task> tasks = mTasksDao.getTasks();
                //切换回主线程调用callback
                mAppExecutors.mainThread().execute(new Runnable() {
                    @Override
                    public void run() {
                        if (tasks.isEmpty()) {
                            // This will be called if the table is new or just empty.
                            callback.onDataNotAvailable();
                        } else {
                            callback.onTasksLoaded(tasks);
                        }
                    }
                });
            }
        };
        //本地IO操作，使用的是单任务的线程池
        mAppExecutors.diskIO().execute(runnable);
    }

    /**
     * Note: {@link GetTaskCallback#onDataNotAvailable()} is fired if the {@link Task} isn't
     * found.
     */
    @Override
    public void getTask(@NonNull final String taskId, @NonNull final GetTaskCallback callback) {
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                //从数据库获取任务
                final Task task = mTasksDao.getTaskById(taskId);
                //切换到主线程回调
                mAppExecutors.mainThread().execute(new Runnable() {
                    @Override
                    public void run() {
                        if (task != null) {
                            callback.onTaskLoaded(task);
                        } else {
                            callback.onDataNotAvailable();
                        }
                    }
                });
            }
        };
        //在单线程的线程池中执行io操作
        mAppExecutors.diskIO().execute(runnable);
    }

    @Override
    public void saveTask(@NonNull final Task task) {
        checkNotNull(task);
        Runnable saveRunnable = new Runnable() {
            @Override
            public void run() {
                //插入数据库
                mTasksDao.insertTask(task);
            }
        };
        mAppExecutors.diskIO().execute(saveRunnable);
    }
}
```
本地数据操作的TasksLocalDataSource(model)实现了getTask等方法，其只会在本地数据库等本地存储位置操作数据，从中可以看到它是进行IO操作，属于耗时操作，会放到单任务的线程池中处理，防止并发，当返回结果时，需要在主线程中调用callback，这样presenter才能在callback中执行ui操作；

- **主要管理本地缓存model和网络缓存model的TasksLocalDataSource(model)**
```
public class TasksRepository implements TasksDataSource {

    private static TasksRepository INSTANCE = null;

    //网络数据
    private final TasksDataSource mTasksRemoteDataSource;

    //本地数据
    private final TasksDataSource mTasksLocalDataSource;

    /**
     * This variable has package local visibility so it can be accessed from tests.
     */
    Map<String, Task> mCachedTasks;

    /**
     *标记当前本地的缓存(本地数据库和内存)是否为垃圾，垃圾则直接从网络加载
     */
    boolean mCacheIsDirty = false;

    // Prevent direct instantiation.
    private TasksRepository(@NonNull TasksDataSource tasksRemoteDataSource,
                            @NonNull TasksDataSource tasksLocalDataSource) {
        mTasksRemoteDataSource = checkNotNull(tasksRemoteDataSource);
        mTasksLocalDataSource = checkNotNull(tasksLocalDataSource);
    }

    public static TasksRepository getInstance(TasksDataSource tasksRemoteDataSource,
                                              TasksDataSource tasksLocalDataSource) {
        if (INSTANCE == null) {
            INSTANCE = new TasksRepository(tasksRemoteDataSource, tasksLocalDataSource);
        }
        return INSTANCE;
    }

    /**
     * Used to force {@link #getInstance(TasksDataSource, TasksDataSource)} to create a new instance
     * next time it's called.
     */
    public static void destroyInstance() {
        INSTANCE = null;
    }

    /**
     * Gets tasks from cache, local data source (SQLite) or remote data source, whichever is
     * available first.
     * <p>
     * Note: {@link LoadTasksCallback#onDataNotAvailable()} is fired if all data sources fail to
     * get the data.
     */
    @Override
    public void getTasks(@NonNull final LoadTasksCallback callback) {
        checkNotNull(callback);

        // Respond immediately with cache if available and not dirty
        //先从内存加载
        if (mCachedTasks != null && !mCacheIsDirty) {
            callback.onTasksLoaded(new ArrayList<>(mCachedTasks.values()));
            return;
        }

        if (mCacheIsDirty) {
            // If the cache is dirty we need to fetch new data from the network.
            //本地数据过期，从网络加载数据
            getTasksFromRemoteDataSource(callback);
        } else {
            //本地数据未过期，从本地加载
            // Query the local storage if available. If not, query the network.
            mTasksLocalDataSource.getTasks(new LoadTasksCallback() {
                @Override
                public void onTasksLoaded(List<Task> tasks) {
                    //将加载内容刷新到内存
                    refreshCache(tasks);
                    callback.onTasksLoaded(new ArrayList<>(mCachedTasks.values()));
                }

                @Override
                public void onDataNotAvailable() {
                    getTasksFromRemoteDataSource(callback);
                }
            });
        }
    }

    private void getTasksFromRemoteDataSource(@NonNull final LoadTasksCallback callback) {
        mTasksRemoteDataSource.getTasks(new LoadTasksCallback() {
            @Override
            public void onTasksLoaded(List<Task> tasks) {
                refreshCache(tasks);
                refreshLocalDataSource(tasks);
                callback.onTasksLoaded(new ArrayList<>(mCachedTasks.values()));
            }

            @Override
            public void onDataNotAvailable() {
                callback.onDataNotAvailable();
            }
        });
    }
}

```
此类将请求（网络、本地）方式进行了封装，使用者无需关心从哪里如何请求的，只需要实例化该类并调用相应model方法即可，可自定义标记本地缓存是否为垃圾；这就是任务管理类实际的model类，任务管理的presenter可以与其交互；

- **fragment对view的实现**
```
public class TasksFragment extends Fragment implements TasksContract.View {

    public static TasksFragment newInstance() {
        return new TasksFragment();
    }
    
    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mListAdapter = new TasksAdapter(new ArrayList<Task>(0), mItemListener);
    }
    
    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        swipeRefreshLayout.setOnRefreshListener(new SwipeRefreshLayout.OnRefreshListener() {
            @Override
            public void onRefresh() {
                //因为用户的下拉手势，fragment去调用了present的加载更多处理，
                //而不是直接去网络或本地加载，因此当需要修改加载逻辑时，只需要修改presenter，实现解耦
                mPresenter.loadTasks(false);
            }
        });                             
    }
    
    //实现BaseView接口，让fragment持有presenter的引用
    @Override
    public void setPresenter(@NonNull TasksContract.Presenter presenter) {
        mPresenter = checkNotNull(presenter);
    }
    
    //展示task列表
    @Override
    public void showTasks(List<Task> tasks) {
        mListAdapter.replaceData(tasks);
        mTasksView.setVisibility(View.VISIBLE);
        mNoTasksView.setVisibility(View.GONE);
    }
}
```
fragment实现了TasksContract.View的接口，因此持有TasksContract.View类型引用的presenter可以调用fragment中的方法进行ui刷新，并且fragment遵循接口内容，presenter也就知道有哪些方法可以调用。fragment中的setPresenter是在presenter初始化时调用，present会将自己的引用传递给fragment，详细可以见下方presenter的构造方法。

- **Presenter的具体实现类**
```
public class TasksPresenter implements TasksContract.Presenter {
    //model
    private final TasksRepository mTasksRepository;

    //view
    private final TasksContract.View mTasksView;
    
    public TasksPresenter(@NonNull TasksRepository tasksRepository, @NonNull TasksContract.View tasksView) {
        //初始化model和view
        mTasksRepository = checkNotNull(tasksRepository, "tasksRepository cannot be null");
        mTasksView = checkNotNull(tasksView, "tasksView cannot be null!");
        //将presenter引用传递给fragment
        mTasksView.setPresenter(this);
    }

    /*实现presenter接口，在其中添加加载任务的逻辑*/
    @Override
    public void loadTasks(boolean forceUpdate) {
        // Simplification for sample: a network reload will be forced on first load.
        loadTasks(forceUpdate || mFirstLoad, true);
        mFirstLoad = false;
    }

    /**
     * @param forceUpdate 是否需要废弃本地缓存的数据
     * @param showLoadingUI 是否需要刷新ui
     */
    private void loadTasks(boolean forceUpdate, final boolean showLoadingUI) {
        if (showLoadingUI) {
            mTasksView.setLoadingIndicator(true);
        }
        if (forceUpdate) {
            //通知model将本缓存清除
            mTasksRepository.refreshTasks();
        }

        // The network request might be handled in a different thread so make sure Espresso knows
        // that the app is busy until the response is handled.
        //表示有操作，并且这些操作中不可以访问ui
        EspressoIdlingResource.increment();
        mTasksRepository.getTasks(new TasksDataSource.LoadTasksCallback() {
            @Override
            public void onTasksLoaded(List<Task> tasks) {
                List<Task> tasksToShow = new ArrayList<Task>();

                //一项任务加载完成，通知EspressoIdlingResource减少一项任务
                if (!EspressoIdlingResource.getIdlingResource().isIdleNow()) {
                    EspressoIdlingResource.decrement(); // Set app as idle.
                }
                //展示逻辑：根据初始选择的类别决筛选哪些task
                for (Task task : tasks) {
                    switch (mCurrentFiltering) {
                        case ALL_TASKS:
                            tasksToShow.add(task);
                            break;
                        case ACTIVE_TASKS:
                            if (task.isActive()) {
                                tasksToShow.add(task);
                            }
                            break;
                        case COMPLETED_TASKS:
                            if (task.isCompleted()) {
                                tasksToShow.add(task);
                            }
                            break;
                        default:
                            tasksToShow.add(task);
                            break;
                    }
                }
                //view已经不存在，不需要刷新ui
                if (!mTasksView.isActive()) {
                    return;
                }
                if (showLoadingUI) {
                    //隐藏加载镇
                    mTasksView.setLoadingIndicator(false);
                }

                processTasks(tasksToShow);
            }

            @Override
            public void onDataNotAvailable() {
                // 必须判断是否fragment绑定在activity上才可进行ui交互
                if (!mTasksView.isActive()) {
                    return;
                }
                mTasksView.showLoadingTasksError();
            }
        });
    }

}
```
presenter同时持有repository(model)和view引用，这也是它作为桥梁的条件，view可以调用presenter的方法，同时presenter可以调用fragment的方法进行相应，presenter可以调用repository的方法进行数据操作请求，同时传递一个callback给repository，当repository对数据操作有结果后可以调用callback将结果返回给presenter，让presenter自己实现接下来的逻辑处理。



- **Activity中初始化fragment(view)和presenter，并且将两者绑定**
```
public class TasksActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        TasksFragment tasksFragment =
                (TasksFragment) getSupportFragmentManager().findFragmentById(R.id.contentFrame);
        if (tasksFragment == null) {
            // Create the fragment
            tasksFragment = TasksFragment.newInstance();
            //AcivityUtil是将fragment添加到activity的某一个布局控件上
            ActivityUtils.addFragmentToActivity(
                    getSupportFragmentManager(), tasksFragment, R.id.contentFrame);
        }
        // 初始化presenter,让其持有model(参数一)和view(参数二)的引用
        mTasksPresenter = new TasksPresenter(
                Injection.provideTasksRepository(getApplicationContext()), tasksFragment);
    }
    
}
```
从中可以看到presenter在初始化时传入了TasksRepository(model)和fragment参数(view)，实现了其引用v和m，成为桥梁。