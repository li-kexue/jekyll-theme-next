---
layout: post
title: 'spring security'
date: 2020-03-13
author: likexue
photos:

- https://pixabay.com/photos/sparkler-holding-hands-firework-677774/
  categories: spring security
  tags: [spring] [security]
---

  


<h1 align="center">Spring Security学习（一）</h1>

# 实现功能：

+ 使用spring security实现一个邮件验证码登录。



## 1.学习如何配置

想要对security进行自定义配置，要先继承WebSecurityConfigurerAdapter然后重写configure()方法。

这里注意一点要确认配置了@EnableWebSecurity，这样才能使配置生效。

准备工作做好后，让我们看如何自定义配置。

#### 对于HttpSecurity的配置

```java
@Override
    protected void configure(HttpSecurity http) throws Exception {
      http
                .formLogin() //表单登录
                .loginPage("/login.html").permitAll() //设置登录页，所有权限都可以访问，也就是无需认证。
                .and()
                .authorizeRequests()//设置具体路由下所需要的权限
                .antMatchers("/admin/**").hasRole("admin") //admin下的所有uri都需要有admin权限
                .antMatchers("/db").access("hasRole(admin) and hasRole(dba)") //对于/db路径下需要admin和dba权限
                .anyRequest().authenticated()
                .and()
                .csrf()
                .disable(); //关闭csrf保护，如果不关闭的话，页面需要提交csrf值。正式开发中，如果安全性较高需要启用。
    }
```





## 2.默认鉴权流程

对于Spring Security默认鉴权的流程：

1. 进入UsernamePasswordAuthenticationFilter执行attemptAuthentication()方法

   1. 在attemptAuthentication()方法中，先实例化未认证通过的UsernamePasswordToken对象
   2. 在attemptAuthentication()方法中，调用getAuthenticationManager()获取一个Manager，这里获取的是ProviderManager。

2. 进入ProviderManager执行authenticate()方法

   1. 在该方法里会找到一个适合的provider去处理传进来的UsernamePasswordAuthenticationToken，这里使用的是DaoAuthenticationProvider。
   2. 执行DaoAuthenticationProvider的authenticate()方法，在这里会进行校验用户名，密码是否正确，正确就调用UsernamePasswordAuthenticationToken的另一个构造函数（带有权限信息），并将是否认证设置为true。

3. 至此，UsernamePasswordAuthenticationFilter执行结束。

结合下图理解：

![图一]({{site.baseurl}}/assets/images/spring/SpringSecurity.png)	


### 1.UsernamePasswordAuthenticationFIlter

在这个类中，我们要看的是下面这部分：

```java
//设置要拦截的路由以及方法	
public UsernamePasswordAuthenticationFilter() {
		super(new AntPathRequestMatcher("/login", "POST"));
	}

	public Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException {
		if (postOnly && !request.getMethod().equals("POST")) {
			throw new AuthenticationServiceException(
					"Authentication method not supported: " + request.getMethod());
		}
        // 获取用户和密码
		String username = obtainUsername(request);
		String password = obtainPassword(request);

		if (username == null) {
			username = "";
		}

		if (password == null) {
			password = "";
		}
        // 去空格
		username = username.trim();
        // 通过用户密码创建一个UsernamePasswordAuthenticationToken类
		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
				username, password);

		// 将详细信息拷贝到authRequest
		setDetails(request, authRequest);

		return this.getAuthenticationManager().authenticate(authRequest);
	}
```

### UsernamePasswordAuthenticationToken

在这个类中，最主要的是两个构造方法以及一个eraseCredentials()方法。

```java
// 该构造方法构造的是一个未认证的token
UsernamePasswordAuthenticationToken(Object principal, Object credentials) {
		super(null);
		this.principal = principal;
		this.credentials = credentials;
		setAuthenticated(false);//未认证
	}
    //该构造方法多了一个authorities参数，构造的是已认证的token
	public UsernamePasswordAuthenticationToken(Object principal, Object credentials,
			Collection<? extends GrantedAuthority> authorities) {
		super(authorities);
		this.principal = principal;
		this.credentials = credentials;
		super.setAuthenticated(true); 
	}
    // 该方法是将credentials（密码）信息擦除
	@Override
	public void eraseCredentials() {
		super.eraseCredentials();
		credentials = null;
	}
```



在UsernamePasswordAuthenticationFilter#attemptAuthentication()方法里最后返回this.getAuthenticationManager().authenticate(authRequest)，这里对我们返回的UsernamePasswordAuthenticationToken进行认证，那我们看看AuthenticationManager。

AuthenticationManager是一个接口，而这个UsernamePasswordAuthenticationFilter调用的是ProviderManager这个类。这里为了便于研究把异常和日志部分给去掉了。

```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
		Class<? extends Authentication> toTest = authentication.getClass();
		AuthenticationException lastException = null;
		AuthenticationException parentException = null;
		Authentication result = null;
		Authentication parentResult = null;
		boolean debug = logger.isDebugEnabled();

        // 先找到一个provider，provider的support方法能够判断是否能够处理UsernamePasswordAUthenticationToken。
		for (AuthenticationProvider provider : getProviders()) {
			if (!provider.supports(toTest)) {
				continue;
			}
			result = provider.authenticate(authentication);
            if (result != null) {
                copyDetails(authentication, result);
                break;
            }
		}
        // 如果找不到适合的provider就会调用parent#authenticate()方法。
		if (result == null && parent != null) {
			result = parentResult = parent.authenticate(authentication);
		}
        // 如果result不为空，验证完之后就可以将credentials信息擦除
		if (result != null) {
			if (eraseCredentialsAfterAuthentication
					&& (result instanceof CredentialsContainer)) {
				((CredentialsContainer) result).eraseCredentials();
			}
			if (parentResult == null) {
				eventPublisher.publishAuthenticationSuccess(result);
			}
			return result;
		}
	}

```









