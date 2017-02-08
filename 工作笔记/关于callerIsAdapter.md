[TOC]
##CalendarUtilities#asSyncAdapter()使用统计
1. 使用ExtendedProperties字段存储数据时
    1. EventInfoFragment中标记完成
    2. EditEventHelper中
        * 创建事件的农历重复
        * 存放finish拓展字段
2. 登录以及切换账户
    1.  CalendarUtilities#createCalendar
3. 在UI层修改Dirty值
    1. DrawerFragment#updateCalendarVisibility
    2. CreateCalendarActivity中
        * createRules
        * saveCalendar
        * updateRules
        * onQueryComplete
    3. PreviewCalendarActivity
        * createRules
        * updateRules
        * saveCalendar
        * onQueryComplete
4. 同步
    1. EventHelper中
        * saveExtendedProperties 
        * saveFinishExtendedProperties
    2. CalendarUtilities#createCalendar

##关于在uri后拼接callerIsAdapter参数的作用
在我看来
1. 对同步数据与用户改变数据的通知做分别处理
2. 对字段的修改作出限制
3. 在同步时候的数据库操作一般都带有而外操作，比如delete字段为1 的数据的删除，以及dirty的设置
4. 某些数值根据来源不同，会进行不同的修改
##目前
1. ui层会有对dirty的修改。我认为这个dirty的数值是不需要ui层关心的。
2. ExtendedProperties的修改限制可以放开
3. 工具方法添加参数来区分调用者是UI层还是数据层
4. 在SycnService中获取下行数据时的数据操作补上asSyncAdapter