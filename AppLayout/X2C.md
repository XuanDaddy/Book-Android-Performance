# 【四】X2C

### 代码写布局

通过Java代码写布局的方式，可以本质解决界面加载慢的性能问题（IO、反射）。

但是不便于开发、可维护性太差。

### X2C介绍

* 保留XML优点，解决其性能问题；
  * 开发人员写XML，加载Java代码；
* APT编译期翻译XML为Java代码；

### X2C使用

1. 引入依赖库

```groovy
annotationProcessor 'com.zhangyue.we:x2c-apt:1.1.2'
implementation 'com.zhangyue.we:x2c-lib:1.0.6'
```

1. 代码实现

```java
@Xml(layouts = "activity_main")
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        X2C.setContentView(this,R.layout.activity_main);
    }
}
```

### X2C问题

1. 部分属性不支持

> - merge标签 ,在编译期间无法确定xml的parent，所以无法支持
> - 系统style,在编译期间只能查到应用的style列表，无法查询系统style，所以只支持应用内style

2. 失去了系统的兼容（AppCompat）

由于它会根据XML直接生成对应的Java代码，所以像TextView并不会转换为AppCompatTextView，失去了兼容性。

**解决办法**：修改X2C源码，在生成组件实例时，如TextView，让其生成AppCompatTextView。