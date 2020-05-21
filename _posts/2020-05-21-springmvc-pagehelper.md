---
title: Spring MVC整合PageHelper
tags:
    - Spring MVC
    - 分页
    - PageHelper
---

> 具体见[官方教程](https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md)，本文仅为实践小记。

## 引入pagehelper依赖

```maven
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper</artifactId>
    <version>5.1.11</version>
</dependency>
```

## 配置applicationContext.xml文件

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="plugins">
        <array>
            <bean class="com.github.pagehelper.PageInterceptor">
                <!-- 这里的几个配置主要演示如何使用，如果不理解，一定要去掉下面的配置 -->
                <property name="properties">
                    <value>
                        <!-- 在这里设定属性，具体见:
                        https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md -->
                    </value>
                </property>
            </bean>
        </array>
    </property>
</bean>
```

## 修改Controller中对应的方法

```java
@RequestMapping("/")
public ModelAndView getList(ModelMap map,@RequestParam(required = false, defaultValue = "1") int pageNum,
                            @RequestParam(required = false, defaultValue = "10") int pageSize) {
    List<Country> countryList = countryService.selectByCountry(pageNum, pageSize);
    map.addAttribute("pageInfo",new PageInfo<Country>(countryList));
    return "/index";
}
```

## 修改Service中对应的方法

```java
public List<Country> selectByCountry(int pageNum, int pageSize) {
    //  开始分页
    PageHelper.startPage(pageNum, pageSize);
    //  查询数据库
    List<Country> countryList = countryMapper.selectByExample(example);
    //  获取分页查询后的数据
    PageInfo<Country> countryPageInfo = new PageInfo<>(countryList);
    //  封装需要返回的分页实体
    return countryPageInfo;
}
```

## 修改视图页（以JSP为例）代码

```jsp
<!-- 表格显示 -->
<table class="table" >
    <thead>
        <tr>
            <th>id</th>
        </tr>
    </thead>
    <tbody>
        <c:forEach items="${pageInfo.list}" var="c">
            <tr>
                <td>${c.id}</td>
            </tr>
        </c:forEach>
    </tbody>
</table>

<div style="float: right;">
    <div style="float: right;">
        当前${pageInfo.pageNum}页,共${pageInfo.pages}页,总共${pageInfo.total}条记录
    </div>
    <div>
        <ul class="pagination">
            <!-- pageContext.request.contextPath表示当前页面路径 -->
            <li>
                <a href="${pageContext.request.contextPath}/?pageNum=${pageInfo.firstPage}">首页</a>
            </li>
            <!-- 如果还有上一页，就显示'<<'按钮 -->
            <c:if test="${pageInfo.hasPreviousPage}">
                <li>
                    <a href="${pageContext.request.contextPath}/?pageNum=${pageInfo.prePage}" aria-label="Previous">
                        <span aria-hidden="true">&laquo;</span>
                    </a>
                </li>
            </c:if>
            <li>
                <!--遍历所有导航页码，如果页码与当前页码相等就高亮显示  -->
                <c:forEach items="${pageInfo.navigatepageNums}" var="page_Nums">
                    <c:if test="${page_Nums==pageInfo.pageNum}">
                        <li class="active"><a href="#">${page_Nums}</a></li>
                    </c:if>
                    <c:if test="${page_Nums!=pageInfo.pageNum}">
                        <li ><a href="${pageContext.request.contextPath}/?pageNum=${page_Nums}">${page_Nums}</a></li>
                    </c:if>
                </c:forEach>
            </li>
            <!-- 如果还有下一页，就显示'>>'按钮 -->
            <c:if test="${pageInfo.hasNextPage}">
                <li>
                    <a href="${pageContext.request.contextPath}/?pageNum=${pageInfo.nextPage}" aria-label="Next">
                        <span aria-hidden="true">&raquo;</span>
                    </a>
                </li>
            </c:if>
            <li><a href="${pageContext.request.contextPath}/?pageNum=${pageInfo.lastPage}">末页</a></li>
        </ul>
    </div>
</div>
```
