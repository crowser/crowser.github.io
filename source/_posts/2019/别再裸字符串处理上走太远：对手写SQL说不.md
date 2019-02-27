---
title: 别再裸字符串处理上走太远：对手写SQL说不
date: 2019-02-28 00:17:58
tag: 技巧
---



SQL作为访问和处理关系数据库的计算机标准语言, 无论用什么编程语言（Java、Python、C++……）编写程序，只要涉及到操作关系数据库，比如，一个电商网站需要把用户和商品信息存入数据库，或者一个手机游戏需要把用户的道具、通关信息存入数据库，都必须通过SQL来完成。

然后当SQL语句出现在程序中时却不那么尽如人意, 他有诸多问题, 笼统地讲就是他对于编程语言来说不够抽象(无法控制sql语句的复杂度).

<!--more-->

> 以下内容引用自 [Python 工匠：使用数字与字符串的技巧](https://www.zlovezl.cn/articles/tips-on-numbers-and-strings/)


所有人都写过这样的代码。有时候我们需要拼接一大段发给用户的告警信息，有时我们需要构造一大段发送给数据库的 SQL 查询语句，就像下面这样：

    def fetch_users(conn, min_level=None, gender=None, has_membership=False, sort_field="created"):
        """获取用户列表
    
        :param int min_level: 要求的最低用户级别，默认为所有级别
        :param int gender: 筛选用户性别，默认为所有性别
        :param int has_membership: 筛选所有会员/非会员用户，默认非会员
        :param str sort_field: 排序字段，默认为按 created "用户创建日期"
        :returns: 列表：[(User ID, User Name), ...]
        """
        # 一种古老的 SQL 拼接技巧，使用 "WHERE 1=1" 来简化字符串拼接操作
        # 区分查询 params 来避免 SQL 注入问题
        statement = "SELECT id, name FROM users WHERE 1=1"
        params = []
        if min_level is not None:
            statement += " AND level >= ?"
            params.append(min_level)
        if gender is not None:
            statement += " AND gender >= ?"
            params.append(gender)
        if has_membership:
            statement += " AND has_membership == true"
        else:
            statement += " AND has_membership == false"
    
        statement += " ORDER BY ?"
        params.append(sort_field)
        return list(conn.execute(statement, params))

我们之所以用这种方式拼接出需要的字符串 - 在这里是 SQL 语句 - 是因为这样做简单、直接，符合直觉。但是这样做最大的问题在于：随着函数逻辑变得更复杂，这段拼接代码会变得容易出错、难以扩展。事实上，上面这段 Demo 代码也只是仅仅做到看上去没有明显的 bug 而已 （谁知道有没有其他隐藏问题）。

其实，对于 SQL 语句这种结构化、有规则的字符串，用对象化的方式构建和编辑它才是更好的做法。下面这段代码用 SQLAlchemy 模块完成了同样的功能：

    def fetch_users_v2(conn, min_level=None, gender=None, has_membership=False, sort_field="created"):
        """获取用户列表
        """
        query = select([users.c.id, users.c.name])
        if min_level is not None:
            query = query.where(users.c.level >= min_level)
        if gender is not None:
            query = query.where(users.c.gender == gender)
        query = query.where(users.c.has_membership == has_membership).order_by(users.c[sort_field])
        return list(conn.execute(query))

上面的 fetch_users_v2 函数更短也更好维护，而且根本不需要担心 SQL 注入问题。所以，当你的代码中出现复杂的裸字符串处理逻辑时，请试着用下面的方式替代它：

    Q: 目标/源字符串是结构化的，遵循某种格式吗？

- 是：找找是否已经有开源的对象化模块操作它们，或是自己写一个
  - SQL：SQLAlchemy
  - XML：lxml
  - JSON、YAML ...
- 否：尝试使用模板引擎而不是复杂字符串处理逻辑来达到目的
  - Jinja2
  - Mako
  - Mustache
