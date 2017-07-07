# -
controller调用日志拦截器


/**
 * controller调用日志拦截器
 */
@ControllerAdvice
public class SysLogInterceptor extends HandlerInterceptorAdapter implements RequestBodyAdvice {

    private static Logger logger = LoggerFactory
            .getLogger(SysLogInterceptor.class);

    private NamedThreadLocal<Long> startTimeThreadLocal = new NamedThreadLocal<Long>("StopWatch-StartTime");

    private final String[] SENSITIVE_FIELDS = new String[]{"password", "pwd", "oldPassword", "oldPwd", "newPassword", "newPwd"};

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response, Object handler) throws Exception {
        long beginTime = System.currentTimeMillis();
        startTimeThreadLocal.set(beginTime);
        String path = request.getServletPath();
        if (path == null || path.trim().length() == 0) {
            path = request.getRequestURI();
        }
        if (logger.isInfoEnabled()) {
            logger.info("Start: {}", path);
        }
        return true;
    }

    @SuppressWarnings("rawtypes")
    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response, Object handler, Exception ex)
            throws Exception {
        long endTime = System.currentTimeMillis();
        long beginTime = startTimeThreadLocal.get();
        long consumeTime = endTime - beginTime;

        String path = request.getServletPath();
        if (path == null || path.trim().length() == 0) {
            path = request.getRequestURI();
        }

        if (null != handler && handler instanceof HandlerMethod) {
            HandlerMethod hand = (HandlerMethod) handler;
            IgnoreLogParam ignoreLogParam = hand.getMethodAnnotation(IgnoreLogParam.class);
            if (ignoreLogParam == null) {
                Map<String, String[]> params = request.getParameterMap();
                Map<String, String[]> newParams = this.filterSensitiveField(params);
                ObjectMapper objectMapper = new ObjectMapper();
                logger.info("Params: {}", objectMapper.writeValueAsString(newParams));
            }
        }

        logger.info("End: {}. executeTime: {} ms.", path, consumeTime);

        if (consumeTime > 3000) {
            logger.warn("slow api ,{} executeTime {} ms", path, consumeTime);
        }
    }

    private Map<String, String[]> filterSensitiveField(Map<String, String[]> params) {
        Map<String, String[]> newParams = new HashMap();
        for (Map.Entry<String, String[]> entry : params.entrySet()) {
            newParams.put(entry.getKey(), entry.getValue());
        }
        for (String field : SENSITIVE_FIELDS) {
            newParams.remove(field);
        }
        return newParams;
    }

    @Override
    public boolean supports(MethodParameter methodParameter, Type type, Class<? extends HttpMessageConverter<?>> aClass) {
        return true;
    }

    @Override
    public Object handleEmptyBody(Object o, HttpInputMessage httpInputMessage, MethodParameter methodParameter, Type type, Class<? extends HttpMessageConverter<?>> aClass) {
        return o;
    }

    @Override
    public HttpInputMessage beforeBodyRead(HttpInputMessage httpInputMessage, MethodParameter methodParameter, Type type, Class<? extends HttpMessageConverter<?>> aClass) throws IOException {
        return httpInputMessage;
    }

    @Override
    public Object afterBodyRead(Object body, HttpInputMessage httpInputMessage, MethodParameter methodParameter, Type type, Class<? extends HttpMessageConverter<?>> aClass) {
        IgnoreLogParam ignoreLogParam = methodParameter.getMethodAnnotation(IgnoreLogParam.class);
        if (ignoreLogParam == null) {
            try {
                logger.info("body Params: {}", new ObjectMapper().writeValueAsString(body));
            } catch (JsonProcessingException e) {
                logger.error("body to json error", e);
            }
        }
        return body;
    }
}
