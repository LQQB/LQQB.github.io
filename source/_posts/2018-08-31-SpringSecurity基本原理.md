---
title: SpringSecurity基本原理分析
date: 2018-08-31 17:25:56
tags: [Spring, Java, SpringSecurity]
keywords: [SpringSecurity基本原理, SpringSecurity]
---
*SpringSecurity* 是一个能够基于 *Spring* 的企业应用系统提供声明式的安全框架<!--more-->
### 它能做什么
> __认证:__ 识别当前用户是谁
> __授权:__ 判断当前用户是否有权限进行相关操作
> __安全防护:__ 防止伪造身份

如果要对 Web 资源进行保护，最好的方法莫过于 Filter, 要项对方法调用进行保护，最好的方法莫过于AOP, `(ﾉ ○ Д ○)ﾉ`。

### 它是如何实现的
下面来简单的介绍一个 __SpringSecurity 基本原理__ :
当一个请求要访问我们受 __SpringSecurity__ 保护的 Web 资源的时候, 会先经过我们在 *Spring* 中设置的关于 __SpringSecurity__ 的多层过滤器链,(如 __UsernamePasswordAuthenticationFilter__, __BasicAuthenticationFilter__),一个过滤器一个过滤器往下走,每通过一个过滤器的认证就会添加在请求中添加一个标记。如下图所示:
![](filter.png)

然后勒，这些 Filter 怎么和我们的数据库联系在一起勒？

这些 __Filter__ 并不直接处理用户的认证，也不直接处理用户的授权，而是把它们交给认证管理器 __AuthenticationManager__ 和决策管理器 __AccessDecisionManager__。SpringSecurity 提供了多个Provider的实现类，拿 __UsernamePasswordAuthenticationFilter__举例，当我们调用这个 __Filter__，并通过__AuthenticationManager__调用到 __DaoAuthenticationProvider__, 这个 __Provider__ 可以调用数据库来存储用户的认证数据。对于__Voter__，我们一般选择__RoleVoter__就够用了，它会根据我们配置中的设置来决定是否允许某一个用户访问制定的Web资源。认证跟授权的流程如下图：
![](认证和授权.png)

 __DaoAuthenticationProvider__不直接操作数据库的，它把任务委托给了 __UserDetailService__，我们要做的就是现实 __UserDetailService__, 它既是连接我们数据库跟SpringSecurity的桥梁。UserDetailService 的要求也很简单，只需要返回一个 __org.springframework.security.userdetails.User__ 对象的 __loadUserByUsername(String userName)__ 方法
。因此，怎么设计数据库都可以，不管我们是用一个表还是两个表还是三个表，也不管我们是用户-授权，还是用户-角色-授权，还是用户-用户组-角色-授权，这些具体的东西 __SpringSecurity__ 统统不关心，它只关心返回的那个User对象，至于怎么从数据库中读取数据，那就是我们自己的事了。


__UserDetailService__ 实现类部分代码,模拟从数据库中获取用户信息:
``` Java
@Component
public class MyUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username);
        if (user == null) {
            throw new UsernameNotFoundException(username);
        }
        return new MyUserPrincipal(user);
    }
}
```

MyUserPrincipal定义如下：
``` Java
public class MyUserPrincipal implements UserDetails {
    private User user;

    public MyUserPrincipal(User user) {
        this.user = user;
    }
    //...
}
```
SpringSecurity 不关心你用了几个表，它只关心UserDetails对象。而决定用户能否访问指定Web资源的，是RoleVoter类，无需任何修改它可以工作得很好，唯一的缺点是它只认ROLE_前缀。

### 它的常用 Filter 总结
SpringSecurity 提供的Filter 不少，不过常用的只有这么几个，下面我们来总结一下：
> __ChannelProcessingFilter:__  制定必须为https连接
> __LogoutFilter:__退出登录
> __UsernamePasswordAuthenticationFilter:__ 账号密码登录
> __RememberMeAuthenticationFilter:__ 记住用户
> __FilterSecurityInterceptor:__ 这个过滤器是整个SpringSecurity过滤器链的最后一环

接下来简短的来谈一谈，__FilterSecurityInterceptor__ 这个过滤器的最后一环，也是最重要的一环。它的父类__AbstractSecurityInterceptor__ 是一个实现了对受保护对象的访问进行拦截的抽象类,该抽象类包含了__AccessDecisionManager(决策管理器)__、__AuthenticationManager(身份认证管理器)__的setter， 可以通过__Spring__ 自动注入，另外，资源角色授权器需要单独自定义注入。

其中有几个比较重要的方法,都会在`FilterSecurityInterceptor`中被调用:
> __beforeInvocation()方法:__ 实现了对访问受保护对象的权限校验，内部用到了`AccessDecisionManager`和`AuthenticationManager`;
> __finallyInvocation()方法:__ 用于实现受保护对象请求完毕后的一些清理工作，主要是如果在`beforeInvocation()`中改变了`SecurityContext`，则在`finallyInvocation()`中需要将其恢复为原来的`SecurityContext`,该方法的调用应当包含在子类请求受保护资源时的finally语句块中;
> __afterInvocation()方法:__ 实现了对返回结果的处理，在注入了AfterInvocationManager的情况下默认会调用其decide()方法。

来看看 `FilterSecurityInterceptor` 的部分源码：
``` Java
public class FilterSecurityInterceptor extends AbstractSecurityInterceptor implements Filter {

	// 。。。。

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        FilterInvocation fi = new FilterInvocation(request, response, chain);
        this.invoke(fi);    //  invoke() 在Filter 的 doFilter（）中被调用
    }

    // FilterSecurityInterceptor 的核心代码
    public void invoke(FilterInvocation fi) throws IOException, ServletException {
        if (fi.getRequest() != null && fi.getRequest().getAttribute("__spring_security_filterSecurityInterceptor_filterApplied") != null && this.observeOncePerRequest) {
            fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
        } else {
            if (fi.getRequest() != null) {
                fi.getRequest().setAttribute("__spring_security_filterSecurityInterceptor_filterApplied", Boolean.TRUE);
            }

	    // 调用父类的beforeInvocation方法，该方法实现了对访问受保护对象的权限校验，
	    // 内部调用到了`AccessDecisionManager`和`AuthenticationManager`
            InterceptorStatusToken token = super.beforeInvocation(fi);

            try {
                fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
            } finally {
            	// 调用父类的finallyInvocation方法, 用于实现受保护对象请求完毕后的一些清理工作
                super.finallyInvocation(token);
            }
	  // 用父类的afterInvocation方法,实现了对返回结果的处理，
	  // 在注入了AfterInvocationManager的情况下默认会调用其decide()方法。
            super.afterInvocation(token, (Object)null);
        }

    }


    // 。。。。

}

```

### 最后说点什么
感觉最后不说点什么，有点奇怪。。。
笔者目前也在学习的__SpringSecurity__， 目前是一个阶段性的总结，以后如果有更深的理解会继续更新。笔者并没有讲具体的实现细节，只是大概的过了一遍SpringSecurity的执行过程，有遗漏的错误的地方欢迎指出。
