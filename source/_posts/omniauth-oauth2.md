---
title: omniauth & oauth2
date: 2017.11.29
tags:
---

## 问题

做oauth2授权登录，遇到了`invalid_grant`问题，流程跟之前几次一样，却出错了，查下来发现是`omniauth-oauth2`的一次merge导致了很多oauth2的gem都出了同样的问题

相关链接：

- [https://github.com/omniauth/omniauth-oauth2/issues/93](https://github.com/omniauth/omniauth-oauth2/issues/93)
- [https://github.com/omniauth/omniauth-oauth2/commit/26152673224aca5c3e918bcc83075dbb0659717f](https://github.com/omniauth/omniauth-oauth2/commit/26152673224aca5c3e918bcc83075dbb0659717f)

相关的gem

- [omniauth(1.7.1)](https://github.com/omniauth/omniauth)
- [oauth2(1.4.0)](https://github.com/intridea/oauth2)
- [omniauth-oauth2(1.4.0)](https://github.com/omniauth/omniauth-oauth2)

## omniauth & oauth2 & omniauth-oauth2 关系

`omniauth`: 帮我们构建了一个授权登录模板的Rack middleware.  
`oauth2`: 实现了OAuth 2.0协议  
`omniauth-oauth2`: 用`omniauth`提供的模板，结合`oauth2`，实现了完整的OAuth 2.0授权流程

## 整个过程

omniauth原理：检查请求路径，匹配授权路径，进行相应的处理  
相关代码(omniauth/lib/omniauth/strategy.rb)：  

```ruby
def call!(env) # rubocop:disable CyclomaticComplexity, PerceivedComplexity
  unless env['rack.session']
    error = OmniAuth::NoSessionError.new('You must provide a session to use OmniAuth.')
    raise(error)
  end

  @env = env
  @env['omniauth.strategy'] = self if on_auth_path?

  return mock_call!(env) if OmniAuth.config.test_mode
  return options_call if on_auth_path? && options_request?
  return request_call if on_request_path? && OmniAuth.config.allowed_request_methods.include?(request.request_method.downcase.to_sym)
  return callback_call if on_callback_path?
  return other_phase if respond_to?(:other_phase)
  @app.call(env)
end
```

### 1、监听`/auth/skylark`(发起授权的路径)请求

```ruby
return request_call if on_request_path? && OmniAuth.config.allowed_request_methods.include?(request.request_method.downcase.to_sym)
```


`request_call`不会去调用下一个Middleware(`@app.call(env)`)，意味着该请求不会流向Rails Application.  
`request_call`真正实现在`omniauth-oauth2`，相关代码(omniauth-oauth2/lib/omniauth/strategies/oauth2.rb)：  

```ruby
def request_phase
  redirect client.auth_code.authorize_url({:redirect_uri => callback_url}.merge(authorize_params))
end
```


以上代码会跳转授权链接(`http://oauth-server.com//oauth/authorize?params....`)

### 2、监听oauth-service的callback`/auth/skylark/callback`请求

```ruby
return callback_call if on_callback_path?
```


向oauth-server请求access_token  
相关代码(omniauth-oauth2/lib/omniauth/strategies/oauth2.rb)：  

```ruby
def callback_phase # rubocop:disable AbcSize, CyclomaticComplexity, MethodLength, PerceivedComplexity
  error = request.params["error_reason"] || request.params["error"]
  if error
    fail!(error, CallbackError.new(request.params["error"], request.params["error_description"] || request.params["error_reason"], request.params["error_uri"]))
  elsif !options.provider_ignores_state && (request.params["state"].to_s.empty? || request.params["state"] != session.delete("omniauth.state"))
    fail!(:csrf_detected, CallbackError.new(:csrf_detected, "CSRF detected"))
  else
    self.access_token = build_access_token
    self.access_token = access_token.refresh! if access_token.expired?
    super
  end
rescue ::OAuth2::Error, CallbackError => e
  fail!(:invalid_credentials, e)
rescue ::Timeout::Error, ::Errno::ETIMEDOUT => e
  fail!(:timeout, e)
rescue ::SocketError => e
  fail!(:failed_to_connect, e)
end
```

omniauth会把相关信息写入request.env，调用下一个Rack Middleware，然后Rails Application就会接受到`/auth/skylark/callback`的请求  
相关代码：  

```ruby
def callback_phase
  env['omniauth.auth'] = auth_hash
  call_app!
end
```

auth_hash会去设置info，  

```ruby
hash.info = info unless skip_info?
```

info获取具体的user信息，而info的具体实现在继承`OmniAuth::Strategies::OAuth2`的类中实现

```ruby
info do
  extra_info = {
    :name => raw_info["name"],
    :uid => raw_info["uid"],
    :extra_info => {
      phone: raw_info["phone"],
      identifier: raw_info["identifier"],
      headimgurl: raw_info["headimgurl"]
    }
  }
end

def raw_info
  @raw_info ||= access_token.get('/api/v1/user.json').parsed
end
```