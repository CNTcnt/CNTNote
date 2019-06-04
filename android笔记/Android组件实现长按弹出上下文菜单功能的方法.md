[TOC]



# Android组件实现长按弹出上下文菜单功能的方法

* 实现简单页面下的长按弹出功能

第一步：在程序合适位置给一个控件注册上下文菜单

组件可以是按钮，文本框，还可以是列表条目，下以listView列表为例

~~~java
ListView contentList=(ListView) findViewById(R.id.blackname_manager_listV);
contentList.setAdapter(mListAdapter);
//先注册
registerForContextMenu(contentList);
~~~



第二步：在activity中复写onCreateContextMenu方法，并添加菜单项目

~~~java
public void onCreateContextMenu(ContextMenu menu, View v,ContextMenuInfo menuInfo) {
  super.onCreateContextMenu(menu, v, menuInfo);
  //设置列表标题
  menu.setHeaderTitle("选择");
  //参数：菜单组的id，菜单子项的id，排列方式，菜单内容
  menu.add(0, MENU_UPDATE, 0, "修改信息");
  menu.add(0, MENU_ADD, 0, "删除记录");
}
//也可以在xml定义好布局然后引用进来即可，新建res/menu/这里放菜单文件
@Override
public void onCreateContextMenu(ContextMenu menu, View v, ContextMenu.ContextMenuInfo menuInfo) {
        super.onCreateContextMenu(menu, v, menuInfo);
      //将文件加载进来即可
        MenuInflater inflater=getMenuInflater();
        inflater.inflate(R.menu.context_menu,menu);
}
~~~

第三步：在activity中复写onContextItemSelected方法，处理菜单条目事件

~~~java
public boolean onContextItemSelected(MenuItem item) {
  //获取上下文菜单适配器
    AdapterContextMenuInfo cmi=(AdapterContextMenuInfo)item.getMenuInfo();
  //获取被选择的菜单位置
    int posMenu=cmi.position;
  //将菜单项与列表视图的条目相关联
    items=(BlackNumber) mListAdapter.getItem(posMenu);
    switch(item.getItemId()){
    case MENU_UPDATE://执行该菜单条目的业务逻辑
      break;
    case MENU_ADD://执行该菜单条目的业务逻辑
      break;
    }
    return super.onContextItemSelected(item);
}
~~~



