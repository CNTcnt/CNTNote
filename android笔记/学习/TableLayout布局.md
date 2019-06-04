### TableLayout布局

---

#### Tablelayout简介 

1. Tablelayout类以行和列的形式对控件进行管理，每一行为一个TableRow对象，或一个View控件。              

   * 当为对象时，可在TableRow下添加子控件，**默认情况下，每个子控件占据一列。**         


   * 为View时，该View将独占一行。

2. TableLayout行列数的确定 

   * TableLayout的行数由开发人员直接指定，即有多少个TableRow对象（或View控件），就有多少行。**TableLayout的列数等于含有最多子控件的TableRow的列数。**如第一TableRow含2个子控件，第二个TableRow含3个，第三个TableRow含4个，那么该TableLayout的列数为4.

3. TableLayout可设置的属性包括全局属性及单元格属性。

   * 全局属性也即列属性，有以下3个参数：
   * * android:stretchColumns    设置可伸展的列。该列可以向**行方向**伸展，最多可占据一整行。

       stretch：伸展；[streth]

     * android:shrinkColumns     设置可收缩的列。当该列子控件的内容太多，已经挤满所在行，那么该子控件的内容将往**列方向**显示。

       shrink:收缩；[srik]                  Columns:列；[calem]

     * android:collapseColumns 设置要隐藏的列。

       collapse:折叠；[kə'læps]

       示例：android:stretchColumns="0"           第0列可伸展

   ​                   android:shrinkColumns="1,2"         第1,2列皆可收缩

   ​                   android:collapseColumns="*"         隐藏所有行

4. 单元格属性，有以下2个参数：

   android:layout_column    指定该单元格在第几列显示
   android:layout_span        指定该单元格占据的列数（未指定时，为1）

   示例：android:layout_column="1"       该控件显示在第1列
   ​           android:layout_span="2"             该控件占据2列

   ​说明：一个控件也可以同时具备这两个特性。​