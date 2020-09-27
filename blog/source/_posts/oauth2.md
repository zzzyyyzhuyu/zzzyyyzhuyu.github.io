---
title: 基于OAuth2和spring-security的token认证的实现
date: 2020-09-27 14:54:34
categories:
    - security
tags: 
     - oauth2
     - spring-security    
---


#  基于oauth2和spring-security的token认证的实现

###  简要说明
 - authenticate: 认证，对用户进行认证
 - authorize: 授权，对认证通过的用户授权，颁发票据，用于访问资源服务器
 - jwt: 后续访问资源服务器的票据（ticket）
 - OAuth2：授权机制的一种，后续实现采用OAuth2中的client credentials方式
 
###  实现思路

#####  对于认证相关请求（登录、刷新token）

- 以auth/form为例,此处不会被OAuth2AuthenticationProcessingFilter拦截
- http.form配置的认证拦截器为UsernamePasswordAuthenticationFilter
```
public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        //校验相关参数
        if (this.postOnly && !request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
        } else {
            String username = this.obtainUsername(request);
            String password = this.obtainPassword(request);
            if (username == null) {
                username = "";
            }

            if (password == null) {
                password = "";
            }

            username = username.trim();
            //封装校验信息
            UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(username, password);
            this.setDetails(request, authRequest);
            //具体认证方法
            return this.getAuthenticationManager().authenticate(authRequest);
        }
    }
```
- ProviderManager中存放多个认证方法provider,会根据filter中存放的认证信息类型，取对应支持的provider进行校验
```
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
        ...
        Class<? extends Authentication> toTest = authentication.getClass();
        ...
		//获取所有provider,逐一比较，支持才进行认证
		for (AuthenticationProvider provider : getProviders()) {
			if (!provider.supports(toTest)) {
				continue;
			}
            
			try {
				result = provider.authenticate(authentication);

				if (result != null) {
					copyDetails(authentication, result);
					break;
				}
			}
			...
        //如果还有父类，后续还会进行父类的认证   
		}
	}
``` 
- 认证成功和失败之后，会调用各自的处理方法,此处以认证成功的方法进行举例
```
public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        String header = request.getHeader(HttpHeaders.AUTHORIZATION);
        if (header == null || !header.startsWith(BEARER_TOKEN_TYPE)) {
            throw new UnapprovedClientAuthenticationException("请求头中无client信息");
        }
        String[] tokens = RequestUtil.extractAndDecodeHeader(header);
        assert tokens.length == 2;
        String clientId = tokens[0];
        String clientSecret = tokens[1];

        /*验证客户端提供的秘钥*/
        ClientDetails clientDetails = clientDetailsService.loadClientByClientId(clientId);
        if (clientDetails == null) {
            throw new UnapprovedClientAuthenticationException("clientId对应的配置信息不存在:" + clientId);
        } else if (!bCryptPasswordEncoder.matches(clientSecret,clientDetails.getClientSecret())) {
            throw new UnapprovedClientAuthenticationException("clientSecret不匹配:" + clientId);
        }
        
        //生成OAuth2授权的token
        TokenRequest tokenRequest = new TokenRequest(new HashMap<>(0), clientId, clientDetails.getScope(), "custom");

        OAuth2Request oAuth2Request = tokenRequest.createOAuth2Request(clientDetails);

        OAuth2Authentication oAuth2Authentication = new OAuth2Authentication(oAuth2Request, authentication);

        OAuth2AccessToken token = authorizationServerTokenServices.createAccessToken(oAuth2Authentication);
        SysUserAuthentication principal = (SysUserAuthentication) authentication.getPrincipal();
        //存储用户信息入redis,为用户注销做准备
        userBiz.handlerLoginData(token, principal);
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write((objectMapper.writeValueAsString(ObjectRestResponse.ok(token))));
    }
```

##### 对于资源请求（请求服务器）
- 请求进入filter链，由OAuth2AuthenticationProcessingFilter进行Token校验
 ```
   public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
    ...
        Authentication authResult = this.authenticationManager.authenticate(authentication);
        if (debug) {
            logger.debug("Authentication success: " + authResult);
        }
        
        this.eventPublisher.publishAuthenticationSuccess(authResult);
        SecurityContextHolder.getContext().setAuthentication(authResult);
    ...
        chain.doFilter(request, response);
   }
```
- 具体认证方法(OAuth2AuthenticationManager管理)
``` 
 public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        if (authentication == null) {
            throw new InvalidTokenException("Invalid token (token not found)");
        } else {
            //加载token相关信息
            String token = (String)authentication.getPrincipal();
            //此处根据AccessToken获取token相关信息，有兴趣可以单步调试进入，token是否过期也在这个方法里
            OAuth2Authentication auth = this.tokenServices.loadAuthentication(token);
            if (auth == null) {
                throw new InvalidTokenException("Invalid token: " + token);
            } else {
                //校验当前客户端ID是否与授权服务器初始化的相匹配
                Collection<String> resourceIds = auth.getOAuth2Request().getResourceIds();
                if (this.resourceId != null && resourceIds != null && !resourceIds.isEmpty() && !resourceIds.contains(this.resourceId)) {
                    throw new OAuth2AccessDeniedException("Invalid token does not contain resource id (" + this.resourceId + ")");
                } else {
                    this.checkClientDetails(auth);
                    if (authentication.getDetails() instanceof OAuth2AuthenticationDetails) {
                        OAuth2AuthenticationDetails details = (OAuth2AuthenticationDetails)authentication.getDetails();
                        if (!details.equals(auth.getDetails())) {
                            details.setDecodedDetails(auth.getDetails());
                        }
                    }

                    auth.setDetails(authentication.getDetails());
                    auth.setAuthenticated(true);
                    return auth;
                }
            }
        }
    }

 ```
- token认证通过后，则可访问并返回对应接口的数据

#### [具体实现](https://github.com/zzzyyyzhuyu/dreamer)
 - 认证相关配置
 - 授权相关配置
 - OAuth2相关配置
 - 验证码配置

#### 总结
 - 配置类中的bean初始化顺序问题，同一个配置类中,@Bean注入的Bean并不一定会在配置方法执行前初始化，尽量把Bean配置单独抽出去作为一个类
 - 关于用户注销问题：
 ```
  1.jwt类型的token是无状态的，并不能根据token是否过期来判断用户是否退出登录
  2.新增用户信息校验和存储，放入redis,在用户退出之后清空，根据token和用户信息联合判断用户的登录状态
 ```
