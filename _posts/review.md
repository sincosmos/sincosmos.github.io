# Review
## Web 开发项目回顾
### 算法管理系统项目架构
项目基于 Spring Boot 搭建，引入了 Spring MVC, Shiro, Mybatis, RedisTemplate 等框架依赖。Spring MVC 做前后端交互支持，Shiro 用来做权限管理，mybatis 作为 ORM 工具，RedisTemplate 用以简化程序与 redis 的交互，将 web session 信息保存到 redis 中。
### Spring MVC
1. Spring MVC 由 DispatcherServlet 统一接受符合 pattern 的所有请求。
2. DispatcherServlet 调用 HandlerMapping（使用 BeanNameUrlHandlerMapping, DefaultAnnotationHandlerMapping 等）的 getHandler(HttpServletRequest request) 方法，该方法解析 request 的资源标志符 URI，获得为该请求配置的 HandlerExecutionChain (Interceptor & Servlet)，HandlerExecutionChain 对象被返回给 DispatcherServlet。handler 和 URI 的映射是在 Spring MVC 的阶段完成的，映射关系会被保存起来供查询。
3. DispatcherServlet 根据获得的 HandlerExecutionChain，选择一个合适的 HandlerAdapter (默认为 HttpRequestHandlerAdapter, SimpleControllerHandlerAdapter, AnnotationMethodHandlerAdapter)。
4. 成功后开始依次执行拦截器 preHandler() 方法。提取 Request 中的模型数据，填充 Handler 入参，开始执行依次执行 Handler (Controller)，填充入参的过程，Spring 会根据配置，额外进行入参格式转换（例如 HttpMessageConveter）、数据验证等工作。Handler 执行完成后，向 DispatcherServlet 返回 ModelAndView 对象。DispatcherServlet 根据获得的 ModelAndView 选择一个合适的 ViewResolver (InternalResourceViewResolver) 对返回给客户端的视图进行渲染，最后把渲染结果返回给客户端。
### Shiro
1. Filter 在 DispatcherServlet 之前拦截请求，并可以修改请求头和数据。实现 Filter 接口的 Bean 会被 Spring boot 自动加入到过滤器链中。在我们的项目中，ShiroFilter 拦截所有请求（如果请求是 http 请求，将其封装为 ShiroHttpServletRequest 后向后发送）。如果请求的资源允许匿名访问，shiroFilter 将允许其通过。否则 shiroFilter 将尝试获取用户 Session
```
package com.hendisantika.utils;

import org.apache.commons.io.FileUtils;
import org.apache.commons.text.StringEscapeUtils;
import org.jsoup.Jsoup;
import org.jsoup.nodes.Document;
import org.jsoup.nodes.Element;
import org.jsoup.nodes.TextNode;
import org.jsoup.select.Elements;

import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.Iterator;
import java.util.Random;

/**
 * @author kleguan
 * @date 2023/02/01
 */
public class HtmlHoneyPotHandler {
    public static final String JSOUP_HTML_SPAN_LEFT_SYMBOL = "JSOUP_HTML_SPAN_LEFT_SYMBOL";
    public static final String JSOUP_HTML_SPAN_RIGHT_SYMBOL = "JSOUP_HTML_SPAN_RIGHT_SYMBOL";
    public static final String JSOUP_HTML_SPAN_END_SYMBOL = "JSOUP_HTML_SPAN_END_SYMBOL";

    public static void main(String[] args) throws IOException {
        HtmlHoneyPotHandler htmlHoneyPotHandler = new HtmlHoneyPotHandler();
        //每次从模版生成新的目标文件，不修改原模版
        //htmlHoneyPotHandler.removeHoneyPot(htmlHoneyPotHandler.templates());
        String path = System.getProperty("user.dir");
        String modulePath = "/";
        String templatePath = "src/main/resources/templates/";
        String source = path + modulePath + templatePath;
        String dest = path + modulePath + "src/main/resources/disturbed-templates/";
        FileUtils.deleteDirectory(new File(dest));
        FileUtils.copyDirectory(new File(source), new File(dest));

        htmlHoneyPotHandler.addHoneyPot(htmlHoneyPotHandler.templates(dest));
    }

    private Iterator<File> templates(String dir){
        File file = new File(dir);
        return FileUtils.iterateFiles(file,null, true);
    }

    private void removeHoneyPot(Iterator<File> templates){
        while (templates.hasNext()) {
            this.removeHoneyPot(templates.next());
        }
    }

    private void removeHoneyPot(File file) {
        System.out.println("removing honey pots: " + file);
        if(!file.toString().endsWith("html")){
            return;
        }
        try {
            Document doc = Jsoup.parse(file, StandardCharsets.UTF_8.name());
            Elements honeyElements = doc.getElementsByClass("honey-candy");
            honeyElements.remove();
            Elements disturbElements = doc.getElementsByClass("disturb-can");
            disturbElements.remove();
            FileWriter fw = new FileWriter(file, false);
            fw.write(doc.outerHtml());
            fw.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void addHoneyPot(Iterator<File> templates){
        int disturb = 0;
        while (templates.hasNext()) {
            this.addHoneyPot(disturb, templates.next());
            disturb++;
        }
    }

    private void addHoneyPot(int disturb, File file) {
        System.out.println("adding honey pots: " + file);
        if(!file.toString().endsWith("html")){
            return;
        }
        String honeySpan = JSOUP_HTML_SPAN_LEFT_SYMBOL + " class=";
        if((disturb & 1) == 0){
            honeySpan += "'honey-candy'" + JSOUP_HTML_SPAN_RIGHT_SYMBOL;
        }else{
            honeySpan += "'disturb-can'" + JSOUP_HTML_SPAN_RIGHT_SYMBOL;
        }
        try {
            Document doc = Jsoup.parse(file, StandardCharsets.UTF_8.name());

            Elements elements = doc.body().children().select("*");
            StringBuilder sb = new StringBuilder();
            for(Element el : elements){
                for(TextNode tn : el.textNodes()){
                    String text = tn.text();
                    System.out.println(text);
                    for(char c: text.toCharArray()){
                        if(this.isChinese(c)){
                            sb.append(JSOUP_HTML_SPAN_LEFT_SYMBOL + " class='disturb-honey'" + JSOUP_HTML_SPAN_RIGHT_SYMBOL
                                    + c + JSOUP_HTML_SPAN_END_SYMBOL);
                            sb.append(honeySpan + this.randomChinese(2) + JSOUP_HTML_SPAN_END_SYMBOL);
                        }else{
                            sb.append(c);
                        }
                    }
                    String es = sb.toString();
                    System.out.println(es);
                    System.out.println(StringEscapeUtils.unescapeHtml4(es));
                    tn.text(es);
                    sb.setLength(0);
                }

            }

            //TODO 直接改 string
            FileWriter fw = new FileWriter(file, false);
            fw.write(doc.outerHtml());
            fw.close();

            String content = FileUtils.readFileToString(file, StandardCharsets.UTF_8);
            FileUtils.writeStringToFile(file,
                    content.replaceAll(JSOUP_HTML_SPAN_LEFT_SYMBOL, "<span")
                            .replaceAll(JSOUP_HTML_SPAN_RIGHT_SYMBOL, ">")
                            .replaceAll(JSOUP_HTML_SPAN_END_SYMBOL, "</span>"), StandardCharsets.UTF_8);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * @param count length of the Chinese string
     * @return chinese string
     */
    private String randomChinese(int count) {
        String str = "";
        for (int i = 0; i < count; i++) {
            try {
                str += randomChinese();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return str;
    }

    /**
     * @return 常用汉字
     * @throws Exception
     */
    private String randomChinese() throws Exception {
        int highPos, lowPos;
        Random random = new Random();
        highPos = (176 + Math.abs(random.nextInt(39)));
        lowPos = (161 + Math.abs(random.nextInt(93)));
        byte[] b = new byte[2];
        b[0] = Integer.valueOf(highPos).byteValue();
        b[1] = Integer.valueOf(lowPos).byteValue();
        return new String(b, "GBK");
    }

    private boolean isChinese(char c) {
        Character.UnicodeScript sc = Character.UnicodeScript.of(c);
        if (sc == Character.UnicodeScript.HAN) {
            return true;
        }

        return false;
    }
}

```
