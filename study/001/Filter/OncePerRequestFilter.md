# Overview

원래 JWT로 인증을 했었는 이번에 하는 프로젝트인 Hanmo는 패스워드 말고 
간편인증 방식을 사용하려고 전화번호만 인증받고 나머지는 임시토큰으로 대체 했었다.
JWT토큰 만들고 필터검증하고는 해봤지만 문자가 뒤섞여서 나오게 해 둔 TempToken은 어떻게 필터링하고
검증을 해야할지, 그냥 문자가 뒤섞여 나와도 괜찮은지 싶었다. 그러면서 Filter를 알아보다가 방법을 조금 대체했다.

하지만 아직 이 방법이 잘 맞는 지 모르겠지만 일단 적어보겠다.

### Filter
1. javax.servlet-api나 tomcat-embed-core를 사용하면 제공되는 Servlet Filter Interface이다.
2. DispatcherServlet이 요청을 받기 전에 Filter에서 먼저 request를 받는다.

```java
package jakarta.servlet;

import java.io.IOException;

public interface Filter {
default void init(FilterConfig filterConfig) throws ServletException {
}

    void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException;

    default void destroy() {
    }
}
```

### GenericFilterBean
Filter를 확장해 Spring에서 제공하는 Filter이다.
기존 Filter에서 얻어 올 수 없는 정보이던 Spring의 설정 정보를 가져올 수 있게 확장 된 추상클래스이다.

```java
public abstract class GenericFilterBean implements Filter, BeanNameAware, EnvironmentAware,
EnvironmentCapable, ServletContextAware, InitializingBean, DisposableBean {

    /** Logger available to subclasses. */
    protected final Log logger = LogFactory.getLog(getClass());

    @Nullable
    private String beanName;

    @Nullable
    private Environment environment;

    @Nullable
    private ServletContext servletContext;

    @Nullable
    private FilterConfig filterConfig;

    private final Set<String> requiredProperties = new HashSet<>(4);

    public void setServletContext(ServletContext servletContext) {
        this.servletContext = servletContext;
    }
}
```

위 코드의 Filter들은 서블릿마다 호출이 된다.
서블릿은 사용자의 요청을 받으면 서블릿을 생성해 메모리에 두고, 같은 클라이언트에 요청을 받으면 
생성해둔 서블릿 객체를 재활용 해 처리한다. 

문제는 의도치 않을 경우 Filter가 두번씩 적용 된다는 것이다.

Spring Security에서 인증과 인가 또한 Filter로 구현되어 있다.
예를 들어, API 0 에서 요청을 처리하고 API 1로 redirect 시킨다고 가정하자.
클라이언트는 한번의 요청을 했지만 , 흐름상 두번을 쓴것과 똑같다. 쓸데없는 자원만 낭비 한 것이다.

이런 두번의 필터를 거치지 않기 위해 OncePerRequestFilter가 생겨났다.

## OncePerRequestFilter
모든 서블릿에서 일관 된 요청을 받기 위한 필터이다.
이 추상 클래스를 구현 한 필터는 사용자의 요청에 딱 한번만 실행된다.
OncePerRequestFilter를 상속해 구현 한 경우 doFilter대신 doFilterInternal메서드를 구현하면 된다.

``` java
@Slf4j
@Component
public class FirstFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
		 // TODO 전처리
        filterChain.doFilter(request , response);
        // TODO 후처리
    }
}
```
모든 서블릿에 일관된 요청을 처리하기 위해 만들어진 필터이다.