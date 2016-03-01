---
layout: post
title: 多列模糊搜索实现
---

### Rails Query Interface: where
Rails提供了一套非常便利而又强大的查询接口（Query Interface）。例如我们经常同其打交道的`where`方法，可以帮我们解决大多数查询条件（Conditions）相关的问题：

```ruby
# Equality Conditions
Device.where('is_root' => true)

# Range Conditions
Device.where(created_at: (Time.now.midnight - 1.day)..Time.now.midnight)

# Subset Conditions
Device.where(os_type: [1,3,5])

# NOT Conditions
Device.where.not(brand: brand)

# OR Conditions(Rails5 Only)
# see https://github.com/rails/rails/pull/16052/
Device.where('id = 1').or(Device.where('id = 2'))
```

### 问题背景
今天我遇到了这么一个的查询问题：

假设数据库中存在一张设备表（devices），表中有多个字段：设备品牌（brand）、系统类型（os type）、分辨率（resolution）、序列号（serial number）、内存（memory）等十多个字段。我们已经实现了一个`index`页面，负责展示所有的设备，现在希望在页面上实现一个搜索框，根据搜索框中输入的文字，针对设备表中的设备品牌、分辨率、序列号这三个字段进行模糊搜索。

大体上长这个样子：
![模糊搜索示例]({{ site.baseurl }}/images/fuzzy_search.png "模糊搜索示例")

### 解决思路
根据需求我实现了一个`search`方法，看起来基本可以满足需求了：

```ruby
# models/devices.rb
def self.search(search)
  if search
    search.strip!
    where("device_brand like ? OR serial_number like ? OR resolution like ?", "%#{search}%", "%#{search}%", "%#{search}%")
  else
    all
  end
end
```
然而深入思考一下，如果有一天需求发生了变更：不再仅仅只针对三个字段进行模糊搜索，而是将模糊搜索的范围扩大到更多字段，那是不是需要拼一个非常长的`where`查询子句：

```ruby
where("aaa like ? OR bbb like ? OR ccc like ? OR ddd like ? OR eee like ? OR fff like ?", "%#{param}%", "%#{param}%", "%#{param}%", "%#{param}%", "%#{param}%", "%#{param}%")
```

有没有更优雅一些的实现方案呢？

### Arel
[Arel](https://github.com/rails/arel)是Rails里用来管理生成AST(Abstract Syntax Tree 抽象语法树)的组件，负责将Rails里一些SQL查询的DSL转化为底层的SQL语句。

我们可以使用Arel灵活地实现一些复杂的查询。

Rails提供了一个叫做`arel_table`的方法，用来访问底层Arel的相关接口：

```ruby
Device.arel_table  # => #<Arel::Table:0x00000004a03e18>
```

这里返回了一个`Arel::Table`对象，我们可以把它理解成一个包含了数据库表中每一列的hash对象，可以通过正常的访问hash元素的方式来访问这些列。

Arel提供了一系列的[方法](https://github.com/rails/arel/blob/master/lib/arel/predications.rb)作用于这些列上，用于构造SQL语句。下面是一段简单的示例：

```ruby
devices = Device.arel_table
devices.where(devices[:brand].eq('HUAWEI'))
# SELECT * FROM devices WHERE devices.brand = 'HUAWEI'
```
回到我们的需求上，我们可以用Arel将原先的代码重构下：

```ruby
# 使用Arel实现多字段模糊查询
def self.search(search)
  if search
    search.strip!
    search_fields = %w(device_brand serial_number resolution)
    # 每个待查询字段都会有一个对应的 matches 匹配条件，最后这些条件之间用or运算合并，语义即“device_brand matches xxx or serial_number matches xxx or ...”
    condition = search_fields.map do |field|
      arel_table[field].matches("%#{search}%")
    end.inject(:or)
    # 使用 Arel 构造的查询子句可以直接用于更高层级的 Query Method，也就是where方法
    where(condition)
  else
    all
  end
end
```
这样即使模糊搜索的范围扩大到了更多的字段，我们只需要将相应的字段名赋值给`search_fields`就可以了，其它代码不需要做任何修改。

### Any way else?
需要注意到的一点是，使用`like '%keyword%'`这种查询方式是无法使用索引的，如果数据库表中的数据非常多，那么必然会成为性能上的瓶颈。

也可以考虑使用一些搜索引擎来实现模糊查询，例如ElasticSearch。

### 参考
[Active Record Query Interface](http://guides.rubyonrails.org/active_record_querying.html)

[Using Arel to compose SQL queries](https://robots.thoughtbot.com/using-arel-to-compose-sql-queries)

[Rails 源码分析之 Arel](http://railscasts-china.com/episodes/kenshin54-source-code-analysis-arel)