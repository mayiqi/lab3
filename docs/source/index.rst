.. lab3 documentation master file, created by
   sphinx-quickstart on Sat Dec 25 22:44:49 2021.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.
   
**Lab 2**: The ORM Magic and The Service Layer
=================================================

.. toctree::
   :maxdepth: 2
   :caption: Contents:

**小组成员**

201932110105沈音棋

201932110104马奕琪

201932110106孙仪杰

201932110107田西芷

**项目GitHub地址**: `lab3`_.

.. _lab3: https://github.com/mayiqi/lab3

**项目Read the Doc地址**: `Read the Doc`_.

.. _Read The Doc: https://readthedocs.org/projects/lanlab3/



Abstract
=================================================

介绍orm.py和services.py的修改内容，以及没有遵循SPR的原因。

Introduction
=================================================

为理解依赖反转相关内容，我们使用SQLAlchemy的ORM将类映射到数据库中，以实现一个供用户阅读文章的服务层，以及实践测试驱动的开发（TDD）。

Methods and materials
=================================================

我们添加了orm.py中start_mappers函数的内容，以及services.py中read函数的内容。

在start_mappers函数中，我们先调用metadata语句连接到数据库，获取数据库资源后，储存到medel中去。

然后我们通过read函数来读取文章。首先判断登陆状态，如果没有登录，抛出异常。确认登录后，判断是否读取过文章。如果没有文章，抛出异常。在有文章的情况下，读取用户的生词库。然后计算文章等级，选取合适的文章推荐给用户，返回被选中的文章的id。

在单一职责原则（SRP：Single Responsibility Principle）的概念中，我们将职责（Responsibility）定义为 "一个变化的原因（a reason for change）"。如果你能想出多于一种动机来更改一个类，则这个类就包含多于一个职责。

我们没有遵循SPR，service.py中read函数包含连接数据库以及选择合适文章推荐给用户两个职责。因为这两个职责是有联系的。选取合适文章需要了解用户的生词库，而生词库需要读取数据库获得，所以并没有将这两个职责分开，而是都放在了read函数中。

Results
=================================================

**orm.py**
::
      from sqlalchemy import Table, MetaData, Column, Integer, String, Date, ForeignKey
      from sqlalchemy.orm import mapper, relationship
      from sqlalchemy import create_engine

      import model

      metadata = MetaData()

      articles = Table(
          'articles',
          metadata,
          Column('article_id', Integer, primary_key=True, autoincrement=True),
          Column('text', String(10000)),
          Column('source', String(100)),
          Column('date', String(10)),
          Column('level', Integer, nullable=False),
          Column('question', String(1000)),
          )

      users = Table(
          'users',
          metadata,
          Column('username', String(100), primary_key=True),
          Column('password', String(64)),
          Column('start_date', String(10), nullable=False),
          Column('expiry_date', String(10), nullable=False),
          )

      newwords = Table(
          'newwords',
          metadata,
          Column('word_id', Integer, primary_key=True, autoincrement=True),
          Column('username', String(100), ForeignKey('users.username')),
          Column('word', String(20)),
          Column('date', String(10)),
          )

      readings = Table(
          'readings',
          metadata,
          Column('id', Integer, primary_key=True, autoincrement=True),
          Column('username', String(100), ForeignKey('users.username')),
          Column('article_id', Integer, ForeignKey('articles.article_id')),
          )

      def start_mappers():
          metadata.create_all(create_engine('sqlite:///EnglishPalDatabase.db'))
          mapper(model.User, users)
          mapper(model.NewWord, newwords)
          mapper(model.Article, articles)
          mapper(model.Reading, readings)


**Services.py**
::
      # word and its difficulty level
      WORD_DIFFICULTY_LEVEL = {'starbucks':5, 'luckin':4, 'secondcup':4, 'costa':3, 'timhortons':3, 'frappuccino':6}

      import model

      class UnknownUser(Exception):
          pass

      class NoArticleMatched(Exception):
          pass

      def read(user, user_repo, article_repo, session):
          u = user_repo.get(user.username)
          if u == None or u.password != user.password:
              raise UnknownUser()

          articles = article_repo.list()

          if articles == None:
              raise NoArticleMatched()

          words = session.execute(
              'SELECT word FROM newwords WHERE username=:username',
              dict(username=user.username),
          )

          sum = 0
          count = 0
          for word in words:
              sum += WORD_DIFFICULTY_LEVEL[word[0]]
              count += 1

          if count == 0:
              count = 1

          average = round(sum / count) + 1
          if average < 3:
              average = 3

          for article in articles:
              if average == article.level:
                  article_id = user.read_article(article)
                  session.add(model.Reading(username = user.username, article_id = article_id))
                  session.commit()
                  return article_id

          raise NoArticleMatched()

References
=================================================

单一职责原则（Single Responsibility Principle） - sangmado - 博客园 (cnblogs.com)
