title: "Restful api url设计建议"
date: 2015-11-24 17:23:19
tags: 
- API
- Restful
---

#基本CRUD
GET /users - 获取user列表
GET /users/12 - 获取id为12的user 
GET /users/12/address 获取id为12的user的address字段
GET /users/12?fields=id,name,address 获取id未12的user，并只返回id，name，address字段
POST /users - 创建user
PUT /users/12 - 更新id为12的user
PATCH /users/12 - 更新id为12的user的部分字段
DELETE /users/12 - 删除id为12的user

#过滤、排序、搜索
GET /users?sort=number - 获取user列表，并根据number字段排序
GET /users?sort=number,name - 获取user列表，并根据number字段,name字段排序

GET /users?state=0 - 获取state=0的user
GET /users?state=0&sex=1 - 获取state=0并且sex=1的user

#分页
GET /users?page=1&size=10 - 获取user列表，第1页，每页10条数据

#数据处理（post与put）
post 处理不能唯一标识的数据
put 处理可以唯一标识的数据
POST /users/bulkCreate 批量更新
POST /users/bulkDelete 批量删除
PUT /users/12/star 关注id为12的user
PUT /users/12/unstar 取消关注id为12的user

#层级关系
对于层级关系比较紧密的实体，某些实体是通过上层实体关联暴露的，比如：
GET /company/1/department/2/users
出于使用的便捷性考虑，用户也可以只需要理解一个实体名称 users。虽然架构上或者物理上仍然作为下层关联实体存在，但是在接口上可以重新设计为
GET users?company=1&&department=2
或者同时保留两种url


参考：
[设计易用的RESTFul API](http://efe.baidu.com/blog/friendly-restful-api/)
[Best Practices for Designing a Pragmatic RESTful API](http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api)
