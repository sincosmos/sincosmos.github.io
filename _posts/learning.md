```
<dependency>
            <groupId>org.jsoup</groupId>
            <artifactId>jsoup</artifactId>
            <version>1.15.3</version>
        </dependency>

        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.12.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-text</artifactId>
            <version>1.10.0</version>
        </dependency>

        <dependency>
            <groupId>commons-io</groupId>
            <artifactId>commons-io</artifactId>
            <version>2.11.0</version>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>31.1-jre</version>
        </dependency>
```
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
