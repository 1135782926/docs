# 表格基本使用


## 简单示例
`Dcat\Admin\Grid`类用于生成基于数据模型的表格，先来个例子，数据库中有`movies`表

```sql
CREATE TABLE `movies` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `title` varchar(255) COLLATE utf8_unicode_ci NOT NULL,
  `director` int(10) unsigned NOT NULL,
  `describe` varchar(255) COLLATE utf8_unicode_ci NOT NULL,
  `rate` tinyint unsigned NOT NULL,
  `released` enum(0, 1),
  `release_at` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `created_at` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `updated_at` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;

```

对应的数据模型为`App\Models\Movie`，对应的数据仓库为`App\Admin\Repositories\Movie`，数据仓库代码如下：

> {tip} 如果你的数据来自`MySQL`，则`数据仓库`不是必须的，你也可以直接使用`Model`。

```php
<?php

namespace App\Admin\Repositories;

use Dcat\Admin\Repositories\EloquentRepository;
use App\Models\Movie as MovieModel;

class Movie extends EloquentRepository
{
    protected $eloquentClass = MovieModel::class;
    
    /**
     * 设置表格查询的字段，默认查询所有字段
     * 
     * @return array
     */
    public function getGridColumns(){
        return ['id', 'title', 'director', 'rate', ...];
    }
}
```

下面的代码可以生成表`movies`的数据表格：

```php
<?php

namespace App\Admin\Controllers;

use App\Admin\Repositories\Movie;
use Dcat\Admin\Grid;
use Dcat\Admin\Controllers\AdminController;

class MovieController extends AdminController
{
    protected function grid()
    {
        return Grid::make(new Movie(), function (Grid $grid) {
            // 第一列显示id字段，并将这一列设置为可排序列
            $grid->column('id', 'ID')->sortable();
            
            // 第二列显示title字段，由于title字段名和Grid对象的title方法冲突，所以用Grid的column()方法代替
            $grid->column('title');
            
            // 第三列显示director字段，通过display($callback)方法设置这一列的显示内容为users表中对应的用户名
            $grid->column('director')->display(function($userId) {
                return User::find($userId)->name;
            });
            
            // 第四列显示为describe字段
            $grid->column('describe');
            
            // 第五列显示为rate字段
            $grid->column('rate');
            
            // 第六列显示released字段，通过display($callback)方法来格式化显示输出
            $grid->column('released', '上映?')->display(function ($released) {
                return $released ? '是' : '否';
            });
            
            // 下面为三个时间字段的列显示
            $grid->column('release_at');
            $grid->column('created_at');
            $grid->column('updated_at');
            
            // filter($callback)方法用来设置表格的简单搜索框
            $grid->filter(function ($filter) {
                // 设置created_at字段的范围查询
                $filter->between('created_at', 'Created Time')->datetime();
            });
        });
    }
}
```

## 表格显示模式

### table_collapse

> {tip} Since `v1.6.5`

在这个版本开始，默认的表格布局将会采用 `table_collapse` 模式，效果如下

<a href="https://cdn.learnku.com/uploads/images/202007/24/38389/4bCfBdtvq5.png!large" target="_blank">
    <img class="img" src="https://cdn.learnku.com/uploads/images/202007/24/38389/4bCfBdtvq5.png!large" />
</a>
<a href="https://cdn.learnku.com/uploads/images/202007/24/38389/35KJXfVXib.png!large" target="_blank">
    <img class="img" src="https://cdn.learnku.com/uploads/images/202007/24/38389/35KJXfVXib.png!large" />
</a>    

如果想要切换回旧版本的表格布局样式，可以在 `app/Admin/bootstrap.php`中加上

```php
Grid::resolving(function (Grid $grid) {
    $grid->tableCollapse(false);
});
```

### 边框模式

通过`withBorder`方式可以让表格显示边框

> {tip} Since `v1.2.0`

```php
$grid->withBorder();
```

效果
<a href="http://103.39.211.179:8080/admin/reports" target="_blank">
    <img style="box-shadow:0 1px 6px 1px rgba(0, 0, 0, 0.12)" width="100%" src="{{public}}/assets/img/screenshots/grid-combination.png">
</a>


> {tip} Since `v1.4.5`
 
禁用边框模式
```php
$grid->withBorder(false);
```

## 基本使用方法

### 添加列

```php
// 添加单列
$grid->column('username', '用户名');

// 添加多列
$grid->columns('email', 'username' ...);
```

### 修改来源数据
```php
$grid->model()->where('id', '>', 100);

$grid->model()->orderBy('id', 'desc');

// 回收站数据
$grid->model()->onlyTrashed();

...
```

同时也可以使用以下方式

```php
protected function grid()
{
    return Grid::make(Model::with('...')->where(...), function (Grid $grid) {
        ...
    });
}
```

其它查询方法可以参考`eloquent`的查询方法.

### 修改显示输出


```php
$grid->column('text')->display(function($text) {
    return str_limit($text, 30, '...');
});

// 允许混合使用多个“display”方法
$grid->column('name')->display(function ($name) {
     return "<b>$name</b>";
 })->display(function ($name) {
    return "<span class='label'>$name</span>";
});

$grid->column('email')->display(function ($email) {
    return "mailto:$email";
});

// 可以直接写字符串
$grid->column('username')->display('...');

// 添加不存在的字段
$grid->column('column_not_in_table')->display(function () {
    return 'blablabla....'.$this->id;
});
```

### 获取当前行数据

`display()`方法接收的匿名函数绑定了当前行的数据对象，可以在里面调用当前行的其它字段数据

```php
$grid->column('first_name');
$grid->column('last_name');

// 不存的字段列
$grid->column('full_name')->display(function () {
    return $this->first_name.' '.$this->last_name;
});
```


<a name="outline"></a>
### 设置工具栏按钮样式

> {tip} Since `v1.4.5`

从这个`v1.4.5`工具栏按钮默认显示`outline`模式，效果如下


用法
```php
$grid->toolsWithOutline();

// 禁止
$grid->toolsWithOutline(false);
```

效果
<a href="{{public}}/assets/img/screenshots/outline.png" target="_blank">
    <img style="box-shadow:0 1px 6px 1px rgba(0, 0, 0, 0.12)" width="100%" src="{{public}}/assets/img/screenshots/outline.png">
</a>

禁用`outline`后的效果

<a href="{{public}}/assets/img/screenshots/n-outline.png" target="_blank">
    <img style="box-shadow:0 1px 6px 1px rgba(0, 0, 0, 0.12)" width="100%" src="{{public}}/assets/img/screenshots/n-outline.png">
</a>

如果你希望某个按钮不使用`outline`模式，可以在按钮的`class`属性中加上`disable-outline`
```php
$grid->tools('<a class="btn btn-primary disable-outline">测试按钮</a>');
```


### 设置表格外层容器
```php
 // 更改表格外层容器
$grid->wrap(function (Renderable $view) {
    $tab = Tab::make();
    
    $tab->add('示例', $view);
    $tab->add('代码', $this->code(), true);

    return $tab;
});
```

### 设置创建按钮

此功能默认开启
```php
// 禁用
$grid->disableCreateButton();
// 显示
$grid->showCreateButton();
```

#### 开启弹窗创建表单

此功能默认不开启

```php
$grid->enableDialogCreate();

// 设置弹窗宽高，默认值为 '700px', '670px'
$grid->setDialogFormDimensions('50%', '50%');
```


### 设置查询过滤器

此功能默认开启

```php
// 禁用
$grid->disableFilter();
// 显示
$grid->showFilter();

// 禁用过滤器按钮
$grid->disableFilterButton();
// 显示过滤器按钮
$grid->showFilterButton();
```


### 设置行选择器
```php
// 禁用
$grid->disableRowSelector();
// 显示
$grid->showRowSelector();
```

#### 设置选择中行的标题字段key
如不设置，默认取 `name`、 `title`、 `username`中的一个。
```php
$grid->full_name;
$grid->age;

...

$grid->rowSelector()->titleColumn('full_name');
```

#### 设置checkbox选择框颜色
默认 `primary`，支持：`default`、 `primary`、 `success`、 `info`、 `danger`、 `purple`、 `inverse`。
```php
$grid->rowSelector()->style('success');
```

#### 点击当前行任意位置选中
此功能默认不开启。
```php
$grid->rowSelector()->click();
```

#### 设置选中行的背景颜色
```php
use Dcat\Admin\Admin;

$grid->rowSelector()->background(Admin::color()->dark20());
```

#### 设置checkbox选择框形状
默认圆形。
```php
// 设置为正方形
$grid->rowSelector()->circle(false);
```

### 设置行操作按钮
```php
// 禁用
$grid->disableActions();
// 显示
$grid->showActions();

// 禁用详情按钮
$grid->disableViewButton();
// 显示详情按钮
$grid->showViewButton();

// 禁用编辑按钮
$grid->disableEditButton();
// 显示编辑按钮
$grid->showEditButton();

// 禁用快捷编辑按钮
$grid->disableQuickEditButton();
// 显示快捷编辑按钮
$grid->showQuickEditButton();

// 设置弹窗宽高，默认值为 '700px', '670px'
$grid->setDialogFormDimensions('50%', '50%');


// 禁用删除按钮
$grid->disableDeleteButton();
// 显示删除按钮
$grid->showDeleteButton();

```

### 设置批量操作按钮
```php
// 禁用
$grid->disableBatchActions();
// 显示
$grid->showBatchActions();

// 禁用批量删除按钮
$grid->disableBatchDelete();
// 显示批量删除按钮
$grid->showBatchDelete();
```

### 设置工具栏
```php
// 禁用
$grid->disableToolbar();
// 显示
$grid->showToolbar();
```

### 设置刷新按钮
```php
// 禁用
$grid->disableRefreshButton();
// 显示
$grid->showRefreshButton();
```

### 设置分页功能
```php
// 禁用
$grid->disablePagination();
// 显示
$grid->showPagination();
```

#### 设置每页显示行数

```php
// 默认为每页20条
$grid->paginate(15);
```

#### 设置分页选择器选项
```php
$grid->perPages([10, 20, 30, 40, 50]);

// 禁用分页选择器
$grid->disablePerPages();
```

### addTableClass

> {tip} Since `v1.5.2`

通过`addTableClass`可以给表格的`table`添加`css`样式

```php
$grid->addTableClass(['class1', 'class2']);
```

## 关联模型

参考[表格关联关系](model-grid-relationship.md)

