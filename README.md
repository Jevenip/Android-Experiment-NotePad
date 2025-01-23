# 期中实验 NotePad

## 实验环境
- 开发环境：Android 7.0
- 软件工具：Android Studio Koala 2024

## 实验目的

基于NotePad 应用实现功能扩展。

## 实验内容

1. NoteList 界面中笔记条目增加时间戳显示。
2. 增加加笔记查询功能，根据标题或内容查询。
3. 增加记事本的偏好设置功能。
4. 增加笔记分类。

## 实验步骤

### **1. 增加时间戳显示（NoteList 界面）**

#### **修改数据库 (NotePadProvider.java)**

```java
@Override
public void onCreate(SQLiteDatabase db) {
    db.execSQL("CREATE TABLE " + TABLE_NAME + " ("
        + NotePad.Notes._ID + " INTEGER PRIMARY KEY,"
        + NotePad.Notes.COLUMN_NAME_TITLE + " TEXT,"
        + NotePad.Notes.COLUMN_NAME_NOTE + " TEXT,"
        + NotePad.Notes.COLUMN_NAME_CREATED_DATE + " INTEGER,"
        + NotePad.Notes.COLUMN_NAME_MODIFIED_DATE + " INTEGER,"
        + NotePad.Notes.COLUMN_NAME_CATEGORY + " TEXT);"
    );
}
```

#### **修改列表项布局 (res/layout/note_list_item.xml)**

```xml
<TextView
    android:id="@+id/text1"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:textSize="18sp" />

<TextView
    android:id="@+id/timestamp"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:textSize="12sp"
    android:textColor="#666"/>
```

#### **修改 NoteList 适配器 (NoteList.java)**

```java
private SimpleCursorAdapter adapter;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.note_list);

    String[] fromColumns = {NotePad.Notes.COLUMN_NAME_TITLE, NotePad.Notes.COLUMN_NAME_MODIFIED_DATE};
    int[] toViews = {R.id.text1, R.id.timestamp};

    adapter = new SimpleCursorAdapter(
        this,
        R.layout.note_list_item,
        null,
        fromColumns,
        toViews,
        0
    );

    adapter.setViewBinder((view, cursor, columnIndex) -> {
        if (view.getId() == R.id.timestamp) {
            long timestamp = cursor.getLong(columnIndex);
            String formattedDate = new SimpleDateFormat("yyyy-MM-dd HH:mm", Locale.getDefault()).format(new Date(timestamp));
            ((TextView) view).setText(formattedDate);
            return true;
        }
        return false;
    });

    setListAdapter(adapter);
    getLoaderManager().initLoader(0, null, this);
}
```

------

### **2. 添加笔记查询功能**

#### **添加搜索栏 (res/layout/note_list.xml)**

```xml
<SearchView
    android:id="@+id/searchView"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"/>
```

#### **修改 NoteList.java**

```java
SearchView searchView = findViewById(R.id.searchView);
searchView.setOnQueryTextListener(new SearchView.OnQueryTextListener() {
    @Override
    public boolean onQueryTextSubmit(String query) {
        searchNotes(query);
        return true;
    }

    @Override
    public boolean onQueryTextChange(String newText) {
        searchNotes(newText);
        return true;
    }
});

private void searchNotes(String query) {
    Cursor cursor = getContentResolver().query(
        NotePad.Notes.CONTENT_URI,
        null,
        NotePad.Notes.COLUMN_NAME_TITLE + " LIKE ? OR " + NotePad.Notes.COLUMN_NAME_NOTE + " LIKE ?",
        new String[]{"%" + query + "%", "%" + query + "%"},
        null
    );
    adapter.swapCursor(cursor);
}
```

------

### **3. 记事本的偏好设置功能**

#### **创建偏好设置界面 (res/xml/preferences.xml)**

```xml
<PreferenceScreen xmlns:android="http://schemas.android.com/apk/res/android">
    <ListPreference
        android:key="font_size"
        android:title="Font Size"
        android:entries="@array/font_size_labels"
        android:entryValues="@array/font_size_values"
        android:defaultValue="14" />

    <SwitchPreferenceCompat
        android:key="dark_mode"
        android:title="Dark Mode"
        android:defaultValue="false"/>
</PreferenceScreen>
```

#### **定义数组 (res/values/arrays.xml)**

```xml
<resources>
    <string-array name="font_size_labels">
        <item>Small</item>
        <item>Medium</item>
        <item>Large</item>
    </string-array>

    <string-array name="font_size_values">
        <item>12</item>
        <item>14</item>
        <item>18</item>
    </string-array>
</resources>
```

#### **创建偏好设置界面 (SettingsActivity.java)**

```java
public class SettingsActivity extends PreferenceActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        getFragmentManager().beginTransaction()
            .replace(android.R.id.content, new SettingsFragment())
            .commit();
    }

    public static class SettingsFragment extends PreferenceFragment {
        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            addPreferencesFromResource(R.xml.preferences);
        }
    }
}
```

#### **在 NoteList 中应用设置**

```java
SharedPreferences prefs = PreferenceManager.getDefaultSharedPreferences(this);
String fontSize = prefs.getString("font_size", "14");
listView.setTextSize(Float.parseFloat(fontSize));
```

------

### **4. 笔记分类功能**

#### **修改数据库**

```java
private static final String SQL_ADD_CATEGORY_COLUMN =
    "ALTER TABLE " + TABLE_NAME + " ADD COLUMN category TEXT DEFAULT 'Uncategorized';";

db.execSQL(SQL_ADD_CATEGORY_COLUMN);
```

#### **添加分类选择到笔记编辑界面 (res/layout/note_edit.xml)**

```xml
<Spinner
    android:id="@+id/category_spinner"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"/>
```

#### **填充分类选项 (NoteEditor.java)**

```java
Spinner categorySpinner = findViewById(R.id.category_spinner);
ArrayAdapter<CharSequence> adapter = ArrayAdapter.createFromResource(this,
    R.array.note_categories, android.R.layout.simple_spinner_item);
adapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
categorySpinner.setAdapter(adapter);
```

#### **保存分类信息 (NoteEditor.java)**

```java
ContentValues values = new ContentValues();
values.put(NotePad.Notes.COLUMN_NAME_CATEGORY, categorySpinner.getSelectedItem().toString());
getContentResolver().update(mUri, values, null, null);
```

#### **按分类筛选 (NoteList.java)**

```java
private void filterByCategory(String category) {
    Cursor cursor = getContentResolver().query(
        NotePad.Notes.CONTENT_URI,
        null,
        NotePad.Notes.COLUMN_NAME_CATEGORY + "=?",
        new String[]{category},
        null
    );
    adapter.swapCursor(cursor);
}
```

## 参考文献

- [NotePad 源码分析](https://blog.csdn.net/llfjfz/article/details/67638499)