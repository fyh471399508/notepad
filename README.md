# NotePad
This is an AndroidStudio rebuild of google SDK sample NotePad<br>
## 扩展的功能：
NoteList中显示条目增加时间戳显示<br>
添加笔记查询功能（根据标题查询）<br>
UI美化<br>
更改记事本的背景<br>

<br>
## 扩展的功能解析：
#### NoteList中显示条目增加时间戳显示：
在`noteslist_item.xml`中添加一个用于显示时间戳的TextView<br>
```
<TextView
            android:id="@+id/text1_time"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:paddingLeft="5dip"
            android:textColor="#000000"
            />
```
已知数据库定义中已存在修改时间和创建时间的项：<br>
```
db.execSQL("CREATE TABLE " + NotePad.Notes.TABLE_NAME + " ("
                   + NotePad.Notes._ID + " INTEGER PRIMARY KEY,"
                   + NotePad.Notes.COLUMN_NAME_TITLE + " TEXT,"
                   + NotePad.Notes.COLUMN_NAME_NOTE + " TEXT,"
                   + NotePad.Notes.COLUMN_NAME_CREATE_DATE + " INTEGER,"
                   + NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE + " INTEGER,"
                   + ");");
```
使用修改时间为时间戳显示，在NoteList中找到PROJECTION添加一个元素<br>
```
private static final String[] PROJECTION = new String[] {
            NotePad.Notes._ID, // 0
            NotePad.Notes.COLUMN_NAME_TITLE, // 1
            NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE,//添加该行
    };
```
修改dataColumns,viewIDs添加时间，然后用Cursor从数据库中取出数据<br>
```
private String[] dataColumns = { NotePad.Notes.COLUMN_NAME_TITLE ,
                    NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE
} ;
// The view IDs that will display the cursor columns, initialized to the TextView in
private int[] viewIDs = { android.R.id.text1,R.id.text1_time };
```
修改时间的显示方式:在NotePadProvider中显示的`insert方法`和NoteEditor中的`updateNote`方法中修改`创建时间`和`修改时间`的格式<br>
```
Long nowTime=Long.valueOf(System.currentTimeMillis());
Date date=new Date(nowTime);
SimpleDateFormat format=new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
String time=format.format(date);
```
```
values.put(NotePad.Notes.COLUMN_NAME_CREATE_DATE, time);
```
结果如下：<br>
![image](https://github.com/he476/NotePad/blob/master/images/1.jpg)

#### 添加笔记查询功能（根据标题查询）:
在`list_options_menu.xml`中添加一个用于搜索的<item><br>
```
<item
        android:id="@+id/menu_search"
        android:title="Search"
        android:icon="@android:drawable/ic_search_category_default"
        android:showAsAction="always"
```
在`onOptionsItemSelected(MenuItem item)`中switch添加<br>
```
case R.id.menu_search:
Intent intent=new Intent(this,NoteSearch.class);
startActivity(intent);
return true;
```
新建一个名为NoteSearch的activity,由于搜索出来的也是笔记列表，所以可以模仿NotesList的activity继承ListActivity。在安卓中有个用于搜索控件：`SearchView`，可以把SearchView跟ListView相结合，动态地显示搜索结果。先布局搜索页面，在layout中新建布局文件`note_search_list.xml`：<br>
```
      <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">
        <SearchView
            android:id="@+id/search_view"

            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:iconifiedByDefault="false"
            android:queryHint="输入搜索内容...">
        </SearchView>
        <ListView
            android:id="@android:id/list"
            android:layout_width="match_parent"
            android:layout_height="wrap_content">

        </ListView>
    </LinearLayout>
```

NoteSearch的activity:<br>

```
public class NoteSearch extends ListActivity implements SearchView.OnQueryTextListener {

    private static final String[] PROJECTION=new String[]{
            NotePad.Notes._ID,
            NotePad.Notes.COLUMN_NAME_TITLE,
            NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE,
            NotePad.Notes.COLUMN_NAME_BACK_COLOR
    };
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.note_search);
        Intent intent =getIntent();
        if(intent.getData()==null){
            intent.setData(NotePad.Notes.CONTENT_URI);
        }
        SearchView searchView=(SearchView)findViewById(R.id.search_view);
        searchView.setOnQueryTextListener(NoteSearch.this);
    }

    @Override
    public boolean onQueryTextSubmit(String s) {
        return false;
    }

    @Override
    public boolean onQueryTextChange(String s) {
        String selection=NotePad.Notes.COLUMN_NAME_TITLE+" LIKE ?";
        String[] selectionArgs={"%"+s+"%"};
        Cursor cursor=managedQuery(
                getIntent().getData(),
                PROJECTION,
                selection,
                selectionArgs,
                NotePad.Notes.DEFAULT_SORT_ORDER
        );
        String[]dataColumns={
                NotePad.Notes.COLUMN_NAME_TITLE,
                NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE,
//                NotePad.Notes.COLUMN_NAME_BACK_COLORACK_COLOR
        };
        int[]viewIDs={android.R.id.text1,R.id.text1_time};
        MyCursorAdapter adapter
                = new MyCursorAdapter(
                this,                             // The Context for the ListView
                R.layout.noteslist_item,          // Points to the XML for a list item
                cursor,                           // The cursor to get items from
                dataColumns,
                viewIDs
        );
        setListAdapter(adapter);
        return true;
    }

    @Override
    protected void onListItemClick(ListView l, View v, int position, long id) {

        // Constructs a new URI from the incoming URI and the row ID
        Uri uri = ContentUris.withAppendedId(getIntent().getData(), id);

        // Gets the action from the incoming Intent
        String action = getIntent().getAction();

        // Handles requests for note data
        if (Intent.ACTION_PICK.equals(action) || Intent.ACTION_GET_CONTENT.equals(action)) {

            // Sets the result to return to the component that called this Activity. The
            // result contains the new URI
            setResult(RESULT_OK, new Intent().setData(uri));
        } else {

            // Sends out an Intent to start an Activity that can handle ACTION_EDIT. The
            // Intent's data is the note ID URI. The effect is to call NoteEdit.
            startActivity(new Intent(Intent.ACTION_EDIT, uri));
        }
    }
}
```
效果如下：<br>
![image](https://github.com/he476/NotePad/blob/master/images/2.png)

#### UI美化
更改原有的应用主题<br>
```
android:theme="@android:style/Theme.DeviceDefault.Light"
```
该项目还使用了开源的TapBarMenu用于编辑页面的菜单显示：
1).Add the dependency to your build.gradle:
```
compile ‘com.github.michaldrabik:tapbarmenu:1.0.5’
```
在`note_editor.xml`中添加TapBarMenu的标签：<br>
```
<FrameLayout
        android:id="@+id/fragment_container"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        >

        <com.michaldrabik.tapbarmenulib.TapBarMenu
            android:id="@+id/tapBarMenu"
            android:layout_width="match_parent"
            android:layout_height="56dp"
            android:layout_gravity="bottom"
            android:layout_marginBottom="24dp"
            android:onClick="onMenuButtonClick"
            app:tbm_backgroundColor="@color/red"
            app:tbm_menuAnchor="bottom"
            >

            <ImageView
                android:id="@+id/item1"
                android:layout_width="0dp"
                android:layout_height="match_parent"
                android:layout_weight="1"
                android:paddingTop="10dp"
                android:paddingBottom="10dp"
                android:onClick="onMenuItemClick"
                android:src="@drawable/ic_menu_save"
                tools:visibility="visible"
                />

            <ImageView
                android:id="@+id/item2"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:paddingTop="10dp"
                android:paddingBottom="10dp"
                android:onClick="onMenuItemClick"
                android:src="@drawable/ic_menu_delete"
                />

            <Space
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                />

            <ImageView
                android:id="@+id/item3"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:paddingTop="10dp"
                android:paddingBottom="10dp"
                android:onClick="onMenuItemClick"
                android:src="@drawable/ic_menu_changecolor"
                />

            <ImageView
                android:id="@+id/item4"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:paddingTop="10dp"
                android:paddingBottom="10dp"
                android:onClick="onMenuItemClick"
                android:src="@drawable/ic_menu_share"
                />

        </com.michaldrabik.tapbarmenulib.TapBarMenu>
    </FrameLayout>
```
在NoteEditor中添加相应的相应函数：
```
public void onMenuButtonClick(View view) {
        tapBarMenu.toggle();
    }
    public void onMenuItemClick(View view) {
        tapBarMenu.close();
        switch (view.getId()) {
            case R.id.item1:
                String text = mText.getText().toString();
                updateNote(text, null);
                finish();
                break;
            case R.id.item2:
                deleteNote();
                finish();
                break;
            case R.id.item3:
                changeColor();
                break;
            case R.id.item4:
                sendTo(this,mText.getText().toString());
                break;
        }
        tapBarMenu.toggle();
    }
```
效果如下：<br>
![image](https://github.com/he476/NotePad/blob/master/images/3.png)<br>
![image](https://github.com/he476/NotePad/blob/master/images/4.png)<br>
![image](https://github.com/he476/NotePad/blob/master/images/5.png)<br>
![image](https://github.com/he476/NotePad/blob/master/images/6.png)<br><br>


#### 更改记事本的背景
在NotePad契约类添加背景色的属性：
```
public static final String COLUMN_NAME_BACK_COLOR = "color";
```
添加不同颜色的属性：
```
public static final int DEFAULT_COLOR = 0; //白
        public static final int YELLOW_COLOR = 1; //黄
        public static final int BLUE_COLOR = 2; //蓝
        public static final int GREEN_COLOR = 3; //绿
        public static final int RED_COLOR = 4; //红
```
NotePadProvider中创建数据库表地方添加颜色的字段：
```
db.execSQL("CREATE TABLE " + NotePad.Notes.TABLE_NAME + " ("
                   + NotePad.Notes._ID + " INTEGER PRIMARY KEY,"
                   + NotePad.Notes.COLUMN_NAME_TITLE + " TEXT,"
                   + NotePad.Notes.COLUMN_NAME_NOTE + " TEXT,"
                   + NotePad.Notes.COLUMN_NAME_CREATE_DATE + " INTEGER,"
                   + NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE + " INTEGER,"
                   + NotePad.Notes.COLUMN_NAME_BACK_COLOR + " INTEGER"//添加该行
                   + ");");
```
`static{}`中中添加<br>
```
sNotesProjectionMap.put(
                NotePad.Notes.COLUMN_NAME_BACK_COLOR,
                NotePad.Notes.COLUMN_NAME_BACK_COLOR);
```
`insert()`中添加<br>
```
if(values.containsKey(NotePad.Notes.COLUMN_NAME_BACK_COLOR)==false){
            values.put(NotePad.Notes.COLUMN_NAME_BACK_COLOR,NotePad.Notes.DEFAULT_COLOR);
        }
```
新建一个Adapter--MyCursorAdapter继承`SimpleCursorAdapter`，用于填充颜色,修改NotesList,和NoteEditor中的Adapter,改为MyCursorAdapter：<br>
```
public class MyCursorAdapter extends SimpleCursorAdapter {
    public MyCursorAdapter(Context context, int layout, Cursor c, String[] from, int[] to) {
        super(context, layout, c, from, to);
    }
    @Override
    public void bindView(View view,Context context,Cursor cursor){
        super.bindView(view,context,cursor);
        int x=cursor.getInt(cursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_BACK_COLOR));

        switch (x){
            case NotePad.Notes.DEFAULT_COLOR:
                view.setBackgroundColor(Color.WHITE);
                break;
            case NotePad.Notes.BLUE_COLOR:
                view.setBackgroundColor(Color.rgb(186,216,230));
                break;
            case NotePad.Notes.GREEN_COLOR:
                view.setBackgroundColor(Color.rgb(195,249,161));
                break;
            case NotePad.Notes.RED_COLOR:
                view.setBackgroundColor(Color.rgb(247,191,196));
                break;
            case NotePad.Notes.YELLOW_COLOR:
                view.setBackgroundColor(Color.rgb(255,253,193));
                break;
            default:
                view.setBackgroundColor(Color.WHITE);
        }
    }
}

```
效果如下：<br>
![image](https://github.com/he476/NotePad/blob/master/images/7.png)<br>
![image](https://github.com/he476/NotePad/blob/master/images/8.png)<br>
![image](https://github.com/he476/NotePad/blob/master/images/9.png)<br>






