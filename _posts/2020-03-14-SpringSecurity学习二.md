<h1 align = "center">Spring Security学习(二)</h1>

根据上一章的学习，基本摸清了spring security的工作脉络，接下来就是要动手实践，来实现我们自己的校验逻辑了。

类比上一张的流程，可以得到下面这张图。

![图一]({{site.baseurl}}/assets/images/spring/SpringSecurity2.png)

<p align="center">图一</p>

按照依赖关系我们从下往上依次实现。

1. 首先实现UserDetailServiceImpl

```java
@Slf4j
@Component
public class UserDetailsServiceImpl implements UserDetailsService {
    @Autowired
    UserRepository repository;

    // 根据email加载用户信息
    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        log.info("登录邮箱：{}",email);
        // 根据邮箱查询用户信息
        UserDo userDo = repository.findUserByEmail(email);
        if (userDo != null){
            System.out.println("user 详情:"+ userDo.toString());
            // 将用户权限进行封装
            String authority = userDo.getAuthorities();
            SimpleGrantedAuthority simpleGrantedAuthority = new SimpleGrantedAuthority(authority);
            Set<GrantedAuthority> authorities = new HashSet<>();
            authorities.add(simpleGrantedAuthority);
            // 返回封装好的用户信息，该User类使用的是org.springframework.security.core.userdetails.User包下的
            User user = new User(userDo.getUsername(),userDo.getPassword(),authorities);    
            return user;
        }
        // 如果根据邮箱查找不到用户信息，抛出异常，这里可以自行实现一个EmailNotFoundException异常类
        throw new EmailNotFoundException("邮箱: ’" + email + "‘不存在");
    }
}
```

2. 由于EmailCodeAuthenticationProvider需要用到EmailCodeAuthenticationToken，所以先实现它，这里我们是仿照UsernamePasswordAuthenticationToken实现的。

```java
public class EmailCodeAuthenticationToken extends AbstractAuthenticationToken {
    private static final long serialVersionUID = 520L;
    private final Object principal;
	// 调用该构造函数实例化的是未经认证的
    public EmailCodeAuthenticationToken(Object email) {
        super((Collection)null);
        this.principal = email;
        this.setAuthenticated(false);
    }
	// 这里两个参数，调用该构造函数表明已通过认证
    public EmailCodeAuthenticationToken(Object principal, Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        this.principal = principal;
        super.setAuthenticated(true);
    }

	// 这里由于继承AbstractAuthenticationToken，必须实现该方法，但我们不需要密码，所以直接返回null就行
    @Override
    public Object getCredentials() {
        return null;
    }

    public Object getPrincipal() {
        return this.principal;
    }

    public void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException {
        if (isAuthenticated) {
            throw new IllegalArgumentException("Cannot set this token to trusted - use constructor which takes a GrantedAuthority list instead");
        } else {
            super.setAuthenticated(false);
        }
    }

    public void eraseCredentials() {
        super.eraseCredentials();
    }
}
```

3. 实现EmailCodeAuthenticationProvider

```java
public class EmailCodeAuthenticationProvider implements AuthenticationProvider {
    @Setter
    private UserDetailsServiceImpl userDetailsService;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        // 通过email加载用户信息
        User user = (User) userDetailsService.loadUserByUsername((String) authentication.getPrincipal());
        if (user == null){
            throw new InternalAuthenticationServiceException("无法获取用户信息");
        }
        EmailCodeAuthenticationToken authenticationResult = new EmailCodeAuthenticationToken(user.getUsername(),user.getAuthorities());
        // 这时返回的就是一个经过认证的Authentication了
        return authenticationResult;
    }

    @Override
    public boolean supports(Class<?> aClass) {
        //判断传进来的是不是EmailCodeAuthenticationToken类型，如果是就用这个provider进行处理
        return EmailCodeAuthenticationToken.class.isAssignableFrom(aClass);
    }
}
```

4. 实现

```java
public class EmailCodeAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
    public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "email";
    private String emailParameter = "email";
    private boolean postOnly = true;

    public EmailCodeAuthenticationFilter() {
        // 对/authentication/email 路径进行拦截（须是post请求）
        super(new AntPathRequestMatcher("/authentication/email", "POST"));
    }

    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException, ServletRequestBindingException {
        if (this.postOnly && !request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
        } else {
            先进行校验
            validate(request);
            
            校验通过进行认证处理
            String email = this.obtainEmail(request);
            if (email == null) {
                email = "";
            }

            email = email.trim();
            // 第一次调用，未经过认证
            EmailCodeAuthenticationToken authRequest = new EmailCodeAuthenticationToken(email);
            this.setDetails(request, authRequest);
            return this.getAuthenticationManager().authenticate(authRequest);
        }
    }
	// 实现验证码校验逻辑
    private void validate(HttpServletRequest httpServletRequest) throws ServletRequestBindingException, AuthenticationException {
        String emailCode = (String) httpServletRequest.getSession().getAttribute("email_code");
        String code = ServletRequestUtils.getStringParameter(httpServletRequest,"code");
        if (emailCode == null){
            throw new ValidateEmailCodeException("验证码不存在");
        }
        if (code == null){
            throw new ValidateEmailCodeException("验证码为空");
        }
        if (!emailCode.equals(code)){
            throw new ValidateEmailCodeException("验证码不匹配");
        }
    }

    @Nullable
    protected String obtainEmail(HttpServletRequest request) {
        return request.getParameter(this.emailParameter);
    }

    protected void setDetails(HttpServletRequest request, EmailCodeAuthenticationToken authRequest) {
        authRequest.setDetails(this.authenticationDetailsSource.buildDetails(request));
    }

    public void setEmailParameter(String emailParameter) {
        Assert.hasText(emailParameter, "email parameter must not be empty or null");
        this.emailParameter = emailParameter;
    }


    public void setPostOnly(boolean postOnly) {
        this.postOnly = postOnly;
    }

    public final String getEmailParameter() {
        return this.emailParameter;
    }

}
```

当然校验逻辑可以单独封装成一个Filter对请求进行处理。

当我们完成这一系列的代码以后，要怎么将这些逻辑串联起来呢？

5. 这就到了我们的最后一步，完成配置信息

```java
@Component
@Slf4j
public class EmailCodeAuthenticationSecurityConfig extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity> {
    @Autowired
    FailureHandler failureHandler;
    @Autowired
    SuccessHandler successHandler;
    @Autowired
    UserDetailsServiceImpl userDetailsService;
    @Override
    public void configure(HttpSecurity builder) throws Exception {
        log.info("userDetailsService:{}",userDetailsService);
        EmailCodeAuthenticationFilter authenticationFilter = new EmailCodeAuthenticationFilter();
        authenticationFilter.setAuthenticationManager(builder.getSharedObject(AuthenticationManager.class));
        authenticationFilter.setAuthenticationFailureHandler(failureHandler);
        authenticationFilter.setAuthenticationSuccessHandler(successHandler);

        EmailCodeAuthenticationProvider emailAuthenticationProvider = new EmailCodeAuthenticationProvider();
        emailAuthenticationProvider.setUserDetailsService(userDetailsService);
        builder.authenticationProvider(emailAuthenticationProvider)
                .addFilterAfter(authenticationFilter, UsernamePasswordAuthenticationFilter.class);
    }
}
```

这一步将我们实现的代码串联起来，但是并没有真正发挥作用，我们还需要在另一个配置文件上使用这一配置

```java
@Configuration
@EnableWebSecurity
public class EmailCodeConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private EmailCodeAuthenticationSecurityConfig securityconfig;

    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .formLogin()
                .loginPage("/login.html")
                .loginProcessingUrl("/authentication/form").permitAll()
                    .and()
                .authorizeRequests()
                .antMatchers("/email/code","/authentication/email").permitAll()
                .anyRequest()
                .authenticated()
                    .and()
                .csrf()//关闭csrf，使自定义登录页面可以使用
                .disable()
                .apply(securityconfig);// 使我们的配置生效
    }
}
```

