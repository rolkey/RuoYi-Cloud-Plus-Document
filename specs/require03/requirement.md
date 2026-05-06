# 租户功能调整

```yaml
数据库连接参数：
  url: jdbc:mysql://192.168.168.128:3306/ry-cloud
  user: ruoyi
  password: Ruoyi@111
```

## 上下级租户功能

* 允许设置上下级租户，在sys_tenant表添加parent_tenant_id字段
* 只能设置两级关系，不能多级

## 用户功能调整

* 用户必须属于一级租户
* 可以给用户分配二级用户的功能
