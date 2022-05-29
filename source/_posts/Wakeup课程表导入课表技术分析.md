---
title: Wakeup课程表导入课表技术分析
date: 2020-10-23 22:22:22
tags:
	- Android
categories: 学习
---

以下源码均来自于[wakeup课程表](https://github.com/YZune/WakeupSchedule_Kotlin)

<!--more-->

#### 选择学校

##### 动态添加学校、网址

```kotlin
/*
	SchoolListActivity.kt
*/
schools.apply {
  add(SchoolInfo("★", "如何正确选择教务类型？", "https://support.qq.com/embed/97617/faqs/59901", TYPE_HELP)) //引导网页项
  getImportSchoolBean()?.let {
    it.sortKey = "★"
    add(it)
  }//上次选择的学校
  //SchoolInfo(var sortKey: String, val name: String, val url: String = "", val type: String?)
  add(SchoolInfo("通", "新 URP 系统", "", TYPE_URP_NEW))
  add(SchoolInfo("通", "URP 系统", "", TYPE_URP))
  ...
  add(SchoolInfo("A", "安徽信息工程学院", "http://teach.aiit.edu.cn/xtgl/login_slogin.html", TYPE_ZF_NEW))
  ...
  add(SchoolInfo("Z", "郑州航空工业管理学院", "http://202.196.166.138/", TYPE_ZF))
}
```

##### 列表跳转

点击某一项跳转到显示网页的界面，传递：校名、导入类型、课表id、教务系统url

```kotlin
/*
	SchoolListActivity.kt
*/
showList.addAll(schools)
val adapter = SchoolImportListAdapter(R.layout.item_apply_info, showList)
adapter.setOnItemClickListener { _, _, position ->
	if (showList[position].type == TYPE_HELP) {//列表第一项的帮助引导网页
    Utils.openUrl(this, showList[position].url)
    return@setOnItemClickListener
  }
  if (fromLocal) {//跳转到学校选择页面选择的是从本地html文件导入
    setResult(Activity.RESULT_OK, 
              Intent().apply{
                putExtra("type", showList[position].type) 
              })
    finish()
  } else {//普通学校网站登录
    launch {
      if (showList[position].type == Common.TYPE_MAINTAIN) {//当前不可用
        Toasty.info(this@SchoolListActivity, "处于维护中哦").show()
        return@launch
      }
      getPrefer().edit { //缓存已经选择的学校
        putString(Const.KEY_IMPORT_SCHOOL, gson.toJson(showList[position]))
      }
      val tableId = tableDao.getDefaultTableId()
      startActivityForResult(Intent(this@SchoolListActivity, LoginWebActivity::class.java).apply {
        //跳转LoginWebActivity[],传递校名、导入类型、课表id、教务系统url
        putExtra("school_name", showList[position].name)
        putExtra("import_type", showList[position].type)
        putExtra("tableId", tableId)
        putExtra("url", showList[position].url)
      }, Const.REQUEST_CODE_IMPORT)
    }
  }
}
```

##### 选择显示页面
显示网页的Activity根据ImportViewModel.imortType判断要加载什么形式的Fragment

```kotlin
/*
	LoginWebActivity.kt
*/
intent.extras?.getString("import_type")?.let {
  viewModel.importType = it
}
intent.extras?.getString("school_name")?.let {
  viewModel.school = it
}
```

```kotlin
/*
	LoginWebActivity.kt
*/
val fragment = when (viewModel.importType) {
  "login" -> LoginWebFragment()  //”从教务导入“（模拟登录）
  "apply" -> SchoolInfoFragment() //”申请适配“
  "file" -> FileImportFragment() //“从文件导入”
  "excel" -> ExcelImportFragment() //“从Excel文件导入”
  "html" -> HtmlImportFragment() //“从html文件导入”
  else -> { //“从教务导入”（网站登录）
    if (viewModel.importType.isNullOrEmpty() || viewModel.school.isNullOrEmpty()) 
    	null
    else //跳转WebView的Fragment，传递教务系统网址url
    	WebViewLoginFragment.newInstance(intent.getStringExtra("url")!!)
  }
}
fragment?.let { frag ->
	val transaction = supportFragmentManager.beginTransaction()
  transaction.add(android.R.id.content, frag, viewModel.school)
  transaction.commit()
  if (viewModel.importType != "apply" && viewModel.importType != "file")
  showImportSettingDialog()
}

```

#### 加载教务网站导出html源码

##### WebView与h5交互获取源码的方法

```kotlin
/*
	WebViewLoginFragment.kt
*/
//h5调用Java的方法类
internal inner class InJavaScriptLocalObj {
  @JavascriptInterface
  fun showSource(html: String) {
    if (viewModel.importType != "apply") {//所有教务系统的导入
      launch {
        try {
          val result = viewModel.importSchedule(html) //！！！重要解析html的方法！！！返回课程数量
          Toasty.success(activity!!, "成功导入 $result 门课程(ﾟ▽ﾟ)/\n请在右侧栏切换后查看").show()
          activity!!.setResult(RESULT_OK)
          activity!!.finish()
        } catch (e: Exception) {
          Toasty.error(activity!!, "导入失败>_<\n${e.message}", Toast.LENGTH_LONG).show()
        } 
      }
    } else { //申请适配————上传html文件
      launch {
        try {
          viewModel.postHtml(
            school = viewModel.schoolInfo[0],
            type = if (viewModel.isUrp) "URP" else viewModel.schoolInfo[1],
            qq = viewModel.schoolInfo[2],
            html = html)
          Toasty.success(activity!!.applicationContext, "上传源码成功~请等待适配哦", Toast.LENGTH_LONG).show()
          activity!!.start<ApplyInfoActivity>()
          activity!!.finish()
        } catch (e: Exception) {
          Toasty.error(activity!!.applicationContext, "上传失败>_<\n" + e.message, Toast.LENGTH_LONG).show()
        }
      }
    }
  }
}
```

##### 导出网页html源码

```kotlin
/*
	WebViewLoginFragment.kt
*/
val js = "javascript:var ifrs=document.getElementsByTagName(\"iframe\");" +
"var iframeContent=\"\";" +
"for(var i=0;i<ifrs.length;i++){" +
"iframeContent=iframeContent+ifrs[i].contentDocument.body.parentElement.outerHTML;" +
"}\n" +
"var frs=document.getElementsByTagName(\"frame\");" +
"var frameContent=\"\";" +
"for(var i=0;i<frs.length;i++){" +
"frameContent=frameContent+frs[i].contentDocument.body.parentElement.outerHTML;" +
"}\n" +
"window.local_obj.showSource(document.getElementsByTagName('html')[0].innerHTML + iframeContent + frameContent);"

fab_import.setOnClickListener { //导入按钮的点击事件
  if (viewModel.importType == Common.TYPE_HNUST) { //一些特殊网站的处理
    if (!isRefer) {
      val referUrl = when (viewModel.school) {
        "湖南科技大学" -> "http://kdjw.hnust.cn/kdjw/tkglAction.do?method=goListKbByXs&istsxx=no"
        "湖南科技大学潇湘学院" -> "http://xxjw.hnust.cn:8080/xxjw/tkglAction.do?method=goListKbByXs&istsxx=no"
        else -> getHostUrl() + "tkglAction.do?method=goListKbByXs&istsxx=no"
      }
      wv_course.loadUrl(referUrl)
      it.longSnack("请在看到网页加载完成后，再点一次右下角按钮")
      isRefer = true
    } else {
      wv_course.loadUrl(js)
    }
  } else if (viewModel.importType == Common.TYPE_CF) {
    if (!isRefer) {
      val referUrl = getHostUrl() + "xsgrkbcx!getXsgrbkList.action"
      wv_course.loadUrl(referUrl)
      it.longSnack("请重新选择一下学期再点按钮导入，要记得选择全部周，记得点查询按钮")
      isRefer = true
    } else {
      wv_course.loadUrl(js)
    }
  } else if (viewModel.importType == Common.TYPE_URP || viewModel.isUrp) {
    if (!isRefer) {
      val referUrl = getHostUrl() + "xkAction.do?actionType=6"
      wv_course.loadUrl(referUrl)
      it.longSnack("请在看到网页加载完成后，再点一次右下角按钮")
      isRefer = true
    } else {
      wv_course.loadUrl(js)
    }
  } else if (viewModel.importType == Common.TYPE_URP_NEW) {
    if (!isRefer) {
      val referUrl = getHostUrl() + "student/courseSelect/thisSemesterCurriculum/callback"
      wv_course.loadUrl(referUrl)
      it.longSnack("请在看到网页加载完成后，再点一次右下角按钮")
      isRefer = true
    } else {
      wv_course.loadUrl("javascript:window.local_obj.showSource(document.getElementsByTagName('html')[0].innerText);")
    }
  } else if (viewModel.importType == Common.TYPE_JNU) {
    if (countClick == 0) {
      val referUrl = getHostUrl() + "Secure/TeachingPlan/wfrm_Prt_Report.aspx"
      wv_course.loadUrl(referUrl)
      it.longSnack("请在看到网页加载完成后，再点一次右下角按钮")
      countClick++
    } else if(countClick == 1){
      val jnujs = "javascript:window.location.href = document.getElementById(\"ReportFrameReportViewer1\").src;"
      wv_course.loadUrl(jnujs)
      it.longSnack("请再点一次右下角按钮")
      countClick++
    }else{
      wv_course.loadUrl(js)
      countClick = 0
    }
  } else { //加载js调用方法showSource(会提取出整个网页的源码)
    wv_course.loadUrl(js)
  }
}
```

###### 附：加载的javascript代码

```javascript
javascript:
var ifrs = document.getElementsByTagName("iframe");
var iframeContent = "";
for(var i=0; i<ifrs.length; i++) {
  iframeContenta = iframeContent + ifrs[i].contentDocument.body.parentElement.outerHTML;
}
var frs = document.getElementsByTagName("frame");
var frameContent = "";
for(var i = 0; i < frs.length; i++) {
  frameContent = frameContent + frs[i].contentDocument.body.parentElement.outerHTML;
}
window.local_obj.showSource(document.getElementsByTagName('html')[0].innerHTML + iframeContent + frameContent);
```

#### 解析html源码

##### 选择解析类

```kotlin
/*
	ImportViewModel.kt
*/
suspend fun importSchedule(source: String): Int {
  val parser = when (importType) { //根据提前存储的教务系统类别，选择不同的解析类
    Common.TYPE_ZF -> ZhengFangParser(source, zfType)
    Common.TYPE_ZF_1 -> ZhengFangParser(source, 1)
    Common.TYPE_ZF_NEW -> NewZFParser(source)
    Common.TYPE_URP -> UrpParser(source)
    Common.TYPE_URP_NEW -> NewUrpParser(source)
    Common.TYPE_QZ -> {
      when (qzType) {
        0 -> QzParser(source)
        1 -> QzBrParser(source)
        2 -> QzWithNodeParser(source)
        else -> QzCrazyParser(source)
      }
    }
    Common.TYPE_QZ_OLD -> OldQzParser(source)
    Common.TYPE_QZ_CRAZY -> QzCrazyParser(source)
    Common.TYPE_QZ_BR -> QzBrParser(source)
    Common.TYPE_QZ_WITH_NODE -> QzWithNodeParser(source)
    Common.TYPE_CF -> ChengFangParser(source)
    Common.TYPE_PKU -> PekingParser(source)
    Common.TYPE_BNUZ -> BNUZParser(source)
    Common.TYPE_HNIU -> HNIUParser(source)
    Common.TYPE_HNUST -> HNUSTParser(source, oldQzType)
    Common.TYPE_JNU -> JNUParser(source)
    else -> null
  }
  
  return parser?.saveCourse(getApplication(), importId) { baseList, detailList ->
  			if (!newFlag) { //覆盖当前课表
        	courseDao.coverImport(baseList, detailList)
        } else { //导入新课表
          tableDao.insertTable(TableBean(id = importId, tableName = "未命名"))
          courseDao.insertCourses(baseList, detailList)
        }
	} ?: throw Exception("请确保选择正确的教务类型，以及到达显示课程的页面")
}
```

##### 解析html字符串

```kotlin
/*
	NewZFParser.kt
	新正方教务系统（华师教务）
*/
class NewZFParser(source: String) : Parser(source) {
  	override fun generateCourseList(): List<Course> {
      	val courseList = arrayListOf<Course>()
        val doc = Jsoup.parse(source) //对源码解析
        val table1 = doc.getElementById("table1") //课表表格标签id
        val trs = table1.getElementsByTag("tr") //以tr标签获取元素
        var node = 0
        var day = 0
        var teacher = ""
        var room = ""
        var step = 0
        var startWeek = 0
        var endWeek = 0
        var type = 0
        var timeStr = ""
        val nodeRegex = Regex("""\(.*节\)""")
      	for (tr in trs) {
          	val nodeStr = tr.getElementsByClass("festival").text() //获取当前是第几节课
            if (nodeStr.isEmpty()) {
                continue
            }
            node = nodeStr.toInt()
            val tds = tr.getElementsByTag("td") //按td标签获取元素列表
            for (td in tds) {
                val divs = td.getElementsByTag("div") //按div标签获取元素列表[一组div是一个课程对全部信息]
                for (div in divs) {
                    val courseValue = div.text().trim()
                    if (courseValue.length <= 1) { //没有内容不是课程
                        continue
                    }
                    val courseName = div.getElementsByClass("title").text() //获取课程名称
                    if (courseName.isEmpty()) { //没有title不是课程
                        continue
                    }
                    day = td.attr("id")[0].toString().toInt()
                    //td.attr("id") = "1-1"（周数-节数） 获取第一位数周数
                    val pList = div.getElementsByTag("p") //按p标签获取元素列表[一组p是一项课程属性]
                    val weekList = arrayListOf<String>()
                    pList.forEach { e ->
                    		when (e.getElementsByAttribute("title").attr("title")) { //获取当前项的属性名称
                          	"教师" -> teacher = e.text().trim()
                            "上课地点" -> room = e.text().trim()
                            "节/周", "周/节" -> {
                              	timeStr = e.text().trim()
                                //(9-10节)4-20周
                              
                                val result = Common.nodePattern.find(timeStr) //查询课程节数
                                //Regex("""\(\d{1,2}[-]*\d*节""")
                              	if (result != null) {
                                  	val nodeInfo = result.value
                                    val nodes = nodeInfo.substring(1, nodeInfo.length - 1).split("-".toRegex()).dropLastWhile { it.isEmpty() } //课程的起止节数

                                    if (nodes.isNotEmpty()) {
                                        node = nodes[0].toInt() //课程的起始节数
                                    }
                                    if (nodes.size > 1) {
                                        val endNode = nodes[1].toInt()
                                        step = endNode - node + 1 //记录课程节数跨度
                                    }
                                }
                                weekList.clear()
                                weekList.addAll(nodeRegex.replace(timeStr, "").split(','))
                              	//课程的起止周数（"5-20周"）
                            }
                        }
                    }
                    weekList.forEach {
                        if (it.contains('-')) { //课程包含连续周
                            val weeks = it.substring(0, it.indexOf('周')).split('-')
                            if (weeks.isNotEmpty()) {
                                startWeek = weeks[0].toInt()
                            }
                            if (weeks.size > 1) {
                                endWeek = weeks[1].toInt()
                            }
                          	//获取起止周数
                            type = when { //是否单双周
                                it.contains('单') -> 1
                                it.contains('双') -> 2
                                else -> 0
                            }
                        } else {
                            try { //没有包含连续周，只上一周的课
                                startWeek = it.substring(0, it.indexOf('周')).toInt()
                                endWeek = it.substring(0, it.indexOf('周')).toInt()
                            } catch (e: Exception) {
                                startWeek = 1
                                endWeek = 20
                            }
                        }

                      	//添加课程
                        courseList.add(
                                Course(
                                        name = courseName, room = room,
                                        teacher = teacher, day = day,
                                        startNode = node, endNode = node + step - 1,
                                        startWeek = startWeek, endWeek = endWeek,
                                        type = type
                                )
                        )
                    }
                }
            }
        }
        return courseList
    }
}
```

###### 附：课表html
```html
<div class="tab-pane fade active in" id="table1" style="">
    <table id="kbgrid_table_0" class="table table-hover table-bordered text-center timetable1"
    style="width:98%;margin-left:10px">
        <tbody>
            <tr>
                <td colspan="8">
                    <div class="timetable_title">
                        <h6 class="pull-left">2020-2021学年第1学期</h6>钟伊凡的课表
                        <h6 class="pull-right">学号：2018212679</h6>
                    </div>
                    <div>
                        <span class="pull-left">○-实验(实践)★-授课</span>
                        <span class="pull-right">
                            <font color="red" size="3"><b>注：</b></font><font color="red" size="3"><i>红色斜体为待筛选</i></font>，<font color="blue" size="3">蓝色为已选上</font>
			</span>
                    </div>
                </td>
            </tr>
            <tr>
              	<td width="5%"><span class="time">节次</span></td>
              	<td width="13.5%"><span class="time">星期一</span></td> 
              	<td width="13.5%"><span class="time">星期二</span></td>
              	<td width="13.5%"><span class="time">星期三</span></td>
              	<td width="13.5%"><span class="time">星期四</span></td>
              	<td width="13.5%"><span class="time">星期五</span></td>
              	<td width="13.5%"><span class="time">星期六</span></td>
              	<td width="13.5%"><span class="time">星期日</span></td>
            </tr>
            <tr>
                <td><span class="festival">1</span></td>
                <td rowspan="2" id="1-1" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">习近平新时代中国特色社会主义思想概论★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(1-2节)4-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N223</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">郭明飞</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">15</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">考试</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:24,实验(实践):4,研讨:4</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">34</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2.0</font>
                        </p>
                    </div>
                </td>
                <td rowspan="1" id="2-1" class="td_wrap"></td>
                <td rowspan="2" id="3-1" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">信息系统建模（UML）★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(1-2节)4周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N312</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">叶光辉</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">1</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:32</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">0</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">信息系统建模（UML）★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(1-2节)5-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N312</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">王光超</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">1</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:32</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">0</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                </td>
                <td rowspan="1" id="4-1" class="td_wrap"></td>
                <td rowspan="2" id="5-1" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">数据库系统开发★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(1-2节)4-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 信管实验室</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">肖毅</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">1</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">无</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:20,实验(实践):12</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">34</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                </td>
                <td rowspan="1" id="6-1" class="td_wrap"></td>
                <td rowspan="1" id="7-1" class="td_wrap"></td>
            </tr>
            <tr>
                <td><span class="festival">2</span>
                </td>
                <td rowspan="1" id="2-2" class="td_wrap"></td>
                <td rowspan="1" id="4-2" class="td_wrap"></td>
                <td rowspan="1" id="6-2" class="td_wrap"></td>
                <td rowspan="1" id="7-2" class="td_wrap"></td>
            </tr>
            <tr>
                <td><span class="festival">3</span>
                </td>
                <td rowspan="2" id="1-3" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">市场营销学★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(3-4节)4-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N312</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">谭春辉</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">1</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:20,研讨:12</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">34</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                </td>
                <td rowspan="2" id="2-3" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">信息系统分析与设计★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(3-4节)4-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N316</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">丁恒</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:32</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">0</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                </td>
                <td rowspan="1" id="3-3" class="td_wrap"></td>
                <td rowspan="2" id="4-3" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">数据挖掘技术★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(3-4节)4-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N312</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">刘向</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:20,实验(实践):12</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">34</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                </td>
                <td rowspan="1" id="5-3" class="td_wrap"></td>
                <td rowspan="1" id="6-3" class="td_wrap"></td>
                <td rowspan="1" id="7-3" class="td_wrap"></td>
            </tr>
            <tr>
                <td><span class="festival">4</span>
                </td>
                <td rowspan="1" id="3-4" class="td_wrap"></td>
                <td rowspan="1" id="5-4" class="td_wrap"></td>
                <td rowspan="1" id="6-4" class="td_wrap"></td>
                <td rowspan="1" id="7-4" class="td_wrap"></td>
            </tr>
            <tr>
                <td><span class="festival">5</span>
                </td>
                <td rowspan="1" id="1-5" class="td_wrap"></td>
                <td rowspan="2" id="2-5" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">供应链管理★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(5-6节)4-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N312</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">陈鹏宇</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">1</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:32</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">34</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                </td>
                <td rowspan="1" id="3-5" class="td_wrap"></td>
                <td rowspan="2" id="4-5" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">社会网络分析★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(5-6节)4-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N312</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">王忠义</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">1</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:24,研讨:8</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">34</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                </td>
                <td rowspan="1" id="5-5" class="td_wrap"></td>
                <td rowspan="1" id="6-5" class="td_wrap"></td>
                <td rowspan="1" id="7-5" class="td_wrap"></td>
            </tr>
            <tr>
                <td><span class="festival">6</span>
                </td>
                <td rowspan="1" id="1-6" class="td_wrap"></td>
                <td rowspan="1" id="3-6" class="td_wrap"></td>
                <td rowspan="1" id="5-6" class="td_wrap"></td>
                <td rowspan="1" id="6-6" class="td_wrap"></td>
                <td rowspan="1" id="7-6" class="td_wrap"></td>
            </tr>
            <tr>
                <td><span class="festival">7</span></td>
                <td rowspan="2" id="1-7" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">系统工程★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(7-8节)4-5周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N316</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">陈静</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:32</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">0</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">系统工程★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(7-8节)6-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N316</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">王光超</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:32</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">0</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                </td>
                <td rowspan="1" id="2-7" class="td_wrap"></td>
                <td rowspan="2" id="3-7" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">优化模型与软件工具★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(7-8节)4-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N316</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">王光超</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:28,研讨:4</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">0</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                </td>
                <td rowspan="2" id="4-7" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">消费心理与行为（通核）</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(7-8节)4-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N207</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">王伟军</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:32</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">34</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2.0</font>
                        </p>
                    </div>
                </td>
                <td rowspan="1" id="5-7" class="td_wrap"></td>
                <td rowspan="1" id="6-7" class="td_wrap"></td>
                <td rowspan="1" id="7-7" class="td_wrap"></td>
            </tr>
            <tr>
                <td><span class="festival">8</span>
                </td>
                <td rowspan="1" id="2-8" class="td_wrap"></td>
                <td rowspan="1" id="5-8" class="td_wrap"></td>
                <td rowspan="1" id="6-8" class="td_wrap"></td>
                <td rowspan="1" id="7-8" class="td_wrap"></td>
            </tr>
            <tr>
                <td><span class="festival">9</span>
                </td>
                <td rowspan="2" id="1-9" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">C#程序设计★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(9-10节)4-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N312</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">程秀峰</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">1</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:20,实验(实践):12</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">34</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                </td>
                <td rowspan="1" id="2-9" class="td_wrap"></td>
                <td rowspan="2" id="3-9" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">XML技术★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(9-10节)4-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N312</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">叶光辉</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">1</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:20,实验(实践):12</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">0</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                </td>
                <td rowspan="2" id="4-9" class="td_wrap">
                    <div class="timetable_con text-left">
                        <span class="title"><font color="blue">用户行为与数据分析★</font></span>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="节/周"><font color="blue"><span class="glyphicon glyphicon-time"></span></font></span>
                            <font color="blue">(9-10节)4-20周</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="上课地点"><font color="blue"><span class="glyphicon glyphicon-map-marker"></span></font></span>
                            <font color="blue">本校 N313</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教师"><font color="blue"><span class="glyphicon glyphicon-user"></span></font></span>
                            <font color="blue">陈烨</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="教学班名称"><font color="blue"><span class="glyphicon glyphicon-home"></span></font></span>
                            <font color="blue">1</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="考核方式"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">未安排</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="选课备注"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">上课地点：南湖综合楼</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="课程学时组成"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">授课:32</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="周学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="总学时"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">34</font>
                        </p>
                        <p>
                            <span data-toggle="tooltip" data-placement="top" title="学分"><font color="blue"><span class="glyphicon glyphicon-tower"></span></font></span>
                            <font color="blue">2</font>
                        </p>
                    </div>
                </td>
                <td rowspan="1" id="5-9" class="td_wrap"></td>
                <td rowspan="1" id="6-9" class="td_wrap"></td>
                <td rowspan="1" id="7-9" class="td_wrap"></td>
            </tr>
            <tr>
                <td><span class="festival">10</span></td>
                <td rowspan="1" id="2-10" class="td_wrap"></td>
                <td rowspan="1" id="5-10" class="td_wrap"></td>
                <td rowspan="1" id="6-10" class="td_wrap"></td>
                <td rowspan="1" id="7-10" class="td_wrap"></td>
            </tr>
            <tr>
                <td><span class="festival">11</span></td>
		<td rowspan="1" id="1-11" class="td_wrap"></td>
                <td rowspan="1" id="2-11" class="td_wrap"></td>
                <td rowspan="1" id="3-11" class="td_wrap"></td>
                <td rowspan="1" id="4-11" class="td_wrap"></td>
                <td rowspan="1" id="5-11" class="td_wrap"></td>
                <td rowspan="1" id="6-11" class="td_wrap"></td>
                <td rowspan="1" id="7-11" class="td_wrap"></td>
            </tr>
            <tr>
                <td><span class="festival">12</span></td>
                <td rowspan="1" id="1-12" class="td_wrap"></td>
                <td rowspan="1" id="2-12" class="td_wrap"></td>
                <td rowspan="1" id="3-12" class="td_wrap"></td>
                <td rowspan="1" id="4-12" class="td_wrap"></td>
                <td rowspan="1" id="5-12" class="td_wrap"></td>
                <td rowspan="1" id="6-12" class="td_wrap"></td>
                <td rowspan="1" id="7-12" class="td_wrap"></td>
            </tr>
        </tbody>
    </table>
</div>
```



