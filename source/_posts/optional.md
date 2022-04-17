---
title: java8之optionAl用法
date: 2022-04-16 10:57:12
tags: 
 - java 
categories: 
 - java
---
# 实战使用
> 以前写法
```
public String getCity(User user)  throws Exception{
        if(user!=null){
            if(user.getAddress()!=null){
                Address address = user.getAddress();
                if(address.getCity()!=null){
                    return address.getCity();
                }
            }
        }
        throw new Excpetion("取值错误"); 
    }
```    
> JAVA8写法
```
public String getCity(User user) throws Exception{
    return Optional.ofNullable(user)
                   .map(u-> u.getAddress())
                   .map(a->a.getCity())
                   .orElseThrow(()->new Exception("取指错误"));
}
```
# 例二
> 以前写法
```
if(user!=null){
    dosomething(user);
}
```
> JAVA8写法
```
 Optional.ofNullable(user)
    .ifPresent(u->{
        dosomething(u);
});
```
# 例三

> 以前写法
```
public User getUser(User user) throws Exception{
    if(user!=null){
        String name = user.getName();
        if("zhangsan".equals(name)){
            return user;
        }
    }else{
        user = new User();
        user.setName("zhangsan");
        return user;
    }
}
```
> java8写法
```
public User getUser(User user) {
    return Optional.ofNullable(user)
                   .filter(u->"zhangsan".equals(u.getName()))
                   .orElseGet(()-> {
                        User user1 = new User();
                        user1.setName("zhangsan");
                        return user1;
                   });
}
```
